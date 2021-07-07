+++
categories = []
tags = ["Python"]
title = "Python中list增加元素的几种方法"
date = 2017-11-30T07:51:59+08:00
url = "/post/python_add_elements_in_list"
+++
## 一

我一般会用append方法来把元素增加到list中.但今天我看到一种新方法：

```python
a = [1]
a += 2,
# a = [1, 2]
```

感觉挺神奇的，所以我查了一些资料，总结如下.

## 二

### 1.元组

上面的2,代表了单元素元组(one element tuple).在元组的表达式中，逗号是必需的，括号则可要可不要.对于单元素元组来说，后面的逗号必须有；而对于多元素元组，只要中间有逗号即可.

```python
1,
1,2,3
```

### 2.a+=b和a=a+b的区别

a+=b

```python
>>> a = [1]
>>> b = a
>>> b += [2]
>>> a
[1, 2]
>>> b
[1, 2]
```

a=a+b

```python>>> a = [1]
>>> b = a
>>> b = b + [2]
>>> a
[1]
>>> b
[1, 2]
```

对于可变对象，+=操作调用**iadd**方法，直接在原对象a上进行更新，该方法的返回值是None；+操作调用**add**方法，返回一个新的对象，原对象不修改，所以b被重新赋值，b指向了一个新的对象.
  
对于不可变对象，只有**add**方法，所以两者效果一样.

不仅如此，在list增加元素方面，两者也有不同.

```python
>>> a = []
>>> a = a + 1
Traceback (most recent call last):
  File "&lt;stdin>", line 1, in &lt;module>
TypeError: can only concatenate list (not "int") to list
>>> a = a + (1,)
Traceback (most recent call last):
  File "&lt;stdin>", line 1, in &lt;module>
TypeError: can only concatenate list (not "tuple") to list
>>> a = a + [1]
>>> a
[1]
```

```python>>> a = []
>>> a += 1,
>>> a += [1]
>>> a += 1
Traceback (most recent call last):
  File "&lt;stdin>", line 1, in &lt;module>
TypeError: 'int' object is not iterable
```

可以看出对于a+=b，b只要是iterable即可；而对于a=a+b，b必须是list类型.其中的原因，可能是**iadd**和**add**方法实现有差异，这里就不做深究了.

### 3.list增加元素几种方法的速度对比

原本我在一个Python文件里写几种方法的实现函数，然后调用timeit模块逐个运行，但是发现无论顺序如何，第一个方法总是时间偏长，不知道原因在哪.所以为了避免这种误差，使用python -m timeit '(CODE)'分别运行，结果如下

```python
>>> python -m timeit 'a=[];a.insert(0,[1])'
1000000 loops, best of 3: 0.354 usec per loop
>>> python -m timeit 'a=[];a.extend([1])'
1000000 loops, best of 3: 0.228 usec per loop
>>> python -m timeit 'a=[];a.append(1)'
10000000 loops, best of 3: 0.153 usec per loop
>>> python -m timeit 'a=[];a+=[1]'
10000000 loops, best of 3: 0.149 usec per loop
>>> python -m timeit 'a=[];a+=1,'
10000000 loops, best of 3: 0.101 usec per loop
```

insert和extend这两种方法不是专门为list增加元素设计的，所以用时较长.append方法和加list时间相近，但是append只适用于一个一个元素的增加，而加list可以一次增加多个元素.加tuple是其中最快的.
  
考虑加list和加tuple两种方法的时间差异可能存在于list和tuple的定义中，所以进一步实验如下：

```python
>>> python -m timeit 'a=[1]'
10000000 loops, best of 3: 0.069 usec per loop
>>> python -m timeit 'a=1,'
10000000 loops, best of 3: 0.0214 usec per loop
```

果然不出所料，tuple定义比list快，直接导致了加tuple比加list快.
