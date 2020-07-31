---
layout: post
title: "Python based quantum softwares: some comparisons"
date: 2020-07-31
excerpt: "基于 Python 的量子编程和微分编程结合的一个调研"
tags: [physics, quantum computation, python, list, machine learning]
comments: true
---

* toc
{:toc}


## 引言

最近一个月，折腾了几种主流的量子软件方案，因此想一篇小文总结一下。想到哪说到哪，纯意识流，而且这种对比和自己做的东西更偏重于哪些方向也有很大的关系。之前已经有很多列表，对比甚至专门的文章来讨论不同量子软件性能的区别，功能的差异等等。但一来这些内容可能不一定 up to date，这个领域的软件开发还很活跃，好多都是挂在 beta 版的形式；二来他们比较关注的点可能并不是我关心的点，有些应用在意能够模拟的尺寸，有些应用在意真实量子硬件的连接性，有些应用更在意软件和其他机器学习框架的结合等等。本文只比较几个主流的，基于 Python 或至少原生前端是 Python 的软件，同时可能关注的点更倾向于 quantum programming 和 differentiable programming 的结合。其他语言当然也有一些很优秀的类似功能的软件，但我没怎么用过，也没有什么发言权，就不在讨论范围之内了，一个比较全的 quantum software 列表可以参考[^list]。

## Qiskit

[Qiskit](https://github.com/Qiskit) 来自 IBM，这在这次的对比中是最重的一个，依赖极多，本身也分成了多个库开发（我得承认我不喜欢这种工作流，很多东西被人为割裂了，而且这导致 qiskit 的命空间嵌套非常恐怖，没有都 reimport 到顶层实现函数的扁平化。这样的问题就是，你想用的每个函数都得去查文档，不然你都不知道从哪个3级或者4级子包里导入它）。如果想安装 qiskit，强烈建议使用单独的 conda 虚拟环境。不然海量依赖和版本要求很容易把其他库的依赖给破坏了。

Qiskit 比较拧巴的地方在于，为了模拟真实硬件的行为，其不能直接输出最后的波函数或者密度矩阵。为了能直接拿到这些量来分析测量期望（要知道这可比跑几千次 sample 求平均快多了，而且更准确，除非想研究测量次数的影响，否则数值模拟时知道波函数不去用，纯属自讨苦吃。）为了绕过这一问题，可以使用所谓 `snapshot` 的操作，该操作可以插入到线路里，相当于一个量子门，但其会记录下当时的波函数等信息，最后模拟的结果里就可以拿到了。但这东西还是很 subtle，你直接用 QuantumCircuit 对象，会报错没有该方法。这再次体现了 Qiskit 劈裂成多个库的设计哲学的问题，snapshot 类方法竟然是在另一个库动态注册上去的，而不在原始的 terra 库的 API 里，虽然文档里还是显示相应函数在 terra 库。Qiskit 虽然在量子软件里算是重些，但在那些大块头的软件工程面前，也只是个一般软件而已，这种多个库的分割，实在是带来的问题多于好处，额外的概念也会给用户带来很多困惑。关于 snapshot 究竟要怎么才能用，请查看 [^aerissue]，我硬是看了源代码，才明白了怎么才能调用的。

关于自动微分，Qiskit 当然是做不了了，只能手动 finite difference 了，这种 finite difference 范式和 Pytorch 的结合，可以参考官方例子[^tutorial]。

## Cirq

[Cirq](https://github.com/quantumlib/Cirq) 来自 Google。首先的一点吐槽，cirq 的 API 设计有点烂，虽然用习惯了也能忍，但就是不太行，各种冗余都需要输入，一点都不漂亮。或者说句难听的，我无法想象怎么才能把量子计算这种接口设计几乎是送分题的东西实现得这么差劲，糟糕到了我不得不专门吐槽一下的程度。还有就是 cirq 挂着 beta 的名义，文档自然是相当糟糕。开个地图炮，这似乎是 Google 开源软件的通病，TensorFlow，TensorNetwork，TensorFlow Probability 文档都极其的不怎么样。

功能上，cirq 内置了 moment 的概念（每列对齐），对于找线路里的 gap 之类的比较方便，而 qiskit 似乎需要手动写算法 schedule 那些 gate，手动找 gap。现在 noisy 线路和 density matrix simulator， cirq 也都实现了，用起来还行。我们关注的微分部分，作为一个传统量子软件，当然是不默认支持了。其实之前有 proposal[^cirqissue] 希望 cirq 把 numpy 迁移到 jax 上去，这样好支持自动微分（因为反正 cirq 是纯 python 的），但似乎开发者都不太感冒。

## TensorFlow Quantum

[TensorFlow Quantum](https://github.com/tensorflow/quantum)同样来自 Google， 设计想法就是基于 cirq，并且使得量子线路可微分，从而可以嵌入到一般的机器学习模型，作为 Keras layer 出现。不过其设计相当局限，首先是微分方式都是 finite difference based （parameter shift 是一种特殊的 finite difference），这相比传统的反向传播，在参数多的时候时间消耗会非常恐怖。其次是可以自动微分的输出量太少了，基本上只有量子线路输出期望可以自动微分。实际上，大多数时候我们需要更灵活的自动微分，包括直接波函数分量就反向传播回去了。不知道为何 tfq 明明有输出 wavefunction 的 layer，但不支持微分。然后就是输入也很局限，不是很能处理任意波函数作为输入[^wavefunctionissue]，而这在量子机器学习模型里可能很必要（已知的 cirq 是支持任意波函数输入，qiskit 似乎也不行？反正大部分软件可能都支持的不好，因为后端模拟器必须是量子态模拟器才能比较好的兼容）。

其次 TensorFlow Quantum 利用了 c++ 后端模拟器，但是也可以用传统的 cirq 里的 python 模拟器，就是慢而已。还有问题就是 tfq 不支持 noise，这个问题也很有趣，分为两个方面。由于 tfq 里的序列化根本就不支持 cirq 中 noise 相关的 gate，那么你就构造不出 noise 线路放进 tfq 框架。但另一方面，又由于 tfq 的模拟器可以替换为 cirq 中具有某种噪声的模拟器，因此可以实现这种所有 gate 前后都插入噪声的模拟。因此本质上，倒不是说 tfq 自己的 c++ 模拟器不支持噪声（也确实不支持），真正的阻碍实际上是 tfq 没实现 cirq 噪声相关 gate 的序列化协议。相关问题可以参考[^noiseissue].

## Pennylane

[Pennylane](https://github.com/PennyLaneAI/pennylane) 出品自 XanaduAI，这个库的设计初心就是怎样更好的结合量子计算和机器学习，因此这个库天然就很适合做机器学习相关的任务，因为该库设计核心就是怎样让量子前向可以微分。整个库的代码很漂亮也很简单，结构可扩展性很好，整个库提供了很多可能的接口，因此一方面量子计算的部分可以 dispatch 到 cirq，qiskit 等不同库上计算，一方面又提供了和 TensorFlow， Pytorch 等兼容的机器学习 layer 接口。而且整个库文档和教学案例非常清楚漂亮。

稍微微观一点讲，这个库采用将量子线路部分封装进一个函数的形式，而该函数的微分操作一般来讲还是 finite difference based，这个也是没有办法的事情。因为 quantum simulation 可能是多后端，多语言 evaluate 的，肯定是没办法 trace 做传统的自动微分的。当然 Pennylane 提供了基于 [TensorNetwork](https://github.com/google/TensorNetwork) 的 simulator，由于这样 quantum simulation 的部分实际上是发生在了 Tensorflow 上，因此是可以用传统反向传播的，这样做的好处自然是多参数时可以保证复杂度只  scale with 线路深度，而不 scale with 参数数目。理论上和实践上，对于量子线路这样的完全可逆计算，我们都可以用 adjoint state 或者说 neural ODE 类似的办法，来实现空间复杂度为常数，而不依赖于线路深度的反向传播，其原理就是反向传播和共轭传播在反向可以同时进行，用来恢复中间值，而不需要前向时记录在内存里。这一想法具体可以参考[^tfqissue]，事实上 Julia 中的 [Yao.jl](https://github.com/QuantumBFS/Yao.jl) 已经实现了这种常数空间复杂度的量子线路自动微分。但据我所知到的信息，**目前还没有 Python based quantum software 支持量子线路常数空间复杂度的自动微分。** 事实上，支持这种量子线路传统自动微分想法的库，也就只有 Pennylane 而已（当然还有下面我自己写的 demo 库，tensorcircuit）。大部分量子软件库不支持量子线路自动微分的理由很简单粗暴：这一操作无法在真实的量子实验实现。其实这根本是他们的接口，本质就是他们懒得去改模拟量子线路的 C++ 代码来添加自动微分的部分，或者同时对自动微分部分和量子模拟算法部分感兴趣又都了解的开发者少而已。

说了这么多，似乎 Pennylane 对于我们的需求堪称完美了，灵活性高，框架漂亮，接口丰富，支持传统反向传播的量子线路微分，除了期望还支持波函数幅度等作为目标函数的一部分自动微分。但是，一个缺点就让我实际项目里放弃了 Pennylane，那就是它太慢了。它的 Tensornetwork simulator 不知道为什么，比我的也没做任何优化的 tensorcircuit 慢上几百到几千倍，更不要提像 tensorflow quantum 或者 qiskit 这种后端是优化过的 C++ 专用量子线路模拟器的软件了。这个速度实在是这一软件的唯一缺点，其他部分的表现都很完美，但是这一缺点却是致命的，直接导致了无法用它上到实际的研究项目。

## Tensorcircuit

最后带点私货，[tensorcircuit](https://github.com/refraction-ray/tensorcircuit) 也是基于 TensorNetwork，我顺手写的。其实一个简单的量子线路模拟器，核心代码也就小几百行。本来有了 pennylane 这么漂亮的库，其又有基于 TensorNetwork 的模拟器并且支持反向传播，我这个库好像有点画蛇添足。要说我这个库和 pennylane 比有啥优势，那就是快。其实我也没做任何优化，我也不知道为什么 pennylane 那么慢。反正现实就是 tensorcircuit 中等规模 size，做项目速度还可以接受，pennylane 的 tensornetwork 模拟器完全无法接受。

最后总结一下，需要微分线路的话，大 size 还得用 tfq，毕竟是 c++ 的模拟器，快很多，当然微分部分会乘以线路参数倍的慢，不过一般时候还是更快。中 size 或者变分参数较多的情形，tensorcircuit 可能更快。而如果需要比较特别的目标函数，不止依赖于观测量期望，而依赖于输出波函数的细节的话，那么就只有 tensorcircuit 能胜任了。

## References

[^list]: <https://github.com/qosf/awesome-quantum-software>

[^tfqissue]: <https://github.com/tensorflow/quantum/issues/256>

[^cirqissue]: <https://github.com/quantumlib/Cirq/issues/1628>

[^aerissue]: <https://github.com/Qiskit/qiskit-aer/issues/839>

[^wavefunctionissue]: <https://github.com/tensorflow/quantum/issues/210>

[^noiseissue]: <https://github.com/tensorflow/quantum/issues/250>

[^tutorial]: <https://qiskit.org/textbook/ch-machine-learning/machine-learning-qiskit-pytorch.html>