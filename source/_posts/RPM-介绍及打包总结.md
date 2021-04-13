---
title: RPM 介绍及打包总结
# excerpt: 简单介绍一下 RPM，并总结制作 RPM 包的流程。（摘要不在文中显示用这种方式）
tags:
  - RPM
categories:
  - [Tech]
abbrlink: 22fd6dd2
date: 2020-08-06 16:18:03
---
<!-- ![](https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/blog/RPM_Logo.svg!600x) -->
<!-- <img src="https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/blog/RPM_Logo.svg" style="zoom:50%" /> style="height:600px" -->
<!-- <img src="https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/blog/RPM_Logo.svg"  width="600px" />  -->
<img src="https://cdn.jsdelivr.net/gh/LiLittleCat/PicBed/images/blog/RPM_Logo.svg" height="200px"/>
RPM Package Manager (RPM) 是一个强大的命令行驱动的软件包管理工具，用来安装、卸载、校验、查询和更新 Linux 系统上的软件包。本文简单介绍了一下 RPM，并总结制作了 RPM 包的流程。

<!-- more -->

## 软件包管理系统和Linux发行版

软件包管理系统：自动安装、配置、卸载和升级软件包的工具组合。

Linux 发行版中常见的有：

- RPM(Redhat Package Manager)
- DPKG(Debian Package)

使用他们的系统被分为两大派系:

- RedHat 系（CentOS、Fedora 等）
- Debian 系（Ubuntu、Deepin 等）

## RPM

一个 RPM 包包含了已经压缩的软件文件集和相关的软件内容信息，通常表现为以 `.rpm` 扩展名结尾的文件，如果是源代码包通常以 `src.rpm` 结尾。

### rpm 包命名

一般格式为：`name-version-release.arch.rpm`。

- `name`：安装包的名称。
- `version`：版本信息。
- `release`：发行号、适应系统等。
- `arch`：适用的硬件平台。包括 `x86_64`、`i686` 等，`noarch` 表示能安装在任何平台。

举例：

> nginx-1.16.1-1.el7.x86_64.rpm

### rpm 命令

安装管理 RPM 包需要用到 `rpm` 命令，常用参数如下：

| 参数 |                含义                 |
| :--: | :---------------------------------: |
| `-q` |           查询 rpm 包的信息           |
| `-i` |              安装 rpm 包              |
| `-U` |              升级 rpm 包              |
| `-e` |              卸载 rpm 包              |
| `-h` |          以 `#` 显示安装进度          |
| `-v` |            显示安装详情             |
| `-l` |          列出安装包中文件           |
| `-p` | 对 rpm 包进行查询、与其他参数一起使用 |

以上参数需要组合使用，举例：

查看安装包信息

```shell
rpm -qip nginx-1.16.1-1.el7.x86_64.rpm
```

列出安装包中的文件

```shell
rpm -qlp nginx-1.16.1-1.el7.x86_64.rpm
```

安装安装包

```shell
rpm -ivh nginx-1.16.1-1.el7.x86_64.rpm
```

查询已安装所有安装包

```shell
rpm -qa
```

卸载安装包

```shell
rpm -evh nginx-1.16.1-1.el7.x86_64 
# 后面跟 rpm 包的全名
```

> 补充：
> - 使用 `--nodeps` 跳过依赖验证，强制操作。
> - rpm默认不允许重复安装和降级安装，可使用 `--force` 强制安装。
> - 可使用 `-qf` 查看某个文件属于哪个rpm包。

## YUM

当一个 rpm 包的安装需要依赖其他包时，就必须先安装依赖包。YUM（Yellowdog Updater, Modified）工具基于 RPM ，可以从指定源空间（服务器、本地目录等）自动下载 rpm 包并处理依赖关系。

### yum 命令

安装指定软件 :

```shell
yum -y install nginx # -y 表示互动时输入 yes
```

列出系统中已安装软件

```shell
yum list
```

列出系统中可升级的所有软件

```shell
yum check-update
```

升级系统中可升级的所有软件

```shell
yum update
```

升级指定软件

```shell
yum update nginx
```

卸载指定软件

```shell
yum remove nginx
```

### repo 文件

yum 使用仓库保存管理 rpm 包，仓库的配置文件以 `.repo` 为扩展名，存放在 `/etc/yum.repo.d/` 目录下。一个配置文件中包含了一个或多个 yum 仓库的配置信息。

### rpm 包下载

#### 使用 `--downloadonly`

没有命令需要先安装 `downloadonly` 插件：

```shell
yum install yum-plugin-downloadonly
```

下载 rpm 包和依赖：

```shell
yum install --downloadonly --downloaddir=/home/rpm-share/yum nginx
```

默认下载路径：`/var/cache/yum/x86_64/[centos/fedora-version]/[repository]/packages`

已安装再次下载：`reinstall`，此时不会下载依赖。

#### 使用 yumdownloader

没有命令需要先安装：

```shell
yum install -y yum-utils
```

下载 rpm 包和依赖：

```shell
yumdownloader --destdir=/home/rpm-share/yum --resolve nginx
```

`--destdir` 指定下载的软件包存放路径，不指定默认当前路径。
`--resolve` 解决依赖关系并下载所需的包。

## rpm 包制作
### spec 基础

要制作 rpm 包，关键在于编写包的 spec 描述文件。
#### 关键字

- `Name`：软件包的名字，后面可使用 `%{name}` 的方式引用

- `Version`：软件版本号，后面可使用 `%{version}` 引用

- `Release`：软件包释出号/发行号，后面可使用 `%{release}` 引用

- `Group`：软件分组，参见 `/usr/share/doc/rpm-4.x.x/GROUPS`

- `License`：软件授权方式，通常是 GPL（自由软件）或 GPLv2，BSD

- `Summary`：软件包的内容概要

- `URL`：软件的主页

- `Packager`：打包者的信息

- `Vendor`：软件开发者的名字，发行商或打包组织的信息，例如 `Fedora Project`

- `Source0`：源代码包的名称，可以带多个用 `Source1`、`Source2` 等源，后面也可以用 `%{source1}`、`%{source2}` 引用，如有补丁也写在此处，如：

  ```shell
  Source0: %{name}-%{version}.tar.gz
  Patch0: some-bugs0.patch                    
  Patch1: some-bugs1.patch                   
  ```

- `BuildRequires`: 制作过程中用到的软件包，构建依赖

- `Requires`: 安装时所需软件包

  - `Requires(pre/post/preun/postun)`: 指定不同阶段的依赖

- `BuildRoot`: 这个是安装或编译时使用的「虚拟目录」，打包时会用到该目录下文件，可查看安装后文件路径，例如：`BuildRoot: %_topdir/BUILDROOT`。

- `%description`：软件包详细说明

#### 主体

主体包括 %prep，%build，%install，%files，%clean 几个关键阶段。

##### %prep 阶段

对源代码包进行解压和打补丁，如：

```sh
%prep
%setup -q    # 将 %sourcedir 目录下的源代码解压到 %builddir 目录下
%pathch -p1  # 给源代码打上补丁 Patch0 ， -p1 是忽略 patch 的第一层目录
```

##### %build 阶段

在 `%_builddir` 目录下执行源码包的编译。一般是执行执行常见的 `configure` 和 `make` 操作，如：

```sh
%configure											 # 执行源代码的 configure 配置
make %{?_smp_mflags} OPTIMIZE="%{optflags}"           # 多核则并行编译
```

##### %install 阶段

执行 `make install` 命令操作。开始把软件安装到虚拟的根目录中。这个阶段会在 `%buildrootdir` 目录里建好目录结构，然后将需要打包到 rpm 软件包里的文件从 `%builddir` 里拷贝到 `%_buildrootdir` 里对应的目录里。

在 `~/rpmbuild/BUILD/%{name}-%{version}` 目录中进行 `make install` 的操作。`%install` 很重要，因为如果这里的路径不对的话，则下面 `%file` 中寻找文件的时候就会失败。

> 注意：
> `%install` 部分使用的是绝对路径，而 `%file` 部分使用则是相对路径。
> 当用户最终用 `rpm -ivh` 安装软件包时，这些文件会安装到用户系统中相应的目录里。

如：

```sh
%install
rm -rf $RPM_BUILD_ROOT 						# 清空下安装目录，实际会自动清除
make install DESTDIR=$RPM_BUILD_ROOT 		# 安装到 buildroot 目录下         
```

`%install` 阶段还可以执行相关脚本

- `%pre` 安装前执行的脚本
- `%post` 安装后执行的脚本
- `%preun` 卸载前执行的脚本
- `%postun` 卸载后执行的脚本

> 注意：
> `%preun` 在升级的时候会执行，`%postun` 在升级的时候不会执行。

##### %file 阶段

说明会将 `%{buildroot}` 目录下的哪些文件和目录最终打包到rpm包里。

定义软件包所包含的文件，分为三类：

- 说明文档（doc）
- 配置文件（config）
- 执行程序

在 `%files` 阶段的第一条命令的语法是：

```sh
%defattr(文件权限,用户名,组名,目录权限) 
```

还可定义文件存取权限，拥有者及组别。如：

```sh
%files
%defattr (-,root,root,0755)                     # 设定默认权限
%config(noreplace) /etc/my.cnf                  # 表明是配置文件，noplace 表示替换文件
%doc %{_mandir}/1.log                           # 表明这个是文档
%attr(755, root, root) %{_sbindir}/mysqld	    # 分别是权限，用户，用户组
%exclude %{_mandir}/2.log					  # 列出不想打包到 rpm 中的文件
```

> 注意：
> - `%files` 里的文件同时需要在 `%install` 中安装。
> - `%{buildroot}` 里的所有文件都要明确被指定是否要被打包到 rpm 里。
> - 如果声明了 `%{buildroot}` 里不存在的文件或者目录也会报错。
> - `%exclude` 指定的文件如果不存在会报错。

##### %clean 阶段

编译完成后一些清理工作，主要包括对 `%{buildroot}` 目录的清空（非必须），如：

```sh
make clean
```

##### %changelog 阶段

主要记录的每次打包时的修改变更日志。标准格式是：

```sh
* date +"%a %b %d %Y" 修改人 邮箱 本次版本 x.y.z-p 
- 本次变更修改了那些内容
```

#### 宏

##### 内建宏

spec 文件中在定义文件的安装路径时通常会使用 RPM 内建的宏定义，可以在 `/usr/lib/rpm/macros` 找到，附录一些常见的宏：

```
%{_sysconfdir}        /etc
%{_prefix}            /usr
%{_exec_prefix}       %{_prefix}
%{_bindir}            %{_exec_prefix}/bin
%{_lib}               lib (lib64 on 64bit systems)
%{_libdir}            %{_exec_prefix}/%{_lib}
%{_libexecdir}        %{_exec_prefix}/libexec
%{_sbindir}           %{_exec_prefix}/sbin
%{_sharedstatedir}    /var/lib
%{_datadir}           %{_prefix}/share
%{_includedir}        %{_prefix}/include
%{_oldincludedir}     /usr/include
%{_infodir}           /usr/share/info
%{_mandir}            /usr/share/man
%{_localstatedir}     /var
%{_initddir}          %{_sysconfdir}/rc.d/init.d 
%{_topdir}            %{getenv:HOME}/rpmbuild
%{_builddir}          %{_topdir}/BUILD
%{_rpmdir}            %{_topdir}/RPMS
%{_sourcedir}         %{_topdir}/SOURCES
%{_specdir}           %{_topdir}/SPECS
%{_srcrpmdir}         %{_topdir}/SRPMS
%{_buildrootdir}      %{_topdir}/BUILDROOT
%{_var}               /var
%{_tmppath}           %{_var}/tmp
%{_usr}               /usr
%{_usrsrc}            %{_usr}/src
%{_docdir}            %{_datadir}/doc
%{buildroot}          %{_buildrootdir}/%{name}-%{version}-%{release}.%{_arch}
$RPM_BUILD_ROOT       %{buildroot}
```

通过命令 `rpm --eval "%{宏名称}"` 来查看具体对应路径，如：

```sh
rpm --eval "%{_topdir}"
```

##### 自定义宏

使用 `%define` 可自定义宏变量。

> 注意：
> 打包时会对所有安装的文件进行 `strip` 操作，去除调试信息等。`strip` 操作定义在宏 `__os_install_post` 中，可通过 `rpmbuild --showrc` 查看，将其设为 `%{nil}` 可跳过 `strip` 操作。即：`%define __os_install_post %{nil}`

### rpmbuild 打包

使用 `rpmbuild` 制作rpm包，`rpmbuid` 的默认工作路径是 `$HOME/rpmbuild`，并且推荐用户在制作 rpm 软件包时尽量不要以 root 身份进行操作。

#### 打包目录

构建打包目录，一般在 `~/rpmbuild` 目录下建立以下目录：

- `SPEC`
- `SOURCE` 
- `BUILD`
- `RPM`
- `SRPM`
- `BUILDROOT`

对应关系：

| 默认位置             | 宏             | 名称              | 用途                                         |
| -------------------- | -------------- | ----------------- | -------------------------------------------- |
| ~/rpmbuild/SPECS     | %_specdir      | spec 文件目录     | 保存 RPM 包配置（.spec）文件                 |
| ~/rpmbuild/SOURCES   | %_sourcedir    | 源代码目录        | 保存源码包（如 .tar 包）和所有 patch 补丁    |
| ~/rpmbuild/BUILD     | %_builddir     | 构建目录          | 源码包被解压至此，并在该目录的子目录完成编译 |
| ~/rpmbuild/RPMS      | %_rpmdir       | 标准 RPM 包目录   | 生成/保存二进制 RPM 包                       |
| ~/rpmbuild/SRPMS     | %_srcrpmdir    | 源代码 RPM 包目录 | 生成/保存源码 RPM 包（SRPM）                   |
| ~/rpmbuild/BUILDROOT | %_buildrootdir | 最终安装目录      | 保存 `%install` 阶段安装的文件               |

#### 打包命令

切换到 `rpmbuild/SPECS` 目录下执行打包编译命令，如：

```sh
rpmbuild -ba 软件名-版本.spec 
```

`rpmbuild`命令参数：

|    参数    |              含义              |
| :--------: | :----------------------------: |
|   `-bp`    |      只解压源码及应用补丁      |
|   `-bc`    |           只进行编译           |
|   `-bi`    |   只进行安装到 `%{buildroot}`   |
|   `-bb`    |      只生成二进制 rpm 包       |
|   `-bs`    |       只生成源码 rpm 包        |
|   `-ba`    | 生成二进制 rpm 包和源码 rpm 包 |
| `--target` |     指定生成 rpm 包的平台      |

#### 修改默认工作路径

`rpmbuild` 默认工作路径通常由 `%_topdir` 的宏变量来定义。如果想更改有以下方法：

##### 修改 `.rpmmacros` 的隐藏文件

在 `root` 下建立一个名为 `.rpmmacros` 的隐藏文件，然后在里面重新定义 `%_topdir`，指向一个新的目录名。`.rpmmacros` 文件内容如下：

```sh
%_topdir %(echo $HOME)/rpmbuild

%_smp_mflags %( \
    [ -z "$RPM_BUILD_NCPUS" ] \\\
        && RPM_BUILD_NCPUS="`/usr/bin/nproc 2>/dev/null || \\\
                             /usr/bin/getconf _NPROCESSORS_ONLN`"; \\\
    if [ "$RPM_BUILD_NCPUS" -gt 16 ]; then \\\
        echo "-j16"; \\\
    elif [ "$RPM_BUILD_NCPUS" -gt 3 ]; then \\\
        echo "-j$RPM_BUILD_NCPUS"; \\\
    else \\\
        echo "-j3"; \\\
    fi )
    
%__arch_install_post \
    [ "%{buildarch}" = "noarch" ] || QA_CHECK_RPATHS=1 ; \
    case "${QA_CHECK_RPATHS:-}" in [1yY]*) /usr/lib/rpm/check-rpaths ;; esac \
    /usr/lib/rpm/check-buildroot
```

##### 使用 `rpmbuild` 命令时定义

使用 `rpmbuild` 时覆盖 `%_topdir` 的宏变量定义来指定此次打包的工作路径：

```sh
rpmbuild --define "_topdir ${dir:-新的目录}" -ba 软件名-版本.spec 
```

