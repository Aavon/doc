## Effective Go (翻译 & 学习)



https://golang.google.cn/doc/effective_go.html#sharing

#### Introduction

Go作为一门新的语言（已经不新了...）。虽然借鉴了现有语言的设计思想，但是Go 语言的一些不同寻常的特性使得Go语言编写的程序和他们拥有不同的特性。简单的将C++或者Java语言编写的程序翻译成Go可能不太会得到令人满意的结果。另一方面，从Go的角度来看待这个问题，竟会产生一个成功但是完全不一样的程序。换句话说，想要将Go写好，了解其特性和风格上非常重要的。同样重要的是了解Go语言中一些既定俗成的规则，例如命名，格式化，程序结构等等，以便其他Go程序员能轻松的理解我们的程序。



这篇文章对于如何写出清晰、纯正的Go语言代码给出了一些建议。同时附加了一些需要提前阅读的资料， [language specification](https://golang.google.cn/ref/spec), [Tour of Go](https://tour.golang.org/), 和 [How to Write Go Code](https://golang.google.cn/doc/code.html)。



#### Examples

Go 语言的src目录不仅仅是用作核心基础库，同时也提供了很多关于如何使用Go语言的例子。同时，许多包包含可独立运行的示例，你可以直接通过[golang.org](https://golang.org) 网站运行。如果你想要解决一个问题或者想要了解它们是怎么实现的，包中的文档、代码和示例将会给你答案。



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



#### Names

和其他语言一样，命名在Go中也是很重要的。甚至有语法上的影响：首字母的大小写决定了在package之外的可见性。因此，花点时间来讨论一下Go语言的命名规范是很值得的。

##### Package names

当一个package被import，那么package的名称将被作为访问的入口。

方便起见，package的名字应该是小写，单个名词，没有必要使用下划线和大小写。也不用担心会有冲突，package的名字只是默认的import之后的名字，他们没有必要保证全局唯一。可以通过重命名的方式解决罕见的混淆问题。在任何情况下，混淆的问题都是很罕见的，因为倒入的文件名决定了具体使用的是哪个package。

因为倒import之后是通过package的名字来引用其内容的，所以暴露出来的命名应该使用这个规则来避免重复冗余。比如bufio中包含带有缓存的reader的实现，但是它的命名是Reader，而不是BufReader，应为使用者看到的是bufio.Reader。

同样的package中只包含了一种实现，这应该使用New()，而不是NewRing()。

另外，也应该避免使用很长的命名，这种情况下添加注释会更加有效，比如once.Do()，而不是once.DoOrWaitUntilDone。

##### Getters

Go没有提供自动的对getters和setters的支持。这是没什么问题的，但是将Get附加到getters命名的前面即不符合规范也没必要的事情，比如Owner()。对于一个setter函数，更好的是使用SetOwner()。这两种命名实际的可读性都是很好的。

```go
owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```

#### Interface names

按照规范，只有一个方法的interface命名一般是在method的名称之后加上er的后缀或者其他类似的名词。比如Reader，Writer，Formatter，CloseNotifier等等

这里有一些惯用的命名，`Read`, `Write`, `Close`, `Flush`, `String`等等，他们都有一些其特殊的含义。为了避免混淆，除非你的函数确实实现了相应的功能，都不应该使用这些命名，比如对于string转换的函数应该叫String，而不是ToString。

##### MixedCaps

最后，按照规范Go语言中使用 `MixedCaps` or `mixedCaps`，而不是使用下划线来表示多个单词的命名。



#### （*）Semicolons（分号）

和C类似，Go 的标准语法中使用分号来断句，但是和C不一样的是，这些分号并不出现在源代码中。而是语法分析器使用一个简单的规则在扫描的时候自动追加上去的，所以输入的文本内容不需要处理它们。

规则是，如果所在行的最后一个词是标识符（包括int，float等），常量，或者以下的词：

```
break continue fallthrough return ++ -- ) }
```

那么语法分析器总是会在它们之后添加一个分号。右大括号之前也可以省略分号，比如，

```go
go func() { for { dst <- <-src } }()
```

Go仅在循环语句中使用分号，同时也很有必要用在分割在同一行的多个语句，如果你那样写代码的话。

这种自动插入分号的方式导致一个结果是，在控制语句中(`if`, `for`, `switch`, or `select`) ，我们不能将大括号新起一行。如果你那样做了，分号将会被插入在大括号之前，导致不想要的影响。

```go
if i < f() {
    g()
}
```

而不是

```go
if i < f()  // wrong!
{           // wrong!
    g()
}
```



#### Control structures

Go中的控制结构和C是类似的，但是却有一些很重要的区别。

没有do和while循环语句，只有一个稍微普适的for；

switch也更加灵活，；

if和switch跟for一样接受一个可选的初始化语句；

break和continue语句接受一个可选的label，来决定break和continue的代码块；

新的结构type switch；

多路复用并发控制select；

语法结构上也有不同：没有括号，代码块内容总是通过大括号来界定；

##### If

一个简单的if结构：

```go
if x > 0 {
    return y
}
```

最好通过大括号来多行编写一个if结构，特别是body中包含return和break语句时；

因为if和switch语句支持初始化语句；他们常被用来初始化一个局部变量。

```go
if err := file.Chmod(0664); err != nil {
    log.Print(err)
    return err
}
```

在没有必要时else也可以被省略。通常是以break，continue，goto，或者return结尾。

```go
f, err := os.Open(name)
if err != nil {
    return err
}
codeUsing(f)
```

这是一个代码中需要排除一系列错误的场景。如果成功的代码在后面将会有很好的可读性，在error出现的地方处理它们。因为error更多的会导致return，这样的话，实际的功能代码就不需要else语句。

```go
f, err := os.Open(name)
if err != nil {
    return err
}
d, err := f.Stat()
if err != nil {
    f.Close()
    return err
}
codeUsing(f, d)
```

##### （*）Redeclaration and reassignment（重复声明和重新赋值）

```go
f, err := os.Open(name)
```

这个语句声明了两个变量，f和err。几行之后，调用`f.Stat`：

```go
d, err := f.Stat()
```

看上去声明了d和err。需要注意的是，尽管err出现在两条语句中，这样的重复是允许的：err被第一条语句声明，俄日在第二条语句中只是被重新赋值了。这就意味着在调用f.Stat时，使用的是之前的err变量，只是赋了新值。

在下面的情况下，变量v 仍然可能出现在`:=`左边，即使之前已经被声明过了：

- 和已存在的声明在相同的作用域（如果v已经在外层的作用域中被声明，那么当前声明会创建新的变量）
- `:=`同时包含多个变量，且至少有一个变量是需要创建的

这不同寻常的特性纯粹是为了实用性，使得使用同一个err变量变得容易了。

> 值得注意的是，函数的参数和返回值和函数体是在相同的作用域，尽管在语法上它们是在大括号之外的。



##### For

Go的for循环和C相似但也有不同。Go统一了for和while并且去除了do-while结构。以下是3中形式，只有一种使用了分号：

```go
// Like a C for
for init; condition; post { }

// Like a C while
for condition { }

// Like a C for(;;)
for { }
```

如果你是在遍历一个array，slice，map，或者是从channel中读取，可以使用range语句来管理循环。

```go
for key, value := range oldMap {
    newMap[key] = value
}
```

range遍历的过程中，可以安全的使用delete

```go
for key := range m {
    if key.expired() {
        delete(m, key)
    }
}
```

对于string，range做了更多的事情。拆分每一个单独的Unicode，如果出现错误，将返回U+FFFD并且占用一个byte（rune是Go的术语，表示一个Unicode字符）。

```
for pos, char := range "日本\x80語" { // \x80 is an illegal UTF-8 encoding
    fmt.Printf("character %#U starts at byte position %d\n", char, pos)
}

prints

character U+65E5 '日' starts at byte position 0
character U+672C '本' starts at byte position 3
character U+FFFD '�' starts at byte position 6
character U+8A9E '語' starts at byte position 7
```

最后，Go没有逗号操作符，同时++和--是`语句而不是表达式`，因此，如果你想要在for 中使用多个变量，你应该使用多变量的赋值语句（虽然这样使得无法使用++和--）。

```go
// Reverse a
for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
    a[i], a[j] = a[j], a[i]
}
```

##### Switch

Go中的switch比C语言的更加通用。表达式没有必要是常量或者整数，**如果switch没有表达式，那么case语句从上至下执行，直到结果为true的分支**。所以，`if-else-if-else`链式结构的代码可以通过switch来代替，这也是符合习惯的。

```go
func unhex(c byte) byte {
    switch {
    case '0' <= c && c <= '9':
        return c - '0'
    case 'a' <= c && c <= 'f':
        return c - 'a' + 10
    case 'A' <= c && c <= 'F':
        return c - 'A' + 10
    }
    return 0
}
```

没有自动的`fall through`（自动执行下一个case分支）,但是case表达式可以通过逗号分隔。

```go
func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
        return true
    }
    return false
}
```

在Go中break语句用来提前终止switch，虽然这个在其他与C类似的语言中并不常见。然而，有的时候确实想要终止外层的循环语句，在Go中可以通过使用Label(标签)来完成。

```go
Loop:
	for n := 0; n < len(src); n += size {
		switch {
		case src[n] < sizeOne:
			if validateOnly {
				break
			}
			size = 1
			update(src[n])

		case src[n] < sizeTwo:
			if n+1 >= len(src) {
				err = errShortInput
				break Loop
			}
			if validateOnly {
				break
			}
			size = 2
			update(src[n] + src[n+1]<<shift)
		}
	}
```

当然，`continue`语句也接受一个可选的标签，但是它只可运用在循环语句中；

最后，下面是一个比较常规的比较两个字节数组的例子。

```go
// Compare returns an integer comparing the two byte slices,
// lexicographically.
// The result will be 0 if a == b, -1 if a < b, and +1 if a > b
func Compare(a, b []byte) int {
    for i := 0; i < len(a) && i < len(b); i++ {
        switch {
        case a[i] > b[i]:
            return 1
        case a[i] < b[i]:
            return -1
        }
    }
    switch {
    case len(a) > len(b):
        return 1
    case len(a) < len(b):
        return -1
    }
    return 0
}
```

##### Type switch

switch结构也可以用于检查一个动态类型的interface类型的变量。`type switch`通过`type`关键词进行类型断言。如果switch痛死声明了一个变量，那么这个变量将会是每个case语句对应的类型。在这种场景中，使用声明的相同名字的变量，但是在不同的case分支中对应不同的类型，也是符合语言习惯的。

```go
var t interface{}
t = functionOfSomeType()
switch t := t.(type) {
default:
    fmt.Printf("unexpected type %T\n", t)     // %T prints whatever type t has
case bool:
    fmt.Printf("boolean %t\n", t)             // t has type bool
case int:
    fmt.Printf("integer %d\n", t)             // t has type int
case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t has type *bool
case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t has type *int
}
```



#### Functions



##### Multiple return values

Go语言一个与众不同的特性是函数可以有多个返回值。这样的语法是为了改善C语言中一些笨拙的语言习惯：错误返回（-1表示EOF），修改指针类型的参数。

在C语言中，一个写操作的错误通过返回返回计数为负来表示，隐藏了实际出错的位置。在Go语言中，Write返回count和error："是的，你写了一些bytes，但是不是所有的因为设备被填满了"。os包下的Write函数的定义：

```go
func (file *File) Write(b []byte) (n int, err error)
```

it returns the number of bytes written and a non-nil `error` when `n` `!=` `len(b)`.

这个是一个常见的编码分风格。

一个类似的建议是避免在返回值中通过传递指针来表示引用参数。

```go
func nextInt(b []byte, i int) (int, int) {
    for ; i < len(b) && !isDigit(b[i]); i++ {
    }
    x := 0
    for ; i < len(b) && isDigit(b[i]); i++ {
        x = x*10 + int(b[i]) - '0'
    }
    return x, i
}
```

你可以用过一下方式来遍历输入b数组中所有的数值：

```go
 for i := 0; i < len(b); {
 	x, i = nextInt(b, i)
 	fmt.Println(x)
 }
```

##### Named result parameters

Go语言的函数中可以给返回值命名，可以像普通变量和参数一样使用。当被命名之后，它们将会在函数开始执行时被初始化为对应类型的零值。如果函数的return语句没有携带参数，那么将用结果参数的当前值作为返回值；

**(*)区分result parameters 和 returned values两个概念**

命名并非是强制的，但是它可以使代码更加简短和清晰。如果我们给nextInt的结果命名，那么int具体的含义就更加明显了。

```go
func nextInt(b []byte, pos int) (value, nextPos int) {
```

因为被命名的结果会被初始化并且和单独的return语句关联，它们可以是很简洁和清晰的。io.ReadFull的例子：

```go
func ReadFull(r Reader, buf []byte) (n int, err error) {
    for len(buf) > 0 && err == nil {
        var nr int
        nr, err = r.Read(buf)
        n += nr
        buf = buf[nr:]
    }
    return
}
```

##### (*)Defer

Go语言中的defer语句安排一个函数调用（the *deferred* function），在函数执行defer退出之前执行。这种方式不常见但是在某些场景却是很有效的，比如资源释放，无论函数通过什么分支退出。典型的例子是释放mutex或者关闭文件。

```go
// Contents returns the file's contents as a string.
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close()  // f.Close will run when we're finished.

    var result []byte
    buf := make([]byte, 100)
    for {
        n, err := f.Read(buf[0:])
        result = append(result, buf[0:n]...) // append is discussed later.
        if err != nil {
            if err == io.EOF {
                break
            }
            return "", err  // f will be closed if we return here.
        }
    }
    return string(result), nil // f will be closed if we return here.
}
```

在函数中延迟执行Close函数有两个优势。首先，它保证了你不会忘记关闭文件，一个很容易犯的错误是在之后添加一个退出的分支。其次，close在open代码附近的位置，这样比把它放在函数结尾会清晰很多。

defer函数的参数是在**defer执行时计算的，而不是defer函数真正被调用时**。此外，避免了虑函数执行时变量的值是否被改变。这意味着单个defer语句可以创建多个延迟执行的函数。这是一个愚蠢的例子。

```
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}
```

defer函数的执行时按照LIFO的顺序，所以这段代码的结果是4 3 2 1 0。更加合理的例子是作为一种简单方式来追踪函数的调用。

```go
func trace(s string)   { fmt.Println("entering:", s) }
func untrace(s string) { fmt.Println("leaving:", s) }

// Use them like this:
func a() {
    trace("a")
    defer untrace("a")
    // do something....
}
```

通过defer函数的参数在定义时计算这一事实，我们可以做一些改进。tracing阶段可以作为untracing的参数。

```
func trace(s string) string {
    fmt.Println("entering:", s)
    return s
}

func un(s string) {
    fmt.Println("leaving:", s)
}

func a() {
    defer un(trace("a"))
    fmt.Println("in a")
}

func b() {
    defer un(trace("b"))
    fmt.Println("in b")
    a()
}

func main() {
    b()
}


func trace() int64 {
	s := time.Now().Unix()
	fmt.Println("entering:", s)
	return s
}

func un(s int64) {
	e := time.Now().Unix()
	fmt.Println("leaving:", e)
	fmt.Println("elapsed:", e-s)
}
```

对于习惯了块级资源管理的程序员来说，defer或许看上去有点奇怪。有趣的是，强大的应用程序正是基于以下事实：它不是基于块级的，而是函数级的。在panic和recover的章节我们将会看到另一种可能性。



#### Data



##### Allocation with `new`

















