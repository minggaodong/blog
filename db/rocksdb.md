# leveldb x详解
##  概述
LevelDB 是 Google 开源的持久化 KV 单机数据库，支持写远高于读的单机应用。

- 写性能远高于读性能，经测试，写速度7w/s，读速度在1.6w/s。
- 同一个库只允许单个进程访问，单个进程内部多线程访问是线程安全的
- 具有插入，删除，查找三种基本操作。
- 支持快照操作，在某个时间创建一个全景快照，读数据时把快照作为参数，则读出的数据是那个时间点时的数据。
- 仅仅是一个 lib 库，没有自己的服务进程，对分布式事务的支持不成熟，机器资源浪费率高。
- leveldb 中 key 和 value 数据尺寸不能太大，在KB级别，如果存储较大，将对 leveldb 的读写性能都有较大的影响。

## 写入流程
- 顺序写入 log 文件，速度非常快，log 文件仅用来故障恢复，属于临时文件，写入 log 文件成功就代表数据写入成功。
- 写入内存中的 memtable 结构，memtable 是一个 skiplist(跳跃表)，数据有序存储。

## 持久化磁盘
- 当 memtable 大小达到上限，会生成一个新的 memtable，当前 memtable 更名为 immutale memtable，并且只读，数据会写入到新的 memtable中。
- immutale memtable 会 dump 到磁盘，存储成 sstabl e文件（.sst后缀），持久化完成后，清除 immutale memtable 对应的 log 文件。
- immutale memtable 每 dump 一次，就会生成一个 .sst 文件，这些文件都属于 level0 文件，这些文件内部是有序的，文件之间是无序的。

## 存储结构
leveldb 磁盘存储的文件分为 level-0 到 level-6, 每一层中有若干个文件, 所有文件长度和最大限制如下
- level-0 10M
- level-1 100M
- level-2 1000M
- level-3 10000M
- level-4 100000M
- level-5 1000000M
- level-6 10000000M

## 合并过程（compact）
- 数据文件中被删除的kv记录占用存储空间需要回收；同一个 key 多次写入，系统维持多个版本的值，需要合并以减少磁盘占用，并提高读取效率
- 判断 immutale memtable 是否为空，不为空则写入 level0 的 .sst 文件。
- level0 层文件数达到上限后，触发compact，通过归并排序进行合并，生成 level1 层文件，并删除 level0 层文件。
- level1 层往上文件在本层之内都是有序的。
- 合并之前，层级与层级之间有大量重复 key，归并之后，变成一层，重复key消失，层数越高，数据版本越老，但是存储空间越大。

## 读数据原理
- 从内存memtable中读取
- 从immutale memtable中读取
- 从磁盘.sst文件中读取，依次level0，levle1，level2，……，level6
- 读到数据或者读到删除标记都返回。
- 在每层查找时，使用二分查找key所在的文件，然后通过快索引二分查找key所在的块，通过块中的布隆过滤器，匹配key值是否存在，不存在直接返回查不到，否则通过二分查找key所在记录。

## 删除原理
- 和写入操作一样，只不过这里写入的是一个删除标记，真正的删除是在 compaction 阶段完成。
- compaction 时，会移除旧的 key，并保留带删除标记的 key。
- compaction 到最顶层时，才删除带删除标记的 key。

## rocksdb 介绍
Rocksdb 是 facebook 开源的 key-value 存储系统，其设计是基于 Google 开源的 Leveldb，因此应用了 LSM (Long Structure Merge Tree)策略。

相比 levledb，有下面特点
- 支持批量获取 key，而且批量操作是一个原子操作。
- 支持单个进程启用多个实例，而 levledb 只能一个实例。
- 支持多线程执行 compaction，防止 put 阻塞。
- 支持管道式的 memtable，也就说允许根据需要开辟多个 Memtable，以解决 Put 与 Compact 速度差异的性能瓶颈问题。
- 支持更多的压缩算法，level 只支持 snappy。

## 存在的问题
### 写延迟
LevelDB 实现上是 L0 达到 4 个时开始触发 compaction, 8个时开始减慢写入, 12个时完全停止写入，一旦写速度>compact速度，就会延迟卡死。

### 写放大
如果真实写入速度是10M/s，由于compact后台操作，会导致10~20倍的磁盘写入，此时瓶颈为磁盘，由于是顺序写入，磁盘缓存ssd也不能提高太多。
