---
title: Java 多线程与并发 - Java 线程状态与切换
tags:
  - Java
categories:
  - Tech
date: 2022-05-07 00:37:02
---

https://www.javatpoint.com/threads-in-operating-system
https://webeduclick.com/life-cycle-of-thread-in-os/
https://www.geeksforgeeks.org/thread-states-in-operating-systems/
https://www.tutorialspoint.com/operating_system/os_multi_threading.htm
https://applied-programming.github.io/Operating-Systems-Notes/2-Process-Management/

[Operating Systems](https://www.pling.org.uk/cs/ops.html)

[Java Thread States and Life Cycle](https://www.uml-diagrams.org/java-thread-uml-state-machine-diagram-example.html)

[Java 线程状态 与 操作系统线程状态](https://www.cnblogs.com/codeclock/p/13803898.html)

[Java 线程和操作系统线程的关系](https://blog.csdn.net/CringKong/article/details/79994511)

[线程状态（操作系统层面和 Java 层面）](https://blog.csdn.net/m0_45025658/article/details/111032780)

[google](https://www.google.com.hk/search?q=thread+state+in+os&source=lmns&hl=zh-CN&sa=X&ved=2ahUKEwjqsJznifb3AhVLdJQKHVifD4AQ_AUoAHoECAEQAA)

本文介绍了操作系统线程的状态和 Java 线程的状态和其转换关系，并对两种线程进行对比。

<!-- more -->

# 操作系统线程状态和转换

在现在的操作系统中，因为线程依旧被视为轻量级进程，所以操作系统中线程的状态实际上和进程状态是一致的模型。

{% plantuml %}
@startuml System Thread State

hide empty description

state SystemThreadState {
    [*] -up-> new
    new -> ready : admitted
    ready -> running : sheduler dispatch
    running -> ready : interrupt
    running -> terminated : exit
    running -> waiting : I/O or event wait
    waiting --> ready : I/O or event completion
    terminated -down-> [*]
}

@enduml
{% endplantuml %}

其中

- new：线程被创建
- ready：等待被执行
- running：正在被执行
- waiting：等待其他事件发生
- terminated：执行结束

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



# 对比

# 参考

https://blog.csdn.net/pange1991/article/details/53860651

https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html
