# 解析器系列之三：生成一个 PEG 解析器

**原题** |  [Generating a PEG Parser](https://medium.com/@gvanrossum_83706/generating-a-peg-parser-520057d642a9)

**作者** | Guido van Rossum（Python之父）

**译者** | 豌豆花下猫（“Python猫”公众号作者）

**声明** | 本翻译是出于交流学习的目的，基于 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权协议。为便于阅读，内容略有改动。



我已经在[本系列第二篇文章](https://mp.weixin.qq.com/s/yUQPeqc_uSRGe5lUi50kVQ)中简述了解析器的基础结构，并展示了一个简单的手写解析器，根据承诺，我们将转向从语法中生成解析器。我还将展示如何使用`@memoize`装饰器，以实现packrat 解析。 

上篇文章我们以一个手写的解析器结束。给语法加上一些限制的话，我们很容易从语法中自动生成这样的解析器。（我们稍后会解除那些限制。）

我们需要两个东西：一个东西读取语法，并构造一个表现语法规则的数据结构；还有一个东西则用该数据结构来生成解析器。我们还需要无聊的胶水，我就不提啦。

所以我们在这创造的是一个简单的编译器编译器（compiler-compiler）。我将语法符号简化了一些，仅保留规则与备选项；这其实对于我在本系列的前面所用的玩具语法来说，已经足够了。

```
statement: assignment | expr | if_statement
expr: expr '+' term | expr '-' term | term
term: term '*' atom | term '/' atom | atom
atom: NAME | NUMBER | '(' expr ')'
assignment: target '=' expr
target: NAME
if_statement: 'if' expr ':' statement
```

使用完整的符号，我们可以为语法文件写出语法：

```
grammar: rule+ ENDMARKER
rule: NAME ':' alternative ('|' alternative)* NEWLINE
alternative: item+
item: NAME | STRING
```

用个花哨的叫法，这是我们的第一个元语法（语法的语法），而我们的解析器生成器将是一个元编译器（**编译器是一个程序，将其它程序从一种语言转译为另一种语言；元编译器是一种编译器，其输入是一套语法，而输出是一个解析器** ）。

有个简单地表示元语法的方法，主要是使用内置的数据类型：一条规则的右侧只是由一系列的条目组成的列表，且这些条目只能是字符串。（Hack：通过检查第一个字符是否为引号，我们可以区分出`NAME`和`STRING`）

至于规则，我用了一个简单的 Rule 类，所以整个语法就是一些 Rule 对象。

这就是 Rule 类，省略了 `__repr__` 与`__eq__` ：

```python
class Rule:
    def __init__(self, name, alts):
        self.name = name
        self.alts = alts
```

调用它的是这个`GrammarParser`类（关于基类`Parser` ，请参阅我之前的帖子）：

```python
class GrammarParser(Parser):
    def grammar(self):
        pos = self.mark()
        if rule := self.rule():
            rules = [rule]
            while rule := self.rule():
                rules.append(rule)
            if self.expect(ENDMARKER):
                return rules    # <------------- final result
        self.reset(pos)
        return None
    def rule(self):
        pos = self.mark()
        if name := self.expect(NAME):
            if self.expect(":"):
                if alt := self.alternative():
                    alts = [alt]
                    apos = self.mark()
                    while (self.expect("|")
                           and (alt := self.alternative())):
                        alts.append(alt)
                        apos = self.mark()
                    self.reset(apos)
                    if self.expect(NEWLINE):
                        return Rule(name.string, alts)
        self.reset(pos)
        return None
    def alternative(self):
        items = []
        while item := self.item():
            items.append(item)
        return items
    def item(self):
        if name := self.expect(NAME):
            return name.string
        if string := self.expect(STRING):
            return string.string
        return None
```

注意 `ENDMARKER` ，它用来确保在最后一条规则后没有遗漏任何东西（如果语法中出现拼写错误，可能会导致这种情况）。 

我放了一个简单的箭头，指向了 grammar() 方法的返回值位置，返回结果是一个存储 Rule 的列表。

其余部分跟上篇文章中的 `ToyParser` 类很相似，所以我不作解释。

只需留意，item() 返回一个字符串，alternative() 返回一个字符串列表，而 rule() 中的 alts 变量，则是一个由字符串列表组成的列表。

然后，rule() 方法将规则名称（一个字符串）与 alts 结合，放入 Rule 对象。

如果把这份代码用到包含了我们的玩具语法的文件上，则 grammar() 方法会返回以下的由 Rule 对象组成的列表：

```python
[
  Rule('statement', [['assignment'], ['expr'], ['if_statement']]),
  Rule('expr', [['term', "'+'", 'expr'],
                ['term', "'-'", 'term'],
                ['term']]),
  Rule('term', [['atom', "'*'", 'term'],
                ['atom', "'/'", 'atom'],
                ['atom']]),
  Rule('atom', [['NAME'], ['NUMBER'], ["'('", 'expr', "')'"]]),
  Rule('assignment', [['target', "'='", 'expr']]),
  Rule('target', [['NAME']]),
  Rule('if_statement', [["'if'", 'expr', "':'", 'statement']]),
]
```

既然我们已经有了元编译器的解析部分，那就创建代码生成器吧。

把这些聚合起来，就形成了一个基本的元编译器：

```python
def generate_parser_class(rules):
    print(f"class ToyParser(Parser):")
    for rule in rules:
        print()
        print(f"    @memoize")
        print(f"    def {rule.name}(self):")
        print(f"        pos = self.mark()")
        for alt in rule.alts:
            items = []
            print(f"        if (True")
            for item in alt:
                if item[0] in ('"', "'"):
                    print(f"            and self.expect({item})")
                else:
                    var = item.lower()
                    if var in items:
                        var += str(len(items))
                    items.append(var)
                    if item.isupper():
                        print("            " +
                              f"and ({var} := self.expect({item}))")
                    else:
                        print(f"            " +
                              f"and ({var} := self.{item}())")
            print(f"        ):")
            print(f"            " +
              f"return Node({rule.name!r}, [{', '.join(items)}])")
            print(f"        self.reset(pos)")
        print(f"        return None")
```

这段代码非常难看，但它管用（某种程度上），不管怎样，我打算将来重写它。

在"for alt in rule.alts"循环中，有些代码细节可能需要作出解释：对于备选项中的每个条目，我们有三种选择的可能：

- 如果该条目是字符串字面量，例如`'+'` ，我们生成`self.expect('+')` 
- 如果该条目全部是大写，例如`NAME` ，我们生成`(name := self.expect(NAME))`  
- 其它情况，例如该条目是`expr`，我们生成 `(expr := self.expr())` 

如果在单个备选项中出现多个相同名称的条目（例如`term '-' term`），我们会在第二个条目后附加一个数字。这里还有个小小的 bug，我会在以后的内容中修复。

这只是它的一部分输出（完整的类非常无聊）。不用担心那些零散的、冗长的 `if (True and … )` 语句，我使用它们，以便每个生成的条件都能够以`and` 开头。Python 的字节码编译器会优化它。

```python
class ToyParser(Parser):
    @memoize
    def statement(self):
        pos = self.mark()
        if (True
            and (assignment := self.assignment())
        ):
            return Node('statement', [assignment])
        self.reset(pos)
        if (True
            and (expr := self.expr())
        ):
            return Node('statement', [expr])
        self.reset(pos)
        if (True
            and (if_statement := self.if_statement())
        ):
            return Node('statement', [if_statement])
        self.reset(pos)
        return None
    ...
```

注意`@memoize` 装饰器：我“偷运”（smuggle）它进来，以便转向另一个主题：使用记忆法（memoization）来加速生成的解析器。

这是实现该装饰器的 memoize() 函数：  

```python
def memoize(func):
    def memoize_wrapper(self, *args):
        pos = self.mark()
        memo = self.memos.get(pos)
        if memo is None:
            memo = self.memos[pos] = {}
        key = (func, args)
        if key in memo:
            res, endpos = memo[key]
            self.reset(endpos)
        else:
            res = func(self, *args)
            endpos = self.mark()
            memo[key] = res, endpos
        return res
return memoize_wrapper
```

对于典型的装饰器来说，它的嵌套函数（nested function）会替换（或包装）被装饰的函数（decorated function），例如 memoize_wrapper() 会包装 ToyParser 类的 statement() 方法。

因为被包装的函数（wrapped function）是一个方法，所以包装器实际上也是一个方法：它的第一个参数是 `self` ，指向 ToyParser 实例，后者会调用被装饰的函数。

包装器会缓存每次调用解析方法后的结果——这就是为什么它会被称为“口袋老鼠解析”（packrat parsing）！

这缓存是一个字典，元素是存储在 Parser 实例上的那些字典。

外部字典的 key 是输入的位置；我将 `self.memos = {}` 添加到 `Parser.__init__()` ，以初始化它。 

内部字典按需添加，它们的 key 由方法及其参数组成。（在当前的设计中没有参数，但我们应该记得 expect()，它恰好有一个参数，而且给它新增通用性，几乎不需要成本。 ）

一个解析方法的结果被表示成一个元组，因为它正好有两个结果：一个显式的返回值（对于我们生成的解析器，它是一个 Node，表示所匹配的规则），以及我们从 `self.mark()` 中获得的一个新的输入位置。

在调用解析方法后，我们会在内部的记忆字典中同时存储它的返回值（res）以及新的输入位置（endpos）。

再次调用相同的解析方法时（在相同的位置，使用相同的参数），我们会从缓存中取出那两个结果，并用 `self.reset()` 来向前移动输入位置，最后返回那缓存中的返回值。

缓存负数的结果也很重要——实际上大多数对解析方法的调用都是负数的结果。在此情况下，返回值为 None，而输入位置不会变。你可以加一个`assert` 断言来检查它。

注意：Python 中常用的记忆法是在 memoize() 函数中将缓存定义成一个局部变量。但我们不这么做：因为我在一个最后时刻的调试会话中发现，每个 Parser 实例都必须拥有自己的缓存。然而，你可以用`(pos, func, args)` 作为 key，以摆脱嵌套字典的设计。

下周我将统览代码，演示在解析示例程序时，所有这些模块实际是如何配合工作的。

我仍然在抓头发中（译注：极度发愁），如何以最佳的方式将协同工作的标记生成器缓冲、解析器和记忆缓存作出可视化。或许我会设法生成动画的 ASCII 作品，而不仅仅是跟踪日志的输出。

本文及示例代码的授权协议： [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)



**译者简介：** 豌豆花下猫，生于广东毕业于武大，现为苏漂程序员，有一些极客思维，也有一些人文情怀，有一些温度，还有一些态度。公众号：「Python猫」（python_cat）。

![](http://ww1.sinaimg.cn/large/68b02e3bly1g2aiq1kpa8j21hc0nmgs4.jpg)

公众号【**Python猫**】， 本号连载优质的系列文章，有喵星哲学猫系列、Python进阶系列、好书推荐系列、技术写作、优质英文推荐与翻译等等，欢迎关注哦。

