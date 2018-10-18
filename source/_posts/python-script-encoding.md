---
title: Python 输出中文乱码问题
date: 2018-10-18 14:15:39
tags:
 - 转载
 - Python
 - 编码   
---

>这几天又折腾电脑，很多东西都又重装了一遍（我为啥要说又。。。），为了学习Cocos2dx，我装上了Python2，但是我电脑上本来就有Python3，处理共存问题倒是问题不大，但是当我用上Python2之后，才发现Py2有多坑，它默认对Py脚本的编码是ASCII，即使你处理好这个问题，仍然可能会遇到打印中文出乱码的情况，对此我上网找了很多资料，总算是找到个比较全面的解决方案。

<!--more-->

# 解决方法

## 增加系统全局变量

以 windows 系统为例，添加系统变量：

PYTHONIOENCODING=UTF8

## 修改 VSC 配置文件

*VSC 即 Visual Studio Code 的缩写*

F1 键调出控制台，输入task,选择任务：配置任务运行程序,打开tasks.json文件，增加以下信息：

```json
    "options": {
    "env":{
    "PYTHONIOENCODING": "UTF-8"
    }
}
```

## 在代码里更改编码

在每个需要中文的 python 文件中添加如下代码：

```python
import io
import sys

#改变标准输出的默认编码
sys.stdout=io.TextIOWrapper(sys.stdout.buffer,encoding='utf8')
```

使用方法1和方法2需要重启 VSC。
方法1可以一劳永逸。

>原作者：木公木  
链接：https://www.jianshu.com/p/615c5aafeb41

*非商业用途，侵权必删 QAQ*
