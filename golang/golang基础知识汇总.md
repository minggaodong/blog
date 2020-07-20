# golang 基础知识汇总

## 包和文件
### 创建包规则
- 包以目录的形式组织，包名就是目录名，同一个目录下的 go 文件属于同一个包。
- main 包用来生成可执行文件，main 包里只有一个 main 函数作为入口函数。
- 包里面需要导出的变量或者函数，首字母必须大写。

### 导入方式
通过 import 关键字导入
```
import     "yuhen/test"      // 默认模式: test.A 
import  M  "yuhen/test"      // 包重命名: M.A 
import  .  "yuhen/test"      // 简便模式: A 
import  _  "yuhen/test"      // ⾮非导⼊入模式: 仅让该包执⾏行init()初始化
```

### init() 函数
go 文件可以定义 init() 函数，init() 在 main() 执行前被系统默认执行，init() 无法被用户程序调用。

init() 执行顺序：全局变量初始化 > 依赖包的init函数 > main包的init函数 > main.main()。

一个 go 文件都可以定义多个 init() ，同一个文件内部顺序执行，不同文件根据文件名顺序执行。

init() 内用 go 关键字创建新 goroutine，新 goroutine 只有在进入 main() 之后才被执行到。

## 常量
常量值必须是编译期可确定的数字、字符串、布尔值，常量定义时可以省略类型，由编译器自动推导。

iota 可理解为 const 语句块中的行索引，可以放在任意一行使用，接下来的行可以省略赋值，默认都是去的行索引
```
const defaultMultipartMemory = 32 << 20 // 32 MB
const s = "Hello, World!"
const (
    MSG_KAFKA = iota  // 0=kafka
    MSG_HTTP          // 1=http
    MSG_BOTH          // 2=both
)
```

## 变量
使用关键字 var 定义变量，自动初始化为零值；如果提供初始化值，可省略变量类型，由编译器自动推导。
```
var x int 
var f float32 = 1.6 
var s = "abc  // 省略变量类型，编译器根据初始化值自动推导为string
```
### 短变量
短变量定义可以省略var关键字，但是只能用于局部变量；短变量定义多个变量时，至少保证一个变量是新的。
```
func main() {
    s := "abc"           // 用":="可以省略关键字var
    s, t := "ab", "ab"   // t是新的变量，这里可以使用":="
}
```

### 多变量赋值
多变量赋值时，先计算所有相关值，然后从左到右依次赋值
```
data, i := [3]int{0, 1, 2}, 0 
i, data[i] = 2, 100                // 这里100赋给了data[0]，而不是data[2]
```

### 同名屏蔽
代码块内部的变量与外部变量同名时，会屏蔽外层同名变量。

## 类型
### nil
nil 并非其他语言中的 NULL 值，nil 可以代表下面这些类型的零值（指针，slice，map，channel，interface，function），不同类型的 nil 值内存占用不一样。

不同类型的 nil 值是不能进行比较的，可以对一个 nil 值的 slice 和 map 进行 range 访问，并不会 panic。

### 引用类型
引用类型的赋值和传参都是浅拷贝，底层都管理指针指向真正数据。

引⽤类型包括 slice、map、channel 和 interface，它们有复杂的内部结构，在创建时，除了申请内存外，还需要初始化相关属性。

### make 和 new
new 只分配内存，不初始化对象；make 除了分配内存，还会对内存进行初始化，map 和 channel 需要复杂的初始化操作。

new 返回的是内存指针，make 返回的是对象值。

make 只能用于创建 slice，map，channel 三种类型；new 可以创建任何类型，new 创建的 map 和 slice 都是一个 nil 指针(零值)，nil 的 map 不能直接使用，会导致 panic。
```
v := make([]int, 100, 100)    // 生成一个元素个数为100，内存容量也为100的切片
v := make([]int, 100)         // 省略第三个参数，默认都是100
m := make(map[string]int, 5)  // 生成一个键string值int类型的字典，长度：5，map的第三个cap参数没有意义
c := make(chan int， 5)       // 生成一个int类型的通道，长度：5

// 使用new
p := new([]int)
*p = append(*p, 1)           // 不会panic，append会初始化对象
p := new(map[int]int)
(*p)[10] = 10               // map未初始化，会导致panic
```

### 接口类型
接口是一种基本类型，是方法的集合，任何方法只要实现了与之对应的全部方法，默认就表示它实现了这个接口，无须显示声明。

接口中不能出现数据字段，但是可出现接口类型字段；接口的名字一般以er结尾。

没有任何方法的接口称作空接口，所有类型都看做是空接口的实现，任何类型都可以赋值给一个空接口，但是反过来不行。

空接口类型通过断言可以推断出原有类型，断言的语法：value,ok := x.(T)，T可以是任意类型，也可以是接口类型。

```
// 定义一个接口，任意方法只要实现这个接口的全部函数，都可以看做
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
    Write([]byte) (int, error)
}

// 定义一个空接口
var str interface{}
str = "hello"        // 任意类型都是空接口的实现，所以任意类型可以赋值给一个空接口


// 类型断言
value, ok := a.(string)
if ok {
    value += "123"
}

// 类型断言，必须在switch内使用
switch a.(type) {
    case string:
        // 一些操作
}
```

### 函数类型
可以把函数当成一种变量，用 type 去定义它，那么这个函数就可以作为值进行传递，相当于 C 语言的函数指针。
```
type HandlerFunc func(*Context) int
type RouteInfo struct {
	Method      string
	Path        string
	HandlerFunc HandlerFunc
}
```

### 结构类型
结构体的成员如果要让其他包访问(导出)，变量首字母要大写。

结构体的成员变量如果都是可以比较的，则结构体也可以比较，也可以作为 map 的 key。

#### 结构体匿名属性
允许结构体定义不带变量名的匿名属性，访问时通过类型名访问，匿名属性同类型只允许一个。

匿名属性如果是一个结构体类型，则表示继承。
```
type Camera struct {
	name string
}
type Phone struct{
    name string
}

func (p *Phone) Call()  {
	fmt.Println("打电话")
}
func (c *Camera) TakePicture()  {
	fmt.Println("拍照片")
}

// 继承关系
type CameraPhone struct {
	Camera
	Phone
}

cp := CameraPhone {
    Camera: Camera {
        name: "camera",
    },
    Phone: Phone {
        name: "phone",
    },
}
```

### 类型转换
golang 支持多种类型转换方式
#### 强制转换
用 “类型(xx)” 的方式强制转换，适用于 int、float、string, byte 这种基础数据类型。
```
// (int32/uint32/int64/uint64/float32/float64)之间可以互转
var a int32
var b uint64 = uint64(a)
var c float32 = float32(a)
var d float64 = float64(c)

// string和[]byte之间可以互转，注意此时底层发生了copy
s := []byte("hello")  // string转byte
str := string(s)      // []byte转string
```
#### unsafe 包
指针的强制类型转换需要用到 unsafe 包中的函数实现，任何指针类型可以在 unsafe.Pointer之间互转，可能不安全。
```
import "unsafe"
var a int =10
var b *int =&a
var c *int64 = (*int64)(unsafe.Pointer(b))
```

#### 断言
针对 interface 类型，可以执行 val.(string) 的方式进行类型断言，返回值为类型转换后的值以及类型断言的结果。

```
// 类型断言
value, ok := a.(string)
if ok {
    value += "123"
}

// 类型断言，必须在switch内使用
switch a.(type) {
    case string:
        // 一些操作
}
```

#### 反射
反射就是程序在运行时，获取变量的类型，如果是结构体，还可以获取它的成员信息(变量名+类型，方法)。

reflect 包用来实现反射，通过 TypeOf() 获取类型信息，通过 ValueOf() 获取值信息。
```
package main

import (
    "fmt"
    "reflect"
)

type Student struct {
    Name    string
    Age     int
    Sex     uint // 0-女性，1-男性
    Address string
}
func (stu Student) Print() {
}

func main() {
    stu := Student{"李四", 18, 1, "中国北京市天安门10000号"}
    
    // 获取类型和值
    st1 := reflect.TypeOf(stu)
    sv1 := reflect.ValueOf(stu)
    fmt.Println(st1, "====", sv1) // 输出：main.Student ==== {李四 18 1 中国北京市天安门10000号}
    
    // 获取结构体名称
    fmt.Println(st1.Name())      // 输出: Student
    
    // 判断Kind类型
    fmt.Println(st1.Kind() == reflect.Struct)     // True
         
    // 获取结构体中每个字段的值
    for i := 0 ; i < st1.NumField() ; i++ {
        fieldName := st1.Field(i).Name        // 取字段名
        fieldType := st1.Field(i).Type
        fieldVal := sv1.Field(i).Interface()  // 取值
        fmt.Printf("字段名：%v，类型：%v，值：%v\n", fieldName, fieldType, fieldVal)   // 输出之一：字段名：Name，类型：string，值：李四 
    }
    
    // 遍历所有方法的名称和类型
    for i := 0 ; i < st1.NumMethod() ; i++ {
        method := st1.Method(i)
        fmt.Println(method.Name, "====", method.Type)    // 输出：Print ==== func(main.Student)
    }
    
    // 通过反射执行方法
    m1 := sv1.MethodByName("Print")
    m1.Call(nil)// 不带参数和返回值
}
```
