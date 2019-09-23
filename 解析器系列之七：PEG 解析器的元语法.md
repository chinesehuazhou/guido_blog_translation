# Python 之父的解析器系列之七：PEG 解析器的元语法

**原题** | [A Meta-Grammar for PEG Parsers](https://medium.com/@gvanrossum_83706/a-meta-grammar-for-peg-parsers-3d3d502ea332)

**作者** | Guido van Rossum（Python之父）

**译者** | 豌豆花下猫（“Python猫”公众号作者）

**声明** | 本翻译是出于交流学习的目的，基于 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权协议。为便于阅读，内容略有改动。本系列的译文已在 Github 开源，项目地址：[https://github.com/chinesehuazhou/guido_blog_translation](https://github.com/chinesehuazhou/guido_blog_translation)



本周我们使解析器生成器完成“自托管”（self-hosted），也就是让它自己生成解析器。

首先我们有了一个解析器生成器，其中一部分是语法解析器。我们可以称之为元解析器（meta-parser）。该元解析器与要生成的解析器类似：`GrammarParser` 继承自`Parser` ，它使用相同的 mark()/reset()/expect() 机制。然而，它是手写的。但是，只能是手写么？         

在编译器设计中有一个传统，即编译器使用它要编译的语言编写。我深切地记得在我初学编程时，当时用的 Pascal 编译器是用 Pascal 本身编写的，GCC 是用 C 编写的，Rust 编译器当然是用 Rust 编写的。

这是怎么做到的呢？有一个辅助过程（bootstrap，引导程序）：对于一种语言的子集或早期版本，它的编译器是用其它的语言编写的。（我记得最初的 Pascal 编译器是用 FORTRAN 编写的！）然后用编译后的语言编写一个新的编译器，并用辅助的编译器来编译它。一旦新的编译器运行得足够好，辅助的编译器就会被废弃，并且该语言或新编译器的每个新版本，都会受到先前版本的编译器的编译能力的约束。

让我们的元解析器如法炮制。我们将为语法编写一个语法（元语法），然后我们将从中生成一个新的元解析器。幸运的是我从一开始就计划了，所以这是一个非常简单的练习。我们在上一篇文章中添加的动作是必不可少的因素，因为我们不希望被迫去更改生成器——因此我们需要能够生成一个可兼容的数据结构。

这是一个不加动作的元语法的简化版：

```
start: rules ENDMARKER
rules: rule rules | rule
rule: NAME ":" alts NEWLINE
alts: alt "|" alts | alt
alt: items
items: item items | item
item: NAME | STRING
```

我将自下而上地展示如何添加动作。参照第 3 篇，我们有了一些带 name 和 alts 属性的 Rule 对象。最初，alts 只是一个包含字符串列表的列表（外层列表代表备选项，内层列表代表备选项的条目），但为了添加动作，我更改了一些内容，备选项由具有 items 和 action 属性的 Alt 对象来表示。条目仍然由纯字符串表示。对于 item 规则，我们有：             

```
item: NAME { name.string } | STRING { string.string }
```

这需要一些解释：当解析器处理一个标识符时，它返回一个 TokenInfo 对象，该对象具有 type、string 及其它属性。我们不希望生成器来处理 TokenInfo 对象，因此这里加了动作，它会从标识符中提取出字符串。请注意，对于像 NAME 这样的全大写标识符，生成的解析器会使用小写版本（此处为 name ）作为变量名。          

接下来是 items 规则，它必须返回一个字符串列表： 

```
items: item items { [item] + items } | item { [item] }
```

我在这里使用右递归规则，所以我们不依赖于第 5 篇中添加的左递归处理。（为什么不呢？保持事情尽可能简单总是一个好主意，这个语法使用左递归的话，不是很清晰。）请注意，单个的 item 已被分层，但递归的 items 没有，因为它已经是一个列表。    

alt 规则用于构建 Alt 对象：

```
alt: items { Alt(items) }
```

我就不介绍 rules 和 start 规则了，因为它们遵循相同的模式。    

但是，有两个未解决的问题。首先，生成的代码如何知道去哪里找到 Rule 和 Alt 类呢？为了实现这个目的，我们需要为生成的代码添加一些 import 语句。最简单的方法是给生成器传递一个标志，该标志表示“这是元语法”，然后让生成器在生成的程序顶部引入额外的 import 语句。但是既然我们已经有了动作，许多其它解析器也会想要自定义它们的导入，所以为什么我们不试试看，能否添加一个更通用的功能呢。      

有很多方法可以剥了这只猫的皮（译注：skin this cat，解决这个难题）。一个简单而通用的机制是在语法的顶部添加一部分“变量定义”，并让生成器使用这些变量，来控制生成的代码的各个方面。我选择使用 @ 字符来开始一个变量定义，在它之后是变量名（一个 NAME）和值（一个 STRING）。例如，我们可以将以下内容放在元语法的顶部：    

```
@subheader "from grammar import Rule, Alt"
```

标准的导入总是会打印（例如，去导入 memoize），在那之后，解析器生成器会打印 `subheader` 变量的值。如果需要多个 import，可以在变量声明中使用三引号字符串，例如：    

```
@subheader """
from token import OP
from grammar import Rule, Alt
"""
```

这很容易添加到元语法中，我们用这个替换 start 规则：  

```
start: metas rules ENDMARKER | rules ENDMARKER
metas: meta metas | meta
meta: "@" NAME STRING NEWLINE
```

（我不记得为什么我会称它们为“metas”，但这是我在编写代码时选择的名称，我会坚持这样叫。:-)

我们还必须将它添加到辅助的元解析器中。既然语法不仅仅是一系列的规则，那么让我们添加一个 Grammar  对象，其中包含属性 `metas` 和 `rules`。我们可以放入如下的动作：

```
start: metas rules ENDMARKER { Grammar(rules, metas) }
     | rules ENDMARKER { Grammar(rules, []) }
metas: meta metas { [meta] + metas }
     | meta { [meta] }
meta: "@" NAME STRING { (name.string, eval(string.string)) }
```

（注意 meta 返回一个元组，并注意它使用 eval() 来处理字符串引号。）    

说到动作，我漏讲了 alt 规则的动作！原因是这里面有些混乱。但我不能再无视它了，上代码吧： 

```
alt: items action { Alt(items, action) }
   | items { Alt(items, None) }
action: "{" stuffs "}" { stuffs }
stuffs: stuff stuffs { stuff + " " + stuffs }
      | stuff { stuff }
stuff: "{" stuffs "}" { "{" + stuffs + "}" }
     | NAME { name.string }
     | NUMBER { number.string }
     | STRING { string.string }
     | OP { None if op.string in ("{", "}") else op.string }
```

这个混乱是由于我希望在描绘动作的花括号之间允许任意 Python 代码，以及允许配对的大括号嵌套在其中。为此，我们使用了特殊标识符 OP，标记生成器用它生成可被 Python 识别的所有标点符号（返回一个类型为 OP 标识符，用于多字符运算符，如 <= 或 ** ）。在 Python 表达式中可以合法地出现的唯一其它标识符是名称、数字和字符串。因此，在动作的最外侧花括号之间的“东西”似乎是一组循环的 NAME | NUMBER | STRING | OP 。       

呜呼，这没用，因为 OP 也匹配花括号，但由于 PEG 解析器是贪婪的，它会吞掉结束括号，我们就永远看不到动作的结束。因此，我们要对生成的解析器添加一些调整，允许动作通过返回 None 来使备选项失效。我不知道这是否是其它 PEG 解析器的标准配置——当我考虑如何解决右括号（甚至嵌套的符号）的识别问题时，立马就想到了这个方法。它似乎运作良好，我认为这符合 PEG 解析的一般哲学。它可以被视为一种特殊形式的前瞻（我将在下面介绍）。

使用这个小调整，当出现花括号时，我们可以使 OP 上的匹配失效，它可以通过 stuff 和 action 进行匹配。

有了这些东西，元语法可以由辅助的元解析器解析，并且生成器可以将它转换为新的元解析器，由此解析自己。更重要的是，新的元解析器仍然可以解析相同的元语法。如果我们使用新的元编译器编译元语法，则输出是相同的：这证明生成的元解析器正常工作。

这是带有动作的完整元语法。只要你把解析过程串起来，它就可以解析自己：

```
@subheader """
from grammar import Grammar, Rule, Alt
from token import OP
"""
start: metas rules ENDMARKER { Grammar(rules, metas) }
     | rules ENDMARKER { Grammar(rules, []) }
metas: meta metas { [meta] + metas }
     | meta { [meta] }
meta: "@" NAME STRING NEWLINE { (name.string, eval(string.string)) }
rules: rule rules { [rule] + rules }
     | rule { [rule] }
rule: NAME ":" alts NEWLINE { Rule(name.string, alts) }
alts: alt "|" alts { [alt] + alts }
    | alt { [alt] }
alt: items action { Alt(items, action) }
   | items { Alt(items, None) }
items: item items { [item] + items }
     | item { [item] }
item: NAME { name.string }
    | STRING { string.string }
action: "{" stuffs "}" { stuffs }
stuffs: stuff stuffs { stuff + " " + stuffs }
      | stuff { stuff }
stuff: "{" stuffs "}" { "{" + stuffs + "}" }
     | NAME { name.string }
     | NUMBER { number.string }
     | STRING { string.string }
     | OP { None if op.string in ("{", "}") else op.string }
```

现在我们已经有了一个能工作的元语法，可以准备做一些改进了。

但首先，还有一个小麻烦要处理：空行！事实证明，标准库的 tokenize 会生成额外的标识符来跟踪非重要的换行符和注释。对于前者，它生成一个 NL 标识符，对于后者，则是一个 COMMENT 标识符。以其将它们吸收进语法中（我已经尝试过，但并不容易！），我们可以在 tokenizer 类中添加一段非常简单的代码，来过滤掉这些标识符。这是改进的 peek_token 方法：       

```python
def peek_token(self):
        if self.pos == len(self.tokens):
            while True:
                token = next(self.tokengen)
                if token.type in (NL, COMMENT):
                    continue
                break
            self.tokens.append(token)
            self.report()
        return self.tokens[self.pos]
```

这样就完全过滤掉了 NL 和 COMMENT 标识符，因此在语法中不再需要担心它们。

最后让我们对元语法进行改进！我想做的事情纯粹是美容性的：我不喜欢被迫将所有备选项放在同一行上。我上面展示的元语法实际上并没有解析自己，因为有这样的情况：

```
start: metas rules ENDMARKER { Grammar(rules, metas) }
     | rules ENDMARKER { Grammar(rules, []) }
```

这是因为标识符生成器（tokenizer）在第一行的末尾产生了一个 NEWLINE 标识符，此时元解析器会认为这是该规则的结束。此外，NEWLINE 之后会出现一个 INDENT 标识符，因为下一行是缩进的。在下一个规则开始之前，还会有一个 DEDENT 标识符。

下面是解决办法。为了理解 tokenize 模块的行为，我们可以将 tokenize 模块作为脚本运行，并为其提供一些文本，以此来查看对于缩进块，会生成什么样的标识符序列：   

```python
$ python -m tokenize
foo bar
    baz
    dah
dum
^D
```

我们发现它会产生以下的标识符序列（我已经简化了上面运行的输出）：

```python
NAME     'foo'
NAME     'bar'
NEWLINE
INDENT
NAME     'baz'
NEWLINE
NAME     'dah'
NEWLINE
DEDENT
NAME     'dum'
NEWLINE
```

这意味着一组缩进的代码行会被 INDENT 和 DEDENT 标记符所描绘。现在，我们可以重新编写元语法规则的 rule 如下：

```
rule: NAME ":" alts NEWLINE INDENT more_alts DEDENT {
        Rule(name.string, alts + more_alts) }
    | NAME ":" alts NEWLINE { Rule(name.string, alts) }
    | NAME ":" NEWLINE INDENT more_alts DEDENT {
        Rule(name.string, more_alts) }
more_alts: "|" alts NEWLINE more_alts { alts + more_alts }
         | "|" alts NEWLINE { alts }
```

（我跨行地拆分了动作，以便它们适应 Medium 网站的窄页——这是可行的，因为标识符生成器会忽略已配对的括号内的换行符。）

这样做的好处是我们甚至不需要更改生成器：这种改进的元语法生成的数据结构跟以前相同。同样注意 rule 的第三个备选项，对此让我们写： 

```
start:
    | metas rules ENDMARKER { Grammar(rules, metas) }
    | rules ENDMARKER { Grammar(rules, []) }
```

有些人会觉得这比我之前展示的版本更干净。很容易允许两种形式共存，所以我们不必争论风格。

在下一篇文章中，我将展示如何实现各种 PEG 功能，如可选条目、重复和前瞻。（说句公道话，我本打算把那放在这篇里，但是这篇已写太长了，所以我要把它分成两部分。）

本文的许可证和显示的代码：[CC BY-NC-SA 4.0](https://translate.google.com/translate?hl=zh-CN&prev=_t&sl=en&tl=zh-CN&u=https://creativecommons.org/licenses/by-nc-sa/4.0/) 