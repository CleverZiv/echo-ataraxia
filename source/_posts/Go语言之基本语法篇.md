---
title: Go语言之基本语法篇
tags:
  - Go 
categories:
  - Go
date: 2018-10-27
---
# 一、变量定义

```go
package main

import (
	"fmt"
	"math"
)

/**
在函数外也可以定义变量，但是此时不能使用":="定义变量
此时该变量是”包变量“，注意go中没有”全局变量“的说法
包变量在下面的函数中也是可以使用的
 */
 /*var aa = 6
 var ss = "kkk"
 var bb = true*/
 //将以上变量定义语句简化为：
 var(
	 aa = 6
     ss = "kkk"
     bb = true
 )

/**
定义空值变量
 */
func variableZeroValue(){
	//变量定义：最前面用 var 声明，然后变量名字在类型前面
	var a int
	var s string
	//fmt是一个函数库
	fmt.Printf("%d  %q\n",a,s)
}

/**
定义有初始值的变量
 */
func variableInitValue(){
	//注意这种赋值的格式
	var a, b int = 3, 4
	var s string = "abc"
	fmt.Println(a, b, s, aa)
}

/**
类型自动识别
 */
 func variableTypeDeduction(){
 	//不同类型也可以放在一个语句中进行定义
 	//如果规定了类型是不能写在一起的
	 var a, b, c, s = 3, 4, true, "def"
	 fmt.Println(a, b, c, s)
 }

 /**
 ":="用于定义变量，可以省略var
 推荐这种
  */
  func variableShorter(){
  	a, b, c, s := 3, 4, true, "def"
  	//而赋值时是不能使用":"的
  	b = 5
  	fmt.Println(a, b, c, s)
  }

  /**
  演示强制类型转换
  可以看到每一个值都需要进行显示类型转换
   */
   func triangle(){
   	var a, b int = 3, 4
   	var c int
   	c = int(math.Sqrt((float64(a*a + b*b))))
   	fmt.Println(c)
   }
func main() {
	fmt.Println("Hello World")
	variableZeroValue()
	variableInitValue()
	variableTypeDeduction()
	variableShorter()
	triangle()
}
```

# 二、內建变量类型

- bool, string
  布尔型和字符串类型
- (u)int, (u)int8, (u)int16, (u)int32, (u)int64, uintptr
  整数类型，加”u”表示无符号整数，否则为有符号整数；后面的数字表示长度，不规定长度的“int”就是根据操作系统来的。uintptr是指针,长度由操作系统决定
- byte, rune
  byte占8字节，rune类比于“char”，长度为32位。
- float32, float64, complex64. complex128
  浮点数类型，complex64和complex128表示复数。

# 三、强制类型转换

go语言中类型转换都是强制的，即都是显示的。

```go
/**
演示强制类型转换
可以看到每一个值都需要进行显示类型转换
 */
 func triangle(){
 	var a, b int = 3, 4
 	var c int
 	c = int(math.Sqrt((float64(a*a + b*b))))
 	fmt.Println(c)
 }
```



# 四、常量定义

```go
  /**
  常量定义演示：
  常量不用全部大写；
  常量类型数值可作为各种类型使用；
  常量一定要赋值。
   */
   func consts(){
   	const a, b = 3, 4
   	var c int
   	c = int(math.Sqrt(a*a + b*b))//常量不需要进行强制类型转换
   	fmt.Println(c)
}

   /**
   利用常量做枚举
    */
    func enums(){
    	const(
    		cpp = iota //iota表示自增值
    		java
    		python
    		golang
	)
 }
```

# 五、常用语句

## 5.1 if

```go
/**
if语句
 */
 func showif(){
   const filename = "abc.txt"
   /*//ioutil是一个库函数，用于读取文件
   //go语言中一个函数可以有多个返回值，按顺序接受即可
   contents, err := ioutil.ReadFile(filename)
   if err != nil {//nil就是null
      fmt.Println(err)
   }else{
      fmt.Printf("%s\n", contents)
   }*/
   //改良版：
   if contents, err := ioutil.ReadFile(filename); err != nil {
      fmt.Println(err)
   }else {
      fmt.Printf("%s\n", contents)
   }
 }
```

- if的条件里可以赋值，新建变量，该变量的作用域只在if语句中
- if中可以进行语句复合，用“;”分割。

## 5.2 switch

```go
/**
 switch语句
  */
  func showswitch(a, b int, op string) int {
  	var res int
	  switch op {
	  case "+":
	  	res = a + b
	  case "-":
	  	res = a - b
	  case "*":
	  	res = a * b
	  case "/":
	  	res = a / b
	  default:
		  panic("不支持的运算符" + op)
	  }
  	return res
  }
```

- switch会自动break，除非使用fallthrough
- switch后面可以没有表达式，在case中计算

## 5.3 for语句

```go
 /**
 for语句
  */
  func showfor1(){
  	sum := 0
  	for i := 0; i <= 100; i++ {
  		sum += i
}
  	fmt.Println(sum)
  }
  //省略初始条件，相当于while，go中没有while
  func showfor2(v int){
  	res := ""
  	for ; v > 0; v /= 2 {
  		//strconv.Itoa(int i)将int 转换为 string
  		res = strconv.Itoa(v%2) + res
}
  	fmt.Println(res)
  }

  //省略一切，就是一个死循环
  func showfor3(){
  	for {
  		fmt.Println("abc")
}
  }
```

# 六、函数定义

```go
/**
 switch语句
  */
  func showswitch(a, b int, op string) int {
  	var res int
	  switch op {
	  case "+":
	  	res = a + b
	  case "-":
	  	res = a - b
	  case "*":
	  	res = a * b
	  case "/":
	  	res = a / b
	  default:
		  panic("不支持的运算符" + op)
	  }
  	return res
  }
```

- 函数名在前，返回值类型在后
  前面提到Go中的函数可以返回多个值，那多个值类型不同怎么做呢？

```go
   /**
   函数定义
还可以为返回值建立变量名，具体这里不演示了
    */
    func div (a, b int) (int, int) {
    	return a / b, a % b
	}
```

当函数有多个返回值，但我只需要其中一个的时候，其他的返回值怎么处理呢？

```go
q, r := div(12,5)
fmt.Println(q)
```



这样是不可以的，因为go中定义的变量一定要使用到，否则就是编译错误。此处的变量r没有使用到，因此不行。怎么解决呢？

```go
q, _ := div(12,5)
fmt.Println(q)
```



# 七、面向函数编程

GO语言是一种面向函数编程的语言，体现在它的函数不仅可以用在函数体中，还可以用作函数的参数，返回值等。

```go
   /**
   函数定义
   还可以为返回值建立变量名，具体这里不演示了
    */
    func div (a, b int) (int, int) {
    	return a / b, a % b
	}

    /**
    函数作为另一函数的参数
     */
     func apply(op func(int, int) int, a, b int) int {
     	//展示出调用的函数的名字
     	p := reflect.ValueOf(op).Pointer()
     	opName := runtime.FuncForPC(p).Name()
     	fmt.Printf("Calling function %s with args" + "(%d, %d)\n", opName, a, b)

     	return op(a, b)
	 }
     func pow(a, b int) int {
     	return int(math.Pow(float64(a), float64(b)))
	 }

	 func main() {
	fmt.Println(apply(pow, 2,4))
}
```



甚至直接定义在函数声明中，定义一个匿名函数

```
fmt.Println(apply(func(a int, b int) int {
	return int(math.Pow(float64(a), float64(b)))
}, 2,4))
```



Go中的函数没有函数重载，函数重写等，但是有一个可变参数列表的用法。

```go
func main() {
	fmt.Println(sum(1,2,3,4,5))
}

     /**
     可变参数列表
      */
      func sum(nums ...int) int {
      	s := 0
      	for i := range nums {
      		s += nums[i]
		}
      	return s
	  }
```



# 八、指针

- 指针不能运算

- 参数传递

  Go语言只有值传递一种方式，但可以通过指针的方式实现引用传递的功能

```go
  func main() {
  	var a int = 2
  	//定义指针 *int; & 表示取a的地址，即可以理解为指针就是一个变量的地址，即pa保存的是a的地址
  	var pa *int = &a
  	//*pa就是取得pa保存的地址对应的变量的值
  	*pa = 3
  	fmt.Println(a)
  	fmt.Println(*pa)
  }
```

指针实现交换两数的函数
```go
  func swap(a, b *int){
     	//a，b两指针指针互换，假设a和b不是指针，由于go中是值传递，所以函数内部
     	//即使进行了交换，也无法在外部实现真正的交换
     	*a, *b = *b, *a
  }
```
<!-- more -->