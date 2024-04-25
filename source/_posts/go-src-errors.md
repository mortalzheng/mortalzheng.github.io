---
title: Go 中的 errors 包
date: 2024-04-18 20:27:19
tags: Go
---
我们先来看下 Go 语言对于 error 的签名定义
```
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
type error interface {
	Error() string
}
```
error 是被定义为包含 ``` Error() string ```方法的接口
而 errors 包就是对 error 接口的实现, 通过方法 New 来创建一个 error 对象, Error() 方法, 简单返回错误文本信息
```
func New(text string) error {
	return &errorString{text}
}
type errorString struct {
	s string
}
func (e *errorString) Error() string {
	return e.s
}
```
由于结构体 errorString 实现了 ``` Error() string ``` 方法, 所以它实现了 error 接口
需要注意的事: 每次调用 New 方法, 创建的 err 对象都具有不同的分配地址, 所以是不同的对象
```
// Different allocations should not be equal.
if errors.New("abc") == errors.New("abc") {
	t.Errorf(`New("abc") == New("abc")`)
}
```

在《Go程序设计语言》的错误处理策略中提到, 当调用函数出现 err 的时候, 我们不仅要返回出错的 err 信息, 还需要返回必要的关键信息, 比如函数调用的入参等
比如下面这个示例
```
doc,err := html.Parse(resp.body)
resp.body.Close()
if err != nil {
    return nil,fmt.Errorf("parseing %s as HTML: %v", url, err)
}
```
返回值创建了一个新的 err 对象, 将原始的 err 对象和入参信息追加进去了, 这样子出错的上下文信息就更加完整了, 但是这样子原始的 err 就会包装在自定义的 fmt.Errorf() 下, 对于后续我们对原始的 err 进行判断处理增加了一定的难度.

为了解决这类问题, Go 在 1.13 中增强了 errors 包, 引入了 wrap error 的概念, 并提供了 Unwrap/Is/As 的方法, 用于对 err 进行二次处理和识别.

简单的使用如下
定义一个 wrap error
```
err := errors.New("internal error")
wrapErr := fmt.Errorf("wrap err: %w", err)
```

Go 不支持 Wrap() 方法, 而是通过扩展 fmt 包的 Errorf 方法返来实现, 通过扩展占位符 %w 来接受 err 对象, 示例中的 wrapErr 就是一个 wrap err

简单介绍一下 Unwrap/Is/AS 等方法
```
func Unwrap(err error) error
func Is(err, target error) bool
func As(err error, target any) bool
```

errors.Unwrap
Unwrap 方法用于获取 wrap 之前的 err 信息, 不管之前的 err 是否被 wrap 过, 也仅只会返回上一次的 err
```
err := errors.New("internal error")
wrapErr := fmt.Errorf("wrap err: %w", err)
wrapErr2 := fmt.Errorf("wrap2 err: %w", wrapErr)

// 打印输出
fmt.Println(errors.Unwrap(wrapErr))

// 结果如下
[Running] go run "/Users/zhengzhilun/Documents/code/study/main.go"
wrap err: internal error
```

errors.Is
Is 方法, 用于判断目标 err 是不是包含在 wrap err 里面, 如果在 wrap 里, 返回 true, 这边需要注意的事: 不过之前的 err 被 wrap 多少次, 只要目标 err 在, 就返回 true
```
err := errors.New("internal error")
wrapErr := fmt.Errorf("wrap err: %w", err)
wrapErr2 := fmt.Errorf("wrap2 err: %w", wrapErr)
fmt.Println(errors.Is(wrapErr2, err))

// 输出结果
[Running] go run "/Users/zhengzhilun/Documents/code/study/main.go"
true
```

errors.As
As 方法, 用于提取指定类型的err, 从 err 树中, 找到第一个符合条件的 err, 并且设置, 否则返回 false.
```
err := errors.New("internal error")
wrapErr := fmt.Errorf("wrap err: %w", err)
var target *os.PathError
errExists := errors.As(wrapErr, &target)

fmt.Println(errExists)

// 输出结果
[Running] go run "/Users/zhengzhilun/Documents/code/study/main.go"
false
```

在 Go 1.20 中又扩展了 errors 包, 提供了 join 方法, 为了处理同时有多个 err 的情况
简单的例子
```
err1 := errors.New("err1")
err2 := errors.New("err2")
err3 := errors.New("err3")

err := errors.Join(err1, err2, err3)
fmt.Println(err)
fmt.Println(errors.Is(err, err1))
fmt.Println(errors.Is(err, err2))

// 输出结果
[Running] go run "/Users/zhengzhilun/Documents/code/study/main.go"
err1
err2
err3
true
true
```


