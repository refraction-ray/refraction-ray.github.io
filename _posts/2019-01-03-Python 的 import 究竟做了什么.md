---
layout: post
title: "Python 的 import 究竟做了什么"
date: 2019-01-03
excerpt: "从一个具体的例子彻底看清 python 的引用运行机制"
tags: [python]
comments: true
---

* toc
{:toc}

## 引言

关于 Python import 机制的文章，可以说成千上万，被大家说烂了。但即使这样，似乎还是没人真的深入到问题的本质。就拿循环引用来说，为什么把引用语句扔到函数里面就可以，为什么有时这样还是会报错？python 文件以脚本模式运行和作为模块被导入时，都有哪些区别？本文围绕一个例子来彻底解决 import 的底层机制问题[^doc]。本文例子全部使用 Python 3.6。

## `from` 引用的例子

这个例子以 package app 为例，其文件目录如下：

```
app
  ├── __init__.py
  ├── a.py
  ├── b.py
  ├── c.py
```

其中 py 文件的内容，参考以下 gist。

{% gist 22b9ab5721d562eba3ab5c8831176986 a.py %}

<hr>

{% gist 22b9ab5721d562eba3ab5c8831176986 b.py %}

<hr>

{% gist 22b9ab5721d562eba3ab5c8831176986 c.py %}

假设我们希望运行 `c.py` ，简单的 `python c.py` 会报错 `ModuleNotFoundError: No module named '__main__.a'; '__main__' is not a package`，这是 python 的设计问题，即 python 的 relative import 机制只能应用于 module 而不能应用于 script。而想让 python 认可是 module ，除了 package 添加 `__init__.py	` 之外（Python 3.3+ 允许 `__init__.py` 缺省[^initpy]），还得是被 import 引入时才会以 module 形式运行。如果想在调试的时候，则需要打开 `-m` 选项，也即在 `app` 的上级路径，运行 `python -m app.c`。此时的 `c.py` 的 `__name__="__main__"`，`__package__="app"`。现在我们开始逐行分析以上命令的输出，来验证 python 的 import 机制。

```bash
------------
you are now entering __main__
c module is now as __main__
in c, import a
------------
you are now entering app.a
before create class A
end create class A
before create obja
initialization of obja
end create obja
end of app.a
------------
in c, import b
------------
you are now entering app.b
in b, import a
initialize list b as [1]
assign attr b to obja
create function modif_list
create function modif_list2
create function pre_defined
begin change the listb
import c in function modify_list2 within b
------------
you are now entering app.c
c module is now as app.c
in c, import a
in c, import b
init tuplec as (2,)
in c, obja.b: 2
in c, listb:  [1]
end of app.c
------------
end change the listb
end of app.b
------------
init tuplec as (2,)
in c, obja.b: 2
in c, listb:  2
end of __main__
------------
```

我们按照分割线分割的部分来分析，分割线代表了模块之间的切换。首先我们进入了直接调用的 c 中，此时 c 是以 `__main__` 的身份出现的，这一点后面很重要。然后执行了 c 中的 `from .a import obja` 语句，python 很自然的切换到了 `a.py` 中来执行。这就进入了第二部分。

在 a 中，值得注意的是，声明类的代码中的函数并不会真的被调用，只有当真的初始化一个实例时，相关的函数代码才会被调用，而没直接用到的代码仍然不会调用，即使里面有语法错误也不会报错。这也是动态语言容易出问题的原因。值得注意的是，即使是用 `from` 语法导入，当 python 找到对应导入的对象的定义之后，还是会继续声明函数和执行后面的语句到结束，这一机制确保了对导入变量在 a 中的后续修改也会被考虑，而不是找到符合的对象就急着返回。a 导入后我们又切换回 c，执行后边的导入 b 模块的语句，从而进入第四部分。

在 b 中，我们一开始就 import a。请注意，此时 a 中的语句没有被执行！在 python 中，每一个模块文件都是天然的单例。每次导入时，一旦 python 解释器在内存中找到了该模块的实例（也即已被其他模块导入过），就会直接使用该实例，而不需要在执行导入一次。对于 `modify_list`，其中的语句没有被打印，再次说明导入模块的执行过程，不会调用未被模块内部直接使用的函数。再注意到 `modify_list2` 中，我们调用了 `pre_defined` 函数，而此调用语句还在该函数的定义之前，但这并不会报错。原因就在于，函数内部内容不会在定义声明时被调用，而只在显式使用时被调用。换句话说，如果把 `pre_defined` 函数的定义语句下移到 b 中 23 行之后，那么程序就会报错。因为在调用该函数时，该函数还没有定义。最后的细节就是 b 和 c 循环引用的处理，我们看到语句 import c 被放到了具体函数之内，原理是一样的，确保该语句在导入模块，声明函数定义时不会执行。同时，执行函数时通过该语句，我们需要导入模块 c，而切换到了第五部分。

这时可能会问，不是说模块只会被导入一次么？为什么导入 c 的时候又切回 c 去执行了？内存中不应该有 c 的模块实例了吗？这一问题的答案就在 c 的 `__name__` 中。第一次执行 c 时，其 name 是 `__main__`，而这次其 name 是 `app.c`。在 python 解释器看来，这并不是一个对象，因此 c 模块还未被导入过，于是再次执行 c 模块内容。而这一次 c 模块中的 a 和 b 的导入语句，都不会再去真的执行，因为其内存中的单例都存在了。同时注意，此时 c 拿到的 `listb` 还没有被修改，这也很自然，因为我们就是在修改之中切回 c 模块来导入的。而 `obja` 虽然是从 a 模块中导入的，但已经具有了 `b` 属性。这说明对象实例在 python 都是单例（似乎是废话，就是一个指针而已），和从哪里导入无关，其他模块文件对其做的修改，如果被执行过，则都会保留。c 彻底执行完毕之后，我们就会切回 b 的修改函数之中，继续修改 `listb`。

最后一部分，我们此时才彻底在作为 `__main__` 的 c 中导入完 b，再次打印 `listb` 发现其内容已经被修改。

## 一些改造

上面是一个不报错的例子，通过对以上 package 的改造，更多会报错的例子，是更有趣的情景，也算是对我们上面运行机制分析正确的验证。这部分就讨论下一些会报错的改造。其中大部分情形，按照上述的分析思路，很容易发现问题的根源，这里不多费口舌，就算留作作业了。

* 在 `b.py` 修改 `listb` 之后定义量，比如 `z＝1`，如果 c 中写为 `from .b import listb, z` 就会报错 `ImportError: cannot import name 'z`。
* app 中添加 `d.py`， 内容为 `from . import c`，执行 `python -m app.d`，报错 `ImportError: cannot import name tuplec`。这就是因为此时第一次执行 c，c 就是模块身份出现的，那么在 b 中列表修改函数再次导入 c 时，不会再去执行 c 了，而此时 c 的模块只卡在执行了第二句 import 的程度，`tuplec` 并没被初始化，因此报错。

## `import` 引用的例子

通过 `import app.d` 这种形式来引入称为绝对引用[^importmethods][^imports]，这种方式也可以避免循环引用的问题（事实上，只要不直接引用模块中的对象，而是导入模块，在 Python 3 中都不会有问题）。我们在 app package 中添加两个模块 e,f 如下。

{% gist 22b9ab5721d562eba3ab5c8831176986 e.py %}

<hr>

{% gist 22b9ab5721d562eba3ab5c8831176986 f.py %}

此外我们在 app 文件夹外添加一个 `ext.py` 模拟对 app package 的调用。

{% gist 22b9ab5721d562eba3ab5c8831176986 ext.py %}

此时执行 `python ext.py` 得到的输出为：

```bash
entering app.e
entering app.f
finish import e in f
finish register of fun f1
finish register of fun f2
finish import f in e
finish register of fun e1
finish register of fun e2
finish import e from ext
finish import f from ext
call e2 in ext
e2
f1
call f2 in ext
f2
e1
```

我们发现，之所以绝对引用可以避免循环引用时来不及注册的问题，在于我们并不需要去找模块中的对应函数，而是直接将模块名注册即可，这总是可以的，即使模块中其实只运行到了第一句 import。也即第一句 `import app.f` 切换到 f 执行时，`import app.e` 不会报错，而之后则将 f 的函数定义执行声明，当切换回 e 之后，e 中的函数才完成声明。这也没什么影响，因为 f 中对 e 函数的调用是在函数内部，并不会被执行，所以不报错。我们注意到 ext import f 时，不再执行 f，这对应了前边模块即单例的概念。之后函数都会正常执行。

此外，我们也可以直接 `python -m app.e` 来运行，其输出如下，理解留作作业吧。

```bash
entering __main__
entering app.f
entering app.e
finish import f in e
finish register of fun e1
finish register of fun e2
finish import e in f
finish register of fun f1
finish register of fun f2
finish import f in e
finish register of fun e1
finish register of fun e2
```

除了这种引用，在 Python 3 中，将 `import app.e` 改成 `from . import e` 或是 `from app import e`，或是增加 `as` 子句都没有任何问题。总结起来就是，以模块形式导入就不需过于担心循环引用之类的问题，即使相应的对象还没来得及执行和注册到模块，只导入模块名也是安全的。注意绝对引用和相对引用（用到 `.` ）的区别在于，绝对引用是在系统路径 `sys.path` 中依次搜索对应的模块名，而相对引用是相对 `__package__` 的路径进行相对路径的查找。

## 其他细节

### script 和 module 模式系统路径区别

可以考察两种模式下 `sys.path[0]` 的不同之处。在 module 模式，也即 `python -m app.file`，路径中加入的是相对路径 `""`, 也即调用的工作目录。而在 script 模式下，也即 `python app/file.py`，路径中加入的是包含该 py 文件的文件夹的绝对路径[^syspath]， 也即 `/root/foo/bar/app`。

### 引用任意路径 package 的解决方案

最快速的办法就是在 import 之前加上 

```python
import sys
sys.path.insert(0, "/abs/path/")
"""
import somepack
"""
```


### 包含 import 函数的字节码

我一直纠结的一个问题是如果函数中的 import 没执行，那么函数对象的字节码，也就是 `__pycache__` 里的东西如何生成。结果证明 import 也是 python 虚拟机 opcode 的一部分。请看以下例子，

```python
import dis
def f(a):
    from ..b import c
    return c+a
dis.dis(f)
"""
  2           0 LOAD_CONST               1 (2)
              2 LOAD_CONST               2 (('c',))
              4 IMPORT_NAME              0 (b)
              6 IMPORT_FROM              1 (c)
              8 STORE_FAST               1 (c)
             10 POP_TOP

  3          12 LOAD_FAST                1 (c)
             14 LOAD_FAST                0 (a)
             16 BINARY_ADD
             18 RETURN_VALUE
"""
```

其实这个内容和 import 的 runtime 不在一个抽象层次，不太需要考虑，看不懂这段的可以略过。

## 总结

可以看出，从根源上来讲， python 并不禁止循环引用，其也没有循环引用的概念。循环引用造成的错误，只是因为引用跳转，导致模块里需要被引用的函数还没来得及注册在模块里造成的而已。解决这个问题，可以把对应的模块引用放在对象声明之后，或对应的类或函数内部避免注册前的跳转。而这样做依旧可能遇到问题，只能沿着运行时的引用关系，慢慢分析。循环引用既不是错误，放在函数里的引用也不一定能解决该问题。import 的 silver bullet 只能是彻底弄懂 import 时到底发生了什么，记一些经验规律而不弄清其中的细节，总有遇到奇怪问题的时候。

## Reference

[^doc]: [Python3 import doc](https://docs.python.org/3/reference/import.html)

[^initpy]: [initpy not required in python3](https://stackoverflow.com/questions/37139786/is-init-py-not-required-for-packages-in-python-3)

[^importmethods]: [different import methods and their effectiveness for circular import](https://stackoverflow.com/a/37126790)

[^syspath]: [sys.path difference in different env](https://stackoverflow.com/a/29549088)

[^imports]: [absolute vs relative python imports](https://realpython.com/absolute-vs-relative-python-imports/)

