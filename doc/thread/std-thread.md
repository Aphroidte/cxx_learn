# std thread

<!-- TOC -->

- [std thread](#std-thread)
  - [1. 线程与进程的简介](#1-线程与进程的简介)
    - [1.1. 什么是线程？](#11-什么是线程)
    - [1.2. 进程与线程的区别](#12-进程与线程的区别)
  - [2. C++11中的 std::thread](#2-c11中的-stdthread)
    - [2.1. 线程的构造&析构](#21-线程的构造析构)
    - [2.2. 常用函数](#22-常用函数)
    - [2.3. 使用过程中需要注意的点](#23-使用过程中需要注意的点)
      - [2.3.1. 回调函数的引用形参传递问题](#231-回调函数的引用形参传递问题)
  - [3. 线程的同步](#3-线程的同步)
    - [3.1. std::mutex](#31-stdmutex)
    - [3.2. std::atomic](#32-stdatomic)
      - [3.2.1. 原子类型支持的原子操作](#321-原子类型支持的原子操作)
  - [4. this_thread命令空间](#4-this_thread命令空间)

<!-- /TOC -->

## 1. 线程与进程的简介

### 1.1. 什么是线程？

**线程** 是<font color=Crimson>操作系统能够进行调度最小单位，是能够独立调度和分派的基本单位。</font>它包含在进程之中，是进程中实际运作的单位。

一个线程指的是进程中一个单一顺序的控制流（相当于是一条流水线）。一个进程中可以并发多个线程。（在 Unix System V 以及 SunOs 中，线程也被称为轻量化进程，轻量化进程更多指的是内核线程，而用户线程则被称为线程）。

### 1.2. 进程与线程的区别

<font color=Crimson>进程是正在运行的程序的实例，而线程是进程中的实际运作单位。</font>

区别：

- 一个程序有且仅有一个同名的进程实例，但可以拥有至少一个线程。
- <font color=DeepPink>不同的进程拥有不同的地址空间，互不相关，而不同线程共同拥有相同的进程的地址空间。</font>

## 2. C++11中的 std::thread

*C++11* 标准中，`<thread>` 头文件提供的 thread 类（位于 `std` 命名空间下），专门用于线程的创建和使用。用于替代 Linux 的 pthread 来进行多线程编程（在 C++11 以前，多线程都是跟操作系统环境相关的。Linux 上是 pthread 类；而 std::thread 则是可移植的，屏蔽掉了对系统的依赖。）

### 2.1. 线程的构造&析构

| 函数 | 类别 | 作用 |
| :--- | :--- | :--- |
| thread() noexcept | 默认构造函数 | 创建一个线程，什么也不做 |
| template\<class Fn, class ... Args\> <br> explicit thread(Fn&& fn, Args&& ... args) | 初始化构造函数 | 创建一个线程，以 Args 为参数（<font color=Crimson>参数是以万能引用的形式传递的</font>），执行 Fn 函数 |
| thread(const thread&) = delete | 拷贝构造函数 | 已删除 |
| thread(thread&& x) noexcept | 移动构造函数 | 构造一个与 x 相同的对象，但是会破坏 x 对象|
| ~thread() | 析构函数 | 析构对象 |

### 2.2. 常用函数

| 函数 |作用 |
| :--- | :--- |
| void join() | 等待线程结束并清理资源(会阻塞) |
| bool joinable() | 返回线程是否可以执行 join 函数 |
| void detach() | 将线程与调用线程分离，彼此独立执行（<font color=Crimson>此函数必须在线程创建时立即调用，且调用此函数会使其不能被 join</font>）。 |
| std::thread::id get_id() | 获取线程 id |
| thread& operator=(thread&& rhs) | 移动赋值，同移动构造（<font color=Crimson>如果对象是 joinable 的，那么会调用 std::terminate 结果程序）</font> |

> 注：每个 *thread* 对象在调用析构函数销毁前，要么调用 `join()` 函数令主线程等待子线程执行完成，要么调用 `detach()` 函数将子线程和主线程分离，两者比选其一，否则程序可能存在以下两个问题：
>
> 1. **线程占用的资源将无法全部释放，造成内存泄漏；**
> 2. **当主线程执行完成而子线程未执行完时，程序执行将引发异常。**

### 2.3. 使用过程中需要注意的点

#### 2.3.1. 回调函数的引用形参传递问题

该问题主要出现在回调函数的形参是左值引用类型时，常发生在线程创建时，或 std::bind 调用时。如以下i情况：

```c++
template<class T>
void changevalue(T& x, T val) {
    x = val;
    val += 100;
}

int main() {
    int a = 10, b = 1;
    std::thread t1(&changevalue<int>, a, b);
    std::cout << a << "," << b << std::endl;
    t1.join();
}
```

上述代码会编译报错：

> error: static assertion failed: std::thread arguments must be invocable after conversion to rvalues

线程的参数都是以右值的形式传递的的，所以 a 和 b 在传递的时候，会先对 a 和 b 进行拷贝，然后绑定到右值形参上，此时如果回调函数的形参时左值引用类型的话，则无法将一个右值引用绑定到左值引用上，编译报错。

这是可以通过 `std::ref` 或 `std::cref` 来解决在绑定含有引用参数的回调函数时进行传参的问题。

正确代码如下：

```c++
template<class T>
void changevalue(T& x, T val) {
    x = val;
    val += 100;
}

int main() {
    int a = 10, b = 1;
    std::thread t1(&changevalue<int>, std::ref(a), b);
    std::cout << a << "," << b << std::endl;
    t1.join();
}
```

与此类似的，我们在采用 std::bind 绑定含有引用形参的函数时，也需要通过 `std::ref` 来实现引用参数的传递，如：

```c++
void func(int& n2) {
    n2++;
}

int main() {
    int n1 = 0;
    auto bind_fn = std::bind(&func, std::ref(n1));

    bind_fn();
    std::cout << n1 << std::endl; // 1
}
```

## 3. 线程的同步

### 3.1. std::mutex

*std::mutex* 是 *C++11* 中最基本的互斥量，一个线程将 *mutex* 锁住时，其他线程就不能操作该 *mutex*（该线程会被阻塞住，即**自旋**），直至该 *mutex* 被解锁。*std::mutex* 的常用成员函数如下：

| 函数 |作用 |
| :--- | :--- |
| void lock() | 将 *mutex* 上锁（如果调用该函数的线程已经锁住了 *mutex*，则会导致死锁）|
| void unlock() | 解除 *mutex*，释放其所有权（如果调用该函数的线程没有锁住 *mutex* 则会引发未定义的异常） |
| bool try_lock() | 尝试将 *mutex* 上锁，如果 *mutex* 未被上锁，则上锁且返回 true；否则直接返回 false（避免自旋等待）。|

### 3.2. std::atomic

*C++11* 在 \<atomic\> 头文件中引入了原子操作类型，如下表所示：

| 原子操作类型 | 对应的内置类型 |
| :----------- | :----------- |
| atomic_flag | |
| atomic_bool | bool |
| atomic_char | char |
| atomic_schar | signed char |
| atomic_uchar | unsigned char |
| atomic_short | short |
| atomic_ushort | unsigned short |
| atomic_int | int |
| atomic_uint | unsigned int |
| atomic_long | long |
| atomic_ulong | unsigned long |
| atomic_llong | long long |
| atomic_ullong | unsigned long long |
| atomic_char16_t | char16_t |
| atomic_char32_t | char32_t |
| atomic_wchar_t | wchar_t |

当我们去看这些类型的定义时会发现，起始它们都是用 `atomic<T>` 模板来定义的。例如 `std::atomic_llong` 就是用 `std::atomic<long long>` 来定义的。

> 注：**原子操作是平台相关的**，原子类型能够实现原子操作是因为C++11对原子类型的操作进行了抽象，定义了统一的接口，并**要求编译器产生平台相关的原子操作的具体实现**。

#### 3.2.1. 原子类型支持的原子操作

| 操作 | atomic_flag | atomic_bool | atomic_integral-type | atomic\<bool\> | atomic<T*> | atomic\<integral-type\> | atomic\<class-type\> |
| :----| :----------| :----------- | :------------------- | :------------- | :---------- | :----------------------- | :------------------- |
| test_and_set | Y | | | | | | |
| clear | Y | | | | | | |
| is_lock_free | | Y | Y | Y | Y | Y | Y |
| load | | Y | Y | Y | Y | Y | Y |
| store | | Y | Y | Y | Y | Y | Y |
| exchange | | Y | Y | Y | Y | Y | Y |
| compare_exchange_weak +strong | | Y | Y | Y | Y | Y | Y |
| fetch_add, += | | | Y | | Y | Y | |
| fetch_sub, -= | | | Y | | Y | Y | |
| fetch_or, \|= | | Y | | | Y | | |
| fetch_and, &= | | | Y | | | Y | |
| fetch_oxr, ^= | | | Y | | | Y | |
| ++, -- | | | Y | | Y | Y | Y |

## 4. this_thread命令空间

`<thread>` 头文件下不仅定义了 *thread* 类，还提供了一个 *this_thread* 命令空间，此空间提供了一些功能实用的函数，如下表所示：

| 函数 |作用 |
| :--- | :--- |
| get_id() | 获取当前线程的 id |
| yield() | 阻塞当前线程，直至条件成熟 |
| sleep_until() | 阻塞当前线程，直至某个时间点为止 |
| sleep_for() | 阻塞当前线程的时间（例如阻塞5秒）|

[//]:参考资料
[//]:[深入理解C++11笔记：原子类型与原子操作](https://blog.csdn.net/WizardtoH/article/details/81111549)
