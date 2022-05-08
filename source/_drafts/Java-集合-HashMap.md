---
title: Java 集合 - HashMap
tags:
  - Java 集合
categories:
  - [Tech]
abbrlink: 2359ae94
date: 2022-03-09 22:55:36
---
<img src="https://typora-1302822277.cos.ap-nanjing.myqcloud.com/blog/java/hashmap/how-does-hashmap-work.png" />

Map 是用于存储 **键值对** 的容器，HashMap 是开发中一种常用的 Map。 
本文介绍了 Java8 中的 HashMap 的特点、底层结构和实现原理，并比较了 HashMap 在 Java7 和 Java8 中的不同。

<!-- more -->

## 底层数据结构

HashMap 的底层由数组和链表实现，Java 8 中增加了红黑树提升查找效率。

存放到 HashMap 的值实际上是放在 `Node` 中，树化时为 `TreeNode`，Node 节点存放在数组 `table[]` 中。其基本结构如下图所示：

HashMap 的基本属性包括：

| 属性                                             | 描述                                                         |
| :----------------------------------------------- | :----------------------------------------------------------- |
| Node<K, V> table                                 | 存放 Node 的数组                                             |
| int size                                         | 实际存放的 K-V 个数                                          |
| int threshold                                    | 扩容阈值，当数组中元素超过该值，开始扩容                     |
| float loadFactor                                 | 负载因子，threshold = capacity * loadFactor                  |
| int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16 | 数组默认大小，数组的大小必须是 2 的幂                        |
| float DEFAULT_LOAD_FACTOR = 0.75f;               | 默认负载因子，0.75                                           |
| int TREEIFY_THRESHOLD = 8;                       | 树化阈值，当链表长度超过 8 开始树化                          |
| int UNTREEIFY_THRESHOLD = 6;                     | 取消树化阈值，当是树的节点数少于 6 开始取消树化              |
| int MIN_TREEIFY_CAPACITY = 64;                   | 树化时数组的最小大小，当数组的大小小于 64 时，如果链表长度超过 8，会优先进行扩容 |

## 映射机制

## 存放机制

## 扩容原理

## 思考
