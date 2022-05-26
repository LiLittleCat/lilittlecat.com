---
title: Java 多线程与并发 - ReentrantLock
tags:
  - Java
categories:
  - - Tech
abbrlink: 28a0bbad
date: 2022-05-22 16:11:18
---

前文 {% post_link Java-多线程与并发-synchronized Java 多线程与并发 - synchronized %} 中介绍了关键字 `synchronized` 来实现同步访问，在 Java 5 之后，java.util.concurrent.locks 包下提供了另外一种方式来实现同步访问，那就是 `Lock`。

本文通过 `Lock` 的常用实现类 `ReentrantLock`，介绍了 `Lock` 的核心概念、`ReentrantLock` 的核心机制以及其公平锁和非公平锁实现的原理。

<!-- more -->

# Lock 核心概念

`Lock` 和 `synchronized` 都可以实现线程同步，`Lock` 提供的功能更多，能应对更复杂的场景，使用也更复杂。它们的功能对比如下：

| 功能                 | synchronized | Lock |
| -------------------- | ------------ | ---- |
| 加锁（等待）         | ✅            | ✅    |
| 尝试加锁（不等待）   | ❌            | ✅    |
| 尝试加锁（限时等待） | ❌            | ✅    |
| 让出锁（等待）       | ✅            | ✅    |
| 唤醒                 | ✅            | ✅    |
| 多条件               | ❌            | ✅    |
| 公平锁               | ❌            | ✅    |
| 释放锁               | ✅            | ✅    |

## Lock 组成结构

<center>
    <img style="border-radius: 8px;
    box-shadow: 0 5px 15px 0px rgba(0, 0, 0, .4);"
    src="https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/drawio/Lock-and-Condition.svg" width = "80%" alt=""/>
    <div style="
  color: #999;
  font-size: 0.875em;
  font-weight: bold;
  line-height: 1;
  margin: 0 auto 15px;
  text-align: center;">
      Lock and Condition
    </div>
</center>

### Lock 锁

JUC 中的锁由 `Lock` 和 `Condition` 组成，`Lock` 负责加锁和释放锁，确保锁同一时刻只能被 一个线程持有。与 `synchronized`  类似，当 `Lock` 锁被一个线程占用时，其他获取该锁的线程会被阻塞，直到锁被释放。不同之处在于 `synchronized` 针对的是其后跟的对象，而 `Lock` 是锁对象本身。

### Condition 条件

`Lock` 锁可以基于业务构建多个 `Condition`，当线程获取锁之后有权操作 `Condition`，没有获取锁时如果操作 `Condition` 会和 `synchronized` 代码块之外调用 Object.wait() 一样报 `IllegalMonitorStateException`。

用一个例子来理解 `Condition`：

假设有一个产品容器，5 个生产者线程负责生产产品到容器中，5 个消费者线程负责从容器中取走产品。生产者线程运行的的条件是 **容器未满**，消费者运行的条件是 **容器非空**，否则阻塞等待。

<center>
    <img style="border-radius: 8px;
    box-shadow: 0 5px 15px 0px rgba(0, 0, 0, .4);"
    src="https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/drawio/Condition-sample.svg" width = "80%" alt=""/>
    <div style="
  color: #999;
  font-size: 0.875em;
  font-weight: bold;
  line-height: 1;
  margin: 0 auto 15px;
  text-align: center;">
      Condition sample 1
    </div>
</center>

如果使用 `synchronized` 实现，当某一个线程获取锁之后，如果其他线程不满足运行条件，调用 `Object.wait()` 进入等待集，线程释放锁之后使用 `Object.notify()` 唤醒等待集中的线程。此时无法确定唤醒的是生产者线程还是消费者线程，因为他们都会放入同一个等待集中，`Object.notify()` 之后和等待获取锁的阻塞线程一起竞争锁。如果等待集中的线程获得了锁，还要继续检测是否满足其运行条件，不满足继续 wait。

<center>
    <img style="border-radius: 8px;
    box-shadow: 0 5px 15px 0px rgba(0, 0, 0, .4);"
    src="https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/drawio/Condition-sample-1.svg" width = "80%" alt=""/>
    <div style="
  color: #999;
  font-size: 0.875em;
  font-weight: bold;
  line-height: 1;
  margin: 0 auto 15px;
  text-align: center;">
      Condition sample 2
    </div>
</center>

上述可知，在这种场景下 `synchronized` 存在局限性。

`Lock` 最大不同是可以有多个条件，每个条件都有一个等待集，可唤醒指定等待集中的线程。在上例中当生产完成之后，指定唤醒消费者的条件队列，而当消费完成后指定唤醒生产者的条件队列。

示例代码如下：

```java
public class LockTest {
    private final Lock lock = new ReentrantLock();
    // 容器未满
    private final Condition notFull = lock.newCondition();
    // 容器非空
    private final Condition notEmpty = lock.newCondition();
    // 容器
    private final Object[] items = new Object[20];

    int putIndex;
    int takeIndex;
    int count;

    public void put(Object o) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                // 容器满了，阻塞生产者
                notFull.await();
            }
            if (++putIndex == items.length) {
                putIndex = 0;
            }
            ++count;
            // 放入产品，唤醒消费者
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                // 容器为空，阻塞生产者消费者
                notEmpty.await();
            }
            final Object item = items[takeIndex];
            if (++takeIndex == items.length) {
                takeIndex = 0;
            }
            --count;
            // 取走产品，唤醒生产者
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }

}
```

写个测试测试一下：

```java
@Test
public void test() throws InterruptedException {
    final LockTest lockTest = new LockTest();
    final ExecutorService producer = Executors.newFixedThreadPool(5);
    final ExecutorService consumer = Executors.newFixedThreadPool(5);
    for (int i = 0; i < 25; i++) {
        String name = "put-" + i;
        producer.submit(() -> {
            try {
                System.out.println("生产：" + name);
                lockTest.put(name);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
    }
    Thread.sleep(10);
    System.out.println("容器中有 20 个产品，生产者线程 5 个等待中");
    for (int i = 0; i < 25; i++) {
        consumer.submit(() -> {
            try {
                System.out.printf("消费：%s\n", lockTest.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
    }
    Thread.sleep(10);
    System.out.println("容器中有 0 个产品，消费者线程 5 个等待中");
}
```

# ReentrantLock 核心机制

## 核心机制

`ReentrantLock` 称为可重入锁，是指一个线程可以**重复获取该锁**，即调用 `lock()` 方法后，没有 `unlock()` 之前再次调用 `lock()` 可取锁。可重入机制可以防止方法递归时发生死锁现象。

## 死锁发生

假设我们自定义一个不可重用的 `Lock`，下面的代码就会发生死锁

```java
public void method() {
    lock.lock();
    try {
        lock.lock();
    } finally {
        lock.unlock();
    }
}
```

第一次调用 `lock()` 状态为占用，第二次获取锁时因为被占用，将会进入阻塞队列，导致永远无法调用 `unlock()` 解锁。

代码示例：

```java
@Test
public void test1() {
    final MyLock myLock = new MyLock();
    lock.lock();
    try {
        lock.lock();
    } finally {
        lock.unlock();
    }
}

private static class MyLock extends AbstractQueuedSynchronizer {
    public void lock() {
        acquire(1);
    }

    public void unlock() {
        release(1);
    }

    @Override
    protected boolean tryAcquire(int arg) {
        return compareAndSetState(0, 1);
    }

    @Override
    protected boolean tryRelease(int arg) {
        setState(0);
        return true;
    }
}
```

`ReentrantLock` 可以解决这个问题，它在 `lock()` 会把当前线程设为锁的当前占有者，当前线程如果继续调用 `lock()` 即使是占有状态，也不会阻塞，而只是状态 +1，调用 `unlock()` 状态 -1。 通过调用 AQS 的 `setExclusiveOwnerThread()` 为设置锁的当前拥有者。

## 公平锁和非公平锁机制

`ReentrantLock` 不仅实现了重入锁机制，同时也实现了公平锁和非公平锁机制。

### 非公平锁

`ReentrantLock` 并发获取锁时，内部机制依赖 CAS 修改状态获取，并不能确定哪个线程能成功修改。这种情况下锁的获取与其调用 `lock()` 顺序无关，这就是非公平锁，但由于这种不用排队的竞争机制性能最高，所以默认情况下 `ReentrantLock` 是使用的公平锁。

### 公平锁

公平锁指严格按照顺序获取锁，后来者不能直接参与锁的竞争。其强制获取锁时先判断在其之前是否有在等待的锁。

下面的代码演示公平锁和非公平锁：

```java
@Test
public void test2() {
    // true 公平锁，false 非公平锁
    final ReentrantLock lock = new ReentrantLock(true);
    ExecutorService executorService = Executors.newFixedThreadPool(3);
    for (int i = 0; i < 3; i++) {
        executorService.submit(() -> {
            for (int k = 0; k < 2; k++) {
                lock.lock();
                System.out.printf("线程%s 获得锁\n", Thread.currentThread().getName());
                lock.unlock();
            }
        });
    }
}
```

公平锁打印结果严格按照调用顺序：

> 线程 pool-1-thread-1 获得锁
> 线程 pool-1-thread-2 获得锁
> 线程 pool-1-thread-3 获得锁
> 线程 pool-1-thread-1 获得锁
> 线程 pool-1-thread-2 获得锁
> 线程 pool-1-thread-3 获得锁

非公平锁结果如下，当线程 1 获得锁以后，其它线程将进入阻塞队列，而当线程 1 释放锁时线程 1 可以和其它线程真接竞争，因为其它线程还需要唤醒，所以线程 1 获得锁的顺在其他线程之前：

> 线程 pool-1-thread-1 获得锁
> 线程 pool-1-thread-1 获得锁
> 线程 pool-1-thread-3 获得锁
> 线程 pool-1-thread-3 获得锁
> 线程 pool-1-thread-2 获得锁
> 线程 pool-1-thread-2 获得锁

# ReentrantLock 底层原理

`ReentrantLock` 底层基于 AQS 实现同步。基本原理是修改 AQS 中的状态值，等于 0 表示锁未占用，修改状态为 1 表示已经占用，如果同一线程重入加锁，状态值会累加。解锁后状态值为-1，为 0 时表示锁已释放。

## 基本结构

<center>
    <img style="border-radius: 8px;
    box-shadow: 0 5px 15px 0px rgba(0, 0, 0, .4);"
    src="https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed@master/drawio/ReentrantLock.svg" width = "80%" alt=""/>
    <div style="
  color: #999;
  font-size: 0.875em;
  font-weight: bold;
  line-height: 1;
  margin: 0 auto 15px;
  text-align: center;">
      ReentrantLock
    </div>
</center>

- `AbstractQueuedSynchronizer` :AQS 同步器（抽象类）
- `Sync`：`ReentrantLock` 中的内部抽象类，用于实现基础同步，继承自 AQS
- `NonfairSync` ：非公平锁同步器实现，继承自 `Sync`
- `FairSync` ：公平锁同步器实现，继承自 `Sync`

## 加锁流程

### 非公平锁

先看代码

```java
/**
 * Sync object for non-fair locks
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        if (compareAndSetState(0, 1))
            // CAS 修改成功，设置当前线程获得锁
            setExclusiveOwnerThread(Thread.currentThread());
        else
            // 否则调用 AQS 中的 acquire() 方法， acquire() 又调用子类的 tryAcquire() 方法
            // 如果 tryAcquire() 方法仍不能获取锁，则进入阻塞队列等待唤醒
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

接下来看 `NonfairSync` 中的 `tryAcquire()` 方法，即 `nonfairTryAcquire(acquires)`。

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // state == 0，没有线程获取到锁
        if (compareAndSetState(0, acquires)) {
            // CAS 修改成功，设置当前线程获得锁
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 有线程获取到锁，判断是不是当前线程获取到的
    else if (current == getExclusiveOwnerThread()) {
        // 是当前线程获取到的，state + 1（acquires 传进来就是 1）
        int nextc = c + acquires;
        // 同一个线程可重入最大次数为 Integer.MAX_VALUE，超出后再加一即为负数
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    // 获取锁失败，会进入 AQS 的等待队列中
    return false;
}
```

总结

<center>
    <img style="border-radius: 8px;
    box-shadow: 0 5px 15px 0px rgba(0, 0, 0, .4);"
    src="https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed@master/drawio/nonfairTryAcquire.svg" width = "80%" alt=""/>
    <div style="
  color: #999;
  font-size: 0.875em;
  font-weight: bold;
  line-height: 1;
  margin: 0 auto 15px;
  text-align: center;">
      非公平锁加锁流程
    </div>
</center>

### 公平锁

主要特性：

1. 加锁时不会尝试修改，直接走 `acquire()` 流程
2. 在 `tryAcquire()` 中修改状态前先判断自己是否处于队列头部

```java
/**
 * Sync object for fair locks
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        // 直接走 AQS 的 acquire() 流程，会调用 tryAcquire() 方法
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // state == 0，没有线程获取到锁
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                // 阻塞等待队列中没有线程，并且 CAS 修改状态成功，设置当前线程获得锁
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 有线程获取到锁，判断是不是当前线程获取到的
        else if (current == getExclusiveOwnerThread()) {
            // 是当前线程获取到的，重入
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        // 阻塞等待队列中有线程，锁也不是当前线程获取，则进入阻塞等待队列
        return false;
    }
}
```

总结

<center>
    <img style="border-radius: 8px;
    box-shadow: 0 5px 15px 0px rgba(0, 0, 0, .4);"
    src="https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed@master/drawio/fairTryAcquire.svg" width = "80%" alt=""/>
    <div style="
  color: #999;
  font-size: 0.875em;
  font-weight: bold;
  line-height: 1;
  margin: 0 auto 15px;
  text-align: center;">
      公平锁加锁流程
    </div>
</center>

## 释放锁

公平锁和非公平锁的释放锁相同，是 `Sync` 中的 `tryRelease()` 方法。

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```
