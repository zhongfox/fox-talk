---
layout: post
tags : [python]
title: Python学习笔记

---


### 编码

Python2.X 中默认的编码格式是 ASCII 格式，在没修改编码格式时无法正确打印汉字，所以在读取中文时会报错。
解决方法为只要在文件开头加入 `# -*- coding: UTF-8 -*-` 或者 `#coding=utf-8` 就行了

Python3.X 源码文件默认使用utf-8编码，所以可以正常解析中文，无需指定 UTF-8 编码

### 标识

以下划线开头的标识符是有特殊意义:

* 以单下划线开头（_foo）的代表不能直接访问的类属性，需通过类提供的接口进行访问，不能用`from xxx import *`而导入
* 以双下划线开头的（__foo）代表类的私有成员
* 以双下划线开头和结尾的（__foo__）代表python里特殊方法专用的标识，如__init__（）代表类的构造函数

### 注释

单行注释: `#`

多行注释用三个引号

```python
'''
这是多行注释，使用单引号。
这是多行注释，使用单引号。
'''

"""
这是多行注释，使用双引号。
这是多行注释，使用双引号。
"""
```

### 命令行参数

* sys.argv 命令行参数列表
* sys.argv[0] 表示脚本名
* len(sys.argv) 是命令行参数个数

### 对象三要素

python 中一切都是对象

* id: 可以通过内置函数`id()`获得
* type: 可以通过内置函数`type()`获得
* vaule

---

## 变量

### 多重赋值

```python
a = b = c = 1
a, b, c = 1, 2, "john"
```

### 删除变量引用

```python
del var
del var_a, var_b
```

### 作用域

* 全局变量作用域
* 局部变量作用域(函数内声明的变量)

#### global

* 每个函数都有自己的命名空间
* 用于函数内部声明全局变量. 
* 在函数内部, 如果不使用global, 对同名全局变量赋值只会创建一个新的局部变量, 此时应该使用global
* 声明和赋值需要分开:

  ```
  global a, b
  a, b = 3, 4
  ```

#### globals() 和 locals() 函数

TODO

---

## 类型

标准数据类型:

* Numbers（数字）
* String（字符串）
* List（列表）
* Tuple（元组）
* Dictionary（字典）

判断数据类型:

* `type()` 返回所属直接类型, 返回值是类型对象, 通常等于`object.__class__`
* `isinstance()` 继承链上的类型都会返回True

### Numbers

不可改变的数据类型

* int（有符号整型）
* long（长整型[也可以代表八进制和十六进制]）
* float（浮点型）
* complex（复数）

### String

可以使用引号( ' )、双引号( " )、三引号( ''' 或 """ ) 来表示字符串; 其中三引号可以由多行组成

* 按索引读: `[index]` 不可写

* 截取:
  * `[头下标:尾下标]` 左闭右开
  * 其中下标是从 0 开始算起，可以是正数或负数，下标可以为空表示取到头或尾, 负数表示倒索引
  * 返回一个新的对象

* 加号（+）是连接运算符

* 星号（*）是重复操作

是否可变?

### List

* 按索引读: `[index]`, 可写

* 截取:
  * `[头下标:尾下标]` 左闭右开
  * 其中下标是从 0 开始算起，可以是正数或负数，下标可以为空表示取到头或尾, 负数表示倒索引
  * 返回一个新的对象

* 删除元素: `del list1[2]` 注意使用方式, 按照索引删除, 不会留空洞

* 加号（+）是连接运算符

* 星号（*）是重复操作

内置函数:

* len(list)
* max(list)
* min(list)
* list(seq): 将元组转换为列表

内置方法:

* `#append(obj)`

### Tuple

类似于List, 元组用"()"标识。内部元素用逗号隔开。但是元组不能二次赋值，相当于只读列表.

* 任意无符号的对象，以逗号隔开，默认为元组

* 元组中只包含一个元素时，需要在元素后面添加逗号: `tup1 = (50,)`

* 访问:
  * 按照索引: `tup1[0]`, 支持负索引
  * 按照片段: `tup2[1:5]` 左闭右开

* 元组无法修改, 但是可以连接创建一个新的元组: `tup3 = tup1 + tup2`

* for in 遍历: `for x in (1, 2, 3): print x`

内置函数:

* len(tuple)
* max(tuple)
* min(tuple)
* tuple(seq) 将列表转换为元组

### Dictionary

* 键必须是不可变的，如字符串，数字或元组. 字符串key不能省略引号

* 读取不存在的键, 将抛出KeyError

* 删除key: `del dict[somekey]`

内置函数:

* len(dict)
* str(dict)

内置方法:

* `#clear()`: 删除字典中所有元素
* `#setdefault(key, default=None)`: set if not exist, 如果不存在该key, 立刻设置上.
* `#items()`
* `#keys()`
* `#values()`

### set

创建: set()

---

## 运算符

* 逻辑运算符: and, or, not

  逻辑假: False, None, 0, ''

* 成员运算符: in, not in, 可用于判断的序列有: 字符串, 列表, 元组, 字典(判断key)

* 比较:
  * id 比较: is
  * vaule比较: ==

---

## 列表解析

列表推导是一个将一个列表（实际上是任意可迭代对象）转换成另一个列表

* 嵌套循环: 顺序从前往后


字段推导:

`flipped = {value: key for key, value in original_dict.items()}`

集合推导:

```python
words = ['abc', 'xyz', 'abc']
first_letters = {w for w in words} # ['xyz', 'abc']
```

---

## 函数

* 参数传递时**值传递**, 传递的内容区分值类型(不可变对象: 整数、字符串、元组)和引用类型(可变对象)
* 函数的第一行语句可以选择性地使用文档字符串—用于存放函数说明
* return 语句可以省略, 默认返回None

参数:

* 必需参数

  `def printme( str ):`

* 关键参数: 关键参数和必需参数的形参列表完全一致, 也就是既可以按照顺序传递(必须参数), 也可以键值对乱序传递(关键参数)

  `def printinfo( name, age ):`

  调用: `printinfo( age=50, name="runoob" )`

* 默认参数

  `def printinfo( name, age = 35 )`

* 不定长参数: 前置星号定义, 不定参数在函数中是一个元组(tuple)

  `def printinfo( arg1, *vartuple ):`

### lambda

* 创建匿名函数, 对象化
* 调用方式同普通函数

`sum = lambda arg1, arg2: arg1 + arg2;`

### 内置函数

#### 数据类型转换

* int(x [,base])
* long(x [,base])
* str(x) 将对象 x 转换为字符串
* repr(x) 将对象 x 转换为表达式字符串

#### print

print 默认输出是换行的，如果要实现不换行需要在变量末尾加上逗号

---

## 模块

* py 文件即模块


### import

* 可导入目标: 全局变量, 函数, 类?, 模块
* 导入后使用模块的函数: `模块名.函数名`
* 一个模块只会被导入一次，不管你执行了多少次import  缓存???
* 如果想重新加载模块, 需要使用reload函数: `reload(module_name)`
* 从模块中导入一个指定的部分到当前命名空间: `from modname import name1[, name2[, ... nameN]]`
* 把一个模块的所有内容全都导入到当前的命名空间: `from modname import *`

### dir 函数

`dir(modname)` 输出该模块导出内容的字符串列表, 包括模块(modname中import的模块), 变量, 函数

模块导出的特殊属性:

* `__builtins__`
* `__doc__`
* `__file__`
* `__name__`
* `__package__`

### 搜索路径

1. 当前目录
2. 遍历 PYTHONPATH
3. 默认路径

---

## 包

* 包就是一个包含若干模块的一个模块, 且包括一个`__init__.py`文件

* 引入包中的模块: `from package_name import module_name` 或者 `import package_name.module_name`

* TODO: 包之间的模块如何import

---

## 面向对象

```python
class ClassName:
   '类的帮助信息'   #类文档字符串
   class_suite  #类体
```

* `ClassName.__doc__` 获取类帮助信息文档

