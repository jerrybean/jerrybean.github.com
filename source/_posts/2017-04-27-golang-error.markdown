---
layout: post
title: "golang-error处理"
date: 2017-04-27 00:14:45 +0800
comments: true
categories: Golang
---

## Golang error

### 什么是 Golang error

1.	Golang 的 error 是内置的类型,无论是标准包还是各大开源项目中都包含着各种对 error 的处理，	[golang-error-blog](https://blog.golang.org/error-handling-and-go) 中有详细的例子。
2.	Golang 的设计之一就是要在调用处处理 error, 而未采用其它语言(类似python,java) 的try+catch 模式，这样会使得逻辑更加清晰，但是代码中会充斥着各种类似如下片段
```
		a, err := doSomeThing()
		if err != nil {
			log.fatal(err)
		}
```

