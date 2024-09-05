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

```c++
std::timed_mutex mtx;
mtx.try_lock_for(std::chrono::seconds(1));
```

- recursive_mutex：递归函数中锁获取的问题，在递归函数中，递归调用本身时，此前获得的锁可能并未释放，因此导致死锁。而递归锁则可以由一个线程进行多次获取，而且只有unlock了相应的次数后其他线程才能获取到该锁。递归锁存在一个调用次数上限，当达到lock上限时，使用lock()会返回std::system_error，而try_lock()则返回false。



#### Lock Guard

使用资源获取即初始化的方式，自动释放所有本地变量

```c++
void increment() {
    std::lock_guard<std::mutex> lock(mtx); // 当lock_guard被实例化时便成功获取了锁
    x++;
} 
```



#### Unique Lock

支持锁策略

- defer_lock，不获取锁的所有权（ownership），需要显式地进行lock
- try_to_lock，不阻塞线程地尝试获取锁的所有权
- adopt_lock，假设本线程已经获取了锁的所有权



#### Condition Variable

（1）利用线程间共享的全局变量进行同步的一种机制。主要包括两个动作：

- 一个线程因等待条件变量的条件成立而挂起。
  - wait：阻塞当前线程，直到条件变量被唤醒
  - wait_for: 阻塞当前线程，直到条件变量被唤醒或指定时限时长后
  - wait_until：阻塞当前线程，直到条件变量被唤醒，或直到抵达指定时间点
- 另一个线程使条件成立，给出信号，从而唤醒被等待的线程
  - notiy_one通知一个等待的线程
  - notify_all通知所有等待的线程

（2）虚假唤醒：wait因为操作系统的原因不满足条件时也会被虚假唤醒

- 内核层面：当你调用notify_one/signal_one等方法时，操作系统并不保证只唤醒一个线程

```c++
while (!(xxx条件) )
{
    //虚假唤醒发生，由于while循环，再次检查条件是否满足，
    //否则继续等待，解决虚假唤醒
    wait();  
}
```





#### promise 和 future

用法一：子线程赋值主线程获取

```c++
void task(int a, int b, std::promise<int>& ret){
    int ret_a = a*a;
    int ret_b = b*2;
    
    ret.set_value(ret_a+ret_b);
}

int main(){
    std::promise<int> p;
    std::future<int> f = p.get_future();
    
    std::thread t(task,1,2,std::ref(p));
    // 一直阻塞
    std::cout<<f.get();
    t.join();
}
```

用法二：主线程赋值子线程获取

```c++
void task(int a, std::future<int>& b, std::promise<int>& ret){
    int ret_a = a*a;
    int ret_b = b.get()*2;
    ret.set_value(ret_a+ret_b);
}

int main(){
    std::promise<int> p_ret;
    std::future<int> f_ret = p_ret.get_future();
    
    std::promise<int> p_in;
    std::future<int> f_in = p_in.get_future();
    std::thread t(task,1,std::ref(f_in),std::ref(p_ret));
    
    p_in.set_val(2);
    std::cout<<f_ret.get();
    t.join();
}
```

get只能获取一次，可以用shared_feature获取多次



#### Packaged Task

包装了一个可调用的目标以便异步调用

promise保存了一个共享状态的值，packaged_task保存的是一个函数，他们内部都有future以便访问异步操作结果

```c++
std::packaged_task<int()> task([](){return 7;});
std::thread t1(std::ref(task));
std::future<int> f1 = task.get_future();
auto r1 = f1.get();
```



#### Async

std::async先将异步操作用std::packaged_task包起来，然后将异步操作的结果放到std::promise重，外面再通过future.get/wait来获取这个未来的结果

原型是async(std::launch::async | std::launch::deferred, f, args)

- std::launch::async：在调用async就开始创建线程
- std::launch::deferred：延迟加载方式创建线程。调用async时不创建线程，直到调用了future的get或者wait时才创建线程。

```c++
std::future<int> f1 = std::async(std::launch::async, [](){
        return 8; 
    });
 
cout<<f1.get()<<endl;
```



#### Atomic

每次操作这个变量，包括读取值更改值，这次操作中执行的唯一一条指令是原子的

（1）原子操作：只能用于纯粹的数据(可以用memcpy拷贝 没有虚函数 构造函数noexcept)

```c++
// 一般推荐使用std::atomic提供的函数而不是数学运算符
std::atomic_int x{1};
x += 1;    //atomic operation
x = x + 1; // not atomic
```

（2）memory order：

- memory_order_relaxed，保证atomic变量本身的访问是原子的
- memory_order_acquired，除了保证原子方存之外还保证排在原子操作之后的读写指令不会由于CPU乱序执行而在原子操作之前被执行
- memory_order_release，与acquired类似，保证排在原子操作之前的读写指令不会由于CPU乱序执行而在原子操作之前被执行

（3）atomic底层原理：锁总线(这里的总线是虚拟的一条总线不一定需要锁物理的总线)，还可以借助于底层的Test And Exchange指令



参考文献：

1. [C++多线程入门](https://zhuanlan.zhihu.com/p/556406170) 
1. [深入浅出c++11 std::async](https://www.cnblogs.com/chengyuanchun/p/5394843.html)
1. [浅析C++ atomic](https://zhuanlan.zhihu.com/p/649663363) 
1. [C++中，std::atomic是真正的原子吗](https://www.zhihu.com/question/469476598/answer/1976150981)