---
layout: post
title: "string&iota"
date: 2018-12-16 14:30
categories: go
---

## string
go 中的字符串都是采用 UTF-8 字符集编码。字符串是用一对双引号（""）或反引号（``）括起来定义，它的类型是 string

##### string 类型的声明以及初始化
``` go
package main

import (
	"fmt"
)

var a string                   //声明 string 类型的一般方法
var b = "xxx"                  //自动推导为 string 类型
var c string = "xxx"           //声明并初始化
var e, f string = "aaa", "xxx" //同时声明多个变量并初始化

var ( //分组形式声明
	g string
	h string
	i string = "xxx"
)

func main() {
	j := "xxx" //简短形式声明并初始化，只能在函数内部使用这种形式
	k, l := "kk", "xx"
	fmt.Println(a, b, c, e, f, g, h, i, j, k, l)
}
```

##### string 变量的值不能被直接改变
``` go
var str string = "xxxxx"
str[0] = 'c' //error cannot assign to str[0]
```

##### 可以通过间接方式改变 string 的值
将字符串 s 转换为 []byte 类型，再转换回 string 类型存储到***另一个*** string 变量中
``` go
package main

import (
	"fmt"
)

func main() {
	str := "xxxx"
	c := []byte(str)  //先将字符串转化为 []byte 类型
	c[0] = 'k'        //修改元素值
	str2 := string(c) //再将 c 转换为 string 类型并存储到变量 str2 中
	fmt.Println(str2) //输出 kxxx
}
```

通过***切片***的方式改变 string 变量的值
``` go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	str := "xxxx"

	//获取 str 的类型
	fmt.Printf("%T\n", str) //输出 string

	str = str[:1] + "s" + str[2:] //通过切片的形式修改 string 的值

	fmt.Println(str) //输出 xsxx
	//获取 str 的类型
	fmt.Println(reflect.TypeOf(str))//输出 string， 即 str 本身还是 string 类型
}
```

## iota
iota 默认开始值是 0，const enum 中每增加一行加 1. 每遇到一个 const 关键字，iota 就会重置为 0, 同一行的 iota 值相同
``` go
package main

import (
	"fmt"
)

const (
	x = iota // x = 0
	y = iota // y = 1
	w        // w = 2
	//常量声明省略值时， 默认和之前一个值的字面相同。 这里隐式地说 w = iota
	//因此 w = 2。 其实上面 y 可同样不用 "= iota"
)

const v = iota // 每遇到一个 const 关键字， iota 就会重置， 此时 v = 0

const (
	h, i, j = iota, iota, iota //h = 0, i = 0, j = 0 iota 在同一行值相同
)

const (
	a       = iota //a=0
	b       = "B"
	c       = iota             //c=2
	d, e, f = iota, iota, iota //d=3,e=3,f=3
	g       = iota             //g = 4
)

func main() {
	fmt.Println(x, y, w)             //0 1 2
	fmt.Println(v)                   //0
	fmt.Println(h, i, j)             // 0 0 0
	fmt.Println(a, b, c, d, e, f, g) // 0 B 2 3 3 3 4
}
```

