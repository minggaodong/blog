# golang 基础知识汇总3

## 运算符
- golang 不支持运算符重载。
- 只支持 i++/i--，不支持 ++i/--i，且 i++/i-- 只能作为独立语句，不能作为表达式，j := i++ 编译报错。
- 没有 "~"，取反运算用 "^"。

## for循环
- golang没有while操作，只有for。
- 单核的情况下，for {} 会导致 cpu 资源独占而其他 goroutine 饿死，可以通过阻塞的方法避免 cpu 占用，改为 select {}。
```
s := "abc"
// 常⻅见的 for 循环，⽀支持初始化语句。
for i, n := 0, len(s); i < n; i++ {       
    println(s[i]) 
}

n := len(s)
for n > 0 {
    n--
}

// 死循环
for {
}
```

## range
- 类似迭代器操作，返回 (索引, 值) 或 (键, 值)。
- 可忽略不想要的第二个返回值，或⽤用 "_" 这个特殊变量。
- range 会复制对象。
- range 遍历时，循环次数在循环开始前就已经确定，循环内改变切片的长度，不影响循环次数。
- 循环条件一开始就计算出来了，循环体内改变chan的大小，不会影响循环次数
```
v := make([]int, 10)
for i := range v { 
    v = append(v, i)
}
```

## 函数
- 不支持嵌套，重载，默认参数。
- 支持不定长变参，支持多返回值，支持命名返回参数，命令返回参数可以看做是局部变量，由 return 隐式返回。
- 多个参数返回时，要么都使用命名返回，要么都不使用，不能掺杂。

### 可变参数
- 变参本质上就是slice，只能有一个，且必须是最后一个，变参类型前加上"..."。
- 使用slice对象做变参时，必须使用"..."展开。
```
func test(s string, n ...int) string {    
    var x int    
    for _, i := range n {        
        x += i    
    }
    return fmt.Sprintf(s, x) 
}

func main() {    
    s := []int{1, 2, 3}    
    println(test("sum: %d", s...))
}
```

## 延迟调用
- 函数的延迟调用是指在函数调用结束前，执行由 defer 定义的延迟语句,多个 defer 的执行是栈的顺序。
- 延迟调用的顺序：return 语句 > defer3 > defer2 > defer1
- 触发 panic 时，先执行触发 panic 处之前的 defer 语句，再执行 panic，可以用于异常发生时关闭或记录一些关键信息。
```
package main
import ("fmt")
func main() {
         defer fmt.Printf("defer 1\n")     // 会后被执行
         defer fmt.Printf("defer 2\n")     // 会先被执行
         a := 0
        fmt.Printf("%d\n", 2/a)
        defer fmt.Printf("defer 2\n")    // 不会被执行
}
```

## 异常处理
- 异常包括 panic 显式产生的异常，以及段错误，除 0 错误，Fatal 日志等产生的系统异常。
- recover() 可以捕获各种异常,可以避免程序 crash，包括段错误产生异常也可以捕获。
- recover() 必须放在 defer 定义的函数中调用。
```
func test() {
	defer func() {        
		if err := recover(); err != nil {            
			println(err.(string))           // 将 interface{} 转型为具体类型。        
		}    
	}()
    panic("panic error!")
}
```


## 匿名函数
- 匿名函数可以像普通变量一样传递和使用。
```
fn := func() { 
    println("Hello, World!") 
} 
fn()
```

## 闭包
golang 中匿名函数作为变量传递时，函数在定义的时候可以引用一些局部变量，而函数在其他地方执行的时候，仍然可以像访问全局变量一样访问局部变量，这个就叫做函数的闭包。

闭包是匿名函数与匿名函数所引用环境的组合。匿名函数有动态创建的特性，该特性使得匿名函数不用通过参数传递的方式，就可以直接引用外部的变量。

```
package main

import (
	"fmt"
)

func fibonacci() func() int {
	b0 := 0
	b1 := 1
   // 返回一个闭包函数，这个闭包函数对b0和b1两个变量随时可以访问，b0和b1就好像全局变量一样
	return func() int {
		tmp := b0 + b1
		b0 = b1
		b1 = tmp
		return b1
	}
}

func main() {
	myFibonacci := fibonacci()
	for i := 1; i <= 5; i++ {
		fmt.Println(myFibonacci())
	}
}

1
2
3
5
8

示例2：因为goroutine启动的慢，所以i的值已经变成了5，所有的goroutine都打印5
func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			fmt.Println(i)
			wg.Done()
		}()
	}
	wg.Wait()
}
输出结果：
5
5
5
5
5
```
