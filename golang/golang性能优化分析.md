# golang 性能优化总结

## 内存优化
- 小对象合并成结构体一次分配，减少内存分配次数，同时可以减少内存碎片的产生。
- 缓冲区内容一次分配的足够大小空间，并适当复用，如对 byte 的操作可以使用 byte.Buffer 预先分配足够大内存后使用。
- slice 和 map 采用 make 创建时，预估大小指定容量。
- 不要在一个 goroutine 中申请较多的临时对象，goroutine 默认栈大小为 2k，超过后 2 倍扩容，影响性能，可以将 goroutine 池化，避免 goroutine 栈空间的变化。
- 尽可能的使用局部变量(栈上分配)，减少 GC 开销。

## 并发优化
- 高并发的任务处理使用 goroutine 池，过多的 goroutine 创建，会影响 go runtime 的调度和 GC 性能。
- 高并发时避免共享对象互斥，goroutine 尽量独立无冲突的执行；若 goroutine 间存在冲突，可以用分区来控制。

## 其他优化
- 避免 CGO 或者 CGO 的调用次数，CGO 调用涉及到 C 和 GO 的栈切换，比较影响性能。
- 减少 []byte 和 string 之间的转换，尽量采用 []byte 来处理字符；[]byte 和 string 在底层结构完全不同，他们之间的转换需要完全深拷贝。

## pprof 使用
pprof 是官方提供的 profiling 工具，用于分析程序运行时从 cpu 和内存占用，goroutine 的运行状态等。

### 代码嵌入
通过下面方式导入，在浏览器中输入 http://ip:8899/debug/pprof/ 可以看到一个汇总页面，点击进去查看详情。
```
import "net/http"
import _ "net/http/pprof"
func main() {
    go func() {
        http.ListenAndServe("0.0.0.0:8888", nil)
    }()
}
```

### 利用go tool分析堆内存占用
##### 下载 pprof 文件
```
wget http://localhost:8888/debug/pprof/heap
```

##### 使用 go tool 分析下载到的heap文件，会进入一个gdb类似环境
```
go tool pprof 可执行程序(用来获取符号) heap
```

##### 查看内存占用前10
```
(pprof) top 10
flat  flat%   sum%        cum   cum%
   43.46MB 60.73% 60.73%    43.46MB 60.73%  git.intra.weibo.com/ad/dmp/package_managment_platfrom/dmp_builder/builder.NewServer
   15.56MB 21.74% 82.48%    15.56MB 21.74%  bufio.NewReaderSize
   10.04MB 14.03% 96.51%    10.04MB 14.03%  bufio.NewWriterSize
    0.50MB   0.7% 97.20%    27.10MB 37.87%  git.intra.weibo.com/ad/dmp/package_managment_platfrom/dmp_builder/builder.NewRedisCluster
    0.50MB   0.7% 97.90%     0.50MB   0.7%  net.dnsPacketRoundTrip
    0.50MB   0.7% 98.60%     0.50MB   0.7%  net.newFD
    0.50MB   0.7% 99.30%     0.50MB   0.7%  github.com/Shopify/sarama.(*TopicMetadata).decode
    0.50MB   0.7%   100%     0.50MB   0.7%  context.WithCancel
         0     0%   100%    15.56MB 21.74%  bufio.NewReader
         0     0%   100%    10.04MB 14.03%  bufio.NewWriter
其中flat是当前函数占用的内存，cum则是当前函数累计内存值，包括调用的函数占用的内存值。
```

##### 保存SVG格式，在本地用浏览器查看各调用占用内存
```
(pprof) svg
```

##### 定位具体代码
```
(pprof)list builder.NewServer
Total: 71.56MB
ROUTINE ======================== git.intra.weibo.com/ad/dmp/package_managment_platfrom/dmp_builder/builder.NewServer in /data0/minggao/dmp_builder/builder/server.go
   43.46MB    43.46MB (flat, cum) 60.73% of Total
         .          .     36:   s.http_server = NewHttpServer(s.conf)
         .          .     37:
         .          .     38:   s.handle = NewHandle(s.conf)
         .          .     39:   s.handle_chan_v = make([]chan KfkMessage, s.conf.Main.MaxHandleRoutine)
         .          .     40:   for i := 0; i < s.conf.Main.MaxHandleRoutine; i++ {
    1.65MB     1.65MB     41:           s.handle_chan_v[i] = make(chan KfkMessage, s.conf.Main.MaxHandleChanLen)
         .          .     42:   }
         .          .     43:
         .          .     44:   s.read = NewPackRead(s.conf)
         .          .     45:   s.read_chan_v = make(chan PackInfo, s.conf.Main.MaxReadChanLen)
         .          .     46:
         .          .     47:   s.load = NewPackLoad(s.conf)
         .          .     48:   s.load_chan_v = make([]chan PackLoad, s.conf.Main.MaxLoadRoutine)
         .          .     49:   for i := 0; i < s.conf.Main.MaxLoadRoutine; i++ {
   41.82MB    41.82MB     50:           s.load_chan_v[i] = make(chan PackLoad, s.conf.Main.MaxLoadChanLen)
         .          .     51:   }
         .          .     52:
         .          .     53:   return s, nil
         .          .     54:}
         .          .     55:
```
