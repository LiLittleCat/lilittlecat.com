---
title: Java 容器 - 概述
excerpt: 本系列旨在归纳总结 Java 容器相关类的知识，并深入分析常用容器类的源代码。本文简单描述了 Java 容器相关概念。
tags:
  - Java 集合
categories:
  - [技术, Java]
index_img: 'https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/blog/logo-java-text-color.svg'
banner_img: 'https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/blog/logo-java-text-color.svg'
abbrlink: 81a892b9
date: 2020-08-17 23:36:04
---

## 前言

一个程序在运行过程中会根据某些条件创建多个对象，这些条件和创建对象的个数在运行之前都是未知的。为此，Java 提供了多种方式来保存对象（对象的引用）。比如数组，但是数组的大小在创建时就已固定，不适合复杂的持有对象的场景。除此之外，Java 类库中还提供了一套容器类来解决这个问题。

其中基本的类型是 **List**、**Set**、**Queue** 和 **Map**，这些对象类型也称为*集合类*，但由于 Java 类库中使用了 **Collection** 这个名词指代某一对象，所以把这些用来保存对象的类统称为”容器“。

## 类图

JDK 8 中 Collection 和 Map 的类图如下所示：
![Collection](https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/blog/Collection.svg)

![Map](https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/blog/Map.svg)


其中单独列出的接口和类：

- Comparator

  比较器接口，是一个函数式接口，可通过实现其方法 `compare()` 对容器中的元素进行排序。

- RandomAccess

  是一个空接口，标识一个 `List` 是否支持快速随机访问，一个 List 支持快速随机访问时，使用 for 循环遍历比采用 Iterator 迭代器遍历更快速。实现类：`ArrayList`，`AttributeList`，`CopyOnWriteArrayList`，`RoleList`，`RoleUnresolvedList`，`Stack`，`Vector`。

- Iterator 和 ListIterator

  `Iterator` 接口使用了迭代器设计模式来对所有的容器进行快速遍历，容器本身不需要关注存储元素的数据类型，这些确定类型和转型的工作由 `Iterator` 负责实现。`Iterator` 只支持`hasNext()`、`next()`、`remove()` 三种操作，而 `ListIterator` 是 List 容器所独有的迭代器，与 `Iterator` 相比，`ListIterator` 还包含 `add()`、`hasPrevious()`、`previous()`、`nextIndex()`、`previousIndex()`、`set()`  等方法，能够在遍历过程中修改集合、逆向顺序遍历、定位遍历索引；而 `Iterator` 只能遍历不能修改、只能顺向顺序遍历、不能定位索引。

- Iterable

  允许实现此接口的对象使用 `for-each loop`。`Iterable` 接口的实现又依赖于实现了 `Iterator` 的内部类(参照 `LinkedList` 中 `listIterator()` 和 `descendingIterator()` 的 JDK 源码)。有的容器类会有多个实现 `Iterator` 接口的内部类，通过返回不同的迭代器实现不同的迭代方式。

- Arrays 和 Collections

  `Arrays` 是关于数组的封装类，封装了对数组操作的多种方法，如 `sort()`、`copyOf()`、`binarySearch()`、`asList()` 等方法；`Collections` 封装了很多对于容器的操作，如 `max()`、`min()`、`reverse()`、`sort()` 等方法，生成同步容器如 `Collections.synchronizedList()`、`Collections.synchronizedSet()` 等。

## Set

Set 的值是不可重复的，无序的，最多只有一个 `null` 值。

其子容器主要包括：

- AbstractSet：抽象集合类，实现了 `equals()` 和 `hashCode()` 方法
- SortedSet：有序（默认自然序）
- NavigableSet：继承自 `SortedSet`
- TreeSet：实现 `NavigableSet` 接口，继承 `AbstractSet`
- HashSet：hash 方式存储（实际上是一个 Map 的实例）
- LinkedHashSet：双向循环链表，不可重复，顺序与插入顺序保持一致，或者实现自定义的顺序
- EmumSet：只能存放 `Emum` 枚举类型

## List

List 的值是可重复的、无序的、可添加 `null` 值。

List 子容器主要包括：

- ArrayList：数组实现，随机存取
- LinkedList：双向循环链表，顺序存取
- Vector：同步，其他与 `ArrayList` 相同
- Stack：同步，继承自 `Vector`，“先进后出”

## Queue

Queue 实现了数据结构中的队列，具有“先进先出”的特点。

主要的子容器包括：

- AbstractQueue：抽象队列
- PriorityQueue：继承自 `AbstractQueue`，数据结构中堆 Heap 的实现
- Deque：双端队列，两端都可以插入和删除 
  - 输出受限的双端队列 
  - 输入受限的双端队列 

## Map

Map 容器是利用映射关系来存储键值对的，独立于 `List`、`Set`、`Queue`。键值对是一一对应的关系，一般允许键值为空，不可重复，是完全抽象类 `Dictionary` 的接口版本。

Map的子容器主要包括：

- AbstractMap：实现了内部 `EntrySet` 接口，实现 `equals()`、`hashCode()` 方法
- SortedMap：有序（默认自然序）
- NavigableMap：继承自 `SortMap`
- TreeMap：基于红黑树，实现 `NavigableMap` 接口，继承 `Abstractmap`
- HashMap：非同步，允许 `null`
- LinkedHashMap：非同步，允许 `null`，双向循环链表，顺序与插入顺序一致（类比于 `LinkedHashSet` ），或者实现自定义顺序。
- HashTable：同步，不允许 `null`
- EmumMap：只能存放 `Emum` 枚举类型
- WeakHashMap：弱键（weak key）映射，允许释放映射所指向的对象，为解决某类特殊问题而设计。如果映射之外没有应用指向某个“键”，则该“键”可以被 GC 回收
- IdentityHashMap：使用“==”代替 `equals()` 对键进行比较的散列映射，专门为解决特殊问题而设计
- ConcurrentHashMap：线程安全的 `Map`，属于 `java.util.concurrent` 并发包中



> 参考：https://blog.csdn.net/WZD2012/article/details/73245493?utm_source=blogxgwz5

