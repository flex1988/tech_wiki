# C++ 11 std::promise std::future

c++ 11 的 std::promise 和 std::future 只是两个轻量级的壳子，他们后面共同绑定着一个associated\_state。promise 是发送端的句柄，future 是接收端句柄，主要用于在多线程间的状态同步。

在没有promise 和 future 时，如果你想用另外一个线程执行一个任务，并在任务完成时同步回当前线程，你需要用一个条件变量加一个 mutex以及一个共享的数据来实现这个功能。而 promise 和 future 本质上是把这些事情帮你实现好了。

#### 1 核心实现

* std::promise ：是向”共享状态“写入数据的唯一入口，它代表一个承诺，我保证会把结果放进去
* std::future：是从“共享状态”读取数据的唯一入口，它代表一个期许，我在等这个结果
* shared state：这是标准库在堆上动态分配的一个对象，它内部一般有3个组件：
  * 实际的数据
  * std::mutex
  * std::condition\_variable

#### 2 工作原理

```
#include <iostream>
#include <thread>
#include <future>

void heavy_compute(std::promise<int> prom) {
    // 模拟耗时计算
    std::this_thread::sleep_for(std::chrono::seconds(2));
    // 步骤 3: 计算完成，履行承诺
    prom.set_value(42); 
}

int main() {
    // 步骤 1: 创建 promise (底层在此刻分配了 Shared State)
    std::promise<int> my_promise;
    
    // 步骤 2: 从 promise 获取配套的 future (它们现在指向同一个 Shared State)
    std::future<int> my_future = my_promise.get_future();
    
    // 启动后台线程，把 promise 转移进去 (通常用 std::move)
    std::thread t(heavy_compute, std::move(my_promise));
    
    std::cout << "主线程: 我去喝杯茶，等结果..." << std::endl;
    
    // 步骤 4: 阻塞等待结果
    int result = my_future.get(); 
    
    std::cout << "主线程: 拿到结果了: " << result << std::endl;
    t.join();
    return 0;
}
```

1. 创建期：当 std::promise 实例化时，c++ 运行时会在堆上 new 出一个“共享状态”，此时锁是空闲的，条件变量没有人在等，数据是空的
2. 获取期：调用 get\_future 时，返回的 future 内部存放了指向那个“共享状态”的指针
3. 等待期：future.get
   1. 主线程调用 .get
   2. 底层立刻检查“共享状态”标志位，看数据准备好了没
   3. 没准备好，主线程调用共享状态内部条件变量的 wait，把自己挂起休眠
4. 唤醒期：promise.set\_value
   1. 后台线程算出了 42，调用 .set\_value(42)
   2. 底层获取“共享状态”里的互斥锁
   3. 将 42 写入共享状态的内存中，并将状态标记为“就绪” ready
   4. 最关键的一步：调用 条件变量的 notify\_all
   5. 主线程被内核唤醒，从 .get 中苏醒，拿出42

#### 3 跨线程传递异常

promise/future 不但支持传递值，还可以传递异常。

#### 4 C++ 11 的实现不支持 .then

在主线程调用 .get 时会阻塞住线程，其实更优雅的方式是使用.then 设置一个 callback。然而 c++ 11 的promise/future 一直不支持 .then。原因是c++ 11 没有原生的线程池，所以不能指定 callback 执行的线程，进而导致 .then 行为的不确定性，boost::future folly::future 均支持了 .then。

对于不支持.then的问题，c++标准协议一直没有解决，直到 c++ 20提供了更加强大的协程，为用户提供了更优雅解决这个问题的方案。

