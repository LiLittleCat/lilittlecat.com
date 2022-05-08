---
title: Java 多线程与并发 - synchronized
tags:
  - Java
categories:
  - - Tech
abbrlink: 74beb013
date: 2022-05-08 11:05:59
---


synchronized 是 Java 多线程编程中常用的关键字，它是一种基于对象的互斥锁。

本文介绍了 synchronized 基本使用规范、底层原理和锁升级流程。

<!-- more -->

## 基本使用规范

synchronized 是一种基于对象的互斥锁，被其装饰的方法或代码快，同时只允许一个线程进入执行并锁定，结束后自动释放锁。 

它的使用有三种方式：

| 作用位置 | 锁的对象                  |
| -------- | ------------------------- |
| 代码块   | 指定对象                  |
| 普通方法 | 方法所属的对象，即 `this` |
| 静态方法 | 方法所属的 `Class` 对象   |

无论哪种方法，都必须与一个对象绑定，这个对象即该同步块的锁。线程进入 synchronized 之后，可以对锁对象进行以下操作：

- `Object.wait()` 该线程让出锁，并进入 wait 状态
- `Object.notify()` 唤醒该锁下任意一个 wait 线程
- `Object.notifyAll()` 唤醒该锁下所有 wait 线程

以上操作需要线程拥有对象锁的情况下才能调用，否则会报 `IllegalMonitorStateException`。

![synchronized](https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/Excalidraw/synchronized.svg)

使用 `notify()` 唤醒线程时，锁资源不会立刻释放，只有当当前线程交出锁资源所有权后，唤醒的线程和阻塞的线程才可以竞争该资源。交出锁资源所有权有两种方式：1. 执行完 synchronized，2. 调用对象的 `wait()` 方法。

### 使用 String 对象作为锁

使用字符串对象作为锁对象时，有可能出现值相同的两个 String 对象并不是同一个对象导致同步失败。此时可以使用 `String.intern()` 将该字符串放入常量池中，并对从常量池中返回的 String 对象加锁。

## 底层原理

编译 java 文件生成的字节码文件中，synchronized 同步代码块开始的位置有 monitorenter 指令，在同步代码块结束的位置有 monitorexit 指令。synchronized 同步方法包含 ACC_SYNCHRONIZED 标识。两者的本质都是占用了当前对象所包含的 monitor 对象。在 HotSpot JVM 中该对象为 [ObjectMonitor](https://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/69087d08d473/src/share/vm/runtime/objectMonitor.hpp#l137)。

### 对象结构

HotSpot JVM（64 位）中，[对象](https://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/69087d08d473/src/share/vm/oops/oop.hpp#l59) 在内存中的布局可以分为三个区域：

- 对象头
  - 标记字段 Mark Word，8 字节
  - 类型指针 Klass Pointer，开启指针压缩为 4 字节，关闭为 8 字节
  - 数组长度（如果是数组对象）
- 实例数据：存放类的属性数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分内存按 4 字节对齐
- 填充数据：虚拟机要求对象必须 8 字节对齐，因为对象的起始地址是 8 字节的整数倍。

我们可以通过 openJDK 提供的工具 [JOL(Java Object Layout) ](https://github.com/openjdk/jol) 打印对象的信息。

例子：

```java
public static void main(String[] args) {
        System.out.println(VM.current().details());
        Object obj = new Object();
        synchronized (obj) {
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        }
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
    }
```

### Mark Word

<table align="center">
	<thead>
		<tr>
			<td><b>对象状态</td>
			<td><b>25 位</td>
			<td><b>31 位</td>
			<td><b>1 位</td>
			<td><b>4 位</td>
			<td><b>1 位（是否偏向锁）</td>
			<td><b>2 位（锁标志）</td>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>无锁</td>
			<td>未使用</td>
			<td>对象 HashCode</td>
			<td>未使用</td>
			<td>分代年龄</td>
			<td>0</td>
			<td>01</td>
		</tr>
		<tr>
			<td>偏向锁</td>
			<td colspan="2">线程 ID（54 位）| epoch（2 位）</td>
			<td>未使用</td>
			<td>分代年龄</td>
			<td>1</td>
			<td>01</td>
		</tr>
		<tr>
			<td>轻量级锁（自旋锁）</td>
			<td colspan="5">线程栈中的 Lock Record 指针</td>
			<td>00</td>
		</tr>
		<tr>
			<td>重量级锁</td>
			<td colspan="5">指向重量级锁 Monitor 指针</td>
			<td>10</td>
		</tr>
		<tr>
			<td>GC 状态</td>
			<td colspan="5">空</td>
			<td>11</td>
		</tr>
	</tbody>
</table>

Mark Word 中有 2bit 的数据用来标记锁的状态。无锁状态和偏向锁标记位为 01，轻量级锁的状态为 00，重量级锁的状态为 10。

- 当对象为偏向锁时，Mark Word 存储了偏向线程的 ID；
- 当状态为轻量级锁时，Mark Word 存储了指向线程栈中 Lock Record 的指针；
- 当状态为重量级锁时，Mark Word 存储了指向堆中的 Monitor 对象的指针。

### Monitor 对象

```c
  ObjectMonitor() {
    _header       = NULL;
    _count        = 0;
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL;
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ;
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
    _previous_owner_tid = 0;
  }
```

Monitor 对象被称为监视器锁。在 Java 中，每一个对象实例都会关联一个 Monitor 对象。这个 Monitor 对象既可以与对象一起创建销毁，也可以在线程试图获取对象锁时自动生成。当这个 Monitor 对象被线程持有后，它便处于锁定状态。

其中：

- **_owner** 用来指向持有 monitor 的线程，它的初始值为 NULL, 表示当前没有任何线程持有 monitor。当一个线程成功持有该锁之后会保存线程的 ID 标识，等到线程释放锁后 _owner 又会被重置为 NULL;
- **_WaitSet** 调用了锁对象的 wait 方法后的线程会被加入到这个队列中；
- **_cxq**  是一个阻塞队列，线程被唤醒后根据决策判读是放入 _cxq 还是 _EntryList；
- **_EntryList** 没有抢到锁的线程会被放到这个队列；
- **_count** 用于记录线程获取锁的次数，成功获取到锁后 count 会加 1，释放锁时 count 减 1。

## 锁升级流程

JDK 1.6 之前只有无锁和重量级锁两种状态，重量级锁需要操作系统在用户态和内核态之间切换，会消耗大量的系统资源，如果程序中有大量的锁竞争，则会引起用户态和内核态的频繁切换，严重影响程序的性能。

JDK1.6 中引入偏向锁和轻量级锁对 synchronized 进行优化。synchronized 一共存在四个状态：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态，并在对象的 Mark Word 中标识出来。锁的状态会随着竞争逐渐升级，即偏向锁->轻量级锁->重量级锁。锁升级的过程是单向不可逆的，即一旦升级为重量级锁就不会再出现降级的情况。

### 偏向锁

大多数情况下，锁是被同一个线程多次获得。因此引入偏向锁，当一个线程获得了锁，锁即进入偏向锁状态。修改对象的 Mark Word，当该线程再次请求锁时，无需同步操作即可获得锁。

### 轻量级锁

当多个线程先后获取锁，但是没有竞争时，锁升级为轻量级锁，Mark Word 的结构也随之改变。JVM 会利用 CAS 尝试把对象原本的 Mark Word 更新为 Lock Record 的指针，成功就说明加锁成功，改变锁标志位为 00，然后执行相关同步操作。

轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间访问同一锁的场合，就会导致轻量级锁就会失效，进而膨胀为重量级锁。

### 自旋锁

轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的优化手段。

自旋锁是基于在大多数情况下，线程持有锁的时间都不会太长。因此自旋锁会假设在不久将来，当前的线程可以获得锁，因此 JVM 会让当前想要获取锁的线程做几个空循环，不断的尝试获取锁。默认允许循环 10 次，可以通过虚拟机参数-XX：PreBlockSpin 更改，如果得到锁，就顺利进入同步代码。如果还不能获得锁，那就会将线程在操作系统层面挂起，即进入到重量级锁。

### 锁消除

JIT 即时编译器会进行逃逸分析，检测加了同步的代码，但是不可能出现共享数据竞争的锁进行锁消除。既同步代码中没有对线程共享数据进行读写，那么就都是线程私有的，可以消除锁。

### 锁粗化

JIT 对代码中出现连续对同一个对象进行加锁，甚至在对同一对象在循环中加锁，会进行锁粗化，扩大锁范围，避免频繁加锁解锁。

### 锁降级

当 JVM 进入安全点 [SafePoint](https://link.segmentfault.com/?enc=6gunoItnaxBFv4gt9ynAYg%3D%3D.VZSBs%2F9Tfs%2BN2zeBh2ZnFhknaAUv8tY52eTA%2FN4nq%2BGlKzXZ78KeyB%2BwRiWhEUK4ZKEP%2FtQJJggU7aJWBdoFrQ%3D%3D) 的时候，会检查是否有闲置的 Monitor，然后试图进行降级。

### 锁升级过程

1. 没有线程执行同步代码块，无锁，Mark Word 最后 3 位为 001；
2. 线程 A 执行同步代码块，偏向锁，Mark Word 记录线程 ID，最后 3 位为 101；
3. 线程 A 再次执行，发现偏向锁，检查线程 ID，一致，继续执行；
4. 线程 B 执行同步代码块，发现偏向锁，检查线程 ID，不一致，使用 CAS 尝试获取锁。如果成功，线程 B 执行；如果失败，进入 5；
5. 偏向锁状态抢锁失败，锁升级为轻量级锁，JVM 开辟空间 Lock Record 保存当前对象锁 Mark Word 指针，同时 Mark Word 保存指向该空间的指针。上述保存操作都是 CAS 操作，如果成功，改变 Mark Word 最后 2 位为 00；如果失败，进入 6；
6. 轻量级锁抢锁失败，JVM 会使用自旋锁，不断的重试，尝试抢锁。从 JDK1.7 开始，自旋锁默认启用，自旋次数由 JVM 决定。如果抢锁成功则执行同步代码，如果失败进入 7；
7. 自旋锁重试之后如果抢锁依然失败，同步锁会升级至重量级锁，改变 Mark Word 最后 2 位为 10。在这个状态下，未抢到锁的线程都会被阻塞。

> 注意：
>
> 1. 由于 Mark Word 结构，无锁状态未生成 HashCode、无线程竞争时才能进入偏向锁状态，如果生成 HashCode，直接进入轻量级锁或重量级锁。
> 2. 偏向锁检查 JDK 1.6 之后是默认开启的，参数 `XX:-UseBiasedLocking=false`，开启后会延缓 JIT 预热进程，预热过程中会直接进入轻量级锁，大约 4s。

参考

https://juejin.cn/post/6973571891915128846

https://www.cnblogs.com/mingyueyy/p/13054296.html

http://www.itabin.com/mark-word/

https://segmentfault.com/a/1190000037645482
