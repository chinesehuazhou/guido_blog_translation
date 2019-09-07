# 解析器系列之二：构建一个 PEG 解析器

**原题** | [Building a PEG Parser](https://medium.com/@gvanrossum_83706/building-a-peg-parser-d4869b5958fb)

**作者** | Guido van Rossum（Python之父）

**译者** | 豌豆花下猫（“Python猫”公众号作者）

**声明** | 本翻译是出于交流学习的目的，基于 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权协议。为便于阅读，内容略有改动。



仅仅理解了 PEG 解析器的小部分，我就受到了启发，决定自己构建一个。结果可能不是一个很棒的通用型的 PEG 解析器生成器——这类生成器已经有很多了（例如 TatSu，写于 Python，生成 Python 代码）——但这是一个学习 PEG 的好办法，推进了我的目标，即用由 PEG 语法构建的解析器替换 CPython 的解析器。

在本文中，**通过展示一个简单的手写解析器，我为如何理解解析器的工作原理奠定了基础。** 

（顺便说一句，作为一个实验，我不会在文中到处放参考链接。如果你有什么不明白的东西，请 Google 之 :-）

最常见的 PEG 解析方式是使用可以无限回溯的递归下降解析器。

以上周文章中的玩具语言为例：

```
statement: assignment | expr | if_statement
expr: expr '+' term | expr '-' term | term
term: term '*' atom | term '/' atom | atom
atom: NAME | NUMBER | '(' expr ')'
assignment: target '=' expr
target: NAME
if_statement: 'if' expr ':' statement
```

这种语言中超级抽象的递归下降解析器将为每个符号定义一个函数，该函数会尝试调用与备选项相对应的函数。

例如，对于`statement`，我们有如下函数：  

```python
def statement():
    if assignment():
        return True
   if expr():
        return True
    if if_statement():
        return True
    return False
```

当然这是极其简化的版本：没有考虑解析器中必要的输入及输出。

我们就从输入端开始讲吧。

经典解析器使用单独的标记生成器，来将输入（文本文件或字符串）分解成一系列的标记，例如关键字、标识符（名称）、数字与运算符。

（译注：标记生成器，即 tokenizer，用于生成标记 token。以下简称为“标记器”）

PEG 解析器（像其它现代解析器，如 ANTLR）通常会把标记与解析过程统一。但是对于我的项目，我选择保留单独的标记器。

对 Python 做标记太复杂了，我不想拘泥于 PEG 的形式来重新实现。

例如，你必须得记录缩进（这需要在标记器内使用堆栈），而且在 Python 中处理换行很有趣（它们很重要，除了在匹配的括号内）。字符串的多种引号也会增加复杂性。

简而言之，我不抱怨 Python 现有的标记器，所以我想保留它。（CPython 有两个标记器，一个是解析器在内部使用的，写于 C，另一个在标准库中，用纯 Python 重写。它对我的项目很有帮助。）

经典的标记器通常具有一个简单的接口，供你作函数调用，例如 `get_token()` ，它返回输入内容中的下一个标记，每次消费掉几个字符。

`tokenize` 模块对它作了进一步简化：它的基础 API 是一个生成器，每次生成（yield）一个标记。

每个标记都是一个 `TypeInfo` 对象，它有几个字段，其中最重要之一表示的是标记的类型（例如 `NAME` 、`NUMBER` 、`STRING`），还有一个很重要的是字符串值，表示该标记所包含的字符（例如 `abc` 、`42` 或者 `"hello world"`）。还有的字段会指明每个标记出现在输入文件中的坐标，这对于报告错误很有用。

有一个特殊的标记类型是 `ENDMARKER` ，它表示的是抵达了输入文件的末尾。如果你忽略它，并尝试获取下一个标记，则生成器会终结。

离题了，回归正题。**我们如何实现无限回溯呢？** 

回溯要求你能记住源码中的位置，并且能够从该处重新解析。标记器的 API 不允许我们重置它的输入指针，但相对容易的是，将标记流装入一个数组中，并在那里做指针重置，所以我们就这样做。（你同样可以使用 `itertools.tee()` 来做，但是根据文档中的警告，在我们这种情况下，效率可能较低。）

我猜你可能会先将整个输入内容标记到一个 Python 列表里，将其作为解析器的输入，但这意味着如果在文件末尾处存在着无效的标记（例如一个字符串缺少结束的引号），而在文件前面还有语法错误，那你首先会收到的是关于标记错误的信息。

我觉得这是种糟糕的用户体验，因为这个语法错误有可能是导致字符串残缺的根本原因。

**所以我的设计是按需标记，所用的列表是惰性列表。** 

基础 API 非常简单。`Tokenizer` 对象封装了一个数组，存放标记及其位置信息。

它有三个基本方法：

- `get_token()` 返回下一个标记，并推进数组的索引（如果到了数组末尾，则从源码中读取另一个标记）
- `mark()` 返回数组的当前索引
- `reset(pos)` 设置数组的索引（参数必须从 mark() 方法中得到）

我们再补充一个便利方法 `peek_token()` ，它返回下一个标记且不推进索引。

然后，这就成了 Tokenizer 类的核心代码：

```python
class Tokenizer:
    def __init__(self, tokengen):
        """Call with tokenize.generate_tokens(...)."""
        self.tokengen = tokengen
        self.tokens = []
        self.pos = 0
    def mark(self):
        return self.pos
    def reset(self, pos):
        self.pos = pos
    def get_token(self):
        token = self.peek_token()
        self.pos += 1
        return token
    def peek_token(self):
        if self.pos == len(self.tokens):
            self.tokens.append(next(self.tokengen))
        return self.tokens[self.pos]
```

现在，仍然缺失着很多东西（而且方法和实例变量的名称应该以下划线开头），但这作为 Tokenizer API 的初稿已经够了。

解析器也需要变成一个类，以便可以拥有 statement()、expr() 和其它方法。

标记器则变成一个实例变量，不过我们不希望解析方法（parsing methods）直接调用 get_token()——相反，我们给 `Parser` 类一个 `expect()` 方法，它可以像解析类方法一样，表示执行成功或失败。

 `expect()` 的参数是一个预期的标记——一个字符串（像“+”）或者一个标记类型（像`NAME`）。

讨论完了解析器的输出，我继续讲返回类型（return type）。

在我初稿的解析器中，解析函数只返回 True 或 False。那对于理论计算机科学来说是好的（解析器要解答的那类问题是“语言中的这个是否是有效的字符串？”），但是对于构建解析器却不是——相反，我们希望用解析器来创建一个 AST。

所以我们就这么办，即让每个解析方法在成功时返回 `Node` 对象，在失败时返回 `None` 。

该 `Node` 类可以超级简单：

```python
class Node:
    def __init__(self, type, children):
        self.type = type
        self.children = children
```

在这里，type 表示了该 AST 节点是什么类型（例如是个“add”节点或者“if”节点），children 表示了一些节点和标记（TokenInfo 类的实例）。

尽管将来我可能会改变表示 AST 的方式，但这足以让编译器生成代码或对其作分析了，例如 linting （译注：不懂）或者是静态类型检查。

为了适应这个方案，expect() 方法在成功时会返回一个 TokenInfo 对象，在失败时返回 None。为了支持回溯，我还封装了标记器的 mark() 和 reset() 方法（不改变 API）。

这是 Parser 类的基础结构：

```python
class Parser:
    def __init__(self, tokenizer):
        self.tokenizer = tokenizer
    def mark(self):
        return self.tokenizer.mark()
    def reset(self, pos):
        self.tokenizer.reset(pos)
    def expect(self, arg):
        token = self.tokenizer.peek_token()
        if token.type == arg or token.string == arg:
            return self.tokenizer.get_token()
        return None
```

同样地，我放弃了某些细节，但它可以工作。

在这里，我有必要介绍解析方法的一个重要的需求：一个解析方法要么返回一个 Node，并将标记器定位到它能识别的语法规则的最后一个标记之后；要么返回 None，然后保持标记器的位置不变。

如果解析方法在读取了多个标记之后失败了，则它必须重置标记器的位置。这就是 mark() 与 reset() 的用途。请注意，expect() 也遵循此规则。

所以解析器的实际草稿如下。请注意，我使用了 Python 3.8 的海象运算符（:=）：

```python
class ToyParser(Parser):
    def statement(self):
        if a := self.assignment():
            return a
        if e := self.expr():
            return e
        if i := self.if_statement():
            return i
        return None
    def expr(self):
        if t := self.term():
            pos = self.mark()
            if op := self.expect("+"):
                if e := self.expr():
                    return Node("add", [t, e])
            self.reset(pos)
            if op := self.expect("-"):
                if e := self.expr():
                    return Node("sub", [t, e])
            self.reset(pos)
            return t
        return None
    def term(self):
        # Very similar...
    def atom(self):
        if token := self.expect(NAME):
            return token
        if token := self.expect(NUMBER):
            return token
        pos = self.mark()
        if self.expect("("):
            if e := self.expr():
                if self.expect(")"):
                    return e
        self.reset(pos)
        return None
```

我给读者们留了一些解析方法作为练习（这实际上不仅仅是为了介绍解析器长什么样子），最终我们将像这样从语法中自动地生成代码。

NAME 和 NUMBER 等常量可从标准库的 `token` 库中导入。（这能令我们快速地进入 Python 的标记过程；但如果想要构建一个更加通用的 PEG 解析器，则应该探索一些其它方法。）

我还作了个小弊：`expr` 是左递归的，但我的解析器用了右递归，因为递归下降解析器不适用于左递归的语法规则。

有一个解决方案，但它还只是一些学术研究上的课题，我想以后单独介绍它。你们只需知道，修复的版本与这个玩具语法并非 100% 相符。

**我希望你们得到的关键信息是： ** 

- 语法规则相当于解析器方法，当一条语法规则引用另一条语法规则时，它的解析方法会调用另一条规则的解析方法
- 当多个条目构成备选项时，解析方法会一个接一个地调用相应的方法
- 当一条语法规则引用一个标记时，其解析方法会调用 expect() 
- 当一个解析方法在给定的输入位置成功地识别了它的语法规则时，它返回相应的 AST 节点；当识别失败时，它返回 None 
- 一个解析方法在消费（consum）一个或多个标记（直接或间接地，通过调用另一个成功的解析方法）后放弃解析时，必须显式地重置标记器的位置。这适用于放弃一个备选项而尝试下一个，也适用于完全地放弃解析

如果所有的解析方法都遵守这些规则，则不必在单个解析方法中使用 mark() 和 reset()。你可以用归纳法证明这一点。   

顺便提醒，虽然使用上下文管理器和 with 语句来替代显式地调用 mark() 与 reset() 很有诱惑力，但这不管用：在成功时不应调用 reset()！

为了修复它，你可以在控制流中使用异常，这样上下文管理器就知道是否该重置标记器（我认为 TatSu 做了类似的东西）。

举例，你可以这样做：

```python
    def statement(self):
        with self.alt():
            return self.assignment()
        with self.alt():
            return self.expr()
        with self.alt():
            return self.if_statement()
        raise ParsingFailure
```

特别地，`atom()` 中用来识别带括号的表达式的 if-语句，可以变成：

```python
        with self.alt():
            self.expect("(")
            e = self.expr()
            self.expect(")")
            return e
```

但我发现这太“神奇”了——在阅读这些代码时，你必须清醒地意识到每个解析方法（以及 expect()）都可能会引发异常，而这个异常会被 with 语句的上下文管理器捕获并忽略掉。

这相当不寻常，尽管肯定会支持（通过从 \_\_exit\_\_ 返回 true）。

还有，我的最终目标是生成 C，不是 Python，而在 C 里，没有 with 语句来改变控制流。

不管怎样，下面是未来的一些主题：

- 根据语法生成解析代码
- packrat 解析（记忆法）
- EBNF 的特性，如(x | y)、[x y ...]、x* 、x+
- tracing （用于调试解析器或语法）
- PEG 特性，如前瞻和“切割”
- 如何处理左递归规则
- 生成 C 代码


本文内容与示例代码的授权协议：[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0) 
