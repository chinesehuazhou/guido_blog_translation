# 解析器系列之五：左递归 PEG 语法

**原题** | [Left-recursive PEG grammars](https://medium.com/@gvanrossum_83706/left-recursive-peg-grammars-65dab3c580e1)

**作者** | Guido van Rossum（Python之父）

**译者** | 豌豆花下猫（“Python猫”公众号作者）

**声明** | 本翻译是出于交流学习的目的，基于 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权协议。为便于阅读，内容略有改动。



我曾几次提及左递归是一块绊脚石，是时候去解决它了。基本的问题在于：使用递归下降解析器时，左递归会因堆栈溢出而导致程序终止。

假设有如下的语法规则：

```
expr: expr '+' term | term
```

如果我们天真地将它翻译成递归下降解析器的片段，会得到如下内容：

```python
def expr():
    if expr() and expect('+') and term():
        return True
    if term():
        return True
    return False
```

也就是`expr()` 以调用`expr()` 开始，后者也以调用`expr()` 开始，以此类推……这只能以堆栈溢出而结束，抛出异常`RecursionError` 。

传统的补救措施是重写语法。在之前的文章中，我已经这样做了。事实上，上面的语法也能识别出来，如果我们重写成这样：

```
expr: term '+' expr | term
```

但是，如果我们用它生成一个解析树，那么解析树的形状会有所不同，这会导致破坏性的后果，比如当我们在语法中添加一个`'-'` 运算符时（因为`a - (b - c)` 与`(a - b) - c` 不一样）。

这通常可以使用更强大的 PEG 特性来解决，例如分组和迭代，我们可以将上述规则重写为：

```
expr: term ('+' term)*
```

实际上，这正是 Python 当前语法在 pgen 解析器生成器上的写法（pgen 与左递归规则具有同样的问题）。  

但是这仍然存在一些问题：因为像`'+'` 和`'-'` 这样的运算符，基本上是二进制的（在 Python 中），当我们解析像`a + b + c` 这样的东西时，我们必须遍历解析的结果（基本上是列表['a'，'+'，'b'，'+'，'c'] ），以构造一个左递归的解析树（类似于 [['a'，'+'，'b'] ，'+'，'c'] ）。        

原始的左递归语法已经表诉了所需的关联性，因此，如果我们可以直接以该形式生成解析器，那将会很好。我们可以！一位粉丝向我指出了一个很好的技巧，还附带了一个数学证明，很容易实现。我会试着在这里解释一下。

让我们考虑输入`foo + bar + baz` 作为示例。我们想要解析出的解析树对应于`(foo + bar)+ baz` 。这需要对`expr()` 进行三次左递归调用：一次对应于顶级的“+” 运算符（即第二个）; 一次对应于内部的“+”运算符（即第一个）; 还有一次是选择第二个备选项（即`term` ）。        

由于我不善于使用计算机绘制实际的图表，因此我将在此使用 ASCII 技巧作演示：

```
expr------------+------+
  |              \      \
expr--+------+   '+'   term
  |    \      \          |
expr   '+'   term        |
  |            |         |
term           |         |
  |            |         |
'foo'        'bar'     'baz'
```

我们的想法是希望在 expr() 函数中有一个“oracle”（译注：预言、神谕，后面就不译了），它要么告诉我们采用第一个备选项（即递归调用 expr()），要么是第二个（即调用 term()）。在第一次调用 expr() 时，“oracle”应该返回 true; 在第二次（递归）调用时，它也应该返回 true，但在第三次调用时，它应该返回 false，以便我们可以调用 term()。

在代码中，应该是这样：        

```python
def expr():
    if oracle() and expr() and expect('+') and term():
        return True
    if term():
        return True
    return False
```

我们该怎么写这样的“oracle”呢？试试看吧......我们可以尝试记录在调用堆栈上的 expr() 的（左递归）调用次数，并将其与下面表达式中“+” 运算符的数量进行比较。如果调用堆栈的深度大于运算符的数量，则应该返回 false。

我几乎想用`sys._getframe()` 来实现它，但有更好的方法：让我们反转调用的堆栈！     

这里的想法是我们从 oracle 返回 false 处调用，并保存结果。这就有了`expr()->term()->'foo'` 。（它应该返回初始的`term` 的解析树，即`'foo'` 。上面的代码仅返回 True，但在本系列第二篇文章中，我已经演示了如何返回一个解析树。）很容易编写一个 oracle 来实现，它应该在首次调用时就返回 false——不需要检查堆栈或向前回看。

然后我们再次调用`expr()` ，这时 oracle 会返回 true，但是我们不对 expr() 进行左递归调用，而是用前一次调用时保存的结果来替换。瞧呐，预期的`'+'` 运算符及随后的`term` 也出现了，所以我们将会得到`foo + bar` 。

我们重复这个过程，然后事情看起来又很清晰了：这次我们会得到完整表达式的解析树，并且它是正确的左递归（（foo + bar）+ baz ）。

然后我们再次重复该过程，这一次，oracle 返回 true，并且前一次调用时保存的结果可用，没有下一步的'+' 运算符，并且第一个备选项失效。所以我们尝试第二个备选项，它会成功，正好找到了初始的 term（'foo'）。与之前的调用相比，这是一个糟糕的结果，所以在这里我们停止并留下最长的解析（即（foo + bar）+ baz ）。     

为了将其转换为实际的工作代码，我首先要稍微重写代码，以将 oracle() 的调用与左递归的 expr() 调用相结合。我们称之为`oracle_expr()` 。代码：

```python
def expr():
    if oracle_expr() and expect('+') and term():
        return True
    if term():
        return True
    return False
```

接着，我们将编写一个实现上述逻辑的装饰器。它使用了一个全局变量（不用担心，我稍后会改掉它）。`oracle_expr()` 函数将读取该全局变量，而装饰器操纵着它：  

```python
saved_result = None
def oracle_expr():
    if saved_result is None:
        return False
    return saved_result
def expr_wrapper():
    global saved_result
    saved_result = None
    parsed_length = 0
    while True:
        new_result = expr()
        if not new_result:
            break
        new_parsed_length = <calculate size of new_result>
        if new_parsed_length <= parsed_length:
            break
        saved_result = new_result
        parsed_length = new_parsed_length
    return saved_result
```

这过程当然是可悲的，但它展示了代码的要点，所以让我们尝试一下，将它发展成我们可以引以为傲的东西。

决定性的洞察（这是我自己的，虽然我可能不是第一个想到的）是我们可以使用记忆缓存而不是全局变量，将一次调用的结果保存到下一次，然后我们不需要额外的`oracle_expr()` 函数——我们可以生成对 expr() 的标准调用，无论它是否处于左递归的位置。

为了做到这点，我们需要一个单独的 @memoize_left_rec 装饰器，它只用于左递归规则。它通过将保存的值从记忆缓存中取出，充当了 oracle_expr() 函数的角色，并且它包含着一个循环调用，只要每个新结果所覆盖的部分比前一个长，就反复地调用 expr()。

当然，因为记忆缓存分别按输入位置和每个解析方法来处理缓存，所以它不受回溯或多个递归规则的影响（例如，在玩具语法中，我一直使用 expr 和 term 都是左递归的）。          

我在第 3 篇文章中创建的基础结构的另一个不错的属性是它更容易检查新结果是否长于旧结果：mark() 方法将索引返回到输入的标记符数组中，因此我们可以使用它，而非上面的parsed_length 。

我没有证明为什么这个算法总是有效的，不管这个语法有多疯狂。那是因为我实际上没有读过那个证明。我看到它适用于玩具语法中的 expr 等简单情况，也适用于更复杂的情况（例如，涉及一个备选项里可选条目背后藏着的左递归，或涉及多个规则之间的相互递归），但在 Python 的语法中，我能想到的最复杂的情况仍然相当温和，所以我可以信任于定理和证明它的人。  

所以让我们坚持干，并展示一些真实的代码。

首先，解析器生成器必须检测哪些规则是左递归的。这是图论中一个已解决的问题。我不会在这里展示算法，事实上我将进一步简化工作，并假设语法中唯一的左递归规则就是直接左递归的，就像我们的玩具语法中的 expr 一样。然后检查左递归只需要查找以当前规则名称开头的备选项。我们可以这样写：   

```python
def is_left_recursive(rule):
    for alt in rule.alts:
        if alt[0] == rule.name:
            return True
    return False
```

现在我们修改解析器生成器，以便对于左递归规则，它能生成一个不同的装饰器。回想一下，在第 3 篇文章中，我们使用 @memoize 修饰了所有的解析方法。我们现在对生成器进行一个小小的修改，对于左递归规则，我们替换成 @memoize_left_rec ，然后我们在memoize_left_rec 装饰器中变魔术。生成器的其余部分和支持代码无需更改！（然而我不得不在可视化代码中捣鼓一下。）   

作为参考，这里是原始的 @memoize 装饰器，从第 3 篇中复制而来。请注意，self 是一个Parser 实例，它具有 memo 属性（用空字典初始化）、mark() 和 reset() 方法，用于获取和设置 tokenizer 的当前位置：

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

@memoize 装饰器在每个输入位置记住了前一调用——在输入标记符的（惰性）数组的每个位置，有一个单独的`memo` 字典。memoize_wrapper 函数的前四行与获取正确的`memo` 字典有关。

这是 @memoize_left_rec 。只有 else 分支与上面的 @memoize 不同：     

```python
    def memoize_left_rec(func):
    def memoize_left_rec_wrapper(self, *args):
        pos = self.mark()
        memo = self.memos.get(pos)
        if memo is None:
            memo = self.memos[pos] = {}
        key = (func, args)
        if key in memo:
            res, endpos = memo[key]
            self.reset(endpos)
        else:
            # Prime the cache with a failure.
            memo[key] = lastres, lastpos = None, pos
            # Loop until no longer parse is obtained.
            while True:
                self.reset(pos)
                res = func(self, *args)
                endpos = self.mark()
                if endpos <= lastpos:
                    break
                memo[key] = lastres, lastpos = res, endpos
            res = lastres
            self.reset(lastpos)
        return res
    return memoize_left_rec_wrapper
```

它很可能有助于显示生成的 expr() 方法，因此我们可以跟踪装饰器和装饰方法之间的流程：  

```python
    @memoize_left_rec 
    def expr(self):
        pos = self.mark()
        if ((expr := self.expr()) and
            self.expect('+') and
            (term := self.term())):
            return Node('expr', [expr, term])
        self.reset(pos)
        if term := self.term():
            return Node('term', [term])
        self.reset(pos)
        return None
```

让我们试着解析 `foo + bar + baz` 。 

每当你调用被装饰的 expr() 函数时，装饰器就会“拦截”调用，它会在当前位置查找前一个调用。在第一个调用处，它会进入 else 分支，在那里它重复地调用未装饰的函数。当未装饰的函数调用 expr() 时，这当然指向了被装饰的版本，因此这个递归调用会再次被截获。递归在这里停止，因为现在 memo 缓存有了命中。       

接下来呢？初始的缓存值来自这行：

```
            # Prime the cache with a failure.
            memo[key] = lastres, lastpos = None, pos
```

这使得被装饰的 expr() 返回 None，在那 expr() 里的第一个 if 会失败（在`expr := self.expr()` ）。所以我们继续到第二个 if，它成功识别了一个 term（在我们的例子中是 ‘foo’），expr 返回一个 Node 实例。它返回到了哪里？到了装饰器里的 while 循环。这新的结果会更新 memo 缓存（那个 node 实例），然后开始下一个迭代。

再次调用未装饰的 expr()，这次截获的递归调用返回新缓存的 Node 实例（一个 term）。这是成功的，调用继续到 expect('+')。这再次成功，然后我们现在处于第一个“+” 操作符。在此之后，我们要查找一个 term，也成功了（找到 'bar'）。

所以对于空的 expr()，目前已识别出 `foo + bar` ，回到 while 循环，还会经历相同的过程：用新的（更长的）结果来更新 memo 缓存，并开启下一轮迭代。

游戏再次上演。被截获的递归 expr() 调用再次从缓存中检索新的结果（这次是 foo + bar），我们期望并找到另一个 ‘+’（第二个）和另一个 term（‘baz’）。我们构造一个 Node 表示 `(foo + bar) + baz` ，并返回给 while 循环，后者将它填充进 memo 缓存，并再次迭代。

但下一次事情会有所不同。有了新的结果，我们查找另一个 '+' ，但没有找到！所以这个expr() 调用会回到它的第二个备选项，并返回一个可怜的 term。当走到 while 循环时，它失望地发现这个结果比最后一个短，就中断了，将更长的结果（（foo + bar）+ baz ）返回给原始调用，就是初始化了外部 expr() 调用的地方（例如，一个 statement() 调用——此处未展示）。

到此，今天的故事结束了：我们已经成功地在 PEG（-ish）解析器中驯服了左递归。至于下周，我打算论述在语法中添加“动作”（actions），这样我们就可以为一个给定的备选项的解析方法，自定义它返回的结果（而不是总要返回一个 Node 实例）。 

如果你想使用代码，请参阅[GitHub仓库](https://github.com/gvanrossum/pegen/tree/master/story4)。（我还为左递归添加了可视化代码，但我并不特别满意，所以不打算在这里给出链接。） 

本文内容与示例代码的授权协议：[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0) 


