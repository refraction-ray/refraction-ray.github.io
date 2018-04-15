---
layout: post
title: "Introduction to Restricted Boltzmann Machine"
date: 2018-04-12
excerpt: "从概率图模型到CD训练：RBM 的一个简介"
tags: [statistics, machine learning, physics]
comments: true
---

* toc
{: toc}

本来想写一篇博客讲讲限制波尔兹曼机的各种扩展，尤其是各种基于 RBM 的深层结构和物理上的一些创意。结果关于 RBM 的引言就已经有一篇博客的篇幅了。那就先把这个引言单独贴出来吧，都是一些基础知识。不过相关基本思路我们会沿用到下一篇关于 RBM 花式扩展的文章上。

## general picture

### probability graphic model
很多现实问题都可以归结为统计问题，具体的说，归结为求多个随机变量的联合分布 $$P(X_1,X_2...X_N)$$ 的问题。即使每个变量只能取两个离散值，该分布一般情况下仍需要 $$2^N$$ 个参数确定，指数大的空间在实践上是不现实的。因此人们总是希望可以将联合概率一定程度上解耦，从而更少的参数就可以有效描述联合分布。
为了实现和表示这一点，引入了概率图模型。概率图模型中，每个节点都代表一个随机变量 $$X_i$$，每条边则代表变量间的一定的联系。如果边是有方向的，则称为有向概率图，有向边本身描述了随机变量间的依赖因果关系。
![](https://user-images.githubusercontent.com/35157286/38677830-fa67f962-3e91-11e8-8774-3814c9c9b34b.jpg)
比如上述有向图的概率可以表示为

$$
P(x_{1}, {\cdots}, x_{n} )=P(x_{1})·P(x_{2}|x_{1} )·P(x_{3}|x_{2} )·P(x_{4}|x_{2} )·P(x_{5}|x_{3},x_{4} ) 
$$

这种有向图常用于各种贝叶斯推断，最简单的例子 naive Bayesian classfier 就可以理解为一个有向图模型。有向图模型的一个著名应用就是隐马尔科夫模型。而对于稀疏的有一定分层的有向图，常被称为信念网络。最简单的例子，就是 sigmoid belief network (SBN)。在这种模型中，每层神经元可以取二值 $$\pm 1$$，各神经元激活（取1）的概率为

$$
P(x^{(l)}_i\vert \mathbf{x^{(l-1)}})=\sigma (W^{(l-1)} \mathbf{x^{(l)}} +\mathbf{b^{(l)}}).
$$

其中 $$\sigma(x)=\frac{1}{1+e^{-x}}$$ 是著名的 sigmoid 或 logistic 函数。而 N 层的 SBN 联合分布概率则为:

$$
P(X)=P(\mathbf{x^{(0)}})\prod_{i=1}^N P(\mathbf{x^{(i)}}\vert \mathbf{x^{(i-1)}}).
$$

根据联合分布的定义，恰好是典型的有向图模型。有向边的条件概率由 sigmoid 函数给出，这种条件概率的传递过程很像一个普通的具有 sigmoid 激活函数的前向神经网络。

### energy based model
与有向图相对的，还有无向图概率模型。对于无向图，边是没有方向的。互相全部通过边连接的节点构成团，若我们可以将节点分割成若干最大团 (maximal clique) $$Yc$$，其中 C 是相应最大团包含的随机变量的集合，那么无向图的联合分布可表示为

$$
P(Y)=\frac{1}{Z}\prod_C \Psi_C(Y_C).
$$

其中 $$Z＝\sum_Y\prod_C\Psi_C(Y_C)$$ 是归一化因子 (关于随机变量族 Y 的求和是关于所有可能构型取值的求和)，而 $$\Psi_C$$ 是定义在最大团 C 上的非负函数。

比如下面无向图的联合分布可以表示为

$$
P(X_1,X_2,X_3,X_4)=\frac{1}{Z}\Psi_1(X_1,X_3,X_4)\Psi_2(X_2,X_3,X_4).
$$

![](https://user-images.githubusercontent.com/35157286/38678913-f9f2129e-3e94-11e8-8acf-17bc53b7dd93.jpg)

由于指数函数，只要指数是任意实函数则非负，我们可以取 $$\Psi_C(Y_C)=e^{-E(Y_C)}$$, 那么整个无向图的概率则表示为

$$
P(Y)=\frac{1}{Z}e^{\sum_{C}-E(Y_C)}.
$$

很明显，整个无向图被一个能量值 $$\sum_{C} E(Y_C)$$ 唯一确定，我们因此也称无向图模型为 energy based model。原因在于，上式给出的概率和统计力学中的系综分布完全相同($$\beta=1$$)。

对于所有的概率图模型，都有两类基本问题。第一，已知部分变量取值的一些数据，如何选取模型的参数，使得模型平衡态的分布尽可能接近已知数据集的分布。即所谓的训练问题，与之对应的是传统神经网络中的反向传播训练。第二，已知模型的各个参数，如何生成符合模型联合分布或相应边际分布的数据。即所谓的取样问题，这一问题对应传统神经网络的前向计算预测过程。我们下面探讨的模型都将按照这两类基本问题的解决这一主线思路进行。

## Boltzmann machine

无向图在自然语言处理中的一个典型例子就是条件随机场模型。不过我们今天要讨论的是另一类，最大团均为两个节点（随机变量）的，每个神经元可以取二值的无向图。这种无向图的能量形式如果写下来，就等价于一个只有两体相互作用的经典自旋模型。仅考虑最简单的自旋自旋耦合二次项，我们就得到了著名的 Boltzman machine(BM)。该模型给出的概率分布实际上就是长程 Ising 模型的热力学分布概率。我们可以强行把该模型视为两层（可见层 V 和隐藏层 H），层间存在连接，能量形式为（公式利用爱因斯坦求和约定，对于诸如$$a_iv_i$$之类的外磁场项已省略，实践上可以保留）

$$
E(\mathbf{v},\mathbf{h})= -(v_i W_{ij }h_j +v_i J_{ij} v_k+ h_i K_{ij}h_j).
$$

该模型如果考虑条件概率依赖的话，可以发现对于每一个神经元激活概率都依赖于其他所用神经元（包括同层）的取值。所有能量模型相关条件概率的推导都基于以下公式：

$$
\frac{P(x_0=1\vert x_1,...x_N)}{P(x_0=0\vert x_1,...x_N)}=e^{-\Delta E(\mathbf{x})}.
$$

其中能量差定义为 $$x_0$$ 取值变化导致的能量变化。


### 两类基本问题

现在考虑该模型的训练问题。对于给定的数据集 V，我们的训练目标是使得 BM 给出的边际分布 $$P_\theta(\mathbf{v})=\sum_h P_\theta(\mathbf{v},\mathbf{h})$$ 尽可能的接近数据真实分布 $$P(\mathbf{v})$$，我们可以利用 KL 散度的最小化来实现这一点：

$$
KL(P\vert P_\theta)=-\sum_\mathbf{v} P(\mathbf{v})\log \frac{P_\theta (\mathbf{v})}{P(\mathbf{v})}.
$$

考虑到 $$P$$ 分布与我们的模型无关，则事实上，我们只需要最大化 $$\sum_\mathbf{v}\log P_\theta(\mathbf{v})$$ 即可，也就是常见的最大可能性估计(MLE)。而这一最优化问题，同样是利用常见的随机梯度下降(SGD)来实现。经过推导可以发现，参数更新的规则为：

$$
\Delta W_{ij} =\alpha (E_{data}[v_ih_j]-E_{model}[v_i h_j]).
$$

其他参数的更新规则类似，$$\alpha$$ 为学习率。这一规则同样适用于后面的 restricted Boltzmann machine (RBM) 以及其很多扩展模型，因此非常重要。这一更新规则非常重要。其中正值部分以数据分布做平均，而负值部分以模型分布做平均，这也是该模型训练的困难所在，一般来讲训练的复杂度是随着神经元增加指数增加的。

具体来说，训练过程为 Gibbs 采样实现的 Markov Chain Monte Carlo(MCMC)，通过神经元取值的不断更新，来实现平衡态。若固定可见层，则对应的采样概率分布为数据的分布。若所有神经元均可更新，则实现的采样分布为模型分布。通过两个分布我们可以计算正负两个均值并更新模型参数来迭代新的取样。这一训练过程有点 EM 算法的意思（毕竟 EM 就是用来做 MLE 的）。但这一训练的问题就是所需时间是实践上不可接受的。同样的 MCMC 更新达到平衡态的过程自然也可用来生成符合概率分布的取样，即第二类基本问题。原则上，以下的所有模型，都可以用最粗暴的 MCMC 达到平衡态采样这一方式，解决训练和取样这两类基本问题，唯一的也是最大的缺陷，是这种算法的时间复杂度过高而难以应用。

为了简化模型，我们令层内连接 J 和 K 均为0则得到了 RBM，如果仅令 J＝0，则得到了所谓的 semirestricted Boltzmann machine.

## vanilla RBM 

对于 RBM，依旧是两层模型，基本的能量模型（忽略层内耦合）和学习规则都不变，示意概率图如下。

![](https://user-images.githubusercontent.com/35157286/38683280-a8a3f4ba-3e9f-11e8-8fa1-0f421722c988.jpg)

此时由于没有层内相互作用，可见层各神经元的依赖隐藏层的激活条件概率相互独立，不再依赖于同层神经元，也即

$$
P(v_i=1\vert h)=\sigma(a_i+ W_ij h_j),
$$

对于隐藏层的条件概率依赖关系亦然。

### 两类基本问题

考虑 RBM 训练过程，对于正值数据平均部分，我们可以直接简单求得，而不再需要像 BM 一样进行 MCMC 取样平衡过程：

$$
E_{data}(v_ih_j)=v_i p(h_j=1\vert v).
$$

对于模型平均部分，我们还是得进行 MCMC 更新取样，不过这次更新更有效率，每一次交替根据条件概率更新可见层或隐藏层整层的神经元取值。于此同时，我们还可以进一步引入对比散度(contrastive divergence)算法对 MCMC 做进一步的简化和加速。我们并不等到 Gibbs 采样迭代到收敛再取样，相反我们根据条件概率进行一次迭代（$$v_0\rightarrow h_0 \rightarrow v_1\rightarrow h_1$$）就开始进行计算，其中起点 $$v_0$$ 选为数据中的样本。如果我们迭代 k 次，对应的训练方式成为 CD－k 算法。k趋于无穷大时，我们就回到了精确的结果。一个 CD 算法的细节是，为了防止每次的取样求均值时涨落太大，对于$$v_1,h_1$$，我们往往直接用神经元均值（也即激活概率）来代替神经元取样。

对于模型平均估计，我们还有一种 CD 算法的变种，被称为 persistive contrastive divergence (PCD)。在该算法里，每次按模型概率采样时，与其从一个数据点出发进行一次迭代，不如只维护一条 Markov Chain，每次需采样时，就在这条 MC 上迭代1(k)次实现。为了理解这点，可以考虑学习率为0的极限，此时每次更新参数不发生变化，所谓的PCD就完全等价于无穷次迭代的 MCMC。

至于第二类取样问题，如果想生成一些符合数据分布的取样，依旧是 Gibbs sampling 来跑 MCMC 即可。如果随机化神经元取值，那么可能平衡态得跑一会。但如果从一个现成的数据点出发，则隐藏层，可见层跑个来回就勉强能作为符合概率的样本用了。

### 表达能力

如同神经网络是通用的函数拟合器，RBM 被证明是通用的概率分布模拟器：只要隐藏层神经元足够多（可能是指数复杂度），理论上 RBM 就可以拟合出任何二元取值的概率分布。具体的，我们可以证明 RBM 增加隐藏神经元增加会减小模型边际分布和实际分布间的 KL 散度。进一步我们还可以证明：如果有k个数据点，k＋1个隐藏层神经元构成的 RBM 总是可以将数据的分布拟合的任意好。注意这里只是存在性证明，不代表有简单算法可以找到拟合任意好的参数（该问题显然具有指数复杂度）。相关具体细节请参考文末文献。

## References

* [Tutorial: RBM](http://www.cs.toronto.edu/~tijmen/csc321/documents/maddison_rbmtutorial.pdf)
* [A Practical Guide to training RBM](http://www.cs.toronto.edu/~hinton/absps/guideTR.pdf)
* [Restricted Boltzmann Machine and its High-Order Extensions](https://www.cs.toronto.edu/~cuty/RBM.pdf)
* [Representation Power of RBM and Deep belief Networks](http://www.iro.umontreal.ca/~lisa/publications2/index.php/attachments/single/22)
* [An Efficient Learning Procedure for Deep Boltzmann Machines](http://www.utstat.toronto.edu/~rsalakhu/papers/neco_DBM.pdf)


EOF

