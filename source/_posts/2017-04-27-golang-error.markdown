---
layout: post
title: "golang-error处理"
date: 2017-04-27 00:14:45 +0800
comments: true
categories: Golang
---

## Golang error

### 什么是 Golang error

*	Golang 的 error 是内置的类型,无论是标准包还是各大开源项目中都包含着各种对 error 的处理，	[golang-error-blog](https://blog.golang.org/error-handling-and-go) 中有详细的例子。
*	Golang 的设计之一就是要在调用处处理 error, 而未采用其它语言(类似python,java) 的try+catch 模式，这样会使得逻辑更加清晰，但是代码中会充斥着各种类似如下片段
``` go
	a, err := doSomeThing()
	if err != nil {
        log.fatal(err)
	}
```	

### 以 error 为线索优化项目结构

我们现在需要实现一个包(Golang package)，需要实现两个整数的加减乘除运算方法。

``` go
	type Operation1 struct {
        Number1 int
        Number2 int
	}
		
	func (opt *Operation1) Add() int {
		return opt.Number1 + opt.Number2
	}
	func (opt *Operation1) Sub() int {
		return opt.Number1 - opt.Number2
	}
	func (opt *Operation1) Multiply() int {
		return opt.Number1 * opt.Number2
	}
	func (opt *Operation1) Divide() int {
		return opt.Number1 / opt.Number2
	}
```

上面这个包的函数是实现了加减乘除的运算，那么有哪些问题呢？ 

- 除数是0的情况 Divide 方法没有处理,程序会报 ```panic: runtime error: integer divide by zero``` ,会使得调用者的程序 panic,如下：

``` go 
	opt := Operation1{}
	opt.Number1 = 1
	opt.Number2 = 0
	opt.Divide()		
```
 
-  Number1 和 Number2 为可导出类型的，即可以在包的外部进行赋值运算，其实是可以通过 New 方法，在初始化以外都避免对这两个要运算的值的修改
	
于是，我们修改如下

``` go 
	import (
		"errors"
	)

	type Operation2 struct {
		number1 int
		number2 int
	}

	func NewOperation2(a, b int) *Operation2 {
		return &Operation2{
			number1: a,
			number2: b,
		}
	}

	func (opt *Operation2) Add() int {
		return opt.number1 + opt.number2
	}

	func (opt *Operation2) Sub() int {
		return opt.number1 - opt.number2
	}

	func (opt *Operation2) Multiply() int {
		return opt.number1 * opt.number2
	}

	func (opt *Operation2) Divide() (int, error) {
		if opt.number2 == 0 {
			err := errors.New("integer divide by zero")
			return 0, err
		}
		return opt.number1 / opt.number2, nil
	}

```

这样的话关于除法部分调用如下:

```go
	opt := NewOperation2(1, 0)
	result, err := opt.Divide()
	if err != nil {
		log.Errof(err.Error())
	}
```
这样的例子是可以实现对于除法的特殊处理，实际上对于业务逻辑来说这样的实现是完全没有问题的，从包的使用的角度来谈，这里是有一些问题的

- 加减乘除应该是一样的返回结果格式，目前只有除法是返回结果和 error 信息
- 当然可以通过给加法、减法、乘法加上相关的 error 返回，及时加上后在调用方也会出现对于 err 是否为 nil 的逻辑判断
- 如果我们限制减法结果不能为负数，即 如果 `number1` 小于 `number2`,我门需要返回一个 `减数不能小于被减数的 error `,在某些情况下是需要打印log,某些情况下是不需要(举这个例子是为了说明业务逻辑中对于error的容忍程度是分场景的，如同样是查 redis，有的业务认为查不到需要报错，有的业务查不到需要去查 db，不报错)。
关于以上几点，其实参考标准包，可以把 `error` 放在 `struct` 中；提供一个 Err() 方法来返回 error，代码调整如下:

``` go
import (
	"errors"
)

type Operation3 struct {
	number1 int
	number2 int
	err     error
}

var (
	ErrDivideByZero = errors.New("divide by zero")
	ErrSubLess      = errors.New("minuend must greater or equal than reduction")
)

func NewOperation3(a, b int) *Operation3 {
	var err error
	if a < b {
		err = ErrSubLess
	}
	return &Operation3{
		number1: a,
		number2: b,
		err: err
	}
}

func (opt *Operation3) Err() error {
	return opt.err
}

func (opt *Operation3) Add() int {
	return opt.number1 + opt.number2
}

func (opt *Operation3) Sub() int {
	return opt.number1 - opt.number2
}

func (opt *Operation3) Multiply() int {
	return opt.number1 * opt.number2
}

func (opt *Operation3) Divide() int {
	if opt.number2 == 0 {
		opt.err = ErrDivideByZero
		return 0
	}
	return opt.number1 / opt.number2
}
	
```

调用的这个包的地方可以用如下方式调用:
``` go
	opt := NewOperation3(1, 0)
	if result := opt.Divide();opt.Err() != nil {
		log.Errorf(opt.Err().Error())
	}
	
```

### 总结

通过调整后的代码相比于最初，有以下改进：

- 除数等于0时会返回 `error` 信息给被调用者
- 调用者不会修改内部数据 `number1` 和 `number2`
- `error` 在结构体重可以通过 `opt.Err()` 获取，可以使得 `New` 及其他方法只关注于本身需要返回的
- 便于后续包的维护，比如 `新增乘法结果最大为1000，超过1000需要返回 error`, 最初版本的需要将乘法的返回类型由 `int` 改为 `(int, error)`,而调整结构后只需要修改 乘法的逻辑，调用者的逻辑都不需要修改,即:
``` go

func (opt *Operation3) Multiply() int {
	result := opt.number1 * opt.number2
	if result > 1000 {
		opt.err = errors.New("max result is 1000")
		return 0
	}
	return result
}
```

- 调用者可以通过 `opt.Err()` 方法来判断是否为某一特定的 `error` 类型,如:
```go
	opt := NewOperation3(1, 0)
	result := opt.Divide()
	if opt.Err() != nil && opt.Err() != ErrSubLess {
		log.Errof(opt.Err.Error())
	}
	...
```
-  本文的所有代码均放在 [golang error 示例](https://github.com/jerrybean/golang-examples/tree/master/golang_errors)
-  欢迎大家交流拍砖
