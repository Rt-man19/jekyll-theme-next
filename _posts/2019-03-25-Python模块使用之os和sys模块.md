---
title: Python模块使用之os和sys模块
tags:
  - Python3
categories:
  - Python
---

### os

os 模块是与操作系统交互的一个接口，它提供了一些方便使用操作系统相关功能的函数。

关于这些函数的可用性的说明：

- 所有 Python 内建的操作系统相关的模块的设计都是为了使得在同一功能可用的情况下，保持接口的一致性；例如，函数 `os.stat(path)` 以相同的格式返回关于 path 的统计信息（这个函数同时也是起源于 POSIX 接口）。
- 针对特定的操作的拓展同样在可用于 os 模块，但是使用它们必然会对可移植性产生威胁。
- 所有接受路径或文件名的函数都同时支持字节串和字符串对象，并在返回路径或文件名时使用相应类型的对象作为结果。

**文件名，命令行参数，以及环境变量**

在 Python 中使用字符串类型表示文件名、命令行参数和环境变量。 在某些系统上，在将这些字符串传递给操作系统之前，必须将这些字符串解码为字节。 Python 使用文件系统编码来执行此转换（请参阅 [sys.getfilesystemencoding()](https://docs.python.org/zh-cn/3/library/sys.html#sys.getfilesystemencoding)）。



**方法和参数说明**

| 方法                             | 说明                                                         |
| :------------------------------- | :----------------------------------------------------------- |
| os.getcwd()                      | 获取当前工作目录，绝对路径，即当前python脚本工作的目录       |
| os.chdir(“dirname”)              | 改变当前脚本工作目录；相当于 shell 下 cd                     |
| os.curdir                        | 返回当前目录: (‘.’)                                          |
| os.pardir                        | 获取当前目录的父目录字符串名：(‘..’)                         |
| os.makedirs(‘dirname1/dirname2’) | 递归创建目录，相当于 shell 的 mkdir -pv                      |
| os.removedirs(‘dirname1’)        | 递归删除空目录，若不是空目录无法删除，会报错                 |
| os.mkdir(‘dirname’)              | 生成单级目录；相当于 shell 的 mkdir dirname                  |
| os.rmdir(‘dirname’)              | 删除单级空目录，若目录不为空则无法删除，报错；相当于 shell 中 rmdir dirname |
| os.listdir(‘dirname’)            | 列出指定目录下的所有文件和子目录，包括隐藏文件，并以列表方式打印 |
| os.remove()                      | 删除一个普通文件，不能删除目录                               |
| os.rename(“oldname”,”newname”)   | 重命名文件/目录，相当于 shell 中的 mv                        |
| os.stat(‘path/filename’)         | 获取 文件/目录 状态信息                                      |
| os.sep                           | 输出操作系统特定的路径分隔符，win下为”\“,Linux下为”/“        |
| os.linesep                       | 输出当前平台使用的换行符，win下为”\t\n”,Linux下为”\n”        |
| os.pathsep                       | 输出系统环境变量中每个路径的分隔符，linux是’:’，windows是’;’ |
| os.name                          | 输出字符串指示当前使用平台 win->’nt’; Linux->’posix’         |
| os.system(“bash command”)        | 运行 shell 命令，直接显示                                    |
| os.popen(“bash command”)         | 运行 shell 命令，获取执行结果                                |
| os.environ                       | 获取系统环境变量                                             |
| os.path.abspath(path)            | 返回 path 规范化的绝对路径                                   |
| os.path.split(path)              | 将 path 分割成目录和文件名二元组返回                         |
| os.path.dirname(path)            | 返回 path 的目录。其实就是 os.path.split(path) 的第一个元素  |
| os.path.basename(path)           | 返回 path 最后的文件名。若 path 以 `/` 或 `\` 结尾，那么就会返回空值。即os.path.split(path) 的第二个元素 |
| os.path.exists(path)             | 如果 path 存在，返回 True；如果 path 不存在，返回 False      |
| os.path.isabs(path)              | 如果 path 是绝对路径，返回 True                              |
| os.path.isfile(path)             | 如果 path 是一个存在的文件，返回 True。否则返回 False        |
| os.path.isdir(path)              | 如果 path 是一个存在的目录，则返回 True。否则返回 False      |
| os.path.join(path1,path2, …)     | 将多个路径组合后返回，第一个绝对路径之前的参数将被忽略       |
| os.path.getatime(path)           | 返回 path 所指向的文件或者目录的最后访问时间                 |
| os.path.getctime(path)           | 返回 path 所指向的文件或者目录的文件属性被修改的时间         |
| os.path.getmtime(path)           | 返回 path 所指向的文件或者目录的文件内容被修改的时间         |
| os.path.getsize(path)            | 返回 path 所指向的文件或者目录的文件大小                     |

注意：`os.stat('path/filename')` 获取 文件/目录信息 的结构说明

```text
st_mode: inode 保护模式
st_ino: inode 节点号。
st_dev: inode 驻留的设备。
st_nlink: inode 的链接数。
st_uid: 所有者的用户ID。
st_gid: 所有者的组ID。
st_size: 普通文件以字节为单位的大小；包含等待某些特殊文件的数据。
st_atime: 最后一次文件被访问的时间。
st_mtime: 最后一次文件内容发生修改的时间。
st_ctime: 最后一次文件属性发生改变的时间。
```

### sys

sys 模块是与 Python 解释器交互的一个接口。它提供了一些变量和函数，这些变量可能被解释器使用，也可能由解释器提供。这些函数会影响解释器。

sys.**argv**

一个列表，其中包含了被传递给 Python 脚本的命令行参数。 `argv[0]` 为脚本的名称（是否是完整的路径名取决于操作系统）。

```python
[root@hadoop003 ~]# cat test.py
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import sys

print(sys.argv)
print(sys.argv[0])
[root@hadoop003 ~]# python3 test.py
['test.py']
test.py
[root@hadoop003 ~]# python3 test.py --help --pidfile abc 1234
['test.py', '--help', '--pidfile', 'abc', '1234']
test.py
```

如果是通过 Python 解释器的命令行参数 `-c` 来执行的， `argv[0]` 会被设置成字符串 ‘-c’ 。如果没有脚本名被传递给 Python 解释器， `argv[0]` 为空字符串

```python
[root@hadoop003 ~]# python3 -c 'import os,sys; print(os.name);print(sys.argv[0])'
posix
-c
```

sys.**version**

获取 Python 解释程序的版本信息

```python
[root@hadoop003 ~]# python3 -c 'import sys; print(sys.version)'
3.7.0 (default, Jun 20 2018, 13:34:30) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-39)]
```

其他方法和参数说明

| 方法                          | 说明                                                     |
| :---------------------------- | :------------------------------------------------------- |
| sys.exit(n)                   | 退出程序，正常退出时exit(0)                              |
| sys.maxint                    | 最大的Int值                                              |
| sys.path                      | 返回模块的搜索路径，初始化时使用 PYTHONPATH 环境变量的值 |
| sys.path.append()             | 添加模块的搜索路径                                       |
| sys.platform                  | 返回操作系统平台名称                                     |
| sys.stdout.write(‘please:’)   | 向屏幕输出一句话,等价于：print(‘please:’)                |
| val=sys.stdin.readline()[:-1] | 获取输入的                                               |

示例:

```python
import os
import sys

# 文件写入
with open('/root/app/test.txt',w) as fw:
    fw.write('dddd')
    fw.write(os.linesep)
    fw.write('taayy')
    fw.write(os.linesep)
# 获取脚本自身名称
me = os.path.basename(sys.argv[0])
```

