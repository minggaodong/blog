# STL 标准库
## 概述
STL 是目前 C++ 内置支持的标准库。底层利用了 C++ 类模板和函数模板的机制，主要由容器、算法和迭代器组成。

### STL 六大组件
- 容器 container
- 算法 algorthm
- 迭代器 iterator
- 仿函数 function object
- 适配器 adaptor
- 空间配置器 allocator

## 容器
- vector：向量
- deque：双端数组
- stack：栈模型
- queue：队列模型
- list：链表模型
- priotriy_queue：优先级队列
- set与multiset容器
- map与multimap容器

## 迭代器
迭代器是一种抽象，提供了访问容器内部元素的方法，但无需暴漏容器的内部结构，它将容器和算法分开，好让二者独立设计。

迭代器用法类似于指针，它对 * 和 -> 进行了运算符重载，所以可以想操作指针一样操作迭代器。

迭代器设计了一套规范，STL每种容器都有自己的迭代器，它们都实现了相同的接口。在使用者看来，迭代器的使用时统一的。

### 迭代器失效
对 map 和 list 的插入（insert）和删除（erase）操作，会导致迭代器失效，需要这样获取新的迭代器：iter = container.erase(iter)。
vector 使用迭代器操作时，一旦发生了扩容，则指向原 vector 的所有迭代器都将失效。
```
std::map<int> m;
for (auto iter = m.begin(); iter != m.end(); ) {
  m.erase(iter++);        // 这里删除元素后，会导致iter迭代器失效，导致crash
  iter = m.erase(iter);   // 正确的删除方式
}
```

## allocator
容器的内存分配是基于 allocator 实现，大部分 STL 版本（比如 SGI ）实现 allocatior 时，其实就是简单的封装了 new/delete。

STL 默认的 allocator 没有采用内存池技术，因为 new 底层调用的 malloc 本身实现了内存池，效率还可以。

### vector 内存分配
vector 动态扩容时，会按倍数重新申请空间，并将原来数据拷贝到新内存，C++11 中底层使用转移构造，不会发生大的拷贝。

在 vs 编译器下是按 1.5 倍扩容，在 gcc 下是按 2 倍扩容，具体用几倍是时间和空间上的一个权衡。

vector 之所以不再尾部重新开辟内存，是因为无法保证原空间之后尚有可供配置的空间。

1.5 倍扩容的优势是分配几次后，下次分配的内存，肯定小于之前分配的内存总和，可以直接复用原来的内存；2倍扩容时，下次分配的内存一定大于之前分配的内存总和，没法重复利用。

2倍扩容的优势是，linux 伙伴系统管理的物理内存页框是2的倍数，2 倍扩容可以充分利用物理内存，避免碎片产生。
