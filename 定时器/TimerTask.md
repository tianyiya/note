## 使用

```markdown
Timer timer = new Timer();
time.schedule(timerTask, 1000, 2000);
```

## 缺点

1、首先Timer对调度的支持是基于绝对时间的，而不是相对时间，所以它对系统时间的改变非常敏感。

2、其次Timer线程是不会捕获异常的，如果TimerTask抛出的了未检查异常则会导致Timer线程终止，同时Timer也不会重新恢复线程的执行，
他会错误的认为整个Timer线程都会取消。同时，已经被安排单尚未执行的TimerTask也不会再执行了，
新的任务也不能被调度。故如果TimerTask抛出未检查的异常，Timer将会产生无法预料的行为

3、Timer在执行定时任务时只会创建一个线程任务，如果存在多个线程，若其中某个线程因为某种原因而导致线程任务执行时间过长，
超过了两个任务的间隔时间，会导致下一个任务执行时间滞后

4. 非线程安全的

## 添加任务

```text
void add(TimerTask task) {
    // Grow backing store if necessary
    if (size + 1 == queue.length)
        queue = Arrays.copyOf(queue, 2*queue.length);

    queue[++size] = task;
    fixUp(size);
}
```

## 执行任务

```text
 private void mainLoop() {
    while (true) {
        try {
            TimerTask task;
            boolean taskFired;
            synchronized(queue) {
                // Wait for queue to become non-empty
                while (queue.isEmpty() && newTasksMayBeScheduled)
                    queue.wait();
                if (queue.isEmpty())
                    break; // Queue is empty and will forever remain; die

                // Queue nonempty; look at first evt and do the right thing
                long currentTime, executionTime;
                task = queue.getMin();
                synchronized(task.lock) {
                    if (task.state == TimerTask.CANCELLED) {
                        queue.removeMin();
                        continue;  // No action required, poll queue again
                    }
                    currentTime = System.currentTimeMillis();
                    executionTime = task.nextExecutionTime;
                    if (taskFired = (executionTime<=currentTime)) {
                        if (task.period == 0) { // Non-repeating, remove
                            queue.removeMin();
                            task.state = TimerTask.EXECUTED;
                        } else { // Repeating task, reschedule
                            queue.rescheduleMin(
                              task.period<0 ? currentTime   - task.period
                                            : executionTime + task.period);
                        }
                    }
                }
                if (!taskFired) // Task hasn't yet fired; wait
                    queue.wait(executionTime - currentTime);
            }
            if (taskFired)  // Task fired; run it, holding no locks
                task.run();
        } catch(InterruptedException e) {
        }
    }
}
```

## 底层调用

```markdown
Object.wait(timeout) -> pthread_cond_timedwait()

pthread_cond_timedwait()，用于在指定的时间内等待条件变量被唤醒；
pthread_mutex_unlock()，用于释放互斥锁；
pthread_mutex_lock()，用于获取互斥锁；
clock_gettime()，用于获取当前时间。
```

```text
pthread_cond_timedwait()和futex()都是用于实现线程同步的系统调用，但是它们的实现方式有所不同。

pthread_cond_timedwait()是POSIX线程库中的一个函数，用于在指定的条件变量上等待，如果条件变量没有被通知就会一直阻塞，直到等待超时或者收到通知才会被唤醒。
在等待过程中，该函数会将调用线程加入到条件变量的等待队列中，当条件变量被通知后，调用pthread_cond_timedwait()的线程就会从等待队列中被移除并唤醒。
该函数需要与互斥锁结合使用，以避免条件变量的竞态条件。

futex()是Linux内核提供的一个系统调用，它是一个快速用户空间互斥量（Fast User-space Mutex，FUTEX）的实现。
该函数主要用于实现同步原语，包括等待（wait）和唤醒（wake），以及比较和交换（compare-and-swap）等操作。
在等待过程中，该函数会将调用线程挂起，直到被唤醒才会继续执行。
相比于pthread_cond_timedwait()，futex()实现了更为高效的线程同步机制，同时避免了额外的系统调用和上下文切换。

在Linux系统中，pthread_cond_timedwait()的实现通常会调用futex()函数。
在等待过程中，pthread_cond_timedwait()会先将线程挂起并将其加入到条件变量的等待队列中，然后通过futex()函数来等待条件变量的唤醒信号。
当条件变量被通知后，pthread_cond_timedwait()会通过futex()函数唤醒挂起的线程，从而实现线程同步。
```