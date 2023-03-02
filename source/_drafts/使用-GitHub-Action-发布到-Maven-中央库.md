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

TL;DR
在你的项目根目录下创建一个 .github/workflows/maven-publish.yml 文件，内容如下：

```

```yaml

name: Maven Publish

on:
  workflow_dispatch:
  release:
    types: [released]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Git Repo
        uses: actions/checkout@v2
      - name: Set up Maven Central Repo
        uses: actions/setup-java@v1
        with:
          java-version: 1.8
          server-id: sonatype-nexus-staging
          server-username: ${{ secrets.OSSRH_USER }}
          server-password: ${{ secrets.OSSRH_PASSWORD }}
          gpg-passphrase:  ${{ secrets.GPG_PASSWORD }}
      - name: Publish to Maven Central Repo
        uses: samuelmeuli/action-maven-publish@v1.4.0
        with:
          gpg_private_key: ${{ secrets.GPG_SECRET }}
          gpg_passphrase: ${{ secrets.GPG_PASSWORD }}
          nexus_username: ${{ secrets.OSSRH_USER }}
          nexus_password: ${{ secrets.OSSRH_PASSWORD }}

```

假定你使用过 [Maven](https://maven.apache.org/)，并且已经有了一个项目在 GitHub 上托管，现在想要发布到 Maven 中央库。

如果你从来没有发布过，则需要注册 [Sonatype](https://issues.sonatype.org/) 账号，然后申请发布权限。这一步骤比较繁琐，但是只需要做一次，之后就可以一直使用了。


后续更新要做的就是提交 Action
