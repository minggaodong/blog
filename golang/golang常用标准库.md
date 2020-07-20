# golang 常用标准库

## contex
### 简介
context 主要用于父子任务之间的同步取消信号，本质上是一种协程调度的方式。

上游使用 context 通知取消下游任务，由下游任务自行决定后续的处理操作，context 的取消操作是无侵入的。

context 是线程安全的，因为 context 本身是不可变的（immutable），因此可以放心地在多个协程中传递使用。

```
使用示例
// 创建contex根节点
func Background() Context

// 创建子节点
//WithCancel函数用来创建一个可取消的context，即cancelCtx类型的context。WithCancel返回一个context和一个CancelFunc，调用CancelFunc即可触发cancel操作
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

//WithDeadline返回一个基于parent的可取消的context，并且其过期时间deadline不晚于所设置时间d。
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)

//WithTimeout也是创建一个定时取消的context，只不过WithDeadline是接收一个过期时间点，而WithTimeout接收一个相对当前时间的过期时长timeout:
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

// 完整示例
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    go func() {
        time.Sleep(3 * time.Second)
        cancel() // 主动调用取消任务
    }()
    log.Println(A(ctx))
    select {}
}

func B(ctx context.Context) string {
    ctx, _ = context.WithCancel(ctx)
    go log.Println(C(ctx))
    select {
    case <-ctx.Done():
        return "B Done"
    }
    return ""
}

func A(ctx context.Context) string {
    go log.Println(B(ctx))
    select {
    case <-ctx.Done():
        return "A Done"
    }
    return ""
}
```

## sync
sync 包提供了常见的并发编程同步原语。

sync 中的同步原语大都不能复制，如果结构体具有这样的成员变量，则必须通过指针传递它，禁止值拷贝。

### sync.Mutex(互斥锁)
```
mutex := &sync.Mutex{} 

mutex.Lock()
// Update共享变量 (比如切片，结构体指针等)
mutex.Unlock()
```

### sync.RWMutex(读写锁)
```
mutex := &sync.RWMutex{}

mutex.Lock()
// Update 共享变量
mutex.Unlock()

mutex.RLock()
// Read 共享变量
mutex.RUnlock(
```

### sync.WaitGroup
sync.WaitGroup 使用场景是在一个 goroutine 中等待另一组 goroutine 执行完成。

sync.WaitGroup 拥有一个内部计数器，当计数器等于0时，则Wait()方法会立即返回，否则一直阻塞下去。
```
wg := &sync.WaitGroup{}
wg.Add(8)
for i := 0; i < 8; i++ {
  go func() {
    // Do something
    wg.Done()
  }()
}

wg.Wait()
```

### sync.Pool(对象池)
- sync.Pool 设计目的是用来保存和复用临时对象，以减少内存分配，降低 GC 压力。
- sync.Pool 是线程安全的，使用起来非常方便。
- sync.Pool 的对象的生命周期受 GC 影响，不适合做连接池，因为连接池需要自己管理对象的生命周期。
- sync.Pool 为每个 P 维护一个私有列表和共享列表，共享列表为双端无锁队列，访问当前 P 的共享列表从头部获取；访问其他 P 的共享列表从尾部获取，并通过 CAS 原子操作保证安全。
```
package main
 
import (
    "bytes"
    "encoding/binary"
    "fmt"
    "sync"
)
func main() {
    var bufferPool = sync.Pool{
        New:func() interface{}{
            return new(bytes.Buffer)
        },
    }
    //获取缓冲区，存储字节序列
    buf := bufferPool.Get().(*bytes.Buffer)
    var a uint32 = 121
    //将数据转为字节序列
    err := binary.Write(buf, binary.LittleEndian, a)
    if err != nil {
        fmt.Println("binary.Write failed:", err)
    }
    var b uint32 = 3434
    //将数据转为字节序列
    err = binary.Write(buf, binary.LittleEndian, b)
    if err != nil {
        fmt.Println("binary.Write failed:", err)
    }
    //拼接后的结果
    fmt.Printf("% x\n", buf.Bytes()) // 79 00 00 00 6a 0d 00 00
    fmt.Printf("% x\n", buf.Bytes()[:4]) // 79 00 00 00
    fmt.Printf("% x\n", buf.Bytes()[4:]) // 6a 0d 00 00
    //缓冲区使用完后，必须重置并放回到Pool中
    buf.Reset()
    bufferPool.Put(buf)
```

### sync.Once（单例）
sync.Once 确保一个函数只被执行一次，内部通过互斥锁来实现。
```
once := &sync.Once{}
for i := 0; i < 4; i++ {
    i := i
    go func() {
        once.Do(func() {
            fmt.Printf("first %d\n", i)
        })
    }()
}
```

## atomic 包
- 原子操作没有锁，基本是在硬件层面实现的。
- 原子操作对整数类型 (int32, int64, uint32, unit64, uintptr) 提供5种类型的原子函数。
```
/*
func AddT(addr *T, delta T)(new T)  // 增加或减少
func LoadT(addr *T) (val T)         // 原子读取，防止读的过程中正在写入
func StoreT(addr *T, val T)         // 原子写入
func SwapT(addr *T, new T) (old T)  // 交换，替换新值，返回旧值
func CompareAndSwapT(addr *T, old, new T) (swapped bool)   // CAS，比较交换
*/

// 使用实例
package main
import (
    "fmt"
    "sync"
    "sync/atomic"
)

func main() {
    var n int32
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            atomic.AddInt32(&n, 1)
            wg.Done()
        }()
    }
    wg.Wait()

    fmt.Println(atomic.LoadInt32(&n)) // 1000
}
```

## errors
error 类型是一个包含了 Error 方法的接口，errors 包实现了这个接口，并提供了 New 方法来创建一个 error 对象。
```
func New(text string) error {
    return &errorString{text}
}

// errorString is a trivial implementation of error.
type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}
```

## time
golang 的定时器使用的 chanel 阻塞来实现的。

定时器获取的 channel 是个单通道 channel，只能读不能写。

```
// 创建一个定时器
t := time.NewTicker(time.Second)
for {
        select {
        case v := <-t.C:
            fmt.Println("tsh", v)
        }
 }
 
 // 获取当前时间戳
 int64 := time.Unix()
 int64 := time.UnixNano()
 
 // 获取指定日期的时间戳
 int64 := time.Parse("2016-01-02 15:04:05")
 
 // 格式化字符串
 string := time.Now().Format("2006-01-02 15:04:05")
 string := time.Now().Format(time.UnixDate)    // Tue Apr 24 09:59:02 CST 2018
 
 // 睡眠
 time.Sleep(time.Duration(10) * time.Second)
 ```

