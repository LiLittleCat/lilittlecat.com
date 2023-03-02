---
title: 使用 GitHub Action 发布到 Maven 中央库
tags:
  - GitHub Action
  - Maven
categories:
  - - Tech
draft: true
abbrlink: 28e29a15
date: 2023-03-03 00:46:57
---


人麻了，今天更新自己开发的 [ChatGPT Java 客户端](https://github.com/LiLittleCat/ChatGPT)，发布时一大堆问题。折腾一个小时后才想起自己好像之前发布的时候用了 GitHub Action，竟然忘了。真是好记性不如烂笔头，于是立马写了这篇文章，以免再次忘记，也希望能帮助到其他人。

<!-- more -->

假定你使用过 [Maven](https://maven.apache.org/)，并且已经有了一个项目在 GitHub 上托管，现在想要发布到 Maven 中央库。

## 1. 创建 