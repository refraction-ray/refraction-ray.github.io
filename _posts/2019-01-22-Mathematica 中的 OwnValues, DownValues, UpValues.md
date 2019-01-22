---
layout: post
title: "Mathematica 中的 OwnValues, DownValues, UpValues"
date: 2019-01-22
excerpt: "从 Mathematica 的几种 Value 看其编程范式和运行机制"
tags: [mathematica, python, cpp]
comments: true
---

* toc
{:toc}

## 引言

关于 mathematica 中的这几种 Value 的区别和理解，参考[^distinct]已经说的比较清楚了。本文算是对这一内容，以及 mathematica 内部一些文档[^doc]的一个笔记，当然也有一些关于这几种 Value 更直观的理解，同时我也会谈到 UpValues 一些可能的应用。贯穿其中的还有对 mathematica 运行时的本质以及编程范式的比较思考。

## 四种 Values

要理解这些 value 的区别，首先要知道 mathematica 的运行机制，是完全基于表达式，规则匹配这些形而上的东西。所谓的函数，变量等等在其他编程语言中所熟悉的事物，只不过是 mathematica 通过这些匹配机制巧妙编织的幻觉。而四种 Values 实际上就直指了这一幻觉的核心。建议读者熟悉 `_`,`__` 等 pattern 的概念和语法来阅读本文。

首先是 `OwnValues`，这是用来构造变量幻觉的主力。例如：

```mathematica
a = 1;
OwnValues[a]
(* {HoldPattern[a] :> 1} *)
a
(* 1 *)
```

mma 实际做的事情，是读到 a 之后，寻找到了 `OwnValues` 中的替换规则，然后将 a 这个字符串，替换成了 1 这个字符串。可以证明这一点，戳破 a 是对应一个占据内存地址变量的例子是 `a[1]`，mma 会给出 `1[1]` 的结果，这正说明了 mma 的运行时机制是按照一定规则无脑的字符串替换，变量只是大多数情况下这个替换带来的幻觉。

然后是 `DownValues`，这是用来构造函数幻觉的主力。例如：

```mathematica
b[x_] := 1
DownValues[b]
(* {HoldPattern[b[x_]] :> 1} *)
b
(* b *)
b[1]
(* 1 *)
```

因此 DownValues 专门用来匹配后跟中括号的 Symbol 的替换。DownValues 可以有多个规则，例如 :

```mathematica
b[x_] := 1;
b[x_, y_] := 2
DownValues[b]
(* {HoldPattern[b[x_]] :> 1, HoldPattern[b[x_, y_]] :> 2} *)
```

对于不同的模式（这里 mma 也不太有本质上的参数概念，其运行时所看到的只是 b 后边的字符串有没有逗号而已），可以采用不同的替换规则，类似函数的重载。注意 `OwnValues` 优先于 `DownValues`， 因此一旦 b 本身也有定义，那么 b 作为函数的身份就会被屏蔽掉。也即无法重载出 `b` 返回1， `b[1]` 返回2的效果，而只会是 `b[1]`  返回 `1[1]`。

关于这里变量幻觉定义中用的 `=`， 而函数幻觉定义中用的是 `:=`，千万不要以为这是 mathematica 定义语法的一部分。两者都可以用在变量和函数定义中，其唯一的区别是变量替换是否延迟到需要时在执行。这一方面只需考察 `c=Random[];{c,c}` 和 `c:=Random[];{c,c}` 的返回就知道我在说什么了。正因为在传统的语言中，变量大多是即时运行，函数都是调用时运行，因此 mma 中这样定义比较符合传统的幻觉，但不是必须的，还需按情况选用 `=` 和 `:=`。

为了充分说明，mma 根本没有函数的概念，参数也只是字符串，请参考以下例子。

```mathematica
g[x_ + y_] := gp[x, y]; 
g[x_ y_] := gm[x, y]
g[Unevaluated[2 + 3]]
(* gp[2,3] *)
g[Unevaluated[2  3]]
(* gm[2,3] *)
```

可以看到，mma 中传统函数的概念完全消解了，连所谓逗号分隔的变量也全是幻觉，有的只是中括号里的内容作为整体的模式匹配而已。

你甚至可以直接操作这些 Values 的 list，请看以下例子。如果能理解这个例子，那么说明对于 mathematica 运行的机制就算入门了。

```mathematica
d[x_, y_] := 1;
d[x__] := 2;
d[1,2]
(* 1 *)
DownValues[d] = Reverse[DownValues[d]];
d[1,2]
(* 2 *)
```

由于 `UpValues` 没有一个传统的幻觉对象，所以是在其中比较难理解的一个。语意上，既然 UpValues 是 DownValues 的反义词，那么 DownValues 是根据符号后面中括号内的内容结合来匹配替换，UpValues 则反其道而行之，根据符号及包含其的中括号对应的函数（外层函数），来决定和重定义内层函数的行为。这样说或许太抽象了，还是直接看例子。

```mathe
g/: f[g]:=1;
f[g]
(* 1 *)
f[a]
(* f[a] *)
f[g[a]]
(* f[g[a]] *)
Clear[g]
g/: f[g[__]]:=1;
f[g]
(* f[g] *)
f[g[a]]
(* 1 *)
f[g[a,b]]
(* 1 *)
```

当然，和 DownValues 一样，一个符号也可以定义多条 UpValues 的规则。这一 `TagSet` 也即 `/:` 的语义是指将其后的替换规则赋值给前边的符号的 `UpValues` 或 `DownValues` 或 `OwnValues`。其中前边符号（上例中是 `g`）处在规则函数内部时，即为 UpValues，而如果写为 `g/: g[f]=1`，则事实上改变了 `DownValues`，其语义和 `g[f]=1` 等价，因此不必使用这种 syntax。`OwnValues` 同理， `g/: g=1` 和 `g=1` 等价。

`UpValues` 还有 UpSet 这一只属于自己的原语。即 `f[g[__]]^:=1`。这种方式和 `TagSet` 区别不大，唯一的一点区别可以参考[^tag]。

说白了 UpValues 就是从内部魔改，考虑到 mma 运行机制是嵌套表达式从里到外，先分析各符号的 OwnValues，再依次分析 UpValues，DownValues。请看一下例子：

```mathematica
Clear[f, g, h];
f[x_] = x;
f[g] ^= 1;
Print[f[g]];
Print[f[h]];
(* 1  h *)
```

最后我们可以通过 `?Symbol` 的 shortcut 快速查看一个变量绑定的所有 Values 的替换规则。

此外还有一个所谓的 `SubValues`，这货 mma 连文档都没。其对应的是形如 `f[a][b]` 这样的定义。

```mathe
f[a][b][c] := 1;
SubValues[f]
(* {HoldPattern[f[a][b][c]] :> 1} *)
d=f[a][b]
(* f[a][b] *)
d[c]
(* 1 *)
```

## UpValues 的几种应用

### 模拟字典和对象

比如我们可以做一个存公式的字典。

```mathematica
square /: Area[square] := s^2;
square /: Circum[square] := 4 s;
circle /: Area[circle] := \[Pi]r^2;
circle /: Circum[circle] := 2 \[Pi]r;

Area[square]
(* s^2 *)
```

我们发现这东西和 python 的 dict 很像，这里我们相当于构造了 Area 和 Circle 两个 dict，不过底层上这和函数一样，依然是一种幻觉而已，本质上还是符号匹配与替换。

当然 mathematica 有了 association 这种数据结构之后，有些需求也可以考虑用 association 实现更加直观。

更进一步的，考虑到 `Head` 在 mathematica 运行和分析时的重要性，我们可以构造一套面向对象的系统。用 Head 来代表对应的内容。这一内容我将结合下小结的重载函数一起来看。

### 重载系统函数

如果我们想要重构系统函数，使得对于一些输入（比如我们新定义的“类”），其函数行为有所不同，那么我们就有两条路径，一条是通过 `Unprotect@func` 来解除系统函数 `func` 的保护，从而重构。细节可以参考[^app]中的解释。这一方案缺点很多，比如修改系统函数行为可能带来不可控的影响，同时由于 mma 的运行机制，函数的每个 DownValues 都会依次评估。比如我们想重载加号，也就是重载 `Plus`，那么我们每次做加法，这一新添加的 DownValues 都会被评估，拖慢了运行时间。毕竟所有做的加法里，只有很小一部分和我们新定义的类有关。

反之，如果我们把这种重构行为绑定在我们新定义的类上，行为就安全了许多。这就需要 UpValues。请参考以下代码，我们实现了一种加法是乘法的数。

```mathematica
SNum /: SNum[a_] + SNum[b_] := a*b;
SNum[1] + SNum[2]
(* 2 *)
```

注意到 `+` 对应了外层函数是 `Plus` ，因此 UpValues 定义符合之前的语法。按照这种思路推广下去，利用 UpValues 我们可以实现较好的封装性并模拟很多面向对象的特征。

## Mathematica 的运行机制

每一句命令，mathematica 都看成是一个表达式。而表达式最重要的特征是具有 `Head` 属性。因此 mathematica 通过从里向外，先匹配 `OwnValues`， 然后匹配 `UpValues` 再匹配 `DownValues`的顺序，依次完成对相应子表达式 pattern 的替换，直到结果没有任何定义的替换规则匹配为止。

注意上面所述的这个运行过程里，根本就没出现传统的函数，内存空间等概念。不严格的说，按照传统的编程语言的观点，mathematica 的运行机制都是在来回操纵字符串而已。为了更清楚的理解这一点，我们从一个加法函数来看静态语言 C++，动态语言 Python 和 Mathematica 运行机制的区别。

```cpp
/* myadd.cpp */
#include <iostream>
int myAdd(int a, int b){
    return a+b;
}
int main(){
    int a=1, b=2;
    std::cout<<myAdd(a,b);
}
```

对于 C++ 代码需要提前通过 gcc 等编译工具生成机器语言，也就是二进制文件。里面就包含了最底层的 cpu 如何操纵内存和进行计算的信息。例如通过 `g++ -S myadd.cpp`，生成的汇编文件中，可以找到 `addl	-8(%rbp), %esi` 这样的句子，对应了 cpu 层次的加法运算。程序运行时，main 中遇到 `myAdd`，可以直接跳转到 `myAdd` 函数二进制存放的内存地址，然后 cpu 按照对应的指令执行操作即可。这里的加法函数，只能计算整数的加法，如果传入实数就会报错，因为占据的内存大小不同，加法计算的 cpu 指令也不同。要想实现统一名称的函数也能计算实数的加法，静态语言就只能求助于重载。但重载的本质，也不过是编译时，编译器自动起了不同名字自动连接而已，运行时的行为都不会发生改变。某种程度上，重载也只能算是静态语言的一个语法糖而已。而想要将函数作为其他函数的参数，则只能通过传递函数指针（也即函数地址），或者函数对象的方式实现。

对于 Python，事情更复杂一些。

```python
"""
myadd.py
"""
def myAdd(a,b):
    return a+b

if __name__ == "__main__":
    print(myAdd(1,2))
```

当执行上面这段代码，也即输入 `python myadd.py` 时，实际上分成了两个阶段。第一阶段是编译，将所有代码都转化为字节码。与 cpp 转化为机器码不同，这里的字节码是 python 解释器可以识别的，自己定义的一套虚拟机机器码。这一字节码可以通过 `python -m dis myadd.py` 查看，函数对应的字节码内容如下。

```python
0 LOAD_FAST                0 (a)
2 LOAD_FAST                1 (b)
4 BINARY_ADD
6 RETURN_VALUE
```

python 执行的第二阶段是运行，由 python 解释器开始逐一调用并执行字节码。python 的灵活之处在于，字节码对应的执行函数是高度抽象和多态的。比如上面的 `BINARY_ADD`，并不一定只能执行整数的加法，也可以执行实数的加法，甚至对象的加法。由于 python 一切皆对象，因此只要加号左边的对象定义了 `__add__` 方法，就可以执行加法。这就造就了动态语言的灵活性和鸭子类型。同时在 Python 里由于函数不过是定义了 `__call__` 方法的对象，因此可以轻松的作为其他函数的参数出现。

而对于 mathematica，运行的机制和以上语言完全不同。

```mathematica
(* myadd.nb *)
myAdd[x_,y_]:=x+y;
Print[myAdd[1,2]];
```

看起来函数定义只是和 python 略有语法区别而已，但实际上底层大不相同。当执行第一句函数时，mathematica 只是在 `myAdd` 这一变量的 `DownValues` 列表里加了一个替换规则。也即

```mathematica
DownValues[myAdd]
(* {HoldPattern[myAdd[x_, y_]] :> x + y} *)
```

注意到，这一过程并为生成任何字节码之类的东西，这里实质上根本就不存在函数的概念。当 mathematica evaluate 第二句的 `myAdd` 函数时，mathematica 将其理解成一个表达式，从里往外找 `Head`，第一个是 `myAdd`，然后 mma 会依次尝试该名称是否有 OwnValues，UpValues，DownValues 对应的规则。找到 DownValues 对应的替换规则后，按照模式匹配，将 `myAdd[1,2]` 这个整体替换为 `1+2` 这个整体。注意这一操作，完全类似于传统语言中对字符串的操作，myAdd 完全没有所谓函数的概念。mma 继续分析表达式，`1+2` 是 `Plus[1+2]`的简写，最终得到 3 这一结果。在 mma 中，函数参数的类型概念就更弱了，因为从头到尾，执行的都不过是一些基于规则的字符串替换而已。那么函数作为其他函数参数就是天然的事情，甚至连 python 中将函数打包封装成一等公民对象这一实现都不需要。

## 简单总结

由以上从几种 Values 发散出的对 mathematica 运行机制和底层的思考来看，mathematica 并不是所谓的函数式编程。准确的说 mma 应该算是一种多范式编程。其变量，函数，对象，都是建立在字符串替换规则上的一套假象，其一等公民并不是函数，而是表达式本身。这一本质特性在众多的编程语言中是无比独特的（~~利用不好也可能是无比慢的~~）。


## Reference

[^distinct]: [what-is-the-distinction-between-downvalues-upvalues-subvalues-and-ownvalues](https://mathematica.stackexchange.com/questions/96/what-is-the-distinction-between-downvalues-upvalues-subvalues-and-ownvalues)
[^doc]: 详见 `tutorial/AssociatingDefinitionsWithDifferentSymbols`，`tutorial/TheStandardEvaluationProcedure`，`tutorial/ManipulatingValueLists`
[^tag]: [TagSet vs. UpSet](https://mathematica.stackexchange.com/questions/18410/upvalues-tagset-and-upset-whats-the-difference-when-should-a-use-each/18411)
[^app]: [Do people actually use UpValues](https://mathematica.stackexchange.com/questions/11435/do-people-actually-use-upvalues)