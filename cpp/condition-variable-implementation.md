# 条件变量（Condition Variable）的实现原理

在 C++ 里，std::condition\_variable 本质上是对操作系统底层同步原语（pthread\_cond\_t / futex）的零开销封装。

它的核心职责是：高效的让线程去休眠，并在合适的时机被别的线程唤醒。

#### 1 条件变量内部

条件变量内部通常有三个核心组件：

1. 等待队列（Wait Queue）: 一个保存了正在休眠的线程 id 的链表
2. 内部自旋锁：用来保护等待队列多线程并发修改
3. 内核接口：用户挂起和唤醒线程，在linux 上就是 futex

#### 2 cv.wait() 内部

为什么条件变量的 wait 一定要和 std::mutex 一起使用？

当你调用 cv.wait(lock)时，底层发生了这些事情：

1. 获取内部锁：cv锁住自旋锁
2. 入队：将当前线程 id 加入 cv wait queue
3. 解锁外部 mutex
4. 进入休眠：调用 futex\_wait
5. 等待
6. 被唤醒：接收到通知，当前线程被加入到可运行队列
7. 重新加锁：在wait 返回前，重新mutex加锁

#### 3 cv.notify\_one 内部

1. 获取内部自旋锁
2. 出队，从等待队列的头部，拿一个正在休眠的线程
3. 唤醒，调用 os 接口（futex\_wake），唤醒线程
4. 释放内部锁

#### 4 伪代码

```
struct condition_variable {
    queue_t wait_queue;
    spinlock_t internal_lock;
};

void cv_wait(struct condition_variable* cv, struct mutex* user_mutex) {
    spin_lock(&cv->internal_lock);
    enqueue(&cv->wait_queue, current_thread_id());
    
    os_unlock_and_sleep(user_mutex, &cv->internal_lock);
    
    mutex_lock(user_mutex);
}

void cv_notify_one(struct condition_variable* cv) {
    spin_lock(&cv->internal_lock);
    if (!is_empty(&cv->wait_queue)) {
        thread_id_t tid = dequeue(&cv->wait_queue);
        os_wake_thread(tid);
    }
    spin_unlock(&cv->internal_lock);
}
```

#### 5 虚假唤醒

cv 有个写法是必须要用 while 循环判断来包住，比如：

```
while (!data_ready) {
    cv.wait(lock);
}

而不能
if (!data_ready) {
    cv.wait(lock);
}
```

为什么线程被唤醒了，但是数据确没有准备好呢？

1. 信号窃取，线程醒来抢锁没有抢到，有一个其他线程先抢到了锁取走了数据
2. 内核里在某些场景真的会发生虚假唤醒，比如不可控的系统级中断等

#### QA

1. 为什么 CV  要和 mutex 一起用
   1. cv 变量总是和一个共享数据一起使用，mutex 用户保护数据修改互斥
   2. 防止丢失唤醒，在 cv.wait 内部内核保证线程睡去和释放锁时原子的，不会有另外一个参与者进来修改状态（比如消费者发现没数据cv.wait前线程切换了，生产者产生了数据交到了队列里，这时消费者切回来 wait，这就永远不会醒来了）
