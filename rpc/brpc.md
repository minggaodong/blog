# brpc 详解
## 概述
brpc 是 baidu 公司开源的 rpc 框架，具有以下特点
- 支持多种第三方协议，比如 grpc，thrif，http 等。
- 引入 bthread 线程库，底层采用 IO 多路复用满足高并发，上层使用 bthread 处理请求任务，支持异步。
- 全面的服务发现，负载均衡，组合访问支持。
- 可视化的内置服务和调试工具。

## 协议
brpc 支持多种协议：baidu_std，http，http2，这里的协议指的是解析包头包体的规则，包头中有认证和压缩信息。

- server 端的包体需要定义为 protobuf 格式，如果服务是一个 http 服务，则 proto 文件中 request 和 response 都定义为空，brpc 提供了获取 get 参数的方法。

- client 端的包体可以使用 json 等格式，server 会自动识别并将 json 内容转换成 pb 格式后使用，server 端的一个接口可以同时识别支持多种协议。

- client 端默认采用 baidu_std 协议，可通过 ChannelOptions.protocol 更换其它协议。

- 支持thrift协议，仅仅是协议，和thrift通信框架没有关系，使用thrift协议时，使用thrift文件定义接口。

## bthread
bthread 是一个 M:N 线程库，类似 goroutine 实现。bthread 工作在 pthread 之上。
- bthread 中调用阻塞的 bthread 函数，pthread 线程会被调度处理其他 bthread。
- bthread 中调用阻塞的系统函数，pthread 会被阻塞，pthread 下等待运行的 bthread 会被调度给其他空闲 pthread。
- server端的 callMethod 和 client 的 Done 都是默认以以 bthread 运行，用户程序中一般不需要调用任何 bthead 函数。
- 可以在 option 选项中设置最大 pthread 线程数，默认是 cpu core 的个数，可以充分利用 cpu 多核资源。
- option 选项中可以设置最大并发数，超过最大并发数返回失败。
- brpc 为每个请求创建一个新的 bthread，请求结束 bthead 就结束了。

## bvar
bvar 是一个多线程环境下的计数器类库，它利用 thread_local 存储减少 cache bouncing(多线程原子操作同一个变量时，cpu需要对cache line一致性同步，性能较低)。

- bvar 适用于读少写多的场景，写时只写到 thread local，读时汇总所有线程的值，因为汇总操作比较慢，所以读不能太频繁。
- bvar 提供了多种计数器，支持多种统计场景，并提供了时间窗口，与变量结合使用。
- bvar 变量名称不能重复，必须唯一，通过 /status 或者 /var 接口，可以在浏览器上查看变量的值。
- 支持导出到 Prometheus，Prometheus 通过 brpc 内置的 /brpc_metrics 接口，定时采集bvar监控指标。

## server 端
通过 proto 文件定义服务接口，业务 server 通过实现 service 基类，来实现具体业务接口。

server 分为 pb 服务和 http 服务，http 服务的 proto 文件中的 message 结构都为空，request 和 response 的对象读写通过 brpc::Controller 获取。

brpc 支持异步返回，通过保存 done 对象，并最终调用 done->Run() 来实现异步返回，通常情况下都是同步模式。

## client 端
客户端使用 service_stub 类调用 rpc 接口，调用之前需要绑定到一个 channel 上。

### channel
与服务端进行通信的通道，内部封装了 tcp 连接，channel 会被所有线程公用，是线程安全的。

#### 建立连接 
channel.init 参数中包括命名服务器列表，负载均衡策略，channel 连接成功后，brpc 会定时更新命名服务器列表。

#### 服务发现
命名服务器列表支持多种格式：文件列表，http 接口，consul 接口。

#### 负载均衡
支持rr（轮询），wrr（权重），la（最低延时），一致性哈希(调用接口前需要设置hash主键值)多种策略。

#### 健康检查
连接断开的 server 会被隔离，不会再被负载均衡算法选中，brpc 会定期连接被隔离的 server，检查他们是否恢复正常，一旦恢复连接，会恢复可用状态。

#### 异步访问
可用设置回调函数，异步的调用 rpc 接口。可用同时发起多个异步 rpc 调用，并通过 brpc::join 逐个等待返回。

#### 超时重试
支持设置 rpc 的超时时间和重试次数，当调用出错时会回调重试函数，重试函数可以自定义那种错误码需要重试，每次重试会避开上次超时的 server。

#### 连接方式
包括连接池和单连接，框架会根据协议类型自动选择连接方式，http 使用的是连接池，badu_std 使用的是单连接：每个 server 只建立一个连接，一个连接上可能同时多个请求，顺序不需要一致。

## 自适应限流
服务处理能力有限的，当请求速度超过服务的处理速度时，服务就会过载，持续过载会导致请求积压，最终所有请求都必须等待较长时间才能被处理，服务处于瘫痪状态。

通常情况下，通过压力测试，计算出最大并发量，并在上线时设置好最大并发量，可以避免服务过载情况发生。当超过设置的最大并发量时，请求会被直接拒掉。

但是实际情况可能比较复杂，在满足时延的情况下，最大可并发数是不太好确定的，自适应限流就是为了解决这种问题。

自适应限流打开后，服务端会根据请求时延进行采样，计算出动态的最大并发数，当请求数超过时，直接拒掉，client 发现因为过载被拒后，需要重发请求到其他 server上，避免流量丢失。
