# golang 基础知识汇总2

## 数组
数组是值类型，数组的赋值和传参会复制整个数组。

数组长度必须是常量，且是类型的组成部分；[2]int 和 [3]int 是不同的数据类型。数组 len() 和 cap() 返回一样。
```
a := [3]int{1, 2}              // 未初始化元素值为 0
b := [...]int{1, 2, 3, 4}      // 通过初始化值确定数组⻓长度
```

## 切片
切片是一个动态数组，底层维护一个数组指针，切片是值传递，深拷贝时采用 copy()。

通过切片创建的新切片，共享底层数组，如：s1 := data[8:] s2 := data[9:]，对s1的内容修改，会导致s2的内容也发生变化。

当使用 append() 添加元素超过 cap 的大小时，底层会分配一个2倍 cap 大小的新数组，并拷贝数据，slice 内部数组指针指向新数组。
```
// 切片底层结构
type SliceHeader struct {
    Data uintptr
    Len  int
    Cap  int
}

// 切片创建
var a []int            // 创建一个nil切片
a := []int{1, 2, 3}   // 通过字面量创建，长度和容量由初始化值决定
a := make([]int, 3)   // 通过make创建，len和cap长度都是3

// 切片删除元素(只是修改了len的大小，底层数组并未改变)
a = a[0:0]         // 清空数组元素 
a = a[1:]          // 删除开头1个元素
a = a[N:]          // 删除开头N个元素
a = a[:len(a)-1]   // 删除尾部1个元素
a = a[:len(a)-N]   // 删除尾部N个元素

// 遍历切片
for index, _ := range a {
    a[index] += 1
}
for index, value := range a {
    value += 1     // 这里的value是一份拷，对value的修改不会改变切片a的值 
}
```

## string
string 是一个结构体，包含了字节数组指针和长度；string 的拷贝是浅拷贝，只拷贝字符数组的指针。

string 底层数组为只读，可以⽤下标访问某字节如 s[i]；不能⽤下标获取字节元素的指针如 &s[i]非法。

string 支持"+="操作，"+="操作的本质是将 string 对象底层指向一个新创建的 byte 数组，并拷贝数据，会带来GC开销，使用 bytes.Buffer 进行字符串的累加比起+=要高效的多。

string 可以直接比较，而 []byte 不可以，所以 []byte 不可以当 map 的 key 值。

string 使用 utf8 存储，ASCⅡ码字符占一个字节，非ASCⅡ码字符占 1~4 个字节不等，用 rune 类型表示。

通过for循环下标遍历得到的是字节，而通过for range遍历得到的是字符，len()返回的是字符串的字节数，而不是字符数。

```
// 定义字符串
str := "abc你d"
str1 := str[0:3]     // str1="abc"
str2 := "hello" + " world"

// 中文字符'你'占用了三个字节，abcd各占用一个字节：for遍历的是字节，for range遍历的是字符
for i, c := range str {
    fmt.Printf("%d:%c\n", i, c)
}
fmt.Println(len(str)))
 0:a
1:b
2:c
3:你
6:d
7

// string转整数和浮点数
int32, err := strconv.ParseInt(string, 10, 32)  // 第2个参数代表10进制，第3个参数对应int32
uint64, err := strconv.ParseUInt(string, 10, 64)  // 第2个参数代表10进制，第3个参数对应int64
float32,err := strconv.ParseFloat(string, 32)
float64,err := strconv.ParseFloat(string, 64)

// string和float的互相转换
string := strconv.FormatInt(int64(int), 10)
string := strconv.FormatUInt(uint64, 10)
string := strconv.FormatFloat(float32,'E',-1,32) // 'E' (-d.ddddE±dd，十进制指数)
string := strconv.FormatFloat(float64,'E',-1,64) // 'E' (-d.ddddE±dd，十进制指数)

// string和int的互转
int, err := strconv.Atoi(string)
string := strconv.Itoa(int)
```

## map
map 为引用类型，底层通过哈希表实现，所以是无序的，并且每次 for range 遍历时是随机 key 出现。

通过 key 索引到的值是一个临时复制，对其修改没有意义，除非 value 是对象指针，map 中的 value 虽然无法被修改，但是可以替换。

map 引用一个不存在的 key 值时，不会发生异常，而是返回一个默认值。

内置函数 delete() 可以删除 map 的元素，但是不会释放底层内存，只有将 map 置为 nil 才可以回收。

```
// map的声明
var m map[int]bool    // 此时m是一个nil值
m[0] = false          // 对一个nil值赋值，会panic掉
a, ok := m[0]         // a=0,ok=false；不会panic

// map的定义
m := make(map[int]bool)
m := map[int]bool {}
m := map[int]bool {1:false}

// map的增删
m[0] = false        // 新增
delete(m, 1)        // 删除，如果key不存在则啥也不干，也不会panic
m[0] = true         // 更新

// 三种查询
val := m[0]
val, ok := m[0]
_, ok := m[0]

// 遍历
for k, v := range m { 
    fmt.Println(k, v)
}
```


## 通道
### CSP 并发模型
CSP(Communicating Sequential Process) 模型是上个世纪七十年代提出的，用于描述两个独立的并发实体通过共享的管道进行通信的并发模型。

golang 的并发模型是基于 CSP 来设计的，多个 goroutine 之间通过 chan 来实现共享通信。

### chan特性
- 使用 "<-" 对 chan 进行读写，发送数据时对端必须有变量，接收数据时对端可以没有变量。
- 读写一个 nil 的 chan，会被阻塞住，close 一个 nil 或者已经 close 的 chan，会 panic。
- 对 close 的 chan 写，会 panic，对 close 的 chan 读，在将缓存数据读完后，每次读取第二个返回值都为 false (永不阻塞)。
- chan 被 close 掉时，等待队列中所有的 goroutine 都会被唤醒。
- chan 中传递的都是数据的拷贝，可能会影响性能，可以存储指针，但是指针又容易引起 data race。

### chan底层原理
chan 其实就是一个优化过的阻塞队列，每秒支持200万次读写操作，性能很高。

chan 的读写都需要互斥锁来同步，chan 底层使用循环链表实现，通过 sendnx 和 recvnx 两个变量保存读写位置。

goroutine 对 chan 读写阻塞时，被放入到发送/接收等待队列中，在 chan 满足条件时再被唤醒。

同步模式下，发送或者接收时从等待队列中寻找一个合作者，直接交换数据；如果找不到则将自己封装成SudoG，放入等待队列。

异步模式下，发送时从缓冲区找空槽，如果有直接拷贝数据，并唤醒等待接收队列，否则排队等待；接收时如果有缓冲项，拷贝数据，否则排队阻塞。

## select
select 为 golang 提供了多路 IO 复用机制，每个 case 必须是一个通信操作（default除外），要么是发送要么是接收。
- 如果任意某个通信可以执行，它就执行，其他被忽略
- 如果多个case都可以执行，随机公平的选取一个执行，其他不会被执行。
- 如果所有case都不可通信，则执行default语句，如果没有定义default语句，则select将阻塞，等待有case可以通信时唤醒。
- select {} 空语句，会阻塞当前 goroutine。
```
// 定义一个chan
c := make(chan int)     // 定义一个阻塞队列
c := make(chan int, 10) // 定义一个长度为10的缓存队列

// 写数据
c <- 1
close(c)
close(c) // close一个已关闭的chan，会panic
c <- 1   // 写一个close的chan，panic
c = nil
c <- 1 // 对nil的chan读会阻塞住

// 读数据
select {
    case v, ret := <-c:
        if !ret {
            // chan被close
        }
}
```
