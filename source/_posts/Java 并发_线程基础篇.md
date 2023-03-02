---
title: Java 并发_线程基础篇
date: 2023-03-01
tags: 并发
categories: JavaSE
toc: true
hide: false
sticky: 0
---

{% note blue 'fas fa-bullhorn' %}
内容包括：
1. 线程状态
2. 线程的使用
3. 基础线程机制
4. 线程中断
5. 互斥同步：synchronized、ReentrantLock
6. 线程协作：join()、wait()、notify()/notifyAll()、await()、signal()/signalAll()
{% endnote %}

---

# 线程状态转换

## 六种状态

![](../pictures/Pasted%20image%2020230301115009.png)

- `创建`：创建后未启动
- `可运行`：可能正在运行，也可能正在等待 CPU 时间片，包含操作系统中的两种状态：Running 和 Ready
- `阻塞`：等待获取一个排他锁（也叫写锁、X锁、独占锁），如果其他线程释放了锁，那么当前线程结束此状态
- `无限期等待`：等待其他线程 “显式” 唤醒，否则一直等待
| 进入方法                              | 退出方法                           |
| ------------------------------------- | ---------------------------------- |
| 没有设置 Timeout 参数的 Object.wait() | Object.notify()/Object.notifyAll() |
| 没有设置 Timeout 参数的 Thread.join() | 被调用的线程执行完毕               |
| LockSupport.park()                    | -                                  |

- `限期等待`：一定时间后会被系统自动唤醒
	- `睡眠`：调用 `Thread.sleep()` （静态方法）
	- `挂起`：调用 `Object.wait()` 
- `死亡`：线程完成任务或产生异常

## 补充
- 睡眠和挂起用来描述行为；阻塞和等待描述状态
- 阻塞是被动的，等待获取一个排他锁；等待是主动的，通过 `Thread.sleep()` 或 `Object.wait()` 等方法进入
- 简单总结就是：`sleep()` ——睡眠|等待，`wait()` ——挂起|等待，`synchronized` ——阻塞

# 线程使用方式

有三种方法：
- 实现 `Runnable` 接口
- 实现 `Callable` 接口
- 继承 `Thread` 类

## 实现 Runnable 接口

实现 `run()` 方法，通过 `Thread.start()` 方法启动线程。

```java
public class MyRunnable implements Runnable {
    public void run() {
        // ...
    }
}

```

```java
public static void main(String[] args) {
    MyRunnable instance = new MyRunnable();
    Thread thread = new Thread(instance);
    thread.start();
}
```

## 实现 Callable 接口

与 `Runnable` 相比，`Callable` 可以有返回值，返回值通过 `FutureTask` 封装。

```java
public class MyCallable implements Callable<Integer> {
    public Integer call() {
        return 123;
    }
}
```

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    MyCallable mc = new MyCallable();
    FutureTask<Integer> ft = new FutureTask<>(mc);
    Thread thread = new Thread(ft);
    thread.start();
    System.out.println(ft.get());
}
```

## 继承 Thread 类

同样需要实现 `run()`，因为 `Thread` 类也是实现了 `Runnable` 接口。

调用 `start()` 时，JVM 将线程放入就绪队列中等待被调度，当一个线程被调度时会执行该线程的 `run()` 方法。

## 实现接口 vs 继承 Thread

实现接口更好：
- Java 不支持多继承，继承了 `Thread` 就无法继承其他类
- 继承整个 `Thread` 开销太大

# 基础线程机制

## Executor 框架

*谷歌翻译：Executor，执行者。*

管理多个异步任务的进行，无需显式地管理线程的声明周期。

主要有三种：CacheThreadPool、FixedThreadPool、SingleThreadExecutor

## Daemon（守护线程）

程序运行时在后台提供服务的进程，不属于程序中不可或缺的部分。

所有非守护线程结束时，程序终止，杀死所有守护线程。

**main() 属于非守护线程。**

通过 `setDaemon()` 方法设置守护线程：

```java
public static void main(String[] args) {
    Thread thread = new Thread(new MyRunnable());
    thread.setDaemon(true);
}
```


## sleep()

`sleep()` 方法可能会抛出 `InterruptedException` 异常，而异常不能跨线程传播回 `main()` 中，所以必须在本地进行处理（其他线程也同理），即：

```java
public void run() {
    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

## yield()

*谷歌翻译：yield，屈服。*

`Thread.yield()` 方法声明了当前线程已经完成了生命周期中最重要的部分，可以切换给其他线程来执行。但是这也仅仅是个建议，并且该建议针对的是同等优先级的其他线程。

```java
public void run() {
    Thread.yield();
}
```

# 线程中断

要么正常执行任务结束，要么发生异常提前结束。

## InterruptedException（中断异常）：interrupt()

通过调用一个线程的 `interrupt()` 来中断该线程，如果该线程处于阻塞、等待或超时等待状态，那么会抛出该异常。但不能中断 I/O 阻塞和 synchronized 锁阻塞。

## interrupted()

如果 `run()` 执行的是一个死循环，且没有 `sleep()` 等可以抛出 `InterruptedException` 异常的操作，那么 `interrupt()` 方法就无法使线程提前结束。

```java
public class InterruptExample {
    private static class MyThread2 extends Thread {
        @Override
        public void run() {
            while (!interrupted()) {
                // 默认 interrupted() 返回值为 false，这里是死循环
            }
            System.out.println("Thread end");
        }
    }
}
```

```java
public static void main(String[] args) throws InterruptedException {
	Thread thread2 = new MyThread2();
    thread2.start();
    thread2.interrupt(); // 调用了 interrupt()，会将 Thread 类里面的参数 interrupted 设置为 true，此时 run() 里的 interrupted() 方法返回值变为 ture，终止了死循环的发生
}

// 输出内容：Thread end
```

`interrupted()` 底层使用了本地方法 `clearInterruptEvent()` 完成的死循环的终止操作。

## Executor 的中断操作

调用 `Executor` 里的 `shutdown()` 方法等待线程全部执行完毕后再关闭，如果调用的是 `shutdownNow()` 方法，则相当于调用每个线程的 `interrupt()` 方法。

如果只想中断其中的一个线程，可以使用 `submit()` 方法来提交一个线程，通过返回的 `Future<?>` 对象的 `cancel(true)` 方法，就可以中断。

```java
Future<?> future = executorService.submit(() -> {
    // ..
});
future.cancel(true);
```

# 线程互斥同步

>Java 的两种锁机制：synchronized （由JVM 实现）、ReentrantLock（由 JDK 实现）

## synchronized

可以同步的有：
- 代码块（只能作用于同一个对象，不同的对象去调用该代码块是不会实现同步的）
- 方法（只能作用于一个对象）
- 类（作用于整个类，不同的对象也会同步）
- 静态方法（作用于整个类）

## ReentrantLock

是 java.util.concurrent 中的锁。

## synchronized vs ReentrantLock

比较：
- `synchronized` 由 JVM 实现；`ReentrantLock` 由 JDK 实现
- 新版本的 `synchronized` 经过优化后，性能差不多
- `ReentrantLock` 可以让正在等待的线程放弃等待，即，等待可中断；`synchronized` 则不行
-  `synchronized` 非公平锁；`ReentrantLock` 默认非公平锁，但可以实现公平锁
- `ReentrantLock` 可以绑定多个 `Condition` 对象

使用选择：除非使用 `ReentrantLock` 的高级功能，否则使用 `synchronized`。
- `ReentrantLock` 不是所有的 JDK 版本都支持
- `synchronized` 不用担心死锁问题，JVM 会确保锁的释放

# 线程之间的协作

>多线程解决某个问题时，如果某些线程的任务必须在其他线程开始之前完成，那么就需要对线程进行协调。

## join()

A 线程中调用 B 线程的 `join()` 方法，会将 A 线程挂起，而不是忙等待，直到 B 线程结束，A 线程才会继续执行。这样，就保证了 B 线程能够先于 A 线完成任务。

## wait()、notify()、notifyAll()

属于 Object，而非 Thread。

调用 `wait()` 会将该线程挂起，其他线程通过 `notify()` 或 `notifyAll()` 来唤醒该线程或所有线程。

只能在同步方法或同步控制块（也就是 `synchronized` ）中使用，否则抛出 `IllegalMonitorStateExeception` 异常。

在 `wait()` 挂起期间，线程会释放锁（与之对比，`sleep()` 不会释放锁），原因是如果不让出锁，其他线程就无法通过 `notify()` 或 `notifyAll()` 唤醒该线程，造成死锁。

## await()、signal()、signalAll()

JUC 提供了 `Condition` 类来实现线程之间的协调，在 `Condition` 上调用 `await()` 方法使线程等待，其他线程调用 `signal()` 或 `signalAll()` 来唤醒该线程或所有线程。

`await()` 和 `wait()` 不同的是，前者可以指定等待条件，更灵活。

```java
public class AwaitSignalExample {
    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    public void before() {
        lock.lock();
        try {
            System.out.println("before");
            condition.signalAll();
        } finally {
            lock.unlock();
        }
    }

    public void after() {
        lock.lock();
        try {
            condition.await();
            System.out.println("after");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

```java
public static void main(String[] args) {
    ExecutorService executorService = Executors.newCachedThreadPool();
    AwaitSignalExample example = new AwaitSignalExample();
    executorService.execute(() -> example.after());
    executorService.execute(() -> example.before());
}
```

```java
before
after
```

