# My blog's source code

My Blog: https://lilittlecat.cn


# 使用

## 安装 Node.js

Node.js 官网：https://nodejs.org

检查安装：控制台输入 `node -v` 和 `npm -v` 出现版本号

### 添加国内镜像源

临时使用：

- 使用命令

  ```sh
  npm --registry https://registry.npm.taobao.org install express
  ```

永久使用：

- 直接配置

  ```sh
  npm config set registry https://registry.npm.taobao.org
  ```

  查看是否配置成功：

  ```sh
  npm config get registry
  
  npm info express
  ```

  如果需要恢复成原来的官方地址只需要执行如下命令：

  ```bash
  npm config set registry https://registry.npmjs.org
  ```

- 使用 cnpm

  安装淘宝的 cnpm，然后在使用时直接将 npm 命令替换成 cnpm 命令即可：

  ```sh
  npm install -g cnpm --registry=https://registry.npm.taobao.org
  ```

  然后安装时使用如下命令：

  ```sh
  cnpm install express
  ```

## 安装 Git

Git 官网：https://git-scm.com/

- 配置用户名邮箱

  右键 `Git Bash` 输入：

  ```sh
  git config --global user.name "username"
  git config --global user.email "username@email.com"
  ```

  命令查看配置是否成功

  ```sh
  git config --global --list
  ```

- 配置 ssh

  继续在 `Git Bash` 输入：

  ```sh
  ssh-keygen -t rsa
  ```

  生成的公钥位置：`C:\Users\用户名\.ssh\id_rsa.pub`

- 在 GitHub 中添加公钥

  登录GitHub，添加公钥位置：Settings > SSH and GPG keys，点击 New SSH key，将上述 `id_rsa.pub` 中的内容复制到 Key 一栏中，Title 随便写。

- 测试公钥是否添加成功

  继续在 `Git Bash` 输入：

  ```sh
  ssh -T git@github.com
  ```

  返回如下表示配置成功

  > Hi LiLittleCat! You've successfully authenticated, but GitHub does not provide shell access.

## 安装 Hexo

Hexo 官方网站：https://hexo.io/

```sh
npm install hexo-cli -g
```

> $ hexo help
> Usage: hexo <command>
>
> Commands:
>   help     Get help on a command.
>   init     Create a new Hexo folder.
>   version  Display version information.
>
> Global Options:
>   --config  Specify config file instead of using _config.yml
>   --cwd     Specify the CWD
>   --debug   Display all verbose messages in the terminal
>   --draft   Display draft posts
>   --safe    Disable all plugins and scripts
>   --silent  Hide output on console
>
> For more help, you can use 'hexo help [command]' for the detailed information
> or you can check the docs: http://hexo.io/docs/

## 安装主题 Fluid

主题官方文档：https://hexo.fluid-dev.com/docs/

所用插件

- 生成文章短链接 [hexo-abbrlink](https://github.com/rozbo/hexo-abbrlink)

  安装插件

  ```sh
  npm install hexo-abbrlink --save
  ```

  修改博客配置文件 _config.yml

  ```yaml
  permalink: posts/:abbrlink/
  
  # abbrlink config
  abbrlink:
    alg: crc32      #support crc16(default) and crc32
    rep: hex        #support dec(default) and hex
    drafts: false   #(true)Process draft,(false)Do not process draft. false(default) 
    # Generate categories from directory-tree
    # depth: the max_depth of directory-tree you want to generate, should > 0
    auto_category:
       enable: true  #true(default)
       depth:        #3(default)
       over_write: false 
    auto_title: false #enable auto title, it can auto fill the title by path
    auto_date: false #enable auto date, it can auto fill the date by time today
    force: false #enable force mode,in this mode, the plugin will ignore the cache, and calc the abbrlink for every post even it already had abbrlink.
  ```

- 渲染 emoji：[hexo-filter-github-emojis](https://github.com/crimx/hexo-filter-github-emojis)

  安装插件

  ```sh
  npm install hexo-filter-github-emojis --save
  ```

  修改博客配置文件 _config.yml

  ```yaml
  githubEmojis:
    enable: true
    className: github-emoji
    inject: true
    styles:
    customEmojis:
  ```

## 写文章

克隆网站源码到本地

```sh
git clone git@github.com:LiLittleCat/my-blog-source-code.git
```

博客配置文件 _config.yml 中配置远程仓库地址

```yaml
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: 
    github: https://github.com/LiLittleCat/lilittlecat.github.io
    coding: git@e.coding.net:lilittlecat/lilittlecat/lilittlecat.git
  branch: master
```

进入该文件夹，安装 npm 相关模板

```sh
npm install
npm install hexo-abbrlink --save
npm install hexo-filter-github-emojis --save
```

新建文章

```sh
hexo new [layout] title
```

新文章页位于 post 文件夹中生成的 title.md，编辑 Front-matter。

Front-matter 是文件最上方以 `---` 分隔的区域，用于指定个别文件的变量。

```markdown
---
title: 标题
# excerpt: 摘要（摘要不在文中显示用这种方式）
tags:
  - 标签
categories: 
  - 分类（多个分类不是平级，父子级别）  
index_img: 首页中的题图
banner_img: 文章中的题图
abbrlink: 自动生成的短链接
comments: true（开启评论）
date: 时间
---

摘要

<!-- more -->
```
> 父子级分类：
>
> ```yaml
> categories:
>   - Java
>   - Servlet
> ```
>
> 或者：
>
> ```yaml
> categories: [Java, Servlet]
> ```
>
> 这样写 Servlet 是 Java 的子类。
>
> 同级分类：
>
> ```yaml
> categories:
>   -[Java]
>   -[Servlet]
> ```
> Tag 以上两种写法均可。

完成编辑后

```sh
hexo g # 本地生成静态网页
hexo s # 本地部署静态网页，访问地址 http://localhost:4000
hexo d # 部署静态网页到远程服务器
```

