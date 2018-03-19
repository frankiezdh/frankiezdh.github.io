---
title: golang中接口与实现
date: 2018-03-13 20:41:11
tags: [golang]
---


golang设计时偏重组合而不是继承或是实现，所以golang没有java中常见的extended和implement关键字。golang实际上比java更灵活，对于接口的实现是一种更“自然而然”的方式——如果某个类型实现了一个接口的所有方法，则默认这个类型实现了这个接口，不用显式的声明。 

```golang
package main

import "fmt"

type readable interface {
	read() string
}

type writable interface {
	write(string) string
}

type text struct {
}

type excel struct {
}

func (this *text) read() string {
	return "text read"
}

func (this *text) write(input string) string  {
	return "text write"
}

func (this *excel) read() string {
	return "excel read"
}

func (this *excel) write(input string) string  {
	return "excel write"
}

func create(typeOf string) (readable, writable)  {
	switch typeOf {
	case "t": return &text{}, &text{}
	case "e": return &excel{}, &excel{}
	}
	return nil, nil
}

func main()  {
	t1, t2 := create("t")
	fmt.Println(t1.read(), t2.write(""))
	e1, e2 := create("e")
	fmt.Println(e1.read(), e2.write(""))
}
```