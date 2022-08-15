---
title: Python 基础知识总结
tags:
  - Python
categories:
  - [Tech]
date: 2020-07-23 19:04:20
---

本文总结了Python基础和使用Python编写脚本所涉及的知识。

<!-- more -->
@[toc]

由于`shell`脚本的书写和阅读都比较困难，并且`CentOS7`自带`python2.7`，拟改用`python`编写部署脚本，记录下学习内容，知识点大体覆盖脚本编写。

### 一、python基础

#### 1.1 变量

数字：`a = 1` 或 `a = 1.0`

常用计算：加（`+`）、减（`-`）、乘（`*`）、除（`/`）、乘方（`**`）

字符串：`a = 'hello'`或`a = "hello"`

查看变量a的类型：`type(a)`

变量a转字符类型：`str(a)`

变量a转整数类型：`int(a)`

注释：`#注释一行`，`'''注释多行'''`

#### 1.2 if语句

格式：

```python
if 表达式 :
    语句（四个空格缩进）
elif 表达式:
    语句
else:
    语句
```

举例：

```python
a = []
if a is None:
    pass
if a is not None:
    pass
```

或

```python
a = 'hello world'
if 'or' in a:
    print 'yes'
```

#### 1.3 while语句

格式：

```python
while 表达式:
    语句
```

举例：

```python
a = 0
while True:
    if a == 6:
        break
    if a < 10:
        a = a + 1
        continue
```

#### 1.4 for语句与range函数

`for`语句格式：

```python
for 变量 in 序列:
    语句
```

`range`函数：生成整数列表

格式：`range(start, stop[, step])`，`start`默认是0，`step`默认是1，`stop`必须有，范围左闭右开，即`[start, stop)`

举例：

```python
for i in range(5):
    print i

c = 'hello'
for i in c:
    print i
```

#### 1.5 输入输出

|            | python2                                                      | python3                                |
| ---------- | ------------------------------------------------------------ | -------------------------------------- |
| 输入       | `a = raw_input()` 输入内容会被转换成字符串，建议使用<br />`a = input()` 输入什么类型就是什么类型 | `a = input()` 输入内容会被转换成字符串 |
| 输出       | `print a`                                                    | `print(a)`                             |
| 不换行输出 | `print a,`                                                   | `print(a, end='')`                     |

#### 1.6 列表

定义列表: `a = [1, 2, 'hello', 3]`

随机访问: `a[下标]`，如`a[0] `，`a[-1]`表示最后一位元素

获取a列表第0、1个元素：`c = a[0:2]`

翻转a列表：`d = a[::-1]`

复制列表：`b = a[:]`

打印列表: `print a`

遍历访问:

```python
for element in a:
    print element
```

修改列表: `a[index] = value`

添加元素:

```python
a.append(element) # 尾部添加

a.insert(index, element) # 位置index+1添加元素
```

删除元素:

```python
del a[index] # 知道下标

a.pop(index) # 知道下标，默认删除尾部数据

a.remove(value) # 删除第一个指定的值 
```

一些补充：

- `len(a)`获取列表元素个数
- `a.sort()`对列表永久排序
- `sorted(a)`对列表临时排序
- `a.reverse()`对列表元素进行翻转

#### 1.7 元组

可以看作列表常量：`a = (1, 2, 'hello', 3)`

如果只有一个元素： `b = (1,)` 用逗号结束，为了避免歧义：`(1)`表示小括号中1

遍历元组：

```python
for element in a:
    print element
```

元组的元素不可更改：`a[0] = 0`会异常

如果元组里面的元素是列表，那么列表里的元素可以修改

```python
c = [1, [1, 2], 3]
c[1][0] = 0 # 修改后：c = [1, [0, 2], 3]
```

#### 1.8 字典

创建字典: `a = dict()`

添加元素:

```python
a = dict()
a['name'] = 'wanghe'
a['age'] = 22 
```

修改元素: `a[ 'age'] = 18`

删除某个键值对: `del a['age']`

遍历字典:

```python
for key, value in a.iterms(): # 遍历key, value
    print key, value
for key in a.keys(): # 遍历所有key
    print key
for value in a.values(): # 遍历所有value
    print value
```

1.9 函数

格式：

```python
def function(参数):
    语句
```

位置参数：`def f(a):`

关键字参数：`def f(a = 1):`

位置参数必须在关键字参数之前

参数顺序必须是位置参数 关键字参数 任意数量参 任意数量的关键字参数

#### 1.10 文件读写

语法:

```python
file = open(file_name [, access_mode][, buffering])
file.close()
'''
file_name：文件名称字符串
access_mode：打开文件的模式，默认只读(r)。
buffering:取值0，没有寄存。取值1，访问文件时会寄存行。取值大于1，表示寄存区的缓冲大小。取值为负，寄存区的缓冲大小为系统默认。
'''
```

推荐:

```python
with open('文件名') as file:
    语句
```

打开文件模式：

| 文件模式 |         描述         |
| :------: | :------------------: |
|    r     |         只读         |
|    w     |         只写         |
|    a     |        只追加        |
|  r+ w+   |       可读可写       |
|    a+    |       读写追加       |
| rb wb ab | 二进制文件读 写 追加 |
| rb+ wb+  |  二进制文件可读可写  |
|   ab+    |  二进制文件读写追加  |

常用函数:

```python
file.read() # 读取所有内容

file.readline() # 读取一行

file.readlines() # 读取所有行，按行分割成列表

file.write(字符串) # 写入文件，必须是字符串类型

file.writelines(字符串或者列表) # 按行写入
```

#### 1.11 类

```python
class person():		# person是类名
    def __init__(self, name, age):
        self.name = name
        self.age = age
    def print_name(self):
        print self.name
    def print_age(self):
        print self.age
    if __name__ == '__main__':	# 执行当前py代码会从这里调用
        p = person('someone', 18)	# 创建类对象
        p.print_name()

```

#### 1.12 异常

语法:

```python
try:
    语句
except 异常类型:
    语句
finally:
    语句
```

通常通过打印异常类型来判断产生某种异常：

```python
try:
    语句
except BaseException as e:
    print type(e)   
```

### 二、python脚本

#### 2.1 python脚本执行shell命令

- 无返回值

  ```python
  import os
  
  os.system(cmd) # 执行命令但不返回结果
  ```

- 有返回值

  ```python
  from subprocess import Popen, PIPE
  
  def run_cmd(cmd):
      res = Popen(cmd, stdin=PIPE, stdout=PIPE, stderr=PIPE, shell=True)
      out, err = res.communicate()
      ret = res.wait()
      return (ret, out, err)
  # ret是命令执行返回码0是成功1是失败
  # out是命令执行结果 err是错误信息
  
  ret, out, err = run_cmd('ls /root') # 调用shell执行
  ```

#### 2.2 python读取json文件

```python
# unicode 转 utf-8
import json

def unicode_convert(input):
    if isinstance(input, dict):
        return {unicode_convert(key): unicode_convert(value) for key, value in input.iteritems()}
	elif isinstance(input, list):
        return [unicode_convert(element) for element in input]
	elif isinstance(input, unicode):
        return input.encode('utf-8')
	else:
        return input
# 读取json文件							
with open('test.json', 'r') as file:
    a = json.load(file)
    print a
    a = unicode_convert(a)
    print a
    
# 一些用法
json.load(文件对象) # 读取json文件变成字典
json.dump(文件对象)	# 写入json文件
json.loads(str)	# json格式字符串转字典
json.dumps(dict) # 字典转json格式字符串
```

#### 2.3 python访问pg数据库

`python2`需要安装`pip2`和`psycopg2`

下载地址：[https://pypi.org/project/psycopg2-binary/](https://pypi.org/project/psycopg2-binary/)

```python
import psycopg2

conn = psycopg2.connect(user='用户名', password='密码', host=主机, port=端口, database='数据库名')
cursor = conn.cursor()
sql = 'sql语句'
cursor.execute(sql)
entries = cursor.fetchall()[0][0] # 获取sql返回结果
cursor.close()
conn.close()
```

#### 2.4 python写日志文件

```python
import logging

logger = logging.getLogger(__name__) # 初始化logger时指定logger名字

def initialization_log():
    logger.setLevel(level=logging.INFO) # 设置异常等级
    filepath = '/var/log/messages' # 日志文件路径
    handler = logging.FileHandler(filepath)
    handler.setLevel(logging.INFO)
    formatter = logging.Formatter('%(asctime)s - %(levelname)s - %(message)s')	# 打印时间、等级、内容
    handler.setFormatter(formatter)
    logger.addHandler(handler)

# 打印日志
logger.info('内容')
logger.error('内容')
logger.warning('内容')

```

日志级别：

设置了输出 `level`，系统便只会输出 `level` 数值大于或等于该 `level` 的的日志结果，例如我们设置了输出日志 `level` 为 `INFO`，那么输出级别大于等于 `INFO` 的日志，如 `WARNING`、`ERROR` 等，`DEBUG` 和 `NOSET` 级别的不会输出。

|   级别   | 数值 |
| :------: | :--: |
| CRITICAL |  50  |
|  FATAL   |  50  |
|  ERROR   |  40  |
| WARNING  |  30  |
|   WARN   |  30  |
|   INFO   |  20  |
|  DEBUG   |  10  |
|  NOTSET  |  0   |

`Formatter`：

|      格式       |                    含义                     |
| :-------------: | :-----------------------------------------: |
|   %(levelno)s   |             打印日志级别的数值              |
|  %(levelname)s  |             打印日志级别的名称              |
|  %(filename)s   |             打印当前执行程序名              |
|  %(pathname)s   | 打印当前执行程序的路径，其实就是sys.argv[0] |
|  %(funcName)s   |             打印日志的当前函数              |
|   %(lineno)d    |             打印日志的当前行号              |
|   %(asctime)s   |               打印日志的时间                |
|   %(thread)d    |                 打印线程ID                  |
| %(threadName)s  |                打印线程名称                 |
|   %(process)d   |                 打印进程ID                  |
| %(processName)s |                打印进程名称                 |
|    %(module)    |                打印模块名称                 |
|   %(message)s   |                打印日志信息                 |

### 三、python进阶

#### 3.1 生成器

列表生成式

```python
 # 生成列表
 a = range(6)
 print a 

 # 生成x*x列表
 a = [x * x for x in range(6)]
 print a

 # 引入if语句的列表生成式
 a = [x * x for x in range(6) if x%2 == 0]
 print a 

 # 双重循环的列表生成式
 a = [m + n for m in 'HMN' for n in 'abc']
 print a
```

生成器

```python
a = (x * x for x in range(6))
# 访问方式：a.next()  	直到抛出StopIteration
for i in a:
    print i
```

引入`yield`关键字

`yield`的作用等价于`return`，但是再次调用函数的时候会从`yield`语句的下一句开始执行

```python
def square(n):
    for x in range(n + 1):
        yield x * x 
for i in square(5):
    print i
```

总结: 生成器就是列表生成式的内存优化

#### 3.2 高阶函数

命令式编程: 关心解决问题的步骤

函数式编程: 关心数据的映射

函数式编程特性：

1. 函数必须有输入和输出
2. 输入不变，输出一定不变
3. 变量不可变，`x = 1`就不能修改`x`的值
4. 高阶函数

高阶函数: 允许把函数本身作为参数传入另一个函数，还允许返回一个函数

命令式（c语言）

```c
int add_abs(int a, int b) {
    int r = 0;
    r += abs(a);
    r += abs(b):
    return r;
}
```

函数式（c语言）

```c
int add_abs(int a, int b) {
    return abs(a) + abs(b);
}
```

`python`中的高阶函数：

- `map`: 对序列中每个元素调用函数并返回成一个新的列表

  格式: `map(函数, 序列)`

- `reduce`: 把函数执行的结果作为一个参数与下一个元素进行计算

  格式: `reduce(函数, 序列)`

  注: 该函数必须两个参数

- `filter`:  对序列中每一个元素调用函数，返回值是false则丢弃

  格式: `filter(函数，序列)`

- `sorted`: 按照指定顺序排序

  格式: `sorted(序列，比较函数)`

  关于比较函数通常认为两个元素`x` , `y`

  `x < y`返回`-1`

  `x == y`返回`0`

  `x > y` 返回`1`

#### 3.3 匿名函数

```python
def f(x):
    return x * x
```

等价于

```python
lambda x: x*x
```

匿名函数通常用作参数传给高阶函数

表达式只能用纯表达式，不能赋值或使用`while`、`try`等语句

```python
a = ['hello', 'name', 'test']

sorted(a, key = lambda word: word[::-1])

# 结果为['name', 'hello', 'test'] 

# 根据反转排序但不会反转单词
```

#### 3.4 装饰器

假设函数func返回的结果并不是需要的，但是又不想改变func的源码，该如何做？

```python
def dec_f(func): # 把函数作为参数传进来
    def new_f():
        func() # 对函数参数进行调用
        print '函数调用结束' # 添加输出
	return new_f # 新函数作为对象返回

@dec_f # @dec_f 等价于 f1 = dec_f(f1)
def f1():
    print 'f1被调用'

@dec_f
def f2():
    print 'f2被调用'

f1()
f2()
print f1
print f2
'''
结果：
f1被调用
函数调用结束
f2被调用
函数调用结束
'''
```

