# C++ 智能指针




<!--more--> 

智能指针（Smart Pointer）是C++ 中用来自动管理动态内存分配的工具，它是一种类模板，能够自动释放内存，避免手动管理动态分配内存时可能发生的内存泄漏和指针悬空等问题。
智能指针主要包含以下几个特性：
1. 自动内存管理：当智能指针超出作用域时，会自动释放其管理的动态内存。
2. RAII（资源获取即初始化）：智能指针在构造时绑定动态内存，并在析构时自动释放资源。
3. 引用计数（部分智能指针支持）：追踪共享对象的引用次数，当引用计数变为0时自动销毁对象。

以上是GPT给出的概念解释，道理大家都懂，具体怎么使用以及怎么实现的呢？

动态内存分配就是我们手动new()或malloc()一块堆上的存储区，最经典的就是定义一个指针，指向新开辟的数组，用完后delete掉，如果忘记delete就会导致内存泄漏，或者重复delete就会导致未定义行为，delete之后再次访问对象还可能出现悬空指针的问题。所以总结一句话，new的时候能用智能指针就用智能指针。
```C++
#include <iostream>

int main() {
    // 动态分配一个大小为5的int数组
    int* arr = new int[5];

    // 初始化数组
    for (int i = 0; i < 5; ++i) {
        arr[i] = i * 10;
    }

    // 使用数组
    std::cout << "Array elements: ";
    for (int i = 0; i < 5; ++i) {
        std::cout << arr[i] << " ";
    }
    std::cout << std::endl;

    // 释放内存
    delete[] arr;

    return 0;
}
```

使用智能指针操心的就比较少，只需引入#include <memory>，写一行智能指针的代码即可。
```C++
#include <iostream>
#include <memory> // 包含智能指针头文件

int main() {
    // 使用 unique_ptr 管理动态分配的数组
    std::unique_ptr<int[]> arr(new int[5]);

    // 初始化数组
    for (int i = 0; i < 5; ++i) {
        arr[i] = i * 10;
    }

    // 使用数组
    std::cout << "Array elements: ";
    for (int i = 0; i < 5; ++i) {
        std::cout << arr[i] << " ";
    }
    std::cout << std::endl;

    // 离开作用域时，unique_ptr 会自动释放数组内存
    return 0;
}
```

智能指针有以下三种：std::unique_ptr、std::shared_ptr、std::weak_ptr，它们的使用就是将模板参数替换为对应new的数据类型。

## std::unique_ptr
独占所有权，只能一个指针指向一块动态内存，无法复制，只能move转移所有权。
```C++
#include <memory>
#include <iostream>

class MyClass {
public:
    MyClass() { std::cout << "Constructor\n"; }
    ~MyClass() { std::cout << "Destructor\n"; }
};

int main() {
    // 创建一个 unique_ptr，管理 MyClass 的对象
    std::unique_ptr<MyClass> ptr1 = std::make_unique<MyClass>();

    // 无法复制 unique_ptr
    // std::unique_ptr<MyClass> ptr2 = ptr1; // 错误

    // 转移所有权
    std::unique_ptr<MyClass> ptr2 = std::move(ptr1);

    // 此时 ptr1 不再拥有对象，ptr2 拥有对象
    if (!ptr1) {
        std::cout << "ptr1 is null\n";
    }

    return 0;
}
```

## std::shared_ptr
共享所有权，允许多个智能指针共享同一块动态内存，使用引用计数来追踪有多少指针引用同一个对象。
每次 shared_ptr 被复制时，引用计数增加。每次 shared_ptr 被销毁或超出作用域时，引用计数减少。当引用计数变为0时，管理的对象被销毁。
```C++
#include <memory>
#include <iostream>

class MyClass {
public:
    MyClass() { std::cout << "Constructor\n"; }
    ~MyClass() { std::cout << "Destructor\n"; }
};

int main() {
    // 创建一个 shared_ptr
    std::shared_ptr<MyClass> ptr1 = std::make_shared<MyClass>();
    
    {
        // 复制 shared_ptr，引用计数增加
        std::shared_ptr<MyClass> ptr2 = ptr1;
        std::cout << "Shared ownership count: " << ptr1.use_count() << "\n"; // 输出 2
    }

    // 离开作用域，ptr2 被销毁，引用计数减少
    std::cout << "Shared ownership count: " << ptr1.use_count() << "\n"; // 输出 1

    return 0;
}
```

但是shared_ptr会存在循环引用问题，导致引用计数永不为0。那么下面讲讲什么是循环引用问题。

很类似于死锁，死锁是A在等B释放锁，而B在等A释放锁，从而导致A与B互相等待的情况。这里的循环应用就是A里的shared_ptr指向了B，而B里的shared_ptr指向了A，两个shared_ptr就算离开作用域，引用计数也都为1，导致A和B的内存无法释放，形成内存泄漏。

死锁的解决方法有：资源（锁）按相同顺序获取；线程主动释放资源；那么循环引用的解决方法是？引入weak_ptr，weak_ptr并不会增加引用计数，那这样weak_ptr在作用域结束时一定会被释放，指向weak_ptr的shared_ptr自然会将引用计数减一，从而正常释放自己。

使用shared_ptr还会出现一个问题，shared_ptr如果直接指向this指针时，可能会出现double free的情况，因为shared_ptr指向this指针会指向一个新的控制块，这样对同一对象就会同时有两个控制块（引用计数），当shared_ptr生命周期结束时所指向的对象会被释放两次。解决方法是：使用std::enable_shared_from_this，它内部实现是一个weak_ptr指向共享的控制块，保证了引用计数的唯一。

## std::weak_ptr
不多说了，上边说了weak_ptr存在的意义。值得注意的是，weak_ptr在观察shared_ptr所管理的对象，当需要访问对象时，可以通过 weak_ptr.lock() 获取一个临时的 std::shared_ptr。如果对象已经被释放，则返回空指针。

```C++
#include <iostream>
#include <memory>

class B; // 前向声明

class A {
public:
    std::shared_ptr<B> b_ptr; // A 持有 B 的 shared_ptr
    ~A() { std::cout << "A destroyed\n"; }
};

class B {
public:
    std::weak_ptr<A> a_ptr; // B 持有 A 的 weak_ptr
    ~B() { std::cout << "B destroyed\n"; }
};

int main() {
    std::shared_ptr<A> a = std::make_shared<A>();
    std::shared_ptr<B> b = std::make_shared<B>();

    a->b_ptr = b; // A 持有 B
    b->a_ptr = a; // B 持有 A（通过 weak_ptr）

    // 离开作用域时，不会发生内存泄漏
    return 0;
}
```

总的来说，智能指针是对原始指针的封装，提供资源管理和安全访问功能的模板类。对于共享型智能指针（如 std::shared_ptr 和 std::weak_ptr），通常引入控制块来管理资源的共享状态。控制块包含强引用计数器、弱引用计数器、资源删除器以及与资源绑定的原始对象指针等信息。多个智能指针实例可以共享一个控制块来管理同一个资源。

---

> 作者: [UAreFree](https://github.com/UAreFree)  
> URL: http://localhost:1313/posts/cpp-smart-pointer/  

