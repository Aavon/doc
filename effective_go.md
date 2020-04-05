## Effective Go (翻译 & 学习)



https://golang.google.cn/doc/effective_go.html#sharing



#### Introduction

Go作为一门新的语言（已经不新了...）。虽然借鉴了现有语言的设计思想，但是Go 语言的一些不同寻常的特性使得Go语言编写的程序和他们拥有不同的特性。简单的将C++或者Java语言编写的程序翻译成Go可能不太会得到令人满意的结果。另一方面，从Go的角度来看待这个问题，竟会产生一个成功但是完全不一样的程序。换句话说，想要将Go写好，了解其特性和风格上非常重要的。同样重要的是了解Go语言中一些既定俗成的规则，例如命名，格式化，程序结构等等，以便其他Go程序员能轻松的理解我们的程序。



这篇文章对于如何写出清晰、纯正的Go语言代码给出了一些建议。同时附加了一些需要提前阅读的资料， [language specification](https://golang.google.cn/ref/spec), [Tour of Go](https://tour.golang.org/), 和 [How to Write Go Code](https://golang.google.cn/doc/code.html)。



#### Examples

Go 语言的src目录不仅仅是用作核心基础库，同时也提供了很多古纳于如何使用Go语言的例子。同时，许多包包含可独立运行的示例，你可以直接通过[golang.org](https://golang.org) 网站运行。如果你想要解决一个问题或者想要了解它们是怎么实现的，包中的文档、代码和示例将会给你答案。



#### Formatting

格式话问题一直是饱受争议的但是同时实际影响却不大。人们能够适应不同的格式化风格，但是如果大家都使用统一的风格，可以不必要去做这些，同时花更少的时间在这个问题上那将会是更好的。那么问题是如何在不用冗长的说明性风格指南下实现这种“乌托邦”式的目标。



在Go语言中，我们使用一种不同寻常的方法来让机器关心绝大多数的格式化问题。gofmt程序（go fmt, 工作在package上，而不是整个源代码目录）将读取Go语言程序并输出标准的缩进，对齐风格的代码，如果有必要也能格式化注释。如果你想了解一些新的格式化场景，运行gofmt就可以了。如果结果好像并不正确，重新调整一下你的程序（或者提交gofmt的bu g）,不要尝试去解决它。

*example....*

所有在Go标准库中的代码都是经过gofmt格式化的。

一些格式化的细节。非常简洁的：

**缩进**

gofmt默认使用tab进行缩进。如果你必须要使用space也是可以的。

**单行长度**

Go没有单行的长度限制。不用担心超出打孔卡（...），如果感觉是在太长了，换行后额外增加一个缩进即可。

**括号**

Go语言相较于C和Java使用跟少的括号：控制结构（if，for，switch）的表达式并不需要括号。同时，操运算符的优先级也更加的简短和清晰，比如：

```go
x<<8 + y<<16
```

空格就表示了其中的优先级关系，不同于其他语言。



#### 注释

Go提供C风格的 `/* */`的块注释和C++风格的`//`行注释。行注释更加常用，块注释更多的出现在包注释上，但是用在描述或者注释一大段代码也是很有效的。

`godoc`服务程序处理所有的Go源代码提取出关于package的文档。那些出现在声明之前的，没有被中断的注释会被单独提取出来作为说明的文本。这些注释的质量和风格决定了最终生成的godoc文档的质量。



每个package应该有一个注释，一个在package语句之前的块注释。对于多个文件的package，package注释只需要在其中任意一个文件。package注释应该提供package相关的整体介绍，它将会出现在godoc对应的页面上。

```go
/*
Package regexp implements a simple library for regular expressions.

The syntax of the regular expressions accepted is:

    regexp:
        concatenation { '|' concatenation }
    concatenation:
        { closure }
    closure:
        term [ '*' | '+' | '?' ]
    term:
        '^'
        '$'
        '.'
        character
        '[' [ '^' ] character-ranges ']'
        '(' regexp ')'
*/
package regexp
```

如果package很简单，那么注视也可以是很简洁的。

```go
// Package path implements utility routines for
// manipulating slash-separated filename paths.
```

注释不需要额外的格式，比如星号横幅。godoc生成的输出甚至可能不会以固定宽度的字体显示，因此不依赖于对齐间距，gofmt会负责这些问题。注释是不会被动态执行的富文本。

godoc甚至不会格式化注释，所以确保他们看上去很优雅：拼写，标点，段落结构，分段等等。

在一个package内，任何在声明之前的注释都被当作说明文档。任何暴露（大写）的命名都应该有文档注释。

文档注释最好是一个完整的句子，可以有多样的展示形式。第一句的开头应该是一个以命名开头的一个总结的句子。

```go
// Compile parses a regular expression and returns, if successful,
// a Regexp that can be used to match against text.
func Compile(str string) (*Regexp, error) {
```

如果每个文档注释都以其描述的对象名称开头，则可以使用`go tool`的doc子命令并通过grep运行输出。 想象一下，你忘记了名称“ Compile”，但正在寻找正则表达式的解析函数，因此您运行了命令，

```sh
go doc -all regexp | grep -i parse
```

如果所有的文档注释都以，“This function...”开头，将对你记住对象名称没有帮助。但是由于每个文档注释都以对象名称开头，你可以看到这样的结果，它帮助你回想起你正在寻找的东西。

```sh
$ go doc -all regexp | grep -i parse
    Compile parses a regular expression and returns, if successful, a Regexp
    MustCompile is like Compile but panics if the expression cannot be parsed.
    parsed. It simplifies safe initialization of global variables holding
$
```

Go的语法支持按组声明。一个单独的文档说明可以用来介绍一组常量和变量声明。因为所有的声明都在一起，所以这样的文档注释一般是很笼统的。

```go
// Error codes returned by failures to parse an expression.
var (
    ErrInternal      = errors.New("regexp: internal error")
    ErrUnmatchedLpar = errors.New("regexp: unmatched '('")
    ErrUnmatchedRpar = errors.New("regexp: unmatched ')'")
    ...
)
```

分组也同样可以表明对象之间的关系，例如一组变量受mutex的保护。

```go
var (
    countLock   sync.Mutex
    inputCount  uint32
    outputCount uint32
    errorCount  uint32
)
```

















