---
layout: post
title: "Cheat Sheet on Python collections itertools and functools module"
date: 2019-02-24
excerpt: "Python 三个模块常用函数应用与实现速览"
tags: [python, algorithm, data structure]
comments: true
---

* toc
{:toc}

## 引言

Python 的 collections, itertools, functools 这三个模块，算是 builtin 模块中，和 Python 语言核心结合最紧密的一部分，甚至其中的一些函数，比如 `functools.reduce()` 本来就是从 Python2 中语言级关键字移过去的。其中的 collections 模块，提供了一些扩展的容器对象，超越了原本的 dict，set，list 和 tuple 的功能，进行了一些特殊功能和效率的改进。itertools 模块，则提供了一系列简单的函数接口，用来根据特定需求，快速生成各种 iterator。functools 模块则比较零碎，主要提供了一些针对函数的小工具，包括最著名的定义装饰器需要的 `wraps` 等。熟悉这几个模块后，很多时候可以写出更简洁，更可读和更函数式的代码。正因为其中提供的大部分功能，都可以简单的自己手写完成，因此这几个模块实际的被重视程度有些不足。本文就基于 Python 3.6 的文档，给出这几个模块主要 API 的用例，同时对稍微复杂的功能的实现和复杂度进行一点介绍。


注意以下的 jupyter notebook，均可以右上角选择在 binder 上执行或直接前往 gist 查看。或者选择直接下载。
{: .notice}

## Collections

<iframe src="https://nbviewer.jupyter.org/gist/refraction-ray/fbb6cb3b527d2ca49606b83cc6427e0e" width="100%" height="4000"></iframe>

简单评述：虽然有很多平衡树的第三方 module，但 Python 官方没有提供基于平衡树的数据结构终归是有点遗憾。虽然大部分时候 dict 可以完成任务，但基于按值排序的键值对存储，就只能求助于平衡树了（或者有类似复杂度的 skiplist）。这样常见的数据结构，似乎还是应该得到语言的直接支持。比如 cpp 的 stl 就同时支持基于 hash 和基于平衡树的键值对存储结构。

## Itertools

<iframe src="https://nbviewer.jupyter.org/gist/refraction-ray/4ac739ff0bc1a08fafc4cb56d376ce14" width="100%" height="5000"></iframe>

简单评述：这些函数里，排列组合的实现稍微需要思考一下，其他都比较容易。有些函数的意义可能比较反直觉，要注意一下。有些返回的 generator 可能是无限的，慎用 `list(genenrator)`。

## Functools

<iframe src="https://nbviewer.jupyter.org/gist/refraction-ray/f001c6297cf6ba824d906b59a8bdcf8d" width="100%" height="3000"></iframe>

简单评述：`lru_cache`无疑是动态规划这类 recursive with memorization 问题解决的利器。`wraps`也许是这篇速查表中最高频的函数了，因为想不出现问题的定义装饰器，`wraps` 是跑不掉的，不然自己手动修改那么多函数属性，将是灾难。`singledispatch` 看起来很强大，但似乎没有一个函数内部判断输入值的性质来得灵活，毕竟 Python 这种动态语言，提供了大量 runtime inspect 的工具，就显得这种基于类型的重载苍白了些。