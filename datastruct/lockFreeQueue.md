# 无锁队列实现
## 前言
在生产者和消费者模型下，对队列的 push 和 pop 操作需要互斥锁保证线程安全，但是加锁操作导致线程让出 CPU，频繁切换代价很高。

无锁队列的设计原理是利用 CAS 原子操作，通过自旋循环重试的去修改队列，适用于数据竞争概率小，对性能要求又很高的场景。

## CAS 
CAS （Compare & Set或是 Compare & Swap），CPU 指令都支持 CAS 的原子操作，X86 下对应的是 CMPXCHG 汇编指令。

CAS 的语义是：“我认为 V 的值应该为 A，如果是，那么将 V 的值更新为 B，否则不修改并告诉 V 的值实际为多少”。

CAS 是项乐观锁技术，当多个线程尝试使用 CAS 同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。

CAS操作需要输入两个数值，一个旧值（期望操作前的值）和一个新值，在操作期间先比较下旧值有没有发生变化，如果没有发生变化，才交换成新值，发生了变化则不交换。

C++11 中的 STL 中的 atomic 类的函数可以实现 CAS
```

template< class T >
bool atomic_compare_exchange_weak( std::atomic<T>* obj,
                                   T* expected, T desired );
template< class T >
bool atomic_compare_exchange_weak( volatile std::atomic<T>* obj,
                                   T* expected, T desired );
```

## 无锁队列实现
### 进队列实现
```
EnQueue(x)//进队列
{
    //准备新加入的结点数据
    q = new record();
    q->value = x;
    q->next = NULL;
    do {
        p = tail;//取链表尾指针的快照
    } while( CAS(p->next, NULL, q) != TRUE);//如果没有把结点链在尾指针上，再试
    CAS(tail, p, q);//置尾结点
}


这里有一个潜在的问题：如果T1线程在用CAS更新tail指针的之前，线程停掉或是挂掉了，那么其它线程就进入死循环了。
下面是改良版的EnQueue()

EnQueue(x)//进队列改良版
{
    q = new record();
    q->value = x;
    q->next = NULL;
    p = tail;
    oldp = p
    do {
        while (p->next != NULL)
            p = p->next;
    } while( CAS(p.next, NULL, q) != TRUE);//如果没有把结点链在尾上，再试
    CAS(tail, oldp, q);//置尾结点
}

我们让每个线程，自己fetch 指针 p 到链表尾。但是这样的fetch会很影响性能。而通实际情况看下来，99.9%的情况不会有线程停转的情况，所以，更好的做法是，你可以接合上述的这两个版本，如果retry的次数超了一个值的话（比如说3次），那么，就自己fetch指针。

```
### 出队列

```
DeQueue()//出队列
{
    do{
        p = head;
        if (p->next == NULL){
            return ERR_EMPTY_QUEUE;
        }
    while( CAS(head, p, p->next) != TRUE );
    return p->next->value;
}

我们可以看到，DeQueue的代码操作的是 head->next，而不是head本身。这样考虑是因为一个边界条件，我们需要一个dummy的头指针来解决链表中如果只有一个元素，head和tail都指向同一个结点的问题，这样EnQueue和DeQueue要互相排斥了。
```

### ABA 问题
ABA 问题可以描述为
- 进程 P1 在共享变量中读到值为A
- P1 被抢占了，进程 P2 执行
- P2 把共享变量里的值从 A 改成了 B，再改回到 A，此时被 P1 抢占。
- P1 回来看到共享变量里的值没有被改变，于是继续执行。

虽然 P1 以为变量值没有改变，继续执行了，但是这个会引发一些潜在的问题。

ABA 问题最容易发生在 lock free 的算法中的，CAS首当其冲，因为 CAS 判断的是指针的地址。如果这个地址被重用，就会出问题

#### 解决 ABA 问题
使用结点内存引用计数 refcnt
```
SafeRead(q)
{
    loop:
        p = q->next;
        if (p == NULL){
            return p;
        }
        Fetch&Add(p->refcnt, 1);
        if (p == q->next){
            return p;
        }else{
            Release(p);
        }
    goto loop;
}
```
其中的 Fetch&Add 和 Release 分别是加引用计数和减引用计数，都是原子操作，这样就可以阻止内存被回收了。
