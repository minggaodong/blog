# c++ 智能指针详解
## 概述
c++ 智能指针的功能是对创建的内存自动释放，原理是在智能指针失去生命周期时，会在析构函数中自动释放管理的内存。

智能指针根据用途可以分为
- auto_ptr：独占指针（c++11 已废弃）
- unique_ptr：独占指针（替代 auto_ptr）
- shared_ptr：共享指针
- weak_ptr：弱指针（解决 shared_ptr 的循环引用）

## auto_ptr
auto_ptr 对象可以通过等号互相传递，传递后，内存的所有权会被转移走，这保证了内存的独占，但是会产生野指针不安全，C++11 中已废弃。

```
auto_ptr<int> p1(new int(10));
auto_ptr<int> p2 = p1;  // 转移控制权，p1为野指针
*p1 += 10;              // crash，可以用p1->get判空做保护
```
## uniq_ptr
unique_ptr 和 auto_ptr 功能一样，在 c++11 中用来替换 auto_ptr，但是更加安全。

unique_ptr 不支持等号传递，保证了智能指针对象和内存绑定关系始终如一。

unique_ptr 虽不支持等号传递，但是重载了等号的右值引用转移，内部将原指针转移给新指针，原指针赋值为空，需要借助 std::move。

```
#include <memory>
std::unique_ptr<int> ptr1(new int(33));
std::unique_ptr<int> ptr2;
ptr2 = ptr1;  // 编译不通过，不支持等号传递
ptr2 = std::move(ptr1); // 允许右值转移，内存所有权从 ptr1 转给了 ptr2，ptr1 变为空。
*ptr1 = 10;   // crash

// 作为函数入参数
std::vector<std::unique_ptr<int> > vec_value;
void push(std::unique_ptr<int> ptr1) {
    vec_value.emplace_back(std::move(ptr));
}
std::unique_ptr<int> ptr1(new int(11));
push(std::move(ptr1));
*ptr1 = 10; // crash, ptr1为null
```

## shared_ptr
shared_ptr 为共享指针，shared_ptr 支持通过等号传递，传递过程中通过引用计数共享内存对象。

shared_ptr 的引用计数是一个堆指针，等号传递时也会传递引用计数指针，引用计数的操作是原子操作，通过 use_count() 可以查看引用个数。

```
#include <memory>
int * p = new int(10);
std::shared_ptr<int> ptr1(p);       // 创建一个 ptr1 指针管理内存p，此时引用计数为 1 。
std::shared_ptr<int> ptr2 = ptr1;   // 创建一个 ptr2 也管理内存p，此时引用计数为 2。
std::shared_ptr<int> ptr3(p);       // 危险方法，一个裸指针只能初始化一个 shared_ptr，shared_ptr 之间只能通过等号传递。
```

## weak_ptr
weak_ptr 和其他智能指针不同，它的出现只是为了解决 shared_ptr 循环引用的问题。

可以将 shared_ptr 等号传递给一个 weak_ptr，而不会修改引用计数，访问前需要通过 lock() 检测管理的内存对象是否被释放。

### shared_ptr 循环引用
两个类 A 和 B，A 和 B 中都包含对方的 shared_ptr 指针作为成员变量，然后创建 A 和 B 两个shared_ptr对象，设置完成员变量后，此时就是循环引用，A 的释放必须依赖 B 的释放，B 同样，这样就产生了循环引用，引用计数始终不会为 0，所以内存永远释放不了。

```
#include <memory>
class B;
class A {
    std::shared_ptr<B> ptr_b;
};

class B{
    std::shared_ptr<A> ptr_a;
};

std::shared_ptr<A> ptr_a(new A);
std::shared_ptr<B> ptr_b(new B);
ptr_a->ptr_b = ptr_b;
ptr_b->ptr_a = ptr_a; // 设置完成后，产生循环依赖
```

### 解决循环引用
将 A 中的成员变量 ptr_b 改成 std::weak_ptr 类型，就解决了循环引用问题。

```
class B;
class A {
public:
        A(int v) { a = v;}
        int a;
        std::weak_ptr<B> ptr_b;
};

class B {
public:
        B(int v) { b = v;}
        int b;
        std::shared_ptr<A> ptr_a;
};

std::shared_ptr<A> ptr_a(new A(10));
std::shared_ptr<B> ptr_b(new B(20));
ptr_b->ptr_a = ptr_a;
ptr_a->ptr_b = ptr_b; // 将shared_ptr赋值给weak_ptr此时不会产生引用计数，这样B内存会被正常释放掉，循环引用解开
std::cout << ptr_a.use_count() << "," << ptr_a->ptr_b.lock()->b << std::endl; // 访问weak_ptr前，需要先调用lock，返回shared_ptr对象后访问。
```
