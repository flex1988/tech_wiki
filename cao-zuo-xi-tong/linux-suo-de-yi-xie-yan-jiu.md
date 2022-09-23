# Linux锁的一些研究

#### 锁能解决什么问题

在讨论锁的实现机制之前，我们可能需要知道为什么需要锁，锁能解决什么问题？

锁存在的意义在于解决了，多核并发时对于内存数据的操作的一致性的问题（事实上就算单核也有这个问题，因为中断可以影响到线程的执行）。

比如下面这段代码：

```c
#include <pthread.h>
#include <stdio.h>

static int count;
void *add_count() {
    int i;
    for (i = 0; i < 10000;i++) {
        count++;
    }
    return NULL;
}

int main() {
    count = 0;
    pthread_t t1, t2;
    pthread_create(&t1, NULL, add_count, NULL);
    pthread_create(&t2, NULL, add_count, NULL);
    pthread_join(t1,NULL);
    pthread_join(t2,NULL);

    printf("count is %d\n", count);
}
```

这个程序很简单，我起了两个线程，每个线程都对count加一，讲道理结果应该是20000，然后执行的结果并不是，而且每次的结果都不一样。

这是因为count++这个操作并不是原子的，它需要经过以下的步骤：

1. 首先寄存器把内存中count的值取出存到寄存器
2. 对寄存器中的值加一
3. 将寄存器中的新值写回到count的内存地址

由于每个线程的寄存器的状态都是独立的，在多个线程并发加一时，假如t1线程加一之后的值还没写回去的时候，t2又读取了地址指向的值，那么接下来t2会将t1写入的值覆盖，数就不对了。

解决这个问题也很简单用锁就可以了，下面是用锁版本的代码：

```c
#include <pthread.h>
#include <stdio.h>

static int count;
pthread_mutex_t m;
void *add_count() {
    int i;
    for (i = 0; i < 10000;i++) {
        pthread_mutex_lock(&m);
        count++;
        pthread_mutex_unlock(&m);
    }
    return NULL;
}

int main() {
    count = 0;
    pthread_t t1, t2;

    pthread_mutex_init(&m,NULL);

    pthread_create(&t1, NULL, add_count, NULL);
    pthread_create(&t2, NULL, add_count, NULL);
    pthread_join(t1,NULL);
    pthread_join(t2,NULL);

    printf("count is %d\n", count);
}
```

你每次运行它，你会发现结果都是20000，这里我们用了mutex来解决问题，mutex能够提供对于临界区(Critical Section)互斥的访问。

除了mutex之外，还有很多其他机制提供了并发线程对于临界区的同步操作，比如semaphore、spinlock、condition等。

#### 同步原语

锁的实现离不开同步原语的支持，而同步原语可以保证原子性的操作，而同步原语的实现需要硬件的支持。

**Test\_and\_Set**

test\_and\_set是实现锁的一种原语，它能够提供一种原子性的操作，这个操作可以将内存中的值和你给的值互换

```c
int test_and_set(int *p,int val){
    int r = *p;
    *p = val;
    return r;
}
```

用c语言来描述就是上面这个样子，test\_and\_set的实现需要汇编指令前缀lock和指令xchg的支持，lock可以禁止系统总线对于指定内存的访问，而xchg可以交换两个值。

上面的操作其实是原子性的用val的值去和内存中的一个值去做交换，假如我们设定0是未锁，1是锁。

那么初始\*p的值都是0表示未锁，假设这时线程t1和t2同时去test\_and\_set，由于lock的存在所以能够保证不会有两个线程同时lock成功。

假设t1 lock成功，并用1去和内存中的值0交换，得到的值是0，表示t1取到了这把锁，他可以对count进行加一的操作，然后释放锁。

而t2 lock失败，只能继续retry，直到交换出0，表示它得到锁。

**Compare\_and\_Swap**

Compare\_and\_Swap比Test\_and\_Set更加强大，很多现代的锁等机制都是用CAS来实现的。

CAS的意思就是先比较两个值，假如内存的值等于你给的值1，那么将值2写入到内存中，用c语言来描述是这个样子：

```c
int compare_and_swap(int *p,int old,int new){
    int r = *p;
    if(r == old) *p = new;
    return r;
}
```

#### 实现锁

好了，有了上面的同步原语，那么我们实现锁就方便了,下面我们先来实现一个简单的锁。

```c
void lock(int *p) {
    while (!__sync_bool_compare_and_swap(p, 0, 1))
        ;
}

void unlock(int *p) { *p = 0; }
```

gcc提供了一些builtins的实现，所以我们可以直接用\_\_sync\_bool\_compare\_and\_swap。

在cas的帮助下，我们实现了一把锁，将原始的程序修改如下：

```c
#include <pthread.h>
#include <stdio.h>
#include <stdbool.h>

static int count;

void lock(int *p) {
    while (!__sync_bool_compare_and_swap(p, 0, 1))
        ;
}

void unlock(int *p) { *p = 0; }

int lock_t;

void *add_count() {
    int i;
    for (i = 0; i < 10000; i++) {
        lock(&lock_t);
        count++;
        unlock(&lock_t);
    }
    return NULL;
}

int main() {
    count = 0;
    pthread_t t1, t2;

    lock_t = 0;

    pthread_create(&t1, NULL, add_count, NULL);
    pthread_create(&t2, NULL, add_count, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("count is %d\n", count);
}
```

编译运行发现每次结果都是20000，我们的锁实现的没有问题，然而这样代码却导致了一个问题while (!\_\_sync\_bool\_compare\_and\_swap(p, 0, 1))。

就是当没有获取到锁的时候，线程是继续试着去抢锁的，事实上这是一个自旋锁(spin lock)。

在临界区很小情况下，自旋锁是很适合的，因为试几次可能就会抢到锁，避免了频繁的上下文切换(context switch)。

而对于临界区很大的情况来说，我们更好的方式是让没取到锁的线程睡眠，假如释放锁的时候还有睡眠的线程在等这个锁，那么唤醒这个线程。

#### 进化

我们的考虑是这样的，每个去lock这把锁的线程都会有两种结果，假设失败，那么我们准备一个阻塞队列保存所有需要这把锁的线程。

为了解决饿死问题，我们希望搞一个优先级队列，先进入睡眠的线程优先抢到锁。

在取到锁的线程释放锁时检查这个队列时，假如队列不空，那么取出第一个线程并唤醒它。

```c
#define _GNU_SOURCE
#include <errno.h>
#include <pthread.h>
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include <linux/futex.h>
#include <sys/syscall.h>

static int count;
struct lock_t _l;

static int futex(int *uaddr, int futex_op, int val, const struct timespec *timeout, int *uaddr2, int val3) {
    return syscall(SYS_futex, uaddr, futex_op, val, timeout, uaddr, val3);
}

struct thread_node {
    pthread_t id;
    int flock;
    pthread_t *t;
    struct thread_node *next;
};

struct lock_t {
    int lock;
    struct thread_node *node;
};

void lock(struct lock_t *l) {
    if (!__sync_bool_compare_and_swap(&l->lock, 0, 1)) {
        struct thread_node *n = malloc(sizeof(struct thread_node));
        n->id = pthread_self();
        n->flock = 0;
        n->next = NULL;
        if (l->node == NULL)
            l->node = n;
        else {
            n->next = l->node;
            l->node = n;
        }

        futex(&n->flock, FUTEX_WAIT, 0, NULL, NULL, 0);
    }
}

void unlock(struct lock_t *l) {
    if (l->node != NULL) {
        struct thread_node *n = l->node;
        l->node = n->next;
        n->flock = 1;

        futex(&n->flock, FUTEX_WAKE, 1, NULL, NULL, 0);
        free(n);
    } else {
        l->lock = 0;
    }
}

void *add_count() {
    int i;
    for (i = 0; i < 10000; i++) {
        lock(&_l);
        count++;
        printf("%d\n", count);
        unlock(&_l);
    }
    return NULL;
}

int main() {
    count = 0;
    pthread_t t1, t2, t3, t4, t5;

    _l.node = NULL;
    _l.lock = 0;

    pthread_create(&t1, NULL, add_count, NULL);
    pthread_create(&t2, NULL, add_count, NULL);
    pthread_create(&t3, NULL, add_count, NULL);
    pthread_create(&t4, NULL, add_count, NULL);
    pthread_create(&t5, NULL, add_count, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    pthread_join(t3, NULL);
    pthread_join(t4, NULL);
    pthread_join(t5, NULL);

    printf("count is %d\n", count);
}
```

futex是linux下的快速同步互斥机制，我们用它来实现线程的sleep和awake，并用队列保存线程的进入次序，避免了线程的饿死。
