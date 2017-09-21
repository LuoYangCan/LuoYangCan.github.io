---
title: PY语言基础
date: 2017-09-21 23:04:40
tags: [Python]
---

# 前言

---

因为最近没事干想学下爬虫玩一下，顺便因为自己是信息安全专业，学一下py应该还蛮有趣的。

安装Python这些就不扯了

直接进入基础吧

# 数据

------

其实Python的数据类型和其他语言的差不多，大致也是以下几种

## 整数

Python可处理任意大小的整数。

由于计算机使用二进制，所以有时候用十六进制表示整数比较方便。

十六进制用`0x`前缀和0~9,a~f表示。例如`0xff00`

## 浮点数

浮点数就是小数。

之所以叫他浮点数，是因为用科学计数法表示时，一个浮点数的小数点位置是可变的。就像`1.23x10^9`和`12.3x10^8`完全相等。

对于很大的数或很小的浮点数，就必须用科学计数法表示

在Python中文名用e替代10。也就是说`1.23x10^9`等价于`1.23e9`

## 字符串

字符串是以单引号`‘`和双引号`"`括起来的任意文本

单引号双引号都可以用，只要不混用。

如果`'`本来就是一个字符，那可以用`""`括起来。

如果字符串内又有`"`又有`'`的话。

熟悉的转义字符就出现了，还是老样子，还是用`\`标识

Python的字符串是以Unicode编码的，所以Python的字符串支持多种语言

Python的字符串类型是`str`，在内存中用Unicode表示，一个字符对应若干个字节。如果要在网络上传输，或者保存到磁盘上，就需要把`str`转化为`bytes`

Python对`bytes`类型的数据用带`b`前缀的单引号或双引号表示：

```python
x = b'ABC'
```

要注意区分`ABC`和`b'ABC'`。后者虽然内容和前者一样，但是`bytes`的每个字符只占用一个字节。

以Unicode表示的`str`都可以通过`encjode()`方法编码为制定的`bytes`。

```python
>>> 'ABC'.encode('ascii')
b'ABC'
>>> '中文'.encode('utf-8')
b'\xe4\xb8\xad\xe6\x96\x87'
>>> '中文'.encode('ascii')
Traceback (most recent call last):
  File"<stdin>", line 1, in <module>
UnicodeEncodeError:'ascii' codec can't encode characters in position 0-1:ordinal not in range(128)
```

纯英文的`str`可以用`ASCII`编码为`bytes`，内容是一样的。含有中文的`str`可以用`UTF-8`编码为`bytes`。含有中文的`str`无法用`ASCII`编码，因为中文编码超过了`ASCII`编码的范围，会报错。

反过来我们如果要从网络上读取字节流，就要把`bytes`变为`str`，需要用到`decode()`方法：

```python
>>> b'ABC'.decode('ascii')
'ABC'
>>> b'\xe4\xb8\xad\xe6\x97\x87'.decode('utf-8')
'中文'
```

要计算长度，可以用`len()`函数

由于Python源代码也是一个问本文件，所以当你的源代码包含中文的时候，为了避免乱码，应当自始至终用UTF-8编码，当Python解释器读取源代码时，为了让它按UTF-8编码读取，我们常在开头上写：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
```

第一行是为了告诉Linux/OS X系统，这是一个Python可执行程序

第二行是为了告诉Python解释器，按照UTF-8编码读取源代码。

我们经常会输出类似`亲爱的xx您好！…`这种字符串。xx的内容是需要变化的

在Python中 ，这样的方式是用`%`实现

```python
>>> 'Hello, %s' % 'world'
'Hello, world'
>>> 'Dear %s , you have $%d' % ('Rues',7)
'Dear Reus , you have $7'
```

和许多语言一样，Python也有很多占位符

| 占位符  | 类型     |
| ---- | ------ |
| %d   | 整数     |
| %f   | 浮点数    |
| %s   | 字符串    |
| %x   | 十六进制整数 |

格式化整数和浮点数还可以指定是否补0和整数与小数的位数。

```python
>>> '%2d-%02d' % (3,1)
'3-01'
>>> '%.2f' % 3.1415926
'3.14'
```

如果你不确定要用什么，那就用`%s`，就好比Objective-C中的`%@`

```python
>>> 'Age: %s. Gender: %s' % (25,True)
'Age: 25. Gender: True'
```



### 打印字符串

我们可以用`print()`打印字符串。

Python允许用`r''`标识`''`内部的字符默认不转义。

```python
>>> print('\\\t\\')
\    \
>>> print(r'\\\t\\')
\\\t\\
```

如果字符串换行很多，写一行看着难受。Python还支持使用`'''...'''`标识多行内容

```python
>>> print('''line1
line2
line3''')

line1
line2
line3
```

## 布尔值

在其他很多语言中我们用`True`、`False`标识布尔值，Python也不例外。

在Python中，布尔值可以用`and`、`or`和`not`运算。

`and`是与运算

`or`是或运算

`not`是非运算

## 空值

空值是Python中一个特殊值，用`None`表示。

注意 `None`不是`0`，`0`是有意义的。

## 变量

变量不仅可以是数字，还可以是任意数据类型。

变量名必须是大小写英文、数字和`_`的组合。且不能以数字开头

Python是一门动态语言，变量本身类型不固定。我们可以把任意数据类型赋值给变量，同一个变量可以反复赋值。

## 常量

Python中常量一般都是全部大写。

事实上Python没有任何机制保证常量不会改变。所以全部大写常量名字只是一种习惯。

## list与tuple

### list

Python中有列表`list`这种数据类型，是有序的集合，可以随时添加删除期中的元素。有点像objc中的MutableArray

```python
//list
>>> classmates = ['Reus', 'Reus1', 'Rues2']
>>> classmates 
['Reus', 'Reus1', 'Reus2']
//空list
>>> c = []
```

我们可以用`len()`获得list元素的个数

```python
>>> len(classmates)
3
>>> len(c)
0
```

还可以用索引访问list的元素，甚至还可以直接获取倒数得到的数据

```python
>>> classmates[0]
'Reus'
>>> classmates[1]
'Reus1'

//溢出
>>> classmates[3]
Traceback (most recent call last):
  File"<stdin>". line 1, in <module>
IndexError: list index out of range
  
//查看最后一个元素
>>> classmates[-1]
'Reus2'
//倒数第二个
>>> classmates[-2]
'Reus1'
//越界溢出
>>> classmates[-4]
Traceback (most recent call last):
  File"<stdin>". line 1, in <module>
IndexError: list index out of range

  //追加元素
>>> classmates.append('Boy')
>>> classmates
['Reus', 'Reus1', 'Reus2', 'Boy']

//插入元素到指定位置
>>> classmates.insert(1,'May')
['Reus', 'May', 'Reus1', 'Reus2', 'Boy']

//删除末尾元素
>>> classmates.pop()
'Boy'
>>> classmates 
['Reus', 'May', 'Reus1', 'Reus2']

//删除指定位置元素
>>> classmates.pop(1)
'May'
>>> classmates
['Reus', 'Reus1', 'Reus2']

//替换元素
>>> classmates[1] = 'Boy'
>>> classmates
['Reus', 'Boy', 'Reus2']

```

`list`里的数据类型也可以不同，可以是整型，可以是字符串，还可以是布尔值

```python
>>> L = ['Apple', 123, True]
```

也可以是另一个list

```python
>>> M = [1, 2, [3, 4], 5]
>>> len(M)
4
```

### tuple

tuple和list非常相似，但是tuple一旦初始化就不能修改（除非里面有个list)。有点像objc中的NSArray

因为tuple不可变，所以相对来说是更安全的。

```python
//定义tuple
>>> t = (1,2)
>>> t
(1, 2)

//定义一个空的tuple
>>> t =()
>>> t 
()

//定义一个元素的tuple
>>> t = (1,)
>>> t 
(1,)
/**Python在显示只有一个的tuple时，也会加一个逗号，防止我们误解成数学上的括号**/
```

# 条件判断

Python中也有`if - else`语句

```python
age = 20
if age >= 18:
  	print('your age is',age)
    print('adult')
```

如果`if`判断为`True`，就把缩进的两行print语句执行了。

```python
age = 3
if age >= 18:
  print('adult')
elif age >=6:
  print('tennager')
else:
  print('kid')
```

和你想的一样，这个`elif`是`else if`的缩写

`if`的特点是从上往下判断，如果上面有一个判断为`True`，就会忽略掉剩下的`elif`和`else`。

# 循环

Python的循环有两种，一种是`for...in`循环，另一种是`while`循环

## for..in

`for x in ...`循环就是把每个元素带入变量`x`

```python
//1-10累加
sum = 0
for x in [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]:
  	sum = sum + x
print(sum)
```

但只做累加  想加到1000页不可能一直手写。还好有`range()`函数，可生成一个整数序列，再通过`list()`可以转换为list

```python
>>> list(range(5))
[0, 1, 2, 3, 4]
```

## while循环

这个循环大家学啥语言基本都会遇到，也就不多赘述了

```python
sum = 0
n = 99
while n > 0:      //只要条件为真就一直循环
  sum = sum + n
  n = n - 2
print(sum)
```



# 参考

---

[廖雪峰的官方网站](https:www.liaoxuefeng.com)

