---
title: Go 错误处理
date: 2022-02-24T13:38:28+08:00
draft: false
tags:
  - go
---

# Go 语言如何表示错误

## 内置接口

Go 语言内置了一个接口类型 `error`

```golang
type error interface {
  Error() string
}
```

所有实现了 `error` 接口的类型被称为错误类型(Error Type)

错误类型的值表示一个错误的发生

`error` 接口只定义了一个方法 `Error()`，该方法返回一个字符串，用来描述错误信息

注意：`Error()` 方法是为人类设计的，而非代码

`Error()` 返回的错误信息只应该被存储在日志中，或是被打印到屏幕上，用于人类查看，绝不能在代码中检查该错误信息，依赖于该错误信息决定程序的执行逻辑

例如，下面的代码是不好的

```go
func main() {
  content, err := os.ReadFile("example.txt")
  errInfo := err.Error()
  switch errInfo {
  case "open example.txt: no such file or directory":
    // handling error: create file
  case "open example.txt: permission denied":
    // handling error: modify permission
  }
}
```

## errors 包中的错误类型

Go 语言在标准库的 `errors` 包中定义了一个非常简单的错误类型

```go
type errorString struct {
  s string
}

func (e *errorString) Error() string {
  return e.s
}
```

由于 `errorString` 不是导出的，`errors` 包同时实现了一个 `New()` 函数用于创建该错误类型的值

```go
// New returns an error that formats as the given text.
// Each call to New returns a distinct error value even if the text is identical.
func New(text string) error {
  return &errorString{text}
}
```

这样的好处是，更加语义化，比如我们创建错误时使用 `errors.New("some error")` 要比 `&ErrorString{"some error"}` 看起来更有语义

第二点是防止出错，例如不小心使用 `ErrorString{"some error"}`，因为 `*errorString` 类型才实现了 `Error()` 方法，`errorString` 类型并没有实现

那为什么不这样定义 `errorString` 错误类型呢

```go
type errorString struct {
  s string
}

func (e errorString) Error() string {
  return e.s
}
```

或者

```go
type errorString string

func (e errorString) Error() string {
	return string(e)
}
```

如果这样定义的话，当我们创建两个字符串内容相同的 `error` 时，这两个 `error` 的是相等的

这样因为 Go 语言对 `struct` 的 `==` 比较会比较其内部的各个字段

```go
type errorString struct {
	s string
}

func (e errorString) Error() string {
	return e.s
}

func main() {
	err1 := errorString{"some error"}
	err2 := errorString{"some error"}
	fmt.Println(err1 == err2) // true
}
```

下面的代码同理

```go
type errorString string

func (e errorString) Error() string {
	return string(e)
}

func main() {
	err1 := errorString("some error")
	err2 := errorString("some error")
	fmt.Println(err1 == err2) // true
}
```

但是 `errorString` 的设计并不希望这样，它希望的是每一个值都是不同的，注意 `New()` 函数上面的注释！

```go
func main() {
  err1 := errors.New("some error")
  err2 := errors.New("some error")
  fmt.Println(err1 == err2) // false
}
```

> 下面会讲到，标准库里使用众多的一个 Go 错误处理的设计模式 sentinel error，就依赖该特性

`fmt` 包还有一个 `Errorf` 函数

```go
// go1.13 之前
func Errorf(format string, a ...interface{}) error {
	return errors.New(Sprintf(format, a...))
}
```

这个函数更方便一点，先格式化后，再调用 `errors.New()`

我们创建一个可能发生错误的函数时，可以使用 `errors.New()`

```go
func Sqrt(f float64) (float64, error) {
  if f < 0 {
    return 0, errors.New("math: square root of negative number")
  }
  // implementation
}
```

抛出错误方还应该包含尽可能多的上下文信息，例如 `os.ReadFile()` 的 `Error()` 方法会返回 `open example.txt: permission denied`，而不只是 `permission denied`

此时，我们可以使用 `fmt.Errorf()`

```go
if f < 0 {
  return 0, fmt.Errorf("math: square root of negative number %g", f)
}
```

## 自定义错误类型

使用 `errors.New()` 或 `fmt.Errorf()` 创建 `errors.errorString` 类型已经及格了，但我们可以做的更好

在实际的业务场景中，我们可以定义自己的错误类型，一般是一个 `struct`，里面是和业务相关的字段，例如

```go
type ParseError struct {
  Msg string
  File string
  Line int
}

func (e *ParseError) Error() string {
  return fmt.Sprintf("%s:%d: %s", e.File, e.Line, e.Msg)
}

func parse() error {
  return &ParseError{"some error", "example.json", 66}
}
```

# Go 语言如何检查错误

上一节介绍了 Go 语言如何表示错误，核心就三句话：
1. Go 语言内置了一个接口类型 `error`
2. 所有实现了 `error` 接口的类型被称为错误类型
3. 错误类型的值表示一个错误

程序要和外界交互，就会不可避免地会出现事先预知的异常情况

这一节讲的是 Go 设计错误以及检查错误的几种方式

最简单的是比较 `err` 是否是 `nil`

```go
if err != nil {
	// something went wrong
}
```

对于大多数情况，这种方法是很好的，错误的细节对于调用者是透明的

但是有些 API 可能发生多种错误，通过上述方法，调用者也只能知道是否发生了错误，而不能知道是哪一种错误

在标准库中很常见的一种做法是定义 sentinel error, 将可能的错误定义为一些错误值，每个错误值对应一种错误

```go
var ErrNotFound = errors.New("not found")

if err == ErrNotFound {
	// something wasn't found
}
```

但有时调用者还需要知道更多的错误信息，因此许多库定义了自己的错误类型并暴露出来，让调用者使用接口断言机制获取更多信息

```go
type NotFoundError struct {
	Name string
}

func (e *NotFoundError) Error() string { return e.Name + ": not found" }

if e, ok := err.(*NotFoundError); ok {
	// e.Name wasn't found
}
```

上面是 3 种设计和检查错误的方式，检查错误是为了更好地处理，错误处理一个很普遍的方式是向上传递

当调用 API 得到 `err != nil` 时，调用方可能并没有责任去处理错误，而应该将错误传给上层，并且希望添加当前的上下文信息

```go
if err != nil {
	return fmt.Errorf("decompress %v: %v", name, err)
}
```

但是，这种行为会破坏以上的所有的错误设计，不论是 sentinel error，还是错误类型

因为 `fmt.Errorf()` 返回的是一个新的 `errorString` 类型的错误值

更上层的调用者无法使用该错误值和 sentinel error 比较，也无法断言为库导出的错误类型

`fmt.Errorf()` 抹除了原有错误的一切类型信息

解决这个问题的方法是包装错误，`fmt.Errorf()` 本质上是创建了一个新的错误，对原始错误的保留仅仅是给人类看的错误信息

那我们自定义错误类型时，可以专门设置一个字段保存原始错误

```go
type QueryError struct {
	Query string
	Err   error
}
```

这种模式很快成为包装错误事实上的标准

当我们检查 sentinel error 时

```go
if err != nil {
	if qerr, ok := err.(QueryError); ok && qerr.Err == ErrNotFound {
		// do something
	}
}
```

当我们检查错误类型时

```go
if err != nil {
	if qerr, ok := err.(QueryError); ok {
		if nerr, ok := qerr.Err.(*NotFoundError); ok {
			// do something	with nerr.Name
		}
	}
}
```

# Go 语言如何包装错误

现在我们有了包装错误的方法，但是具体还需要实现很多功能

例如，希望在当前调用的层级添加调用栈信息

或是，上面的检查错误太麻烦了，我们希望能更简单的检查错误

`github.com/pkg/errors` 和 Go1.13 版本对标准库的改进就是为了解决这些问题

## github.com/pkg/errors

因为 Go 的错误处理依赖契约规范(错误类型必须实现的只是一个为人类设计的 `Error()` 方法，没有任何其他能力!)，而不在语言编译器或运行时层面，

因此 Go 语言给了开发者极大的自由自己去定义错误处理的整个体系

由于 Go 语言 errors 包提供的错误类型太过简陋，一些开发者选择自己去定义错误类型和相关的错误处理方法

最有名的是 github.com/pkg/errors 包，这个包已经成为 Go 语言错误处理的事实标准，该包主要实现了错误链，包装调用栈和上下文信息等功能

pkg/errors 包定义了三个错误类型 `fundamental`, `withMessage`, `withStack`

```go
type fundamental struct {
	msg string
	*stack
}

func (f *fundamental) Error() string { return f.msg }

type withMessage struct {
	cause error
	msg   string
}

func (w *withMessage) Error() string { return w.msg + ": " + w.cause.Error() }
func (w *withMessage) Cause() error  { return w.cause }

type withStack struct {
	error
	*stack
}

func (w *withStack) Cause() error { return w.error }
```

这些错误类型有一个共同点就是都有一个 `cause` 字段, 来保存底层的错误，同时有一个 `Cause()` 方法返回 `cause` 字段

这样就可以实现一个 `Cause()` 函数，沿着 cause 字段构成的错误链，一只找到最底层的那个没有实现 `Cause()` 方法的原始错误

```go
func Cause(err error) error {
	type causer interface {
		Cause() error
	}

	for err != nil {
		cause, ok := err.(causer)
		if !ok {
			break
		}
		err = cause.Cause()
	}
	return err
}
```

pkg/errors 实现了一些函数来创建错误或包装错误，代码都很简单，如下所示

```go
func New(message string) error {
	return &fundamental{
		msg:   message,
		stack: callers(),
	}
}

func Errorf(format string, args ...interface{}) error {
	return &fundamental{
		msg:   fmt.Sprintf(format, args...),
		stack: callers(),
	}
}


func WithMessage(err error, message string) error {
	if err == nil {
		return nil
	}
	return &withMessage{
		cause: err,
		msg:   message,
	}
}

func WithMessagef(err error, format string, args ...interface{}) error {
	if err == nil {
		return nil
	}
	return &withMessage{
		cause: err,
		msg:   fmt.Sprintf(format, args...),
	}
}

func WithStack(err error) error {
	if err == nil {
		return nil
	}
	return &withStack{
		err,
		callers(),
	}
}

func Wrap(err error, message string) error {
	if err == nil {
		return nil
	}
	err = &withMessage{
		cause: err,
		msg:   message,
	}
	return &withStack{
		err,
		callers(),
	}
}

func Wrapf(err error, format string, args ...interface{}) error {
	if err == nil {
		return nil
	}
	err = &withMessage{
		cause: err,
		msg:   fmt.Sprintf(format, args...),
	}
	return &withStack{
		err,
		callers(),
	}
}
```

总结一下就是
- New/Errorf
  - 创建错误
  - 新的错误是 `fundamental` 类型
  - 带有 call stack 信息
- WithMessage/WithMessagef
  - 包装错误
  - 新的错误是 `withMessage` 类型
  - 增添了上下文信息
- WithStack
  - 包装错误
  - 新的错误是 `withStack` 类型
  - 增添了 call stack 信息
- Wrap/Wrapf
  - 包装错误
  - 先调用 `withMessage`，再调用 `withStack`
  - 增添了 call stack 信息和上下文信息

## Go1.13

在过去 Go 语言错误处理方面的实践中，自定义错误类型使用一个字段来指向底层错误的做法已经非常普遍

不管是上面通用目的的 `fundamental`, `withMessage`, `withStack`，还是一些面向特定场景的错误类型，例如

```go
type QueryError struct {
	Query string
	Err   error
}

func (e *QueryError) Error() string { return e.Query + ": " + e.Err.Error() }
```

因此 Go 在 1.13 版本增加了对这种模式的支持，主要是为 errors 包添加了以下三个函数

```go
func Unwrap(err error) error {}
func Is(err, target error) bool {}
func As(err error, target interface{}) bool {}
```

同时修改了 `fmt.Errorf()` 的实现


# 如何设计错误处理

Go 语言的错误处理有下面三种设计模式：
1. sentinel error
2. custom error type
3. opaque error

## sentinel error

## custom error type

## opaque error





# Go 语言如何处理错误

这一节将介绍处理错误的方法和原则

首先，错误处理有一条通用的原则：对于错误，要么不处理，要么完全处理

我不知道怎么做，不处理，直接将错误传给上层, 如下示例

```go
func main() {
	content, err := readFile("example.txt")
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(content)
}

func readFile(path string) (string, error) {
	content, err := os.ReadFile(path)
	if err != nil {
		return "", err
	}
	return string(content), nil
}
```

我知道怎么做，完全处理好，屏蔽掉底层错误，如下示例

```go
func main() {
	content := readFile("example.txt")
	fmt.Println(content)
}

func readFile(path string) string {
	content, err := os.ReadFile(path)
	if err != nil {
		return "default text"
	}
	return string(content)
}
```

总之一定要清楚自己在干什么，不要又处理一部分，又要抛给上层

> Don't just check errors, handle them gracefully.

最常见的场景是记录日志，因为记录日志本身也是处理错误

```go
func readFile(path string) (string, error) {
	content, err := os.ReadFile(path)
	if err != nil {
		log.Println(err)
		return "", err
	}
	return string(content), nil
}
```

错误处理一般有以下几种选择
1. 直接将错误向上传，即直接 return err
2. 包装当前错误，添加上下文信息, 再向上传
3. 屏蔽掉底层错误, 返回自己定义的错误
4. 处理错误，不返回错误

对于应用来说，不需要定义自己的错误类型，使用 github.com/pkg/errors 包

可以把应用的调用链抽象为 layer1 -> layer2 -> layer3 -> lib

layer3 就是调用 API 的地方，即应用的边界，对于 lib 返回的 error，使用 WithStack/Wrap/Wrapf 包装, 这里的重点是带上调用栈信息

layer2 直接返回 layer3 的 error，或者使用 WithMessage/WithMessagef 包装，或者直接处理好，不返回 error

layer1 是最上层了，只能把 error 处理好，日志记录就在这里

对于库来说，要使用上面讲的 opaque error 设计

# 错误处理技巧

长久以来，对于 Go 语言的一种批评就是"错误处理太繁琐了，要在代码里写非常多的 `if err != nil`"

但这可能是习惯于 `try...catch` 的程序员并没有发挥 Go 语言错误处理的能力

他们把 `if err != nil` 当作一个定式，像 `try...catch` 那样, 而忽略了 Go 语言错误处理的本质，即 Errors are value

因此，错误在编程中是非常灵活的

> Values can be programmed, and since errors are values, errors can be programmed.

下面是一个示例

```go
func parse(r io.Reader) (*Point, error) {
  var p Point
  var err error
  err = binary.Read(r, binary.BigEndian, &p.Longitude)
  if err != nil {
    return nil, err
  }
  err = binary.Read(r, binary.BigEndian, &p.Latitude)
   if err != nil {
    return nil, err
  }
  err = binary.Read(r, binary.BigEndian, &p.Distance)
  if err != nil {
    return nil, err
  }
  err = binary.Read(r, binary.BigEndian, &p.ElevationGain)
  if err != nil {
    return nil, err
  }
  err = binary.Read(r, binary.BigEndian, &p.ElevationLoss)
  if err != nil {
    return nil, err
  }
  return &p, nil
}
```

要解决这个问题，我们可以使用闭包

```go

func parse(r io.Reader) (*Point, error) {
  var p Point
  var err error
  read := func(data interface{}) {
    if err != nil {
      return
    }
    err = binary.Read(r, binary.BigEndian, data)
  }

  read(&p.Longitude)
  read(&p.Latitude)
  read(&p.Distance)
  read(&p.ElevationGain)
  read(&p.ElevationLoss)

  if err != nil {
    return &p, err
  }
  return &p, nil
}
```

```go
type Reader struct {
  r io.Reader
  err error
}

func (r *Reader) read(data interface{}) {
  if r.err != nil {
    return
  }
  r.err = binary.Read(r.r, binary.BigEndian, data)
}


func parse(input io.Reader) (*Point, error) {
  var p Point
  r := Reader{r: input}

  r.read(&p.Longitude)
  r.read(&p.Latitude)
  r.read(&p.Distance)
  r.read(&p.ElevationGain)
  r.read(&p.ElevationLoss)

  if r.err != nil {
    return nil, r.err
  }

  return &p, nil
}
```

使用这种方式，我们还可以实现流式 API



实际上，`bufio` 包中的 `Scanner` 就是这么做的

```go
scanner := bufio.NewScanner(input)
for scanner.Scan() {
  token = scanner.Text()
  // process token
}
if err := scanner.Err(); err != nil {
  // process error
}
```


# References

- https://go-proverbs.github.io/
- https://go.dev/blog/error-handling-and-go
- https://go.dev/blog/errors-are-values 🌟
- https://go.dev/blog/go1.13-errors 🌟
- https://dave.cheney.net/2014/12/24/inspecting-errors
- https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully 🌟
- https://dave.cheney.net/2016/06/12/stack-traces-and-the-errors-package
- https://dave.cheney.net/2019/01/27/eliminate-error-handling-by-eliminating-errors
- https://crawshaw.io/blog/xerrors
- https://www.ardanlabs.com/blog/2014/10/error-handling-in-go-part-i.html
- https://www.ardanlabs.com/blog/2014/11/error-handling-in-go-part-ii.html
- https://www.ardanlabs.com/blog/2017/05/design-philosophy-on-logging.html
- https://medium.com/gett-engineering/error-handling-in-go-53b8a7112d04
- https://medium.com/gett-engineering/error-handling-in-go-1-13-5ee6d1e0a55c
- https://pkg.go.dev/golang.org/x/xerrors
- https://pkg.go.dev/github.com/pkg/errors
- https://pkg.go.dev/errors@go1.17.7
