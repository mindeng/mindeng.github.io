
---
title: "Python Tips"
date: 2022-12-03T10:40:00.000Z
lastmod: 2022-12-04T13:38:00.000Z
tags: ['python']
draft: false
---



## 函数工具 functools


### 缓存类装饰器

**``@cache``**

用于自动缓存函数的返回结果（即其他语言常见的 *memoize*）。

使用起来非常简单，给函数增加 ``@cache`` 装饰器（Decorator）即可：

```python
from functools import cache

@cache
def factorial(n):
    return n * factorial(n-1) if n else 1

>>> factorial(10)      # 无缓存结果, 触发 11 次递归调用
3628800
>>> factorial(5)       # 直接返回缓存结果
120
>>> factorial(12)      # 触发 2 次递归调用, 其余 10 次直接使用缓存的结果
479001600
```

> 由于缓存是通过字典来实现的，key 就是函数的参数，所以，参数必须是 *hashable* 的。  

那如果想缓存属性（property）呢？很简单，直接在 ``@property`` 基础上，叠加一个 ``@cache`` 即可：

```python
from functools import cache
import statistics

class DataSet:
    def __init__(self, sequence_of_numbers):
        self._data = sequence_of_numbers

    @property
    @cache
    def stdev(self):
        return statistics.stdev(self._data)
```

**``@cached_property``**

上述这个是只读属性，如果想改成可写属性呢？functools 还提供了一个可写的版本 ``@cached_property``：

```python
from functools import cached_property
import statistics

class DataSet:
    def __init__(self, sequence_of_numbers):
        self._data = sequence_of_numbers

		# 该属性是可写版本
    @cached_property
    def stdev(self):
        return statistics.stdev(self._data)
```

**``@lru_cache``**

如果想限制缓存的 size 呢？可以使用 ``@lru_cache`` ：

```python
# maxsize 为可选参数，默认为 128
@lru_cache(maxsize=32)
def get_pep(num):
    'Retrieve text of a Python Enhancement Proposal'
    resource = 'https://peps.python.org/pep-%04d/' % num
    try:
        with urllib.request.urlopen(resource) as s:
            return s.read()
    except urllib.error.HTTPError:
        return 'Not Found'
```

> 和 ``@cache`` 类似，参数也必须是 *hashable* 的。  


### Partial object

``functools.partial`` 函数可以创建一个 *partial object* ， ``partial object`` 是一个可调用的对象。

字面上稍微有点抽象，看个例子就很清晰了：

```python
import functools

def add(a, b):
    return a + b

plus3 = functools.partial(add, 3)

plus3(1)
# out: 4

plus3(7)
# out: 10
```


## 闭包示例：对函数进行求导

```python
def derivative(f):
    # Computes the numerical derivative of a function.
    def df(x, dx=1e-6):
        return (f(x+dx) - f(x)) / dx
    return df

def g(x):
    return x ** 2

# first derivative of g
dg = derivative(g)

# second derivative of g
d2g = derivative(dg)
```


## Iterate over a list 2 items at a time

```python
for (c1, c2) in zip(s[0::2], s[1::2]):
    print c1, c2

```

No-copy version:

```python
from itertools import izip, islice

for (c1, c2) in izip(islice(s, 0, None, 2), islice(s, 1, None, 2)):
    print c1, c2

```


## Hex ↔ Bin

Hex string to bin string:

```python
s = 'bd2866849ae03b59861d40dd8bb0d4'
''.join([chr(int(''.join(two), 16)) for two in zip(s[0::2], s[1::2])])
```

Bin string to hex string:

```python
binstr = '\\xbd(f\\x84\\x9a\\xe0;Y\\x86\\x1d@\\xdd\\x8b\\xb0\\xd4'
''.join(('{0:02x}'.format(ord(c)) for c in binstr))
```