## 实现

`Delayed`

## 成员

```markdown
private final transient ReentrantLock lock = new ReentrantLock();
// 优先级队列，存放工作任务。
private final PriorityQueue<E> q = new PriorityQueue<E>();
// 等待队列头的元素的线程
private Thread leader = null;
// 依赖于重入锁的condition。
private final Condition available = lock.newCondition();
```

## 放入元素

```markdown
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        q.offer(e);
        if (q.peek() == e) {
            leader = null;
            available.signal();
        }
        return true;
    } finally {
        lock.unlock();
    }
}
```

## 获取元素

```markdown
public E take() throws InterruptedException {
    // 获取锁。每个延迟队列内聚了一个重入锁。
    final ReentrantLock lock = this.lock;
    // 获取可中断的锁。
    lock.lockInterruptibly();
    try {
        for (;;) {
            // 尝试从优先级队列中获取队列头部元素，获取但不移除
            E first = q.peek();
            if (first == null)
                //无元素，当前线程节点加入等待队列，并阻塞当前线程
                available.await();
            else {
                // 通过延迟任务的getDelay()方法获取延迟时间
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0)
                    //延迟时间到期，获取并删除头部元素。
                    return q.poll();
                first = null; // don't retain ref while waiting
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        // 线程节点进入等待队列 x 纳秒。
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        // 若还存在元素的话，则将等待队列头节点中的线程节点移动到同步队列中。
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
```

## 线程阻塞特定时间

```markdown
Condition.awaitNanos -> LockSupport.parkNanos -> unsafe.parkNanos ->futex
```

```markdown
futex 是一种 Linux 内核提供的用户空间与内核空间之间用于线程同步的原语。

futex 全称为 "Fast Userspace Mutex"，旨在提供一种高效的线程同步机制， 它是通过系统调用 syscall() 在用户空间和内核空间之间进行通信的。

futex 的主要思想是将线程的阻塞操作从内核空间转移到用户空间，从而避免了不必要的上下文切换和系统调用。

具体来说，当一个线程调用 futex() 函数时，它会将自己的状态设置为阻塞，并陷入内核空间等待条件变量的触发。

如果此时条件变量的值与线程期望的值不一致，则该线程会一直等待。

当条件变量被触发后，内核会将状态设置为唤醒，并且将该线程从内核空间返回到用户空间继续执行。

futex 的优点在于它避免了内核空间和用户空间之间的频繁切换和系统调用，从而提高了线程同步的效率。

同时，由于 futex 可以通过等待条件变量来实现阻塞和唤醒操作，因此它还可以支持一些高级的线程同步机制，比如信号量和读写锁等。
```

```markdown
从性能上来讲，定时等待的确比非定时等待更加耗费性能。这是因为在定时等待过程中，
系统需要定期地唤醒线程并进行一些额外的操作，例如更新定时器、检查超时时间等等，这些操作都会增加系统的负担和开销。

不过，需要注意的是，对于大多数应用来说，性能的影响通常是非常小的，
因为定时等待的唤醒间隔一般都比较长，通常在几毫秒甚至几十毫秒级别，
而对于一个现代的计算机系统来说，这个时间非常短，对于整体性能的影响很小。
```

## leader的作用

最大限度的减少定时等待，即只有leader线程进行定时等待即可，其它线程使用非定时等待