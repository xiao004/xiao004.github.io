---
layout: post
title: "interface"
date: 2018-12-20 13:10:00 +0800
categories: go
---

interface 类型定义了一组方法，如果某个对象实现了某个接口的所有方法(实现 interface 中的一个 method 即函数名，形参类型，个数，顺序，返回参数类型，个数，顺序对应)，则此对象就实现了此接口。interface 可以被任意的对象实现。一个对象可以实现任意多个 interface

##### interface 的声明及实现
``` go
package main

import (
	"fmt"
)

type Human struct {
	name  string
	age   int
	phone string
}

type Student struct {
	Human  //匿名字段
	school string
	loan   float32
}

type Employee struct {
	Human   //匿名字段
	company string
	money   float32
}

//定义 interface
type Men interface {
	SayHi()
	Sing(lyrics string)
	Guzzle(beerStein string)
}

type YoungChao interface {
	SayHi()
	Sing(song string)
	BorrowMoney(amount float32)
}

type ElderlyGent interface {
	SayHi()
	Sing(song string)
	SpendSalary(amount float32)
}

//Human 对象实现 Sayhi 方法
func (h *Human) SayHi() {
	fmt.Println("hi, i am a Human")
}

//Human 对象实现 Sing 方法
func (h *Human) Sing(lyrics string) {
	//形参的名字没有任何影响, 虽然三个 interface 中的 Sing mehod 形参名字不同， 但是他们的类型相同
	//所以 Human 实现了一个 Sing 等于三个 interface 的 Sing method 都实现了
	fmt.Println("lalalala")
}

//Human 对象实现 Guzzle 方法
func (h *Human) Guzzle(beerStein string) {
	fmt.Println("Guzzle")
}

func (h *Human) BorrowMoney(amount float32) {
	fmt.Println("BorrowMoney")
}

//Student 覆盖继承自 Human 中的 SayHi method
func (s *Student) SayHi() {
	fmt.Println("hi, i am a student")
}

func main() {
	var cnt Men
	var gel YoungChao

    //如果我们定义了一个interface的变量，那么这个变量里面可以存实现这个interface的任意类型的对象
	cnt = new(Human)
	gel = new(Human)

	cnt.SayHi()     //hi, i am a Human
	gel.Sing("xxx") //lalalala

	//Student 含有匿名字段 Human, 所以能继承 Human 中的 method， 即 Human 实现了的 interface 在 Student 中也能用
	cnt = new(Student)
	//Student 中覆盖了继承自 Human 中的 SayHi method
	cnt.SayHi() //hi, i am a student

}
```
**注意**<br>
若多个 interface 中包含同一个 method（即方法名，形参，返回参数全部一致），则某个类型实现该 method 一次即等于实现了所有 interface 中的该方法<br>
匿名字段的 method 可以继承，命名字段不可。即类型 x 包含了匿名字段 y，则 y 实现的方法 x 也具有。若 x 包含了命名字段 y，则 y 中的 method 不能直接被 x 类型对象调用，只能由 x 对象中的 y 类型对象间接调用<br>
继承自匿名字段的 method 可以被覆盖<br>
如果我们定义了一个 interface 的变量，那么这个变量里面可以存实现这个 interface 的任意类型的对象。由这些特性可知，***go 可以实现类似于 c++ 中的多态***<br>

