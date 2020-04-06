---
layout: post
tags : [java, 并发]
title: Java并发学习

---

## 线程

### 创建线程

`Thread t = new Thread(runnableObject)`

其中runnableObject是实现了接口Runnable的对象, 接口定义如下:

```java
public interface Runnable {
  void run();
}
```

### 线程状态:

* 新建 new: `new Thread(r)`
* 可运行 runnable: 调用start后进入可运行状态, 受操作系统调度控制, runnable线程在某个时刻可能是运行也可能没有运行
* 阻塞 blocked: 试图获得一个被其他线程占用的内部锁
* 等待 waiting: 等待另一个线程通知调度器的一个条件, 如`Object.wait` `Thread.join`或等待`java.util.concurrent`的Lock/Condition
* 计时等待 timed_waiting: 带超时参数的方法: `Thead.sleep` `Object.wait` `Thread.join` `Lock.tryLock` `Condition.await`
* 终止 terminated: run正常结束; 未捕获异常导致run结束

### 方法:


#### 静态方法:

* `Thread.sleep(毫秒)`

  sleep的线程进入阻塞状态

  阻塞的线程(sleep/wait)如果调用interrupt, 会造成InterruptedException异常, 需要try catch. 阻塞的调用会被InterruptedException中断, 并且在抛出异常后立即将线程的中断标示位清除，即重新设置为false

  中断的线程并不是终止, 线程可以自行决定后续流程

* `Thread.currentThread`: 当前执行的线程对象

* `Thread.interrupted` 判断当前线程是否终端, 同时此方法会清除该线程的中断状态

#### 成员方法:

* `start`

* `isInterrupted` 判断线程的中断状态, 不会改变中断状态

* `interrupt` 向线程发送中断请求, 将会设置该线程的中断状态位，即设置为true.

  通常线程的执行代码中应该去(不断)检查中断状态(`Thread#isInterrupted`), 如果出现中断, 应该有相应处理

  如果线程当前阻塞, 会造成InterruptedException异常

* `join()` 当前线程等待指定线程终止

* `join(毫秒)`

* `Thread.State getState()` 获得线程状态

* `setDaemon(boolean)` 设置是否为守护线程

* `setUncaughtExceptionHandler(Thread.UncaughtExceptionHandler eh)` 未捕获异常处理器: TODO

---

## 锁

### 类 ReentrantLock

实现了接口Serializable, Lock

临界区: lock和unlock之间的代码, 临界区通常需要try finally, 用以确保unlock

多个线程对**同一个**锁的lock请求, 可以将原来非同步的线程控制为同步的线程(临界区的代码).

锁是**可重入的**, 一个线程可以重复lock已经持有的锁, 锁保持一个**持有计数**来跟踪锁的嵌套使用, 线程每次lock都要有相应的unlock来释放锁.

#### 成员方法

* `lock()`
* `unlock()`

### 接口 Condition

条件对象: 线程进入临界区后还需要满足一些前置条件才能执行, java中通过条件对象进行控制

创建: `Interface Lock`#`Condition newCondition()`

一个锁可以有多个相关的条件对象

常规的使用方式:

```
theLock = new ReentrantLock()
conditionOfTheLock = theLock.newCondition()

......

theLock.lock()
try {
  while !(ok to process)
    conditionOfTheLock.await() // 当前线程放弃了锁, 进入waiting状态. 进入条件等待集

  ...dowork...
  conditionOfTheLock.signalAll() // 激活所有等待该条件的线程, 把线程从等待集合中移出, 成为可运行状态
}
finally {
  theLock.unlock()
}
```

* `await()` 会放弃锁, 进入waiting状态, 进入条件等待集, 当被signalAll唤醒后, 且锁可用时, 只有一个waiting线程会从await中返回.
* `signalAll()`
* `signal()`

---

## 内部锁

`synchronized` 关键字用于方法前, 用于保护该对象的方法同步访问. 成员方法和静态方法都可以使用synchronized关键字.

`synchronized` 方法中`wait` `notifyAll`用于实现条件对象访问.

---

## 接口 BlockingQueue

多线程问题, 可以通过一个或者多个队列转换为安全的同步处理模式.

| 应用场景 | 方法    | 作用               | 异常处理              |
|----------|---------|--------------------|-----------------------|
| 线程管理 | put     | 添加元素           | 如果队列满, 则阻塞    |
| 线程管理 | take    | 移除并返回头元素   | 如果队列空, 则阻塞    |
| 强制异常 | add     | 添加元素           | 如果队列满, 抛出异常  |
| 强制异常 | remove  | 移除并返回头元素   | 如果队列空, 抛出异常  |
| 强制异常 | element | 返回头元素         | 如果队列空, 抛出异常  |
| 普通用法 | offer   | 添加元素, 返回true | 如果队列满, 返回false |
| 普通用法 | poll    | 移除并返回头元素   | 如果队列空, 返回null  |
| 普通用法 | peek    | 返回头元素         | 如果队列空, 返回null  |


用作线程管理工具:

* put: 添加元素;  如果队列满, 则阻塞
* take: 移除并返回头元素;








