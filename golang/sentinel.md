# Sentinel-golang源码解析
## Sentinel简介
Sentinel是阿里开源的一款轻量级流控框架，主要以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度来帮助用户保护服务的稳定性。

相比于Hystrix，Sentinel的设计更加简单，在 Sentinel中资源定义和规则配置是分离的，也就是说用户可以先通过Sentinel API给对应的业务逻辑定义资源（埋点），然后在需要的时候再配置规则，通过这种组合方式，极大的增加了Sentinel流控的灵活性。

引入Sentinel带来的性能损耗非常小。只有在业务单机量级超过25W QPS的时候才会有一些显著的影响（5% - 10% 左右），单机QPS不太大的时候损耗几乎可以忽略不计。

## 项目地址
[sentinel-golang](https://github.com/alibaba/sentinel-golang/tree/v0.4.0)

## 源码解析
### [api](https://github.com/alibaba/sentinel-golang/tree/v0.4.0/api)
api包面向户程序，提供了使用Sentinel功能的入口函数。

#### 访问资源
```
func Entry(resource string, opts ...EntryOption) (*base.SentinelEntry, *base.BlockError)
func TraceError(entry *base.SentinelEntry, err error) 
```
用户程序在访问资源前，调用api.Entry函数对资源进行埋点，Sentinel同时会对该资源做流控规则检查，检查失败，则返回base.BlockError错误，此时应用程序执行自定义的Fallback函数对请求直接拒绝或者降级处理。

资源检查成功时，api.Entry函数返回新创建的base.SentinelEntry对象，此时应用程序可以访问资源，如果资源访问出错，需要主动调用api.TraceError函数反馈错误给Sentinel，并在最后无论如何要调用base.SentinelEntry对象的Exit方法，Sentinel会在Exit方法内检查本次资源访问是否出错，并更新该资源的监控指标。

### [core/base](https://github.com/alibaba/sentinel-golang/tree/v0.4.0/core/base)
#### 插槽
```
type SlotChain struct {
	statPres   []StatPrepareSlot
	ruleChecks []RuleCheckSlot
	stats      []StatSlot
	// EntryContext Pool, used for reuse EntryContext object
	ctxPool sync.Pool
}

func (sc *SlotChain) Entry(ctx *EntryContext) *TokenResult
func (sc *SlotChain) exit(ctx *EntryContext) 
```
SlotChain是一个全局对象，应用程序调用的api.Entry函数内部调用了SlotChain.Entry，作为一个全局插槽，可以支持三种Slot在初始化的时候插入进来，并在SlotChain.Entry函数内链式调用。

执行流程
- 顺序执行statPres：执行前置处理，stat统计模块会为当前资源查找StatNode埋点，如果没有找到就创建一个出来,
- 顺序执行ruleChecks：执行流控规则检查，检查结果保存到base.TokenResult中。
- 顺序执行stats：根据流控规则检查结果，分别执行StatSlot接口的成功和失败两个方法，stat统计模块就实现了StatSlot接口，用于对埋点进行数据统计。

### [core/stat](https://github.com/alibaba/sentinel-golang/tree/v0.4.0/core/stat)
#### 埋点
```
type ReadStat interface {
	GetQPS(event MetricEvent) float64
	GetSum(event MetricEvent) int64

	MinRT() float64
	AvgRT() float64
}

type WriteStat interface {
	AddMetric(event MetricEvent, count uint64)
}

type StatNode interface {
	MetricItemRetriever

	ReadStat
	WriteStat

	CurrentGoroutineNum() int32
	IncreaseGoroutineNum()
	DecreaseGoroutineNum()

	Reset()
}

type BaseStatNode struct {
	sampleCount uint32
	intervalMs  uint32

	goroutineNum int32

	arr    *sbase.BucketLeapArray
	metric *sbase.SlidingWindowMetric
}

type ResourceNode struct {
	BaseStatNode

	resourceName string
	resourceType base.ResourceType
	// key is "sampleCount/intervalInMs"
	readOnlyStats map[string]*sbase.SlidingWindowMetric
	updateLock    sync.RWMutex
}
```
埋点的具体实现是ResourceNode对象，stat包中创建一个全局的resNodeMap，保存Resource到ResourceNode之间的关系，用于查找。

ResourceNode通过BaseStatNode的sbase.BucketLeapArray对象对各种event事件进行原子计数，事件类型包括资源访问通过数，资源访问失败时，资源访问超时数，总体资源访问数等。

ResourceNode通过BaseStatNode的sbase.SlidingWindowMetric对象实现对埋点数据的格式化获取，比如QPS计算等。

#### 统计
StatisticSlot对象实现了StatSlot接口，具体处理逻辑：
- OnEntryPassed：goroutine并发数加1；sbase.BucketLeapArray计数器对成功事件累加。
- OnEntryBlocked：sbase.BucketLeapArray计数器对失败事件累加。
- OnCompleted：goroutine并发数减1；统计访问资源耗时rt，sbase.BucketLeapArray计数器分别对rt事件，完成事件进行计数，


### [core](https://github.com/alibaba/sentinel-golang/tree/v0.4.0/core)

### [adapter](https://github.com/alibaba/sentinel-golang/tree/v0.4.0/adapter)
