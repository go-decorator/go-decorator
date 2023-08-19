# 使用指引

## 要求

- go1.18+
- go.mod project

## 安装

```shell
$ go install github.com/dengsgo/go-decorator/cmd/decorator@latest
```

运行 `decorator` 显示当前版本。

提示：请经常更新以安装最新版本。获得最佳体验。

## 使用 

### 添加参数

在 `go` 命令中添加参数 `-toolexec decorator` 即可。

例如 'go build **-toolexec decorator**'、'go run **-toolexec decorator** main.go'.

它适用于大多数 `go` 的子命令。

### 目标函数和装饰器

> 目标函数：即使用了装饰器的函数，也称为被装饰的函数。  
> 比如 A 函数使用了装饰器 B 来装饰自己，A 即为目标函数。

装饰器是一种概念，它通常用来对目标函数进行包含装饰。当代码运行到目标函数的时候，并不会真的执行它，而是运行它所使用的装饰器。实际的目标函数逻辑被包装到了装饰器中，并允许装饰器来控制它。

### 自定义装饰器

装饰器是普通的 `go` 一级函数(Top-level Function)，它的类型是 `func(*decor.Context)`. 只要函数满足这个类型，就可以在项目代码里作为装饰器来装饰其他函数。

例如, 这里定义一个 logging 装饰器，它可以打印被调用函数的参数：

```go
package main

import "github.com/dengsgo/go-decorator/decor"

func logging(ctx *decor.Context) {
	log.Println("logging print target in", ctx.TargetIn)
	ctx.TargetDo()
	log.Println("logging print target out", ctx.TargetOut)
}
```

这个函数 `logging` 就是一个合法的装饰器，它可以用在任意的一级函数上。

关于 `ctx *decor.Context` 参数的含义，查看这里。

### 使用装饰器

在任意一级函数上都可以通过注释 `//go:decor ` 来使用装饰器。

例如，我们有一个函数 `datetime`, 它可以把时间戳转换成字符串日期格式。通过 `//go:decor logging` 来使用 `logging` 装饰器打印函数出入参：

```go
// Omitted code ...

// Convert timestamp to string date format.
//
//go:decor logging
func datetime(timestamp int64) string {
    return time.Unix(timestamp, 0).String()
}

// Omitted code ...
```

`datetime` 在编译时会被识别并注入`logging`的调用。当其他地方调用 `datetime` 函数时，就会自动执行 `logging`。

比如，我们在 `main` 入口函数中调用 `datetime`:

```go
func main() {
	t := 1692450000
	s := datetime(t)
	log.Printf("datetime(%d)=%s\n", t, s)
}
```

使用下面的命令编译、运行:

```shell
$ go build -toolexec decorator
$ ./datetime
```
将会看到如下输出：

```shell
2023/08/19 21:12:21 logging print target in [1692450000]
2023/08/19 21:12:21 logging print target out [2023-08-19 21:00:00 +0800 CST]
2023/08/19 21:12:21 datetime(1692450000)=2023-08-19 21:00:00 +0800 CST
```
我们代码中只调用了 `datetime` 函数，但是可以看到使用的 logging 装饰器也被执行了！

完整代码可以查看 [example/datetime](example/datetime). 

### 使用多个装饰器

`decorator` 允许同时使用多个装饰器来装饰目标函数, 但并不推荐这样做。 

通过多个 `go:decor ` 来使用多个装饰器。

例如，下面的 `datetime` 使用了3个装饰器,分别是 `logging`、`appendFile`、`timeFollowing`：

```go
// Omitted code ...

// Convert timestamp to string date format.
//
//go:decor logging
//go:decor appendFile
//go:decor timeFollowing
func datetime(timestamp int64) string {
    return time.Unix(timestamp, 0).String()
}

// Omitted code ...
```

如果使用了多个装饰器，装饰器执行的优先级为从上往下，也就是说先定义的先被执行。上面的装饰器中，执行顺序为 `logging` -> `appendFile` -> `timeFollowing`.

多个装饰器的使用，可能会导致代码的可读性变差，逻辑流程理解困难，尤其是装饰器本身的代码又特别复杂的情况。因此并不推荐这样使用。

## Context

`decor.Context` 是装饰器函数的入参，它是目标函数（即使用了这个装饰器的函数，也称为被装饰的函数）上下文。

在装饰器中可以使用这个上下文来修改目标函数的出入参，调整执行逻辑等操作。

### ctx.Kind

目标函数类型，目前只有 `decor.KFunc`, 函数类型。

### ctx.TargetIn

目标函数的入参列表。它是一个[]any slice, 其中每个元素的类型和目标函数的入参类型一致。 如果目标函数没有入参，列表为空。

`ctx.TargetDo()` 会使用这个 slice 来当作真实调用的入参，因此改变它的元素值可以修改目标函数的入参。只在 `ctx.TargetDo()` 调用前修改有效。

### ctx.TargetOut

目标函数的出餐列表。它是一个[]any slice, 其中每个元素的类型和目标函数的出参类型一致。 如果目标函数没有出参，列表为空。

`ctx.TargetDo()` 会使用这个 slice 来接收真实调用的结果，因此改变它的元素值可以修改目标函数的出参。只在 `ctx.TargetDo()` 调用后修改有效。

### ctx.TargetDO()

执行目标函数。它是对目标函数的无参化包装，调用它才会真正的执行目标函数逻辑。

它从 `ctx.TargetIn` 获取到目标函数入参值，然后执行目标函数代码，再把得到的结果赋值到 `ctx.TargetOut`。

如果装饰器中没有执行 `ctx.TargetDO()` ，意味着目标函数真实的逻辑不会被执行，调用目标函数得到的结果是零值（在没有修改 ctx.TargetOut 的情况下）。

## 包引用

上面的 `datetime` [example/datetime](example/datetime) 例子中，我们的装饰器和目标函数都是在一个包中的，我们无需考虑包的问题。

go 规范中，没有被当前文件里的代码使用到的包无法导入，这就导致了 `//go:decor` 这样的注释无法真正的导入包，因此需要我们使用匿名导入包的方式来导入对应的包。像这样 `import _ "path/to/your/package"`.

有下面两种情况需要使用匿名导入包：

一、包的函数使用了装饰器注释，但没有导入 `github.com/dengsgo/go-decorator/decor` 这个包，需要我们添加匿名导入包：

```go
import _ "github.com/dengsgo/go-decorator/decor"
```

二、如果包(A)引用了另外一个包(B)的装饰器, 而 B 没有被 A `import`,我们需要使用匿名导入包的方式导入它。

例如：

```go
package main

// other imports
import _ "github.com/dengsgo/go-decorator/example/packages/fun1"

//go:decor fun1.DecorHandlerFunc
func test() {
	//...
}
```

当然，如果包已经被文件里其他代码用到了，已经导入，那么就不需要再匿名导入了。

完整的例子可以查看 [example/packages](example/packages) .

## 条件和限制

以下几种情况需要注意：

- 使用装饰器的目标函数范围**仅限当前项目内**。依赖的其他库即使使用 `//go:decor`也**无法**被装饰。

例如，你的项目module名称时 `a/b/c` ，那么 `//go:decor` 只在 `a/b/c` 及其子包中生效（`a/b/c/d` 有效，`a/m/`无效）。

但是`//go:decor`可以使用任意包的装饰器，没有范围限制。

- **不能**在同一个目标函数上同时使用相同的装饰器重复装饰；  
- **不能**对装饰器函数应用装饰器；

## 性能

尽管 `decorator` 在编译时会对目标函数做额外的处理，但是相对于原始go代码直接调用装饰器函数来讲，性能几乎没有损失。

// TODO 提供性能指标对比

## 更多

// TODO