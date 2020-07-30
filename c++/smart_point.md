C++里面的四个智能指针
C++里面的四个智能指针: auto_ptr, shared_ptr, weak_ptr, unique_ptr 其中后三个是c++11支持，并且第一个已经被11弃用。

auto_ptr(c++11已废弃)
auto_ptr对象在生命周期结束时，其析构函数会自动释放指向的内存。
auto_ptr对象对指向的内存是独享的，如果同一块内存被分配给多个auto_ptr，则容易因为内存被其中一个对象释放掉，导致另一个对象变成野指针，不安全。
auto_ptr对象可以互相赋值，赋值后，原对象对内存的所有权会被转移走，这保证了内存的独享。
auto_ptr对象不能作为函数传参，因为传参时，参数发生拷贝，导致原有auto_ptr对象的内存所有权被释放掉，转移给了形参，形参在函数调用结束也会被释放掉。
#include <memory>
int *p = new int(33)
auto_ptr<int> ptr1(p); // 赋值
auto_ptr<int> ptr2(p); // 禁止，ptr2也指向了p，如果ptr2生命周期结束，会释放p，导致ptr1变成野指针。
auto_ptr<int> ptr3;   // 空指针
ptr3 = ptr1;          // ptr3指向了p，ptr1自动释放掉


unique_ptr(c++11用来替代auto_ptr)
unique_ptr和auto_ptr的功能一样，但是更安全。
unique_ptr不能直接用等号互相赋值，保证了智能指针对象和内存绑定关系始终如一。
如果确实想改变unique_ptr的内存，可以使用std::move转移所有权。
#include <memory>
int *p = new int(33);
std::unique_ptr<int> ptr1(p);
std::unique_ptr<int> ptr2;
ptr2 = ptr1;  // 编译不通过，禁止对象拷贝
ptr2 = std::move(ptr1); // 允许，所有权发生转移，ptr1变为空指针，ptr2指向内存p


shared_ptr(共享指针)
shared_ptr共享被管理对象，同一时刻可以有多个shared_ptr拥有对象的所有权，当最后一个shared_ptr对象销毁时，被管理对象自动销毁。
shared_ptr对象隐藏了一个保存引用计数的堆指针，多个shared_ptr指针通过赋值共享管理内存时，也会赋值引用对象堆指针，并原子访问引用计数值，通过use_count()可以查看引用个数。
shared_ptr对象被创建时，会在堆上创建一个引用计数对象，并保存这个引用计数对象指针，并共享给后面的所有创建的shared_ptr对象。各对象修改计数值为原子操作，计数器是线程安全且无锁的。
#include <memory>
int * p = new int(10);
std::shared_ptr<int> ptr1(p);
std::shared_ptr<int> ptr2(p);  // 这不是正确的共享方法，一个内存指针只能初始化一个shared_ptr对象，其他shared_ptr对象的初始化都通过其他shared_ptr进行初始化。
std::shared_ptr<int> ptr3 = ptr2; // 这是正确的共享方法，引用计数变为2



weak_ptr(弱指针)
weak_ptr和其他智能指针不同，它的出现只是为了解决shared_ptr循环引用的问题。
weak_ptr不会修改引用计数，就像一个普通的指针，访问前需要通过lock()检测管理的内存对象是否被释放。
shared_ptr循环引用：定义两个类A和B，A和B中都包含对方的shared_ptr指针作为成员变量，然后创建A和B两个shared_ptr对象，设置完成员变量后，此时就是循环引用，A的释放必须依赖B的释放，B同样。
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

将成员变量由shared_ptr改成weak_ptr，就可以解决循环引用，因为将shared_ptr对象赋值给weak_ptr不会使引用计数增加。
weak_ptr指针不能直接使用，需要调用lock()方法，返回shared_ptr后使用。
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
ptr_a->ptr_b = ptr_b; // 将shared_ptr赋值给weak_ptr此时不会产生引用计数
std::cout << ptr_a.use_count() << "," << ptr_a->ptr_b.lock()->b << std::endl; // 访问weak_ptr前，需要先调用lock，返回shared_ptr对象后访问。

