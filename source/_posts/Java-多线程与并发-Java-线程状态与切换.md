---
title: Java 多线程与并发 - Java 线程状态与切换
tags:
  - Java
categories:
  - - Tech
abbrlink: d7729ff
date: 2022-05-07 00:37:02
---

本文介绍了操作系统线程的状态和 Java 线程的状态和其转换关系，并对两种线程进行对比。

<!-- more -->

# 操作系统线程状态和转换

程序（Program）是通过一些符号表示的算法，而进程（Process）是程序执行的过程。操作系统中的进程可能的状态有以下几种：

- NEW：正在创建进程
- READY：进程等待被运行
- RUNNING：正在执行指令
- WAITING：进程正在等待某个事件发生
- TERMINATED：进程执行结束

<center>
    <img style="border-radius: 8px;
    box-shadow: 0 5px 15px 0px rgba(0, 0, 0, .4);"
    src="https://www.pling.org.uk/cs/opsimg/processstates.png" width = "80%" alt=""/>
    <div style="
  color: #999;
  font-size: 0.875em;
  font-weight: bold;
  line-height: 1;
  margin: 0 auto 15px;
  text-align: center;">
      "Process State" from <a href=https://www.pling.org.uk/cs/ops.html > https://www.pling.org.uk/cs/ops.html</a>
    </div>
</center>

在现在的操作系统中，因为线程（Thread）依旧被视为轻量级进程，所以操作系统中线程的状态实际上和进程状态是类似的模型。

<center>
    <img style="border-radius: 8px;
    box-shadow: 0 5px 15px 0px rgba(0, 0, 0, .4);"
    src="https://www.pling.org.uk/cs/opsimg/threadstates.png" width = "80%" alt=""/>
    <div style="
  color: #999;
  font-size: 0.875em;
  font-weight: bold;
  line-height: 1;
  margin: 0 auto 15px;
  text-align: center;">
      "Thread State" from <a href=https://www.pling.org.uk/cs/ops.html > https://www.pling.org.uk/cs/ops.html</a>
    </div>
</center>

# Java 线程状态和转换

Java 线程，即 JVM 线程，实际上是通过系统调用，将程序的线程交给了操作系统内核进行调度。本质是操作系统中的线程，Linux 下是基于 pthread 库实现的轻量级进程，Windows 下是原生的系统 Win32 API 提供系统调用从而实现多线程。

Java 线程有 6 种状态，定义在 Thread 类内部的枚举中。

```java
public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED;
}
```

| 状态          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| NEW           | 新建一个线程对象，还没有调用 start() 方法                      |
| RUNNABLE      | 线程等待被调度或正在运行中，Java 线程中将就绪（READY) 和运行中（RUNNING）两种状态统一为 RUNABLE |
| BLOCKED       | 线程阻塞在进入 synchronized 修饰的方法或代码块时的状态，表示线程阻塞于锁 |
| WAITING       | 让出 CPU，等待被显示的唤醒，被唤醒后进入 BLOCKED 状态，重新获取锁，进行该状态的线程需要等待其他线程做出一些特定动作（通知或中断） |
| TIMED_WAITING | 超时等待，让出 CPU，时间到了自动唤醒，进入 BLOCKED 状态，重新获取锁，不同于 waiting，它可以在指定的时间后自行返回 |
| TERMINATED    | 表示线程已经执行完毕                                         |

它们的转换关系为：
<center>
    <img style="border-radius: 8px;
    box-shadow: 0 5px 15px 0px rgba(0, 0, 0, .4);"
    src="https://www.uml-diagrams.org/examples/state-machine-example-java-6-thread-states.png" width = "80%" alt=""/>
    <div style="
  color: #999;
  font-size: 0.875em;
  font-weight: bold;
  line-height: 1;
  margin: 0 auto 15px;
  text-align: center;">
      "Java Thread States and Life Cycle" From <a href=https://www.uml-diagrams.org/java-thread-uml-state-machine-diagram-example.html>https://www.uml-diagrams.org/</a>
    </div>
</center>

# 对比

| 操作系统线程 | Java 线程                        |
| ------------ | ------------------------------- |
| NEW          | NEW                             |
| READY        | RUNNBALE                        |
| RUNNING      | RUNNBALE                        |
| WAITING      | BLOCKED，WAITING，TIMED_WAITING |
| TERMINATED   | TERMINATED                      |

从实际意义上来讲，操作系统中的线程除去 NEW 和 terminated 状态（并不存在于线程运行中），一个线程真实存在的状态，只有：

- READY：表示线程已经被创建，正在等待系统调度分配 CPU 使用权
- RUNNING：表示线程获得了 CPU 使用权，正在进行运算
- WAITING：表示线程等待（或者说挂起），让出 CPU 资源给其他线程使用

对于 Java 中的线程状态，无论是 TIMED_WAITING，WAITING 还是 BLOCKED，对应的都是操作系统线程的 WAITING 状态。
而 RUNNBALE 状态，则对应了操作系统中的 READY 和 RUNNING 状态，因为 JVM 并不关心操作系统线程的实际状态，从 JVM 看来，等待 CPU 使用权（READY）与等待 I/O（WAITING）没有区别，都是在等待某种资源，所以都归入 RUNNABLE 状态。

正如，Thread 类中 State 的注释中所说：

```java
/**
 * A thread can be in only one state at a given point in time.
 * These states are virtual machine states which do not reflect
 * any operating system thread states.
 */
```

# Java 线程状态代码示例

```java
public class ThreadStateTest {
    @Test
    public void test() throws InterruptedException {
        //1、NEW
        Thread t1 = new Thread(() -> {
        });
        System.out.printf("t1:[%s]%n", t1.getState());

        //2、RUNNABLE
        Thread t2 = new Thread(() -> {
            while (true) {

            }
        });
        t2.start();
        System.out.printf("t2:[%s]%n", t2.getState());

        //3、TERMINATED
        Thread t3 = new Thread(() -> {
        });
        t3.start();
        Thread.sleep(1000);
        System.out.printf("t3:[%s]%n", t3.getState());

        //4、TIMED_WAITING
        Thread t4 = new Thread(() -> {
            try {
                Thread.sleep(1000 * 3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        t4.start();
        Thread.sleep(1000);
        System.out.printf("t4:[%s]%n", t4.getState());

        //5、WAITING
        Thread t5 = new Thread(() -> {
            try {
                // 等待 t2 线程完成
                t2.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        t5.start();
        Thread.sleep(1000);
        System.out.printf("t5:[%s]%n", t5.getState());

        //6、BLOCKED
        // test 线程提前拿到锁
        Thread test = new Thread(() -> {
            synchronized (ThreadStateTest.class) {
                try {
                    Thread.sleep(1000 * 1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        test.start();

        Thread t6 = new Thread(() -> {
            synchronized (ThreadStateTest.class) {
                try {
                    Thread.sleep(1000 * 1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        t6.start();
        Thread.sleep(1000);
        System.out.printf("t6:[%s]%n", t6.getState());
    }
}
```

运行结果：

> t1:[NEW]
>
> t2:[RUNNABLE]
>
> t3:[TERMINATED]
>
> t4:[TIMED_WAITING]
>
> t5:[WAITING]
>
> t6:[BLOCKED]

# 参考

[Operating Systems](https://www.pling.org.uk/cs/ops.html)

[Java Thread States and Life Cycle](https://www.uml-diagrams.org/java-thread-uml-state-machine-diagram-example.html)

[Java 线程状态 与 操作系统线程状态](https://www.cnblogs.com/codeclock/p/13803898.html)

[Java 线程的 6 种状态及切换（透彻讲解）](https://blog.csdn.net/pange1991/article/details/53860651)

[线程状态（操作系统层面和 Java 层面）](https://blog.csdn.net/m0_45025658/article/details/111032780)
