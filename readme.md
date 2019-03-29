# benhoyt/goawk [![explain]][source] [![translate-svg]][translate-list]

<!-- [![size-img]][size] -->

[explain]: http://llever.com/explain.svg
[source]: https://github.com/chinanf-boy/Source-Explain
[translate-svg]: http://llever.com/translate.svg
[translate-list]: https://github.com/chinanf-boy/chinese-translate-list
[size-img]: https://packagephobia.now.sh/badge?p=Name
[size]: https://packagephobia.now.sh/result?p=Name

「 用 Go 编写的 AWK 解释器 」

[中文](./readme.md) | [english](https://github.com/benhoyt/goawk)

---

## 校对 ✅

<!-- doc-templite START generated -->
<!-- repo = 'benhoyt/goawk' -->
<!-- commit = 'a75cecd04d8aa8829c04b97bf370c8afaf53a68e' -->
<!-- time = '2018-11-17' -->
翻译的原文 | 与日期 | 最新更新 | 更多
---|---|---|---
[commit] | ⏰ 2018-11-17 | ![last] | [中文翻译][translate-list]

[last]: https://img.shields.io/github/last-commit/benhoyt/goawk.svg
[commit]: https://github.com/benhoyt/goawk/tree/a75cecd04d8aa8829c04b97bf370c8afaf53a68e

<!-- doc-templite END generated -->

| **详细了解 GoAWK 如何在工作和执行.**(网页) | github                         |
| ---------------------------------------- | ------------------------------ |
| [原文`en`][writing] === [中文][llever]                                 | [./goawk.zh.md](./goawk.zh.md) |

> [作者同意][issue]

[writing]: https://benhoyt.com/writings/goawk/
[issue]: https://github.com/benhoyt/goawk/issues/12
[llever]: http://llever.com/2018/11/19/译-goawk一个用go编写的awk解释器/

### 贡献

欢迎 👏 勘误/校对/更新贡献 😊 [具体贡献请看](https://github.com/chinanf-boy/chinese-translate-list#贡献)

## 生活

[If help, **buy** me coffee —— 营养跟不上了，给我来瓶营养快线吧! 💰](https://github.com/chinanf-boy/live-need-money)

---

# GoAWK:用 Go 编写的 AWK 解释器

[![GoDoc](https://godoc.org/github.com/benhoyt/goawk?status.png)](https://godoc.org/github.com/benhoyt/goawk)
[![TravisCI Build](https://travis-ci.org/benhoyt/goawk.svg)](https://travis-ci.org/benhoyt/goawk)
[![AppVeyor Build](https://ci.appveyor.com/api/projects/status/github/benhoyt/goawk?branch=master&svg=true)](https://ci.appveyor.com/project/benhoyt/goawk)

AWK 是一种引人入胜的文本处理语言,在阅读了令人愉快的[_AWK 编程语言_](https://ia802309.us.archive.org/25/items/pdfy-MgN0H1joIoDVoIC7/The_AWK_Programming_Language.pdf)简洁之后,我很高兴能在 Go 中为它写一个解释器。所以,我宣布这里是,功能完整的测试套件"一个真正的 AWK"。

[**详细了解 GoAWK 如何工作和执行的.**][writing]

## 基本用法

要使用命令行版本,只需使用`go get`安装它,然后使用`goawk`(假设`$GOPATH/bin`在你的`PATH`):

```bash
$ go get github.com/benhoyt/goawk
$ goawk 'BEGIN { print "foo", 42 }'
foo 42
$ echo 1 2 3 | goawk '{ print $1 + $3 }'
4
```

要在 Go 程序中使用它,您可以调用`interp.Exec()`，其直接满足简单需求:

```go
input := bytes.NewReader([]byte("foo bar\n\nbaz buz"))
err := interp.Exec("$0 { print $1 }", " ", input, nil)
if err != nil {
    fmt.Println(err)
    return
}
// Output:
// foo
// baz
```

或者你可以使用`parser`模块，然后使用`interp.ExecProgram()`控制执行,设置变量等:

```go
src := "{ print NR, tolower($0) }"
input := "A\naB\nAbC"

prog, err := parser.ParseProgram([]byte(src), nil)
if err != nil {
    fmt.Println(err)
    return
}
config := &interp.Config{
    Stdin: bytes.NewReader([]byte(input)),
    Vars:  []string{"OFS", ":"},
}
_, err = interp.ExecProgram(prog, config)
if err != nil {
    fmt.Println(err)
    return
}
// Output:
// 1:a
// 2:ab
// 3:abc
```

请阅读[GoDoc 文档](https://godoc.org/github.com/benhoyt/goawk)更多细节.

## 与 AWK 的差异

目的是让 GoAWK 遵守`awk`的行为和对[POSIX AWK 规格](http://pubs.opengroup.org/onlinepubs/9699919799/utilities/awk.html),但本节介绍了一些不同的地方.

GoAWK 的其他功能优于 AWK:

- 它可以嵌入你的 Go 程序中!
- I/O 绑定的 AWK 脚本(大多数脚本)明显快于`awk`,与`gawk`和`mawk`相比.
- 解析器同时支持`'单引号 strings'`，除了`"double-quoted strings"`,主要是为了让 Windows 单行更容易(Windows`cmd.exe`shell 使用`"`作为引用字符).

AWK 超过 GoAWK 的:

- CPU 绑定的 AWK 脚本比`awk`慢,和也比`gawk`和`mawk`大约慢两倍.
- AWK 由 Brian Kernighan 编写.

## 稳定性

这个项目有一套很好的测试,我亲自使用它,但它肯定没有经过实战测试，或大量使用,所以请自担风险使用。我不打算大改 Go API.

## 执照

GoAWK 是根据开源许可的[MIT 许可证](https://github.com/benhoyt/goawk/blob/master/LICENSE.txt).

## 结束

玩得开心,请[联络我](https://benhoyt.com/)，如果你正在使用 GoAWK 或有任何反馈!
