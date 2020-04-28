---
layout: post
title: "Universal duck array and Universal ML"
date: 2020-04-27
excerpt: "后端无关的线性代数和自动微分编程"
tags: [python, machine learning]
comments: true
---

* toc
{:toc}


## 引言

写这篇东西的起因是最近在做 [tensorcircuit](https://github.com/refraction-ray/tensorcircuit) 这个项目，由于项目是构建在 [TensorNetwork](https://github.com/google/TensorNetwork) 的基础上，而 tensornetwork 这个库采用了多后端机制，同时支持用 numpy，jax，tensorflow 表达的张量。自然地，我的开发也努力地维持这一多后端机制，这就带来了一个有趣的副作用：不只前向操作，甚至求导，jit 等操作都可以做成统一的接口，从而实现写一次代码，可以在不同的引擎下运行，有点 universal ML language 的意思。当然我无意去丰富这一点，这不过是该项目的副产品，但是作为一个讨论的话题，还是蛮有意思的。众所周知，在 python 里有着众多类似 numpy.array 但又不是 numpy.array 的东西，尤其是机器学习框架兴起之后。比如 scipy 的 sparse，tensorflow，pytorch，jax 里的 tensor 或 array，等等。当然还有一些用于大规模分布式计算的库比如 dask 和 mars 也会提供类似的 array 界面。这些形似神也似，但 API 上却总是有着细微区别的 array 界面，无疑有时会给开发者一些困扰。本文就尝试讨论下统一这些 array 界面的有趣尝试。而文章最后，会更进一步地讨论一下不止统一了 array 界面，也统一了自动微分界面的尝试，这样一种尝试，可以让我们写出 universal ML programming，毕竟整个机器学习就是线性代数加自动微分而已。 从而一段代码，多引擎运行。注意这里的引擎，将不只是不同硬件 CPU/GPU/TPU 的区别，而是同时用 tensorflow， pytorch 或者 jax 运行。

## autoray

由于 Python 的反射机制非常好用，动态的导入库和在运行时生成注册及调用函数都非常方便，因此做一个简单的粗糙 wrapper 使得代码可以支持不同的 array 后端是很容易的。一个典型的例子就是 [autoray](https://github.com/jcmgray/autoray/blob/master/autoray/autoray.py)，其代码只有几百行，就已经可以完整支持通用的 array 处理了。这一代码短小精悍，如果是不熟悉  python 动态性和反射机制的读者，其源码也是相当好的入门教程。autoray 最后提供了两种通用接口，一种是 ``do("tensordot", a, b)``，这种接口想来是没什么人用的。另一种是 ``pseudonumpy.tensordot(a, b)``，这种方案就和 jax 等采取的方案一样，提供一揽子和 numpy 相同的 API，但是后端不同。这两者其实是一回事，后者在 ``pseudonumpy`` 这个类的 ``__getattribute__`` 直接调用 do 就好了。而前者也很简单，do 里边找一下注册在某个模块或字典的 ``backend`` 信息，然后直接 ``getattr(backend, "tensordot")`` 就好了。这就是我为什么说，简单实现一个通用 wrapper 是很容易的，核心逻辑也就不到100行。

当然这样一个封装的问题是并没有提供一个统一的 API。也就是实际上做不到一段代码多后端运行。举个最简单的例子，我想求一个数的实部，那应该怎么通用的实现呢。``do("real", a)`` 是不行的，这行代码在 numpy 后端是没问题的，但是切换到 tensorflow 后端呢，你必须得 ``do("math.real", a)``。这个问题可能还不严重，可以通过把 ``tf.math`` 下面的东西导入到 tf 名空间解决。但这只是一个例子，由于不同 array-like 库的 API 存在各种微妙的差异和名空间层级，使得 autoray 的方法论无法做到同样代码，处处运行。

## TensorNetwork

Google Tensornetwork 库的处理方式其实和 autoray 差不多，区别就是 TN 库进一步尝试对外提供统一的 API。也就是说 TN 多后端的实现要比 autoray 繁琐得多。因为这意味着对于每一个 API，都要在不同后端实现一次，虽然大部分实现只不过是一句简单的返回而已。但做了这一步之后，整个 API 就统一了。``backend.tensordot(a, b)`` 也不用担心，是不是 pytorch 里 API 不叫这个名字之类的问题了。实际上 TN 库从来也没有尝试对这些 array-like 的库做大统一，其提供的少量封装 array API 都是给库内部用的。甚至简单的 sin，cos，expm 这些函数都没有提供接口，还是我最近 [PR](https://github.com/google/TensorNetwork/pull/544) 上去了一些接口。当然我十分理解这种哲学，因为多后端只是 tensornetwork 实现和做 benchmark 的需要，完全没必要维护 tensornetwork 操作用不到的 array 接口。只不过作为下游开发者，如果想保持 TN 库 backend-agnostic 这一优良特性的话，可能需要自己继承 backend 类，实现更多的 array 操作通用接口。如果想知道怎么做的，可以去我的 tensorcircuit 库，翻一下 backends.py 的代码。

除了 Tensornetwork 库，类似的 backend-agnostic 机制也被用在了 [tensorly](https://github.com/tensorly/tensorly) 上。这个我没有去读源码，但看 API 接口，大差不差应该和 TN 库是一个方案。

## NEP-18

从上面的讨论我们看出，想要维护一套 API 一致的 array 引擎得以在不同后端运行，需要大量的重复工作，来做不同库类似 API 的 wrapper。因此根本的改变还是需要反向来。“鼻祖” numpy 提供一整套的接口和方案，而实现这些是其他 array-like 库自己的义务。一旦实现了，那么这些数值库就可以和 numpy 无缝对接，从而 numpy 代码不需修改，可以直接换后端来运行。

和 PEP 类似，numpy enhanced proposal 是一个 numpy 新 feature 提案的讨论记录系统，[NEP-18](https://numpy.org/neps/nep-0018-array-function-protocol.html) 描述了 A dispatch mechanism for NumPy’s high level array functions，就是为了解决 array-like 库的统一问题。其想法大致是这样的，考虑到不同库的 array 都是 class，我们给这些 class 带一个 ``__array_function__(self, func, types, args, kwargs)`` 的 magic methods。对于array like 库，需要做的是，验证 func 函数该库是否支持，验证 types 对应的 args 等的类型，是否是自己库 array 的实例，如果有不是的，并且自己库没有信心强行 cast，那就 `raise NotImplemented`，否则就调用自己库的 func 函数实现，并且 ``return func(*args, **kwargs)`` 即可。这样实现一致 API，并且支持 numpy 这个 dispatch 协议的任务就落在了各个 array 库自己身上。而 numpy 需要做的是，对于每一个 ``np.func``，用装饰器将其快速改写，在调用 func 本体之前，先行检查个 args 的类型，以及其是否带有 ``__array_fucntion__``，如果有则尝试调用该方法计算，如果遇到 NotImplemented，就继续看下一个参数是否带有该方法，依此类推。当然纯 python 实现这一套，会导致 numpy 调用有一个不可忽略的 overhead，这也需要仔细考量。

注意到这一方式只适合 ``np.fun(a)`` 的 dispatch，而对于 `a.dtype` 这种取属性的代码，还是没办法做到 universal。

## uarray

 [uarray](https://github.com/Quansight-Labs/uarray) 是另一个尝试提供 backend-agnistic array 的库。uarray 与 [NEP-22](https://numpy.org/neps/nep-0022-ndarray-duck-typing-overview.html) 有关。不过 NEP22 基本上就是一些 duck array 的兼容原则和路线图，没有任何技术细节，uarray 可以看成基于这之上的某种尝试。不过该项目文档生态明显没有成型，这里就不多讨论了。大体思路还是一样的，其想实现的东西，就是：

```python
with backend("tensorflow"):
    np.tensordot(a, b)
```

这样每次只需要换 backend 名称，就可以把同样的代码跑在不同 backend。一个简单的实例可以看[这里](https://github.com/Quansight-Labs/uarray/blob/master/notebooks/01_user_facing.ipynb). uarray 同样是对其他 array 库的作者提出了协议要求，需要其他库主动实现 ``__ua_function__``，这一方法论和 NEP18 究竟怎么协调，关系如何，我想还有待时间观察。可以说对于 duck type array 的 dispatch，numpy 还在一个纠结的过程，没有最后一锤定音，强力推行某种协议和规范。

## tensorcircuit

终于说到我这个库，当然我这个库不是专门用来做 universal array 的，整个项目也只是随便写写，没有特别想持续维护或者做大生态。不过仅仅很短的行数，带来了这么多有趣的副产品还是值得提下的。这个副产品就是在 TN 的 backend-agnostic array 的基础上，我顺手就实现了下 backend-agnostic AD。要知道线性代数和自动微分就构成机器学习的全部了，也就是这个代码库，天生就支持一段代码，多引擎运行的通用机器学习编程。当然这只是随便吹吹，我是没时间也懒得去实现那些基础网络结构和优化器的。只是说原则上，有了通用线性代数和通用求导，甚至还有通用试验性的 JIT，那么你总是可以构造出通用的机器学习代码的。作为一个简单的例子，可以参考这个 [notebook](https://github.com/refraction-ray/tensorcircuit/blob/master/examples/Unified%20AD%20model.ipynb) 感受下通用代码的魅力。从此让 ``import torch as tf`` 的操作失去了光彩。当然，还是要在此强调，这种 universal ML programming 只是一个玩票的 demo，并不能大规模使用。不过这里 universal AD 的想法还是比较新的，没什么人尝试把不同的反向传播代码封装成相同的 API，我只了解到有一个把 pytorch 的求导代码封装成 jax 风格的 wrapper，叫做 [difftorch](https://github.com/gbaydin/difftorch).



总之 unify all duck arrays 的任务还道阻且长，主要是牵涉少太多的库了，是一个很生态的问题。但感觉由 numpy 牵头推动这一统一还是很有必要的，谁也不想要一个 array 就有不兼容的十几种，这样的 python 科学计算生态是有点野蛮生长的。一旦统一，其潜力还是巨大的。想象一下以前的祖传 numpy 老代码，只要加一行就可以轻松的通过 tensorflow 跑在 GPU 上或者通过 dask 进行分布式运算，前景还是很美妙的。反过来看，Julia 的 multidispatch 的设计还是超强的，直接从头就在语言层面解决了统一 array 的问题，并且有更清晰的接口可供其他 array 库的作者重载实现函数即可。不需要像 python 这边，统一接口的设计都很拧巴，还需要 numpy 层面去统一生态，费力推动。这方面 Julia 对 Python 的优势是巨大的。