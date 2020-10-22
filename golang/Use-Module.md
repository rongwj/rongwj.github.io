<!-- TOC -->

- [1. 使用Go模块](#1-使用go模块)
  - [1.1. 创建一个模块](#11-创建一个模块)
  - [1.2. 添加一个依赖](#12-添加一个依赖)
  - [1.3. 更新依赖](#13-更新依赖)
  - [1.4. 添加新的major版本的依赖](#14-添加新的major版本的依赖)
  - [1.5. 更新一个依赖的major版本](#15-更新一个依赖的major版本)
  - [1.6. 删除无用的依赖](#16-删除无用的依赖)
  - [1.7. 总结](#17-总结)
  - [1.8. 引用](#18-引用)

<!-- /TOC -->

# 1. 使用Go模块

Go 1.11和1.12最先开始支持模块，Go新的依赖管理系统使得依赖包的版本信息更加清晰、简单和便于管理。这篇文章从模块的使用来开始介绍基本的模块操作。

一个模块是一个保存Go包集合的文件，该文件是它根目录下的go.mod文件。go.mod文件定义模块的模块路径，该路径不仅包含导入的路径，还包含它能成功编译所需要的依赖条件，这些依赖条件是模块依赖的其他模块。每一个单独的依赖条件是一个模块路径和特定的语义上的版本。

自从Go 1.11，在当前目录或任何的父目录包含一个go.mod文件，go的命令就可以使用模块。这个目录不属于$GOPATH/src的子目录，但是兼容$GOPATH/src的子目录情况，这种情况下即使有go.mod文件，go命令任然运行在老的GOPATH模式。从Go 1.13后，模块模式是所有开发方式的默认模式。

这篇文章介绍了一系列在使用go模块的常见操作：

- 创建一个模块
- 添加依赖
- 更新依赖
- 在一个新版本添加依赖
- 在一个新版本更新依赖
- 删除无用的依赖
  
## 1.1. 创建一个模块

再$GOPATH/src外创建一个空目录，然后创建一个源文件，hello.go。

```go
package hello

func Hello() string {
    return "Hello, world."
}
```

接着写一个测试，hello_test.go

```go
package hello

import "testing"

func TestHello(t *testing.T) {
    want := "Hello, world."
    if got := Hello(); got != want {
        t.Errorf("Hello() = %q, want %q", got, want)
    }
}
```

现在因为没有go.mod文件，导致这个目录包含一个包，但不是一个模块。如果我们在/home/gopher/hello运行go test命令：

```bash
# go test
PASS
ok  _/home/gopher/hello 0.008s
```

最后一行解释整个包的测试结果。因为我们工作目录不是在$GOPATH和任何模块子目录下，所以go命令知道当前目录没有导入路径，就编造一个基于_/home/gopher/hello伪模块：_/home/gopher/hello。

接着使用go mod init在模块的根目录创建一个模块，然后在运行go test。

```bash
# go mod init example.com/hello
go: creating new go.mod: module example.com/hello
# go test
PASS
ok  example.com/hello 0.004s
```

go mod init命令会生成一个go.mod文件：

```bash
# cat go.mod
module example.com/hello

go 1.14
```

go.mod文件只会出现在模块的root目录下。模块的子目录的导入路径由模块路径加上子目录名，例如我们创建一个world子目录，我们不需要运行go mod init。这个包默认被当作模块的一部分，导入路径example.com/hello/world。

## 1.2. 添加一个依赖

Go模块的主要目的时提高使用三方代码的体验。

现在我们更新hello.go文件，导入一个rsc.io/quote并使用它实现Hello：

```go
package hello

import "rsc.io/quote"

func Hello() string {
    return quote.Hello()
}
```

现在再次运行测试用例：

```bash
# go test
go: finding module for package rsc.io/quote
go: downloading rsc.io/quote v1.5.2
go: found rsc.io/quote in rsc.io/quote v1.5.2
go: downloading rsc.io/sampler v1.3.0
go: downloading golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
PASS
ok  example.com/hello 0.007s
```

go 命令解析go.mod保留的特定版本的依赖模块并导入。当依赖的包不属于任何一个模块的go.mod。go命令会自动查找模块包含的包并加入到它的go.mod中，并依次使用最新的版本（最新标识最新的稳定tag），或者是最新的预发布tag版本，或最新untagged版本。  

在我们的例子中，go test解析最新导入rsc.io/quote到模块的 rsc.io/quote v1.5.2。
同时会下载rsc.io/quote的两个依赖，rsc.io/sampler 和 golang.org/x/text。唯一直接依赖加入到go.mod文件中：

```bash
# cat go.mod
module example.com/hello

go 1.14

require rsc.io/quote v1.5.2
```

再次使用go test将不会重复这些工作，go.mod现在是最新的并且下载的模块缓存在本地的$GOPATH/pkg/mod目录：

```bash
PASS
ok  example.com/hello 0.007s
```

当go命令使添加一个新的依赖快速和简单并没有额外的代价，你的模块现在仅仅通过几个名称，就可以正确地、安全地和合适的许可地使用依赖。更多详见[Our Software Dependency Problem](https://research.swtch.com/deps)

正如上面所见，添加一个直接依赖通常会添加其他的间接依赖。go list -m all可以查看当前模块所有的依赖：

```bash
# go list -m all
example.com/hello
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
```

在go list的输出中，当前模块是主模块，总是在第一行，接下来的依赖按照模块的路径排序。

golang.org/x/text的版本v0.0.0-20170915032832-14c0d48ead0c是[pseudo-version](https://golang.org/cmd/go/#hdr-Pseudo_versions)的一个示例, 这是一个untagged的提交。

除了go.mod之外，go命令维护一个go.sum文件，改文件包含模块内容的[cryptographic hashes](https://golang.org/cmd/go/#hdr-Module_downloading_and_verification)。

```bash
# cat go.sum
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZOaTkIIMiVjBQcw93ERBE4m30iBm00nkL0i8=
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ=
rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3Y=
rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPXsUe+TKr0=
rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/QiW4=
rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9JXDnKaTXpA=
```

go命令使用go.sum文件去保证将来下载模块的正确性，从而保证你的工程不会依赖一个不正确的模块。go.mod和go.sum同样应该被版本控制检验。

## 1.3. 更新依赖

再Go模块中，版本时引用一个语义的版本tag。一个语义的版本有三个部分：major，minor，和patch。例如v0.1.2，major是0，minor是1，patch是2。

在go list -m all我们看到untagged版本的golang.org/x/text。现在我们更新最新的tagged版本并测试所有的事是否正常运行。

```bash
# go get golang.org/x/text
go: golang.org/x/text upgrade => v0.3.3
go: downloading golang.org/x/text v0.3.3
# go test
PASS
ok  example.com/hello 0.008s
```

一切正常，接下来看看go list -m all和go.mod文件：

```bash
# go list -m all
example.com/hello
golang.org/x/text v0.3.3
golang.org/x/tools v0.0.0-20180917221912-90fa682c2a6e
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
# cat go.mod
module example.com/hello

go 1.14

require (
    golang.org/x/text v0.3.3 // indirect
    rsc.io/quote v1.5.2
)
```

golang.org/x/text包已经被更新到最新的tagged版本。go.mod文件也更新了。indirect注释说明一个依赖不是模块的直接依赖，仅仅是其他模块的依赖。详细见go help modules。

现在我们更新rsc.io/quote的minor版本。同样的方式，运行go get和go test：

```bash
go get rsc.io/sampler
go: rsc.io/sampler upgrade => v1.99.99
go: downloading rsc.io/sampler v1.99.99
# go test
--- FAIL: TestHello (0.00s)
    hello_test.go:8: Hello() = "99 bottles of beer on the wall, 99 bottles of beer, ...", want "Hello, world."
FAIL
exit status 1
FAIL example.com/hello 0.005s
```

测试失败，输出显示rsc.io/sampler的最新版本和我们使用的不兼容。让我们列出模块可使用的tagged：

```bash
# go list -m -versions rsc.io/sampler
rsc.io/sampler v1.0.0 v1.2.0 v1.2.1 v1.3.0 v1.3.1 v1.99.99
```

我们使用的是v1.3.0，v1.99.99明显不可用。我们试试v1.3.1：

```bash
# go get rsc.io/sampler@v1.3.1
go: downloading rsc.io/sampler v1.3.1
# go test
PASS
ok  example.com/hello 0.009s
```

> 在go get命令中显示的使用v1.3.1参数，通常会获取一个显示的版本，默认的是@latest。

## 1.4. 添加新的major版本的依赖

我们在我们包添加一个新函数：func Proverb返回一个Go并发格言，通过调用quote.Concurrency，该函数是rsc.io/quote/v3提供。

```bash
package hello

import (
    "rsc.io/quote"
    quoteV3 "rsc.io/quote/v3"
)

func Hello() string {
    return quote.Hello()
}

func Proverb() string {
    return quoteV3.Concurrency()
}
```

测试文件hello_test.go文件添加：

```bash
func TestProverb(t *testing.T) {
    want := "Concurrency is not parallelism."
    if got := Proverb(); got != want {
        t.Errorf("Proverb() = %q, want %q", got, want)
    }
}
```

测试输出：

```bash
# go test
go: finding module for package rsc.io/quote/v3
go: downloading rsc.io/quote/v3 v3.1.0
go: found rsc.io/quote/v3 in rsc.io/quote/v3 v3.1.0
go test
PASS
ok  example.com/hello 0.006s
```

> 我们的模块先在更新rsc.io/quote和rsc.io/quote/v3：

```bash
# go list -m rsc.io/q...
rsc.io/quote v1.5.2
rsc.io/quote/v3 v3.1.0
```

每一个不同major版本(v1、v2等等)Go模块使用不同模块路径：路径必须以major版本结尾。在这个例子中，v3版本的rsc.io/quote不再是rsc.io/quote：而是由rsc.io/quote/v3定义的模块。这个约定称之为[semantic import versioning](https://research.swtch.com/vgo-import),这样可以给出不同版本的兼容包。比较起来v1.6.0版本quote会向前兼容v1.5.2，因此重复使用rsc.io/quote。

go命令最多允许构建包含一个特定major版本的模块，意味着major版本最多一个：quote、quote/v2、quote/v3等等。这样对模块的作者对单个模块路径的多副本有一个清晰的规则：它运行程序用v1.5.2和v1.6.2构建成为可能。同时运行不同major版本模块，使模块的使用在升级到新的major更加的平滑。

## 1.5. 更新一个依赖的major版本

先在让我们完成quote/v3到quote的转换。因为major版本的改变，一些API可能已经被删除、重命名或改成兼容方式。通过阅读docs，我们可以看到Hello变成HelloV3：

```bash
# go doc rsc.io/quote/v3
package quote // import "rsc.io/quote/v3"

Package quote collects pithy sayings.

func Concurrency() string
func GlassV3() string
func GoV3() string
func HelloV3() string
func OptV3() string
```

这里的Output有一个已知的bug，导入路径缺少了/v3。

我们更新hello.go文件的quote.Hello()为quoteV3.HeeloV3():

```go
package hello

import quoteV3 "rsc.io/quote/v3"

func Hello() string {
    return quoteV3.HelloV3()
}

func Proverb() string {
    return quoteV3.Concurrency()
}
```

现在我们不再需要重命名导入了：

```go
package hello

import "rsc.io/quote/v3"

func Hello() string {
    return quote.HelloV3()
}

func Proverb() string {
    return quote.Concurrency()
}
```

运行go test保证一切正常

```bash
# go test
PASS
ok  example.com/hello 0.008s
```

## 1.6. 删除无用的依赖

我们已经删除了quote的使用了，但是go list -m all和go.mod任然包含它：

```bash
# go list -m all
example.com/hello
golang.org/x/text v0.3.3
golang.org/x/tools v0.0.0-20180917221912-90fa682c2a6e
rsc.io/quote v1.5.2
rsc.io/quote/v3 v3.1.0
rsc.io/sampler v1.3.1

# cat go.mod
module example.com/hello

go 1.14

require (
    golang.org/x/text v0.3.3 // indirect
    rsc.io/quote v1.5.2
    rsc.io/quote/v3 v3.1.0
    rsc.io/sampler v1.3.1 // indirect
)
```

go build或者go test可以知道缺少的和需要添加的依赖，但是不能安全的取删除一些依赖。删除一个依赖仅仅在检验了模块所有的依赖，还有这些依赖所有可能的 build tag组合后才可以删除一个依赖。一个普通的构建命令不会加载这些信息，因此不能安全删除依赖。

go mod tidy命令会清除不需要的依赖：

```bash
# go mod tidy
# go list -m all
example.com/hello
golang.org/x/text v0.3.3
golang.org/x/tools v0.0.0-20180917221912-90fa682c2a6e
rsc.io/quote/v3 v3.1.0
rsc.io/sampler v1.3.1

# cat go.mod
module example.com/hello

go 1.14

require (
    golang.org/x/text v0.3.3 // indirect
    rsc.io/quote/v3 v3.1.0
    rsc.io/sampler v1.3.1 // indirect
)
```

## 1.7. 总结

Go modules是Go依赖管理，模块的功能在Go 1.11之后都可以使用。

这篇文件介绍使用Go模块的流程：

- go mod init 创建一个新的模块，初始化go.mod文件。
- go build，go test，和其他的包构建命令添加需要的新依赖到go.mod。
- go list -m all查看当前模块所有的依赖。
- go get改变当前模块的依赖版本(或添加一个新依赖)。
- go mod tidy 删除无用的依赖。
  
## 1.8. 引用

- [using-go-modules](https://blog.golang.org/using-go-modules)
