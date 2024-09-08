STL六大组件：

（1）容器container：存放数据的数据结构

（2）算法algorithm：可以与容器组合，sort() erase() find()

（3）迭代器iterators：一种泛型指针

（4）适配器adapters：用来修饰容器接口的东西，STL提供的queue和stack，看似容器其实只是容器适配器

（5）配置器allocators：负责空间配置与管理

（6）仿函数functors：行为类似函数



线程安全

（1）C++标准库中的容器没有直接的线程安全保障，需要用std::mutex等线程支持库来加锁保护

（2）C++11开始引入了一些并发数据结构，提供了原子操作和线程安全功能

- std::atomic确保在多线程中对共享变量的读写是原子的
- std::shared_ptr，引用计数的管理是线程安全的，但不能保证对象本身的线程是安全的