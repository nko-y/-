主要参考自[C++多线程入门](https://zhuanlan.zhihu.com/p/556406170) 



#### 创建一个线程

函数指针

```c++
void countdown(int x) {
    while (true) std::cout << x << std::endl;
}

int main() {
    std::thread t(countdown, 10); // 使用函数指针创建线程
}
```

Lambda函数：

```c++
int main() {
    std::thread t([] (int x) {
        while (true) std::cout << x << std::endl;
    }, 10); // 使用Lambda函数创建线程
}
```

函数对象

```c++
class Base {
public:
    void operator() (int x) {
        while (true) std::cout << x << std::endl;
    }
};

int main() {
    std::thread t(Base(), 10); // 使用函数对象指针创建线程
}
```



#### Join和Detach

thread.join()的含义是等待（类似js中的await）该线程执行结束，可以使用thread.joinable()来检查该线程是否可调用join()

thread.detach()的含义是将子线程从父线程中分离出去



#### Mutex(互斥量)

（1）两个线程同时对一个关键变量（Critical Section，即只能有一个线程或进程同时进行访问的一个或一组变量或状态）进行读写的问题

```c++
#include <iostream>
#include <thread>
#include <mutex>

int x = 0;
std::mutex mtx;

void increment() {
    mtx.lock();
    x++;
    mtx.unlock();
}

int main() {

    std::thread t1(increment);
    std::thread t2(increment);

    t1.join();
    t2.join();

    std::cout << "x = " << x << std::endl;

    return 0;
}
```

（2）类型：

- try_lock：尝试获得互斥锁成功返回true失败返回false，不会阻塞线程

- timed_mutex：阻塞线程直到timeout或者成功返回锁



try_lock























参考文献：

1. [C++多线程入门](https://zhuanlan.zhihu.com/p/556406170) 