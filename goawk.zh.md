| 原文                                               | 时间          |
| -------------------------------------------------- | ------------- |
| [benhoyt blog](https://benhoyt.com/writings/goawk) | 2018 年 11 月 |

## [GoAWK，一个用 Go 编写的 AWK 解释器](https://benhoyt.com/writings/goawk/)

> 简介：阅读 *AWK 编程语言之后* 我受到启发，用 Go 语言 为 AWK 编写了一个解释器。本文向你概述了什么是 AWK，描述了 GoAWK 的工作原理，我是如何进行测试的，以及我如何衡量和改进其性能。


> **转至：** [AWK 概述](#awk-%E6%A6%82%E8%BF%B0) | [代码演练](#%E4%BB%A3%E7%A0%81%E6%BC%94%E7%BB%83) | [测试](#%E6%88%91%E6%98%AF%E5%A6%82%E4%BD%95%E6%B5%8B%E8%AF%95%E5%AE%83%E7%9A%84t) | [性能](#%E6%8F%90%E9%AB%98%E6%80%A7%E8%83%BD)

### 大目录

<details>

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [AWK 概述](#awk-%E6%A6%82%E8%BF%B0)
- [代码演练](#%E4%BB%A3%E7%A0%81%E6%BC%94%E7%BB%83)
  - [词法分析器](#%E8%AF%8D%E6%B3%95%E5%88%86%E6%9E%90%E5%99%A8)
  - [解析器](#%E8%A7%A3%E6%9E%90%E5%99%A8)
  - [分解器](#%E5%88%86%E8%A7%A3%E5%99%A8)
  - [解释器](#%E8%A7%A3%E9%87%8A%E5%99%A8)
  - [主程序](#%E4%B8%BB%E7%A8%8B%E5%BA%8F)
- [我是如何测试它的](#%E6%88%91%E6%98%AF%E5%A6%82%E4%BD%95%E6%B5%8B%E8%AF%95%E5%AE%83%E7%9A%84)
  - [Lexer 测试](#lexer-%E6%B5%8B%E8%AF%95)
  - [解析测试](#%E8%A7%A3%E6%9E%90%E6%B5%8B%E8%AF%95)
  - [解释器测试](#%E8%A7%A3%E9%87%8A%E5%99%A8%E6%B5%8B%E8%AF%95)
  - [命令行测试](#%E5%91%BD%E4%BB%A4%E8%A1%8C%E6%B5%8B%E8%AF%95)
  - [AWK 测试套件](#awk-%E6%B5%8B%E8%AF%95%E5%A5%97%E4%BB%B6)
  - [模糊测试](#%E6%A8%A1%E7%B3%8A%E6%B5%8B%E8%AF%95)
- [提高性能](#%E6%8F%90%E9%AB%98%E6%80%A7%E8%83%BD)
  - [我是如何分析的](#%E6%88%91%E6%98%AF%E5%A6%82%E4%BD%95%E5%88%86%E6%9E%90%E7%9A%84)
  - [性能改进](#%E6%80%A7%E8%83%BD%E6%94%B9%E8%BF%9B)
  - [性能表现](#%E6%80%A7%E8%83%BD%E8%A1%A8%E7%8E%B0)
- [从哪里来？](#%E4%BB%8E%E5%93%AA%E9%87%8C%E6%9D%A5)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

</details>

---

AWK 是一种引人入胜的文本处理语言，而[_AWK 编程语言_](https://ia802309.us.archive.org/25/items/pdfy-MgN0H1joIoDVoIC7/The_AWK_Programming_Language.pdf)是一本描述它的,非常简洁的书。AWK 中的 A，W 和 K 代表三位原创作者的姓氏：Alfred Aho，Peter Weinberger 和 Brian Kernighan。Kernighan 也是*The C Programming Language*（“K＆R”）的作者，这两本书具有相同的每页风格感觉。

AWK 于 1977 年发布，超过 40 年！对于这种特定领域的语言来说并不常见，而今它仍大量用于 Unix 命令行上的单行控制。

在实现了我个人的一种[语言](https://benhoyt.com/writings/littlelang/) , 还有 Bob Nystrom 的[在Lox中实现的Lox解释器](https://benhoyt.com/writings/loxlox/)后，我好像处在一个“高级解释员”的位置 。在阅读了 AWK 书之后，我认为用 Go 语言 为它编写一个解释器会相当有趣（或一些与“乐趣”想近的意思）。它能有多难？

事实证明，让它在基本功能级别上工作并不是很难，但在获得正确的[POSIX AWK](http://pubs.opengroup.org/onlinepubs/9699919799/utilities/awk.html)语义，并使其快速的上面，有点棘手。

首先，简要介绍 AWK 语言（如果您已经知道，请跳到下一部分）。

## AWK 概述

如果您不熟悉 AWK，这里有一句话摘要：`AWK 逐行读取文本文件，对于与模式表达式匹配的每一行，它执行一个操作动作（通常打印输出）`。

因此，给出一个输入文件示例（Web 服务器日志文件），其中每行使用以下格式

`"timestamp method path ip status time"` ：

```
2018-11-07T07:56:34Z GET /about 1.2.3.4 200 0.013
2018-11-07T07:56:35Z GET /contact 1.2.3.4 200 0.020
2018-11-07T07:56:37Z POST /contact 1.2.3.4 200 1.309
2018-11-07T07:56:40Z GET /robots.txt 123.0.0.1 404 0.004
2018-11-07T07:57:00Z GET /about 2.3.4.5 200 0.014
2018-11-07T08:00:00Z GET /asdf 3.4.5.6 404 0.005
2018-11-07T08:00:01Z GET /fdsa 3.4.5.6 404 0.004
2018-11-07T08:00:02Z HEAD / 4.5.6.7 200 0.008
2018-11-07T08:00:15Z GET / 4.5.6.7 200 0.055
2018-11-07T08:05:57Z GET /robots.txt 201.12.34.56 404 0.004
2018-11-07T08:05:58Z HEAD / 5.6.7.8 200 0.007
2018-11-07T08:05:59Z GET / 5.6.7.8 200 0.049

```

如果我们想要查看`/about`页面中，所有命中的 IP 地址（字段 4），我们可以写：

```bash
$ awk '/about/ { print $4 }' server.log
1.2.3.4
2.3.4.5

```

上面的模式是斜线分隔的正则表达式`/about/`，操作是打印第四个字段（`$4`）。默认情况下，AWK 以空白格，将行拆分字段，但字段分隔符很容易配置，并可以是正则表达式。

通常，正则表达式模式匹配整行，但您也可以匹配任意表达式。上面写法也会匹配 URL `/not-about`，但你可以收紧它，通过测试路径（字段 3）是否正好`"/about"`：

```bash
$ awk '$3 == "/about" { print $4 }' server.log
1.2.3.4
2.3.4.5

```

如果我们想确定所有 GET 请求的平均响应时间（字段 6），我们可以将响应时间相加，并计算 GET 请求的数量，然后打印`END`块中的平均值 - **18 毫秒**，不错：

```bash
$ awk '/GET/ { total += $6; n++ } END { print total/n }' server.log
0.0186667

```

AWK 支持哈希表（称为“关联数组”），因此您可以像这样打印每个请求方法的计数 - 请注意该模式是可 pe 的，并在此处省略：

```bash
$ awk '{ num[$2]++ } END { for (m in num) print m, num[m] }' server.log
GET 9
POST 1
HEAD 2

```

AWK 有两个标量类型，字符串和数字，但它被描述为“字符串类型”，因为如果数据来自用户输入并作为数字解析，就用`==`和`<`这样的比较运算符进行数字比较，否则进行字符串比较。这听起来很草率，但对于文本处理，它通常就是你想要的。

该语言支持的类 C 的表达式和控制结构（通常的`if`，`for`区块范围等等）。它还具有一系列内置函数，如`substr()`和`tolower()`，它支持用户定义的函数，以及局部变量和数组参数。

所以它绝对是完整图灵性(符合)的，实际上这是一个非常好的，强大的语言。你甚至[可以用](https://github.com/benhoyt/goawk/blob/master/examples/mandel.awk)几十行代码就[生成 Mandelbrot 集合](https://github.com/benhoyt/goawk/blob/master/examples/mandel.awk)：

```bash
$ awk -f examples/mandel.awk

```

```
......................................................................................................................................................
............................................................................................................-.........................................
.....................................................................................................----++-*------...................................
...................................................................................................--------$+---------................................
................................................................................................-----------++$++--+++---..............................
..............................................................................................--------------++*%#+++------............................
............................................................................................--------------++%*%@*++----------.........................
.........................................................................................------------++**++*#    *++++%----------.....................
......................................................................................--------------+++             %#%+-------------.................
..................................................................................-------------------+* @           %*+------------------.............
.............................................................................---------------------+++++              +++--------------------..........
........................................................................----------*%+**#@++++++$ %++****%%        $%**+**+++#++---------+*+---........
.................................................................-----------------+*$% $ #  ++*   $                       #   *++++#*+++**++----......
..........................................................------------------------+++@      #                                  @**#   @  *#+-----.....
....................................................-----------------------------++++*#                                                 %*+-------....
...............................................------------------------------+$+*%**#                                                 %*++--------....
..........................................---+-------------------------------++                                                        %#+---------...
......................................--------+ +----------++---------------++**%$                                                       *%*+* ----...
..................................------------+*+++++*+++++ *++++++-----+++++$@                                                               +----...
...............................----------------+++#% $$**%* @ $**#%+++++++++                                                              %++------...
............................------------------+++*%$                $ *++++*                                                              # **-----...
..........................-------------------+*+**#                    @%**%                                                              #$+------...
.......................---------------%++++++++*#                         ##                                                               $+------...
....................-----------------+++#**%***#                                                                                          *--------...
.......-----------++--------------++++**%$                                                                                              +----------...
......                                                                                                                              %*++-----------...
.......-----------++--------------++++**%$                                                                                              +----------...
....................-----------------+++#**%***#                                                                                          *--------...
.......................---------------%++++++++*#                         ##                                                               $+------...
..........................-------------------+*+**#                    @%**%                                                              #$+------...
............................------------------+++*%$                $ *++++*                                                              # **-----...
...............................----------------+++#% $$**%* @ $**#%+++++++++                                                              %++------...
..................................------------+*+++++*+++++ *++++++-----+++++$@                                                               +----...
......................................--------+ +----------++---------------++**%$                                                       *%*+* ----...
..........................................---+-------------------------------++                                                        %#+---------...
...............................................------------------------------+$+*%**#                                                 %*++--------....
....................................................-----------------------------++++*#                                                 %*+-------....
..........................................................------------------------+++@      #                                  @**#   @  *#+-----.....
.................................................................-----------------+*$% $ #  ++*   $                       #   *++++#*+++**++----......
........................................................................----------*%+**#@++++++$ %++****%%        $%**+**+++#++---------+*+---........
.............................................................................---------------------+++++              +++--------------------..........
..................................................................................-------------------+* @           %*+------------------.............
......................................................................................--------------+++             %#%+-------------.................
.........................................................................................------------++**++*#    *++++%----------.....................
............................................................................................--------------++%*%@*++----------.........................
..............................................................................................--------------++*%#+++------............................
................................................................................................-----------++$++--+++---..............................
...................................................................................................--------$+---------................................
.....................................................................................................----++-*------...................................
............................................................................................................-.........................................
```

而这也就不过是，AWK 的一个小能力。

## 代码演练

GoAWK 在编译器设计方面并不突破。它由词法分析器(lexer)，解析器(parser)，分解器(resolver)，解释器(interpreter)和主程序(main)（[GitHub repo](https://github.com/benhoyt/goawk)）组成。只用 Go 标准库包，制作此程序。

### 词法分析器

这一切都始于[词法分析器](https://github.com/benhoyt/goawk/blob/master/lexer/lexer.go)，它将 AWK 源代码转换为`tokens(标记)`流。词法分析器的核心是`scan()`方法，其会跳过空格和注释，然后解析下一个`token`：例如，`DOLLAR`，`NUMBER`，或`LPAREN`。返回每个`token(标记)`及其源代码位置（行和列），以便解析器可以在语法错误消息中,包含此信息。

大部分代码（`Lexer.scan`方法）只是一个大的 **switch** 语句，来自`token`的第一个字符。下面是一个片段：

```go
// ...
switch ch {
case '$':
    tok = DOLLAR
case '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', '.':
    start := l.offset - 2
    gotDigit := false
    if ch != '.' {
        gotDigit = true
        for l.ch >= '0' && l.ch <= '9' {
            l.next()
        }
        if l.ch == '.' {
            l.next()
        }
    }
    // ...
    tok = NUMBER
case '{':
    tok = LBRACE
case '}':
    tok = RBRACE
case '=':
    tok = l.choice('=', ASSIGN, EQUALS)
// ...

```

关于 AWK 语法的一个奇怪的事情是解析`/`和`/regex/`并不明确 - 你必须知道解析上下文，以知道是返回一个`DIV`还是一个`REGEX`的标记。因此，词法分析器暴露了一个针对普通标记的`Scan`和一个解析器调用正则标记时，所需的`ScanRegex`方法。

### 解析器

接下来是[解析器](https://github.com/benhoyt/goawk/blob/master/parser/parser.go)，一个相当标准的深度递归解析器，它创建一个[抽象语法树（AST）](https://github.com/benhoyt/goawk/blob/master/internal/ast/ast.go)。我不喜欢学习如何驱动解析器生成器，像`yacc`或引入外部依赖，所以 GoAWK 的解析器是用爱，手工制作的。

AST 节点是简单的 Go 结构，每个表达式和语句结构都实现`Expr`和`Stmt`作为接口。AST 节点也可以通过调用`String()`方法来自行打印 - 这对于调试解析器非常有用，您可以命令行上指定`-d`，启用它：

```bash
$ goawk -d 'BEGIN { x=4; print x+3; }'
BEGIN {
    x = 4
    print (x + 3)
}
7

```

AWK 语法在某些地方有点古怪，其中最重要的是`print`语句中，不支持表达式`>`或`|`(除括号内)。这些应该能使重定向或管道输出更简单。

- `print x > y`表示打印变量`x`重定向到具有名称的文件`y`
- `print (x > y)`表示 打印 布尔:true（1），若`x`大于`y`

我无法想出一个更好的方法来做，两个这个深度递归树路径 - `expr()`和`printExpr()`在代码中：

```go
func (p *parser) expr() Expr      { return p.getLine() }
func (p *parser) printExpr() Expr { return p._assign(p.printCond) }

```

特别解析内置函数调用，以便在解析时，检查参数的数量（在某些情况下是类型）。例如，解析`match(str, regex)`：

```go
case F_MATCH:
    p.next()
    p.expect(LPAREN)
    str := p.expr()
    p.commaNewlines()
    regex := p.regexStr(p.expr)
    p.expect(RPAREN)
    return &CallExpr{F_MATCH, []Expr{str, regex}}

```

许多解析函数提出，无效语法或意外标记的错误。不在每一步都检查这些错误，使生活更容易的是`panic`后，在顶层的特定的`ParseError`类型，才`recover`。这避免了深度递归代码中的大量重复错误处理。以下是顶级`ParseProgram`函数：

```go
func ParseProgram(src []byte, config *ParserConfig) (
        prog *Program, err error) {
    defer func() {
        if r := recover(); r != nil {
            // 转换为ParseError或重新发生恐慌
            err = r.(*ParseError)
        }
    }()
    // ...
}

```

### 分解器

该[分解器](https://github.com/benhoyt/goawk/blob/master/parser/resolve.go)实际上是解析器包的一部分。它对数组和标量进行基本类型检查，并为所有变量引用分配整数索引（以避免执行时，较慢的映射查找）。

我认为我完成分解器的方式是非传统的：解析器不是对 AST 进行完整传递，而仅记录分解器找出类型（函数调用和变量引用列表）所需的内容。这可能比走遍整棵树更快，但它也可能使代码不那么简单。

事实上，分解器是我写了一段时间的代码之一。这是我 对 GoAWK 不太满意的的一部分。它有效，但它很混乱，我仍然不确定我是否覆盖了所有边缘情况。

复杂性来自这样一个事实，即在调用函数时，您不知道调用点的参数是，顺序参数还是数组。您必须仔细阅读被调用函数中的类型（并且可能在它调用的函数中）以确定它。考虑这个 AWK 程序：

``` go
function g(b, y) { return f(b, y) }
function f(a, x) { return a[x] }
BEGIN { c[1]=2; print f(c, 1); print g(c, 1) }

```

该程序只打印`2`,两次。但是当我们在`g`里面调用`f`时，我们还不知道参数的类型。这是解析器工作的一部分，以迭代的方式解决这个问题。（参见[`resolveVars`](https://github.com/benhoyt/goawk/blob/a75cecd04d8aa8829c04b97bf370c8afaf53a68e/parser/resolve.go#L223)，在`resolve.go`）。

找出未知参数类型后，解析器将整数索引分配给所有变量引用，全局和局部。

### 解释器

解释器是一个简单的围绕虚拟树解释器。解释器实现，`语句执行和表达式求值，输入/输出，函数调用和基本值类型`。

- *语句执行*开始在[interp.go](https://github.com/benhoyt/goawk/blob/master/interp/interp.go)的`ExecProgram`，这需要一个解析了的`Program`，建立起解释器，然后执行`BEGIN`块，模式和动作，和`END`块。执行操作包括评估模式表达式，并确定它们是否与当前行匹配。其中包括“范围模式” `NR==4, NR==10`，它匹配开始和停止模式之间的行。

语句由`execute`方法执行，该方法采用任何类型的一个`Stmt`，在其`switch`上执行大型类型'认亲'活动，以确定它是什么类型的语句，并执行该语句的行为。

- *表达式求值*以相同的方式工作，除了它发生在`eval`方法中，它接受`Expr`，并 switch 到表达式类型。

大多数二进制表达式（除短路的`&&`和`||`）都通过`evalBinary`进行求值，其中包含对运算符标记的进一步 switch，如下所示：

```go
func (p *interp) evalBinary(op Token, l, r value) (value, error) {
    switch op {
    case ADD:
        return num(l.num() + r.num()), nil
    case SUB:
        return num(l.num() - r.num()), nil
    case EQUALS:
        if l.isTrueStr() || r.isTrueStr() {
            return boolean(p.toString(l) == p.toString(r)), nil
        } else {
            return boolean(l.n == r.n), nil
        }
    // ...
}

```

在`EQUALS`这种情况下，您可以看到 AWK 的“字符串类型”特性：如果任一操作数绝对是“真正的字符串”（来自用户输入的非数字字符串），请进行字符串比较，否则进行数字比较。这意味着像`$3 == "foo"`是字符串比较，但`$3 == 3.14`做一个数字比较，这是你所期望的。

AWK 的关联数组很好地映射到了 Go 的`map[string]value`类型，因此可以轻松实现这些。说到这个，Go 的垃圾收集器意味着我们不必担心编写自己的 GC。

- *输入和输出*在[io.go 中](https://github.com/benhoyt/goawk/blob/master/interp/io.go)处理。所有 I/O 都经过缓冲以提高效率，我们使用 Go 的`bufio.Scanner`来读取输入记录和`bufio.Writer`缓冲输出。

输入记录通常是行（`Scanner`默认行为），但记录分隔符`RS`也可以设置为要拆分的另一个字符，或者设置为空字符串，这意味着可在两个连续的换行符（空行）上，进行拆分以处理多行记录。这些方法仍使用`bufio.Scanner`，但使用自定义拆分函数，例如：

```go
// 拆分函数，用于拆分给定分隔符字节上的记录
type byteSplitter struct {
    sep byte
}

func (s byteSplitter) scan(data []byte, atEOF bool)
        (advance int, token []byte, err error) {
    if atEOF && len(data) == 0 {
        return 0, nil, nil
    }
    if i := bytes.IndexByte(data, s.sep); i >= 0 {
        // 我们有一个完整的sep终止符记录
        return i + 1, data[0:i], nil
    }
    // 如果在EOF，我们有最终的，未终止的记录; 把它返还
    if atEOF {
        return len(data), data, nil
    }
    // 请求更多数据
    return 0, nil, nil
}

```

`print`或`printf`的输出可以重定向到文件，附加到文件或通过管道输出到命令：这些在`getOutputStream`处理。输入可以来自 stdin，文件，也可以来自命令。

- *调用的函数*在[functions.go](https://github.com/benhoyt/goawk/blob/master/interp/functions.go)中实现，包括内置函数和用户定义函数。

该`callBuiltin`方法再次使用一个大的 switch 语句，来确定我们正在调用的 AWK 函数，例如`split()`或`sqrt()`。内置`split`函数需要特殊处理，因为它需要一个未计算的数组参数。类似地，`sub`，`gsub`实际上采用分配到的“左值”参数。对于其余的函数，我们首先评估参数并执行操作。

大多数函数都是使用 Go 标准库的一部分实现的。例如，所有数学函数都`sqrt()`使用标准`math`包，`split()`用到了`strings`和`regexp`函数。GoAWK 重新使用 Go 的正则表达式，因此模糊的正则表达式语法可能与“one true awk”的行为不同。

说到正则表达式，我使用简单的有界缓存来缓存正则表达式的编译，这足以加速几乎所有的 AWK 脚本：

```go
// 编译正则表达式字符串（或从正则表达式缓存中获取）
func (p *interp) compileRegex(regex string) (*regexp.Regexp, error) {
    if re, ok := p.regexCache[regex]; ok {
        return re, nil
    }
    re, err := regexp.Compile(regex)
    if err != nil {
        return nil, newError("invalid regex %q: %s", regex, err)
    }
    // 哎呀，非LRU 缓存：只缓存前N个正则表达式
    if len(p.regexCache) < maxCachedRegexes {
        p.regexCache[regex] = re
    }
    return re, nil
}

```

我也对 AWK 的`printf`声明作弊了，将 AWK 格式的字符串和类型转换为 Go 类型，这样我就可以重用 Go 的`fmt.Sprintf`函数了。同样，缓存此转换的格式字符串。

用户定义的调用使用`callUser`，它评估函数的参数，并将它们推送到本地堆栈。这比你想象的要复杂得多，原因有二：

- 首先，你可以将数组作为参数传递（通过引用），
- 其次，你可以调用一个，输入参数少于规定参数的函数。

它还检查调用深度（当前最大值为 1000），以避免在无限递归的panic。

- *基本值类型*实现在[value.go](https://github.com/benhoyt/goawk/blob/master/interp/value.go)。GoAWK 值是字符串或数字（或“数字字符串”），并使用`value`，这个值传递的结构，其定义如下：

```go
type value struct {
    typ      valueType // 值类型（nil，str或num）
    isNumStr bool      // 如果str值是“数字字符串”，则为True
    s        string    // 字符串值（typeStr）
    n        float64   // 数值（typeNum和数字字符串）
}

```

本来我做出了 GoAWK 值类型的`interface{}`，用它来持有`string`和`float64`。但你无法分辨常规字符串和数字字符串之间的区别，所以才决定使用结构。我的预感是，通过，值传递一个小的 4字 结构，比指针传递更好，所以这就是我所做的（虽然我没有验证）。

要检测“数字字符串”（请参阅 ​​`numStr`），我们只简单修剪空格，并使用 Go 的`strconv.ParseFloat`函数。但是，当字符串值使用`value.num()`显式转换为数字时，我不得不自力更生，'卷'自己的解析函数，因为这些转换允许`"1.5foo"`这样的事情，而`ParseFloat`不能。

### 主程序

主程序,在[goawk.go](https://github.com/benhoyt/goawk/blob/master/goawk.go)中，卷好上面的所有内容，放到命令行实用程序`goawk`中。同样，这里没什么好看的 - 它甚至使用标准的 Go `flag`包来解析命令行参数。

该`goawk`实用程序具有一个小辅助函数`showSourceLine`，它会显示语法错误的错误行和位置。例如：

```bash
$ goawk 'BEGIN { print sqrt(2; }'
--------------------------------------------
BEGIN { print sqrt(2; }
                    ^
--------------------------------------------
parse error at 1:21: expected ) instead of ;

```

没有什么特别的`goawk`：它只是调用`parser`和`interp`包。GoAWK 有一个非常简单的 Go API，所以如果你想从你自己的 Go 程序中调用它，请查看[GoDoc API 文档](https://godoc.org/github.com/benhoyt/goawk)。

## 我是如何测试它的

### Lexer 测试

[词法分析器测试](https://github.com/benhoyt/goawk/blob/master/lexer/lexer_test.go)使用了[表格驱动的测试](https://dave.cheney.net/2013/06/09/writing-table-driven-tests-in-go)，比较源输入和词法分析器字符串化版本的输出。这包括检查标记位置（行：列）以及标记的字符串值（用于`NAME`，`NUMBER`，`STRING`，和`REGEX`标记）：

```go
func TestLexer(t *testing.T) {
    tests := []struct {
        input  string
        output string
    }{
        // 名称和关键字
        {"x", `1:1 name "x"`},
        {"x y0", `1:1 name "x", 1:3 name "y0"`},
        {"x 0y", `1:1 name "x", 1:3 number "0", 1:4 name "y"`},
        {"sub SUB", `1:1 sub "", 1:5 name "SUB"`},

        // 字符串标记
        {`"foo"`, `1:1 string "foo"`},
        {`"a\t\r\n\z\'\"b"`, `1:1 string "a\t\r\nz'\"b"`},
        // ...
    }
    // ...
}

```

### 解析测试

解析器实际上没有明确的单元测试，除了[`TestParseAndString`](https://github.com/benhoyt/goawk/blob/a75cecd04d8aa8829c04b97bf370c8afaf53a68e/parser/parser_test.go#L17)，它测试一个包含所有语法结构的大程序 - 测试只是它解析，并可以通过漂亮的打印再次序列化。我的目的是在解释器测试中，测试解析逻辑。

### 解释器测试

该[解释器单元测试](https://github.com/benhoyt/goawk/blob/master/interp/interp_test.go)是表格驱动的测试，其具有漫长的列表。它们比词法分析器测试稍微复杂一点 - 它们采用 AWK 源，预期输入和预期输出，以及预期的错误字符串和 AWK 错误字符串（如果测试是应为导致错误的）。

您可以选择通过指定命令行`go test ./interp -awk=gawk`，运行针对某外部 AWK 解释器的解释器测试。我要确保测试是能针对`awk`和`gawk`这两种情况的 - 比如说，错误讯息，两者是完全不同的，我已经考虑到这一点，尝试只针对错误消息的一个子字符串，。

有时`awk`和`gawk`都有不同的已知行为，或者不会捕获与 GoAWK 相同的错误，因此在一些测试中我必须按名称排除解释器 - 也就是在源字符串中使用特殊的`!awk`（“非 awk”）注释来完成的。。

以下是解释器单元测试的样子：

```go
func TestInterp(t *testing.T) {
    tests := []struct {
        src    string
        in     string
        out    string
        err    string // GoAWK的错误必须等于此
        awkErr string // 来自 awk/gawk的错误 必须包含此内容
    }{
        {`$0`, "foo\n\nbar", "foo\nbar\n", "", ""},
        {`{ print $0 }`, "foo\n\nbar", "foo\n\nbar\n", "", ""},
        {`$1=="foo"`, "foo\n\nbar", "foo\n", "", ""},
        {`$1==42`, "foo\n42\nbar", "42\n", "", ""},
        {`$1=="42"`, "foo\n42\nbar", "42\n", "", ""},
        {`BEGIN { printf "%d" }`, "", "",
            "format error: got 0 args, expected 1", "not enough arg"},
        // ...
    }
    // ...
}

```

### 命令行测试

我也想测试`goawk`命令行处理，所以在[`goawk_test.go`](https://github.com/benhoyt/goawk/blob/master/goawk_test.go)有，另一套测试的东西，表格驱动测试像`-f`，`-v`，`ARGV`，和相关的命令行其他的事情：

```go
func TestCommandLine(t *testing.T) {
    tests := []struct {
        args   []string
        stdin  string
        output string
    }{
        {[]string{"-f", "-"}, `BEGIN { print "b" }`, "b\n"},
        {[]string{"-f", "-", "-f", "-"}, `BEGIN { print "b" }`, "b\n"},
        {[]string{`BEGIN { print "a" }`}, "", "a\n"},
        {[]string{`$0`}, "one\n\nthree", "one\nthree\n"},
        {[]string{`$0`, "-"}, "one\n\nthree", "one\nthree\n"},
        {[]string{`$0`, "-", "-"}, "one\n\nthree", "one\nthree\n"},
        // ...
    }
    // ...
}

```

这些是针对`goawk`二进制，以及外部 AWK 程序（默认为`gawk`）进行测试的。

### AWK 测试套件

我还针对 Brian Kernighan 的“one true awk”测试套件，测试了 GoAWK 。它们是`testdata`目录中的`p.*`和`t.*`文件。[`goawk_test.go`](https://github.com/benhoyt/goawk/blob/master/goawk_test.go)的`TestAWK`函数会驱动这些测试。将测试程序的输出与外部 AWK 程序的输出（再次默认为`gawk`）进行比较，以确保其匹配。

一些测试程序，例如那些调用`rand()`的不会真正与 AWK 区别开来的测试，所以我将其排除在外。对于其他程序，例如循环遍历数组的测试（迭代顺序未定义），我会在 不同 之前，对输出中的行进行排序。

### 模糊测试

我使用的最后一种测试是“模糊测试”。这是一种向解释器发送随机输入直到它中断的方法。我通过这种方式捕获了几次崩溃（panic），甚至 Go 编译器中的[一个错误](https://github.com/benhoyt/goawk/commit/89cf2a2c3958f2e602d553a9abc418aa0031a0f0)导致了对 segfault 的越界切片访问（虽然我发现在 Go 1.11 中已经修复了）。

为了驱动模糊测试，我简单使用了[go-fuzz](https://github.com/dvyukov/go-fuzz)库的`Fuzz`函数：

```go
func Fuzz(data []byte) int {
    input := bytes.NewReader([]byte("foo bar\nbaz buz\n"))
    err := interp.Exec(string(data), " ", input, &bytes.Buffer{})
    if err != nil {
        return 0
    }
    return 1
}

```

模糊测试发现了一些我没在其他测试方法抓到的漏洞和边缘情况。大多数情况下，这些都是你不会用实际代码编写的东西，但是让一个不知疲倦的机器人帮助你增加一层健壮性是很好的。在 GoAWK 中，模糊测试发现至少存在以下问题：

- [c59731f](https://github.com/benhoyt/goawk/commit/c59731f5543bf9b48cf92a981b66696a5ab0ceae)：使用数组上下文中的内置（scalar）修复恐慌
- [59c931f](https://github.com/benhoyt/goawk/commit/59c931fa42e6bd436c64391fd743f6e259beabef)：修复尝试从输出流中，读取时，的崩溃（反之亦然）
- [b09e51f](https://github.com/benhoyt/goawk/commit/b09e51f64689e12c466e951ed1b8add17742be9f)：禁止将 NF 和 $n 设置为 1,000,000（fuzzer 发现此信息）
- [6d99151](https://github.com/benhoyt/goawk/commit/6d99151bc918fc602bc0274221b8ca93f80c7095)：修改，转换为浮点数时的'值超出范围'恐慌（go-fuzz 发现这个）

有关如何运行 fuzzer 的详细信息，请参见[fuzz/README.txt](https://github.com/benhoyt/goawk/blob/master/fuzz/README.txt)。

## 提高性能

[跳过 叙述，直接查看性能表现！](#%E6%80%A7%E8%83%BD%E8%A1%A8%E7%8E%B0)

性能问题往往是由以下几个方面的瓶颈造成的，从多到少影响排序（感谢 Alan Donovan 对此的简洁思考方式）：

1.  输入/输出
2.  内存分配
3.  CPU 周期

如果您要做很多很多的 I/O 或系统调用，`goAWK`的血量会`-99999`。

下一个是内存分配：它们是昂贵的，重要事情之一，为此 Go 提供了内存分配很大的控制权（例如，`make()`的“cap”容量参数）。

最后是 CPU 周期 - 这通常是影响最小的，尽管有时人们在谈论“性能”时，会想到这一点。

在 GoAWK 中，我在所有三个方面都做了一些优化。对于典型的 AWK 程序来说，其最大优性能点都与 I/O 有关 - 正如预期的那样，因为 AWK 程序通常会读取输入，处理它，并写入输出。但，仍可在内存分配和 CPU 周期，有一些重要的收获。

### 我是如何分析的

瓶颈往往不 🐶 直观，因此*测量*是关键。让我们来看看你如何衡量 Go 代码中，正在发生的事情。

使用标准库[`runtime/pprof`包](https://golang.org/pkg/runtime/pprof/)来检索，分析代码相当容易。（您可以[在此处](https://blog.golang.org/profiling-go-programs)阅读有关[分析 Go 程序的](https://blog.golang.org/profiling-go-programs)更多信息。）

首先，我添加了一个`-cpuprofile`命令行标志，若设置了这个，会启用 CPU 分析。下面是代码：

```go
if *cpuprofile != "" {
    f, err := os.Create(*cpuprofile)
    if err != nil {
        errorExit("could not create CPU profile: %v", err)
    }
    if err := pprof.StartCPUProfile(f); err != nil {
        errorExit("could not start CPU profile: %v", err)
    }
}
// ...运行 interp.Exec ...
if *cpuprofile != "" {
    pprof.StopCPUProfile()
}

```

然后，您可以运行，要分析的 AWK 程序：

```bash
$ ./goawk -cpuprofile=prof 'BEGIN { for (i=0; i<100000000; i++) s++ }'

```

最后使用该`pprof`工具查看输出（该`-http`标志会在 Web 浏览器中，激活一个 tab 选项卡，并提供几种查看数据的好方法）：

```bash
$ go tool pprof -http=:4001 prof

```

这是“top”视图的屏幕截图，我发现它最有用：

![pprof的“顶级”视图](/images/goawk-pprof.png)

从这个截图中，你可以立即看到几件事：

- 通过**map**的变量访问让我们慢慢的（`getVar`，`mapassign`，`mapaccess`）
- 基于`panic`的错误处理相当缓慢（所有`defer`行）

为了解决了这两个问题，我在下面叙述了。我在不同类型的 AWK 程序上，运行了很多次分析器，并从 I/O 开始发现了许多问题。

### 性能改进

果不期然，GoAWK 有 I/O 问题 - 我没有缓冲对 stdout 的写入。因此微基准测试看起来没问题，但真正的 AWK 程序运行速度比之慢很多倍。因此，**加速输出**是我做的第一个优化（后来我意识到我也忘了为重定向输出做这一点）：

- [60745c3](https://github.com/benhoyt/goawk/commit/60745c3503ba3d99297816f5c7b5364a08ec47ab)：缓冲标准输出（和 stderr），加速 10 倍
- [6ba004f](https://github.com/benhoyt/goawk/commit/6ba004f5fbf9b84bc6196d50c2a0dd496ed1771b)：缓冲重定向输出以提高性能

接下来我改为使用**switch/case 进行二进制操作，**而不是在`map`中查找函数,并调用它。但这并没有明显变快，特别是`switch`在 Go 跳过`case`列表，并且不使用“computed gotos”时。但我想在很多情况下，调用函数所涉及的常数因素超过了这个因素：

- [ad8ff0e](https://github.com/benhoyt/goawk/commit/ad8ff0e5f6cc89fdd480099614187ee23b20a8c9)：通过从 map 移动到 switch/case 来加速二进制操作

```bash
benchmark                           old ns/op     new ns/op     delta
BenchmarkComparisons-8              975           739           -24.21%
BenchmarkBinaryOperators-8          1294          993           -23.26%
BenchmarkConcatSmall-8              1312          1120          -14.63%
BenchmarkArrayOperations-8          2542          2350          -7.55%
BenchmarkRecursiveFunc-8            64319         60507         -5.93%
BenchmarkBuiltinSub-8               16213         15305         -5.60%
BenchmarkForInLoop-8                3886          4092          +5.30%
...

```

有趣的是，我的一些改进，将了*完全不相关*路径的代码变慢。我还是不知道为什么。它是测量的’噪音‘吗？我不这么认为，因为它似乎非常一致。我的猜测是，机器代码已被重新排列，并在某种程度上导致代码的其他部分中的，缓存未命中或分支预测更改。

下一个重大变化是，**在解析时**将**变量名称分解为索引。**以前，我是使用`map[string]value`在'运行时'，执行所有变量查找，但是 AWK 中的变量引用可以在解析时,分解，然后解释器可以在一个`[]value`中找到它们。它还避免了内存分配，当因，随着 map 的增长分配了变量等情况：

- [e0d7287](https://github.com/benhoyt/goawk/commit/e0d7287ac1580bd0f144c763b222b9db8a858c54)：大的性能改进：在解析时分解变量

```bash
benchmark                           old ns/op     new ns/op     delta
BenchmarkFuncCall-8                 13710         5313          -61.25%
BenchmarkRecursiveFunc-8            60507         30719         -49.23%
BenchmarkForInLoop-8                4092          2467          -39.71%
BenchmarkLocalVars-8                2959          1827          -38.26%
BenchmarkForLoop-8                  15706         10349         -34.11%
BenchmarkIncrDecr-8                 2441          1647          -32.53%
BenchmarkGlobalVars-8               2628          1812          -31.05%
...

```

最初我用`interp.eval()`，只是为了`value`和运行时错误的特殊错误而恐慌返回，但这是一个显着的减速，所以我切换到使用更详细，但更类 Go **错误返回值。**使用[建议的`check`关键字](https://go.googlesource.com/proposal/+/master/design/go2draft-error-handling-overview.md)会更好，但是哦！这一变化在许多基准测试中提高了 2-3 倍：

- [aa6aa75](https://github.com/benhoyt/goawk/commit/aa6aa75368afeb40897b180c5a36501012e94907)：通过消除 panic/recover 来提高`interp`性能

```bash
benchmark                           old ns/op     new ns/op     delta
BenchmarkIfStatement-8              885           292           -67.01%
BenchmarkGlobalVars-8               1812          672           -62.91%
BenchmarkLocalVars-8                1827          682           -62.67%
BenchmarkIncrDecr-8                 1647          714           -56.65%
BenchmarkCondExpr-8                 604           280           -53.64%
BenchmarkForLoop-8                  10349         6007          -41.96%
BenchmarkBuiltinLength-8            2775          1616          -41.77%
...

```

下一个改进是对`evalIndex`一些小但有效的调整，它评估一片数组表达式，以产生一个关键字符串。在 AWK 中，数组可以被多个下标索引`a[1, 2]`，实际上只是将它们一起混合成`"1{SUBSEP}2"`（下标分隔符默认为`\x1c`）。

但大多数时候你只有一个下标，所以我**优化了常见的情况。**对于多下标的情况，我做了一个初始分配 - 希望在堆栈上 - 使用`make([]string, 0, 3)`以避免最多 3 个下标的堆分配。

- [af99309](https://github.com/benhoyt/goawk/commit/af993094e3e8aca2b7ab709ffcda437996c906fe)：加速数组操作

```bash
name                    old time/op  new time/op  delta
ArrayOperations-8       1.80µs ± 1%  1.13µs ± 1%  -37.52%

```

另一个例子是**减少分配**，通过确保具有多达七个参数的调用不需要堆分配，来加速函数调用。内置调用增加了 2 倍。

- [e45e209](https://github.com/benhoyt/goawk/commit/e45e2090d44c08b340555f483e1f5bf42160e199)：通过减少分配，来加速对内置函数的调用

```bash
name                    old time/op  new time/op  delta
BuiltinSubstr-8         3.11µs ± 0%  1.56µs ± 5%  -49.77%
BuiltinIndex-8          3.00µs ± 2%  1.56µs ± 3%  -48.17%
BuiltinLength-8         1.62µs ± 0%  0.93µs ± 6%  -42.92%
ArrayOperations-8       1.80µs ± 1%  1.13µs ± 1%  -37.12%
BuiltinMatch-8          3.77µs ± 1%  3.04µs ± 0%  -19.39%
SimpleBuiltins-8        5.50µs ± 1%  4.68µs ± 0%  -14.83%
BuiltinSprintf-8        14.3µs ± 4%  12.6µs ± 0%  -12.50%
...

```

下一个优化是**避免使用重量级工具**（`text/scanner`），仅为了简单地将字符串转换为数字。我正在使用`Scanner`，因为它允许你解析，像`1.23foo`（当字符串不是来自用户输入时， AWK 允许），且因`strconv.ParseFloat`并不处理它。

我只是简单地编写了自己的 lexing 函数，来查找字符串中实际浮点数的结尾，然后在其上调用`ParseFloat`。这样可以将显式字符串到数字的转换速度提高 10 倍以上！

- [12b8520](https://github.com/benhoyt/goawk/commit/12b8520948e78ef19e3ed99bcffe25b3e893e447)：优化，通过不使用 text/scanner 程序 - 字符串到数字转换

```bash
$ cat test.awk
BEGIN {
    for (i=0; i<1000000; i++) {
        "1.5e1"+"1"; "1.5e1"+"1"; "1.5e1"+"1"; "1.5e1"+"1"; "1.5e1"+"1";
        "1.5e1"+"1"; "1.5e1"+"1"; "1.5e1"+"1"; "1.5e1"+"1"; "1.5e1"+"1";
    }
}
$ time ./goawk_before test.awk
real    0m10.692s
$ time ./goawk_after test.awk
real    0m0.983s

```

我做的另一优化是，通过在 lexing 期间**避免 UTF-8 解码**，可加速词法分析器。没有充分的理由不会将所有内容保留为字节，并且它使得词法分析器，在这些提交后，速度提升了 2-3 倍：

- [0fa32f9](https://github.com/benhoyt/goawk/commit/0fa32f929b27bc55bcb8d68507853f1083d8ae02)：通过避免 UTF-8 解码，加速词法分析器
- [43af0cb](https://github.com/benhoyt/goawk/commit/43af0cbd2f7b19273b58a75bf0fab20f91a755bf)：通过从 rune 改为 byte 类型, 加速词法分析器
- [c5a32eb](https://github.com/benhoyt/goawk/commit/c5a32eb08f817b4622ce11e7ad858ed131e3cad7)：通过减少分配来, 加速词法分析器

### 性能表现

那么 GoAWK 与其他 AWK 实现相比如何呢？挺好的！在下图中：

- `goawk`是指当前版本的 GoAWK（[commit 109e8a9](https://github.com/benhoyt/goawk/commit/109e8a9d645cb454e13582ab34f0f9d6d3fbdcfd)）
- `orig`是指第一个“正常工作”的 GoAWK 版本，没有优化（[提交 8ab5446](https://github.com/benhoyt/goawk/commit/8ab54463f01a7d7d018be26a2f618cbd3c82538d)）
- `awk` 是“one true awk”版本 20121220
- `gawk` 是 GNU Awk 版本 4.2.1
- `mawk` 是 mawk 版本 1.3.4（20171017）

下面的数字表示在 3 次运行中,运行给定测试所需的平均时间，标准是`goawk`运行时间 - **越低越好** 。正如你所看到的，大多数情况 GoAWK 比`awk`要快得多，而相比`gawk`也不算太差！

| 测试   | goawk     | orig      | awk       | gawk      | mawk      |
| ------ | --------- | --------- | --------- | --------- | --------- |
| tt.01  | 1.000     | 1.123     | 5.818     | 0.455     | 0.465     |
| tt.02  | 1.000     | 1.107     | 5.015     | 1.331     | 0.963     |
| tt.02a | 1.000     | 1.149     | 4.115     | 1.356     | 0.892     |
| tt.03  | 1.000     | 1.183     | 5.574     | 0.467     | 0.738     |
| tt.03a | 1.000     | 2.013     | 5.965     | 0.362     | 0.794     |
| tt.04  | 1.000     | 1.386     | 1.222     | 0.800     | 0.434     |
| tt.05  | 1.000     | 1.450     | 1.425     | 0.545     | 0.430     |
| tt.06  | 1.000     | 1.360     | 5.175     | 0.628     | 0.756     |
| tt.07  | 1.000     | 1.177     | 6.160     | 1.140     | 0.961     |
| tt.big | 1.000     | 1.540     | 1.314     | 0.757     | 0.447     |
| tt.x1  | 1.000     | 2.591     | 0.866     | 0.575     | 0.427     |
| tt.x2  | 1.000     | 2.348     | 0.511     | 0.411     | 0.296     |
| **总** | **1.000** | **1.486** | **3.074** | **0.708** | **0.521** |

## 从哪里来？

我很想知道您是否使用[GoAWK](https://github.com/benhoyt/goawk)，请向我发送错误报告和代码反馈。如果你不需要 GoAWK，至少你已经了解了常用的 AWK，以及它是多么有用的工具。

谢谢阅读！
