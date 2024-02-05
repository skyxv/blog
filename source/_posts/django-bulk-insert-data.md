---
title: django bulk_create 插入大量数据的正确打开方式
description: 
date: 2024-01-26
scheduled: 2024-01-26
draft: false
---

先看这段`bulk_create`代码:
```python
from django.db import models

class DemoModel(models.Model):

    name = models.CharField(max_length=255)
    age = models.IntegerField()

# 模拟待插入数据
objs = [(f"name{i}", i) for i in range(1000000)]

objs_to_create = []

for obj in objs:
    objs_to_create.append(
        DemoModel(name=obj[0], age=obj[1])
    )

DemoModel.objects.bulk_create(objs_to_create)

```

这段代码确实可以运行，但它存在两个严重问题:
* `objs_to_create` 内存占用量随着对象数量和大小增加而增加，碰到大量插入数据的场景，很容易导致内存溢出。
* `bulk_create` 方法有一个`batch_size`参数，是用来支持将传入的对象分批保存的，默认为`None`, 即不分批，如果对象很多，一是有可能会超出sql长度限制，二是会对数据库造成负面影响：例如内存压力，表长时间被锁定，连接超时，甚至直接挂掉等。

第二个问题好解决，直接加上batch_size参数即可。那么第一个问题呢?

答案是`生成器`。我们知道python提供了`yield`原语，它逐个产生值而不是一次性生成所有值保存在内存里，这会使生成器对处理大量数据时非常高效。

接下来是优化后的代码:
```python
from itertools import islice

from django.db import models

class DemoModel(models.Model):

    name = models.CharField(max_length=255)
    age = models.IntegerField()

# 模拟待插入数据
objs = [(f"name{i}", i) for i in range(1000000)]

# 生成器
def obj_generator(objs):
    for obj in objs:
        yield DemoModel(name=obj[0], age=obj[1])

objs_to_create = obj_generator(objs)

# bulk_create 无法直接接收一个生成器作为参数，因此我们需要按batch_size提取
batch_size = 500
while True:
    batch = list(islice(objs_to_create, batch_size))
    if not batch:
        break
    DemoModel.objects.bulk_create(batch, batch_size)

```



