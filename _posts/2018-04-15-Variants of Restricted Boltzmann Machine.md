---
layout: post
title: "Variants of Restricted Boltzmann Machine"
date: 2018-04-15
excerpt: "RBM 的花式变种：更现实，更深度，更物理"
tags: [statistics, machine learning, physics]
comments: true
---

* toc
{: toc}


这一篇继上一篇简介之后，继续介绍下各种 RBM 的变形。主要包括三个部分，第一部分是考虑 RBM 输入的变化，从二元的向量变形到多元，连续向量或者矩阵输入等。第二部分是 RBM 的深层扩展，如何利用 RBM 搭建一些深层结构来增强模型的表达和生成能力。第三部分是物理学家对 RBM 模型的一些改造和创意，不得不说 RBM 是物理学家最喜欢使用的机器学习模型，甚至没有之一。

## RBM with count input

### softmax unit and multinominal unit
很多时候，可见层的神经元不一定只有两个取值，而是有多个不同的取值（注意这和从0到K的整数取值不太一样）。比如考虑一篇文档作为可见层，文档有 M 个词，而字典有 K 个词汇，我们采取 one-hot 编码，整个文档可以表示为 $$M\times K$$ 的可见层矩阵。对于每一行，只有一个元素取值为1 ($$v_{ij}=1$$)，其余取为0，这表示对于第 i 行所表示的文档的第 i 个词是字典中的第 j 个词。对于这样的可见层我们可以建立一个 RBM，但我们假设任意神经元到同列所有行的神经元的权重连接都相等，同列可见层的偏置参数也相等（这由文章词语出现的顺序无关来保证，当然这取决于一个人关心的语言处理的具体问题）。此时隐藏层依旧是二元取值的神经元，且隐藏层激活条件概率形式不变，但可视层神经元激活的条件概率变为 softmax 函数形式：(所有公式若非注明，均默认使用爱因斯坦求和约定)

$$
P(v_{ik}=1\vert \mathbf{h})=\frac{e^{h_jW_{jk}+a_k}}{\sum_{q=1}^Ke^{h_jW_{jq}+a_q}}
$$

由于这一条件概率恰好为 softmax 函数，故将这种可以取多个离散值的神经元称为 softmax unit。一个 softmax unit 就等价于上述的二元取值，且存在有且只有一个神经元被激活限制的一行神经元。像上面描述这样我们将一个可以取多个值的 softmax unit 理解成一行神经元的好处是 RBM 的结构，算法等都得到了保持。否则直接理解成单个 softmax 的神经元的话，每个可见层神经元将和同一个隐藏层神经元有多种不同的连接权重，并取决于可见层的取值情况，这就偏离了原始的 RBM 的定义。

另外通过上式注意到条件概率的计算与行数 i 无关，这也验证了我们模型成立的先验条件是文档词汇出现的顺序不是我们关心的问题。进一步，可以利用 $$\sum_{i=1}^M v_{ik}$$ 来判定词汇 k 在文档中出现的总次数。对应的，我们将每行合并过的一列softmax神经元进行进一步合并（M次有放回抽样），这被称为 multinominal unit。如果行不合并成 softmax unit 而直接按列进行求和合并，我们就退化到了神经元取值为0到M的整数的情形。由此可见层的输入为整数（1,2...）或是一些与数字无关的可能性（a,b,c...）处理方式并不相同，是两种情形。

拆解成标准的 RBM 的话，输入取若干整数的模型，可以看成同一二元神经元连续采样 M 次。而输入取若干可能性的模型，可以看成是有 K 个互斥的二元神经元。一旦拆解成标准 RBM，所有的训练采样的过程，就可以约化过去。无论哪种情况，和所谓的高自旋模型还不一样！（这种模型里能量表达式和 RBM 完全相同，唯一区别是神经元可能取不止两个的若干数值）

## RBM with real value input

更多的情况 RBM 的可见层输入数据取值范围是实数，比如输入图片的灰度等。最简单的处理方案，就是作弊。只需将每个神经元的条件激活概率视为该神经元的取值即可，所有计算都和之前的 RBM 相同。由于图片上每个像素点的取值几乎总是严格等于周边像素点取值的平均值，这种比较窄的取值范围很难用 sigmoid 激活函数实现，所以这种作弊可能只在 MNIST 这种数据上有用武之地。下面我们介绍另两种比较靠谱的办法。

### truncated exponential

这种想法也很简单，RBM 的能量模型不变，只将可见层神经元的可能取值放宽到实数域。由此隐藏层激活条件概率形式不变，而可见层取任意值的概率则由能量差的指数形式给出。考虑到去不同实数值的情况互斥，最后给出的条件概率则为类似 softmax 的积分形式:

$$
P(v\vert \mathbf{h})=\frac{e^{y\Delta E(\mathbf{z})}}{\int_v e^{v\Delta E(\mathbf{z})}}
$$

如果神经元取值为实数的一个区间的话，则称为截断指数形式。该模型对应的 CD 训练算法与 RBM 完全相同。


### Gaussian unit

处理实数取值更好的办法是在可见层引入高斯神经元。首先我们对图片数据进行预处理，使得数据的每个维度（神经元取值）都是均值为0，标准差为1的分布。之后我们定义 RBM 的能量函数为：

$$
E(\mathbf{v},\mathbf{h})=(v_i-a_i)^2-b_ih_i-v_iW_{ij}h_j.
$$

通过该能量，可以求得可见层的激活条件概率为：
$$
P(v_i=x\vert \mathbf{h})\sim N(b_i+W_{ij}h_j,1)
$$
也即可见层的激活函数变成了高斯分布。

虽然整个训练过程，除了可见层 Gibbs 取样按照高斯分布以外，和普通RBM依旧没什么区别，但学习率需要调的更低。因为实数没有上界，不像二元取值，一旦学跑偏了，神经元取值变得很大，就很容易越学越跑偏，一去不复返。

你可以进一步的将隐藏层的二元取值神经元也替换为高斯型的神经元，对应的模型双向的条件概率均为按高斯分布取样，但该模型易燃易爆炸，更难训练。还有另一种操作，就是隐藏层上带噪声的 ReLU 做激活函数，可以降低训练的难度，这种神经元被称为 Noise ReLU unit。具体可参考文末文献。

## convolutional RBM
考虑 RBM 的输入具有矩阵结构，比如说图片或文档等，用普通的 RBM 训练，这种矩阵还是在顶向量用，矩阵的局域性没有得到利用。一种想法是所谓 further restricted RBM，对于这种 RBM，隐藏层和可见层的神经元都保持矩阵形式，同时每个隐藏神经元只和局域的若干可见层神经元有非0的连接。这种结构就一定程度利用了矩阵输入的二维特征。我们还可以进一步把参数做的更少，比如每个隐藏神经元和紧邻可见神经元的局域连接都共享一组参数。如果熟悉 CNN 的话，就发现我们上面说到的就是一种卷积结构。这就是所谓的卷积 RBM。这种模型同时具有卷积结构对局域特征的提取特性，和 RBM 的能量模型概率解释的特征。

卷积这种事情，文字描述比较费劲，具体的，看下面的图就懂了。

<img src="https://user-images.githubusercontent.com/35157286/38741822-714c967e-3f6d-11e8-99ff-3a5266254c2f.png" style="zoom:70%">

除了卷积结构外，为了同时保持RBM的特征，我们可以写下其能量函数(忽略偏置项，如果考虑平移不变性，不同神经元偏置项可取相同的值)
$$
E(\mathbf{v},\mathbf{h})=-h_{kq} w_k\odot v_q
$$

其中$$\odot$$表示矩阵元素层面的内积，q 是窗口的移动指标，而k是 filter w 的特征指标。由此可以得到条件概率依赖，隐藏层的条件概率是卷积运算，而可见层的条件概率是转置卷积运算。此外可见层的边界部分由于转置卷积无法进行统计推断，因此最后做训练时只能以优化 $$P(V_{in}\vert V_{boundary})$$ 作为目标，这点也不同于普通的 RBM。而训练时，则需要固定可见层图像边缘的像素取值来进行 Gibbs 采样，其他训练细节和 RBM 相同。


## deep structure generalization
在这一节，我们将讨论 RBM 在深度方向的扩展，虽然 RBM 本身已经被证明可以拟合任意分布，但可能需要的隐藏层神经元的数目是计算上不可接受的。另一方面，传统神经网络的经验已经告诉我们，以深度换宽度，可以在同样的计算复杂度下得到更强的拟合能力。

### deep belief nets

##### 一个等价关系

将 RBM 向深度方向扩展，来自这样一个观察，RBM 和 infinite sigmoid belief network with tied weights 等价。

<img src="https://user-images.githubusercontent.com/35157286/38712840-cf67bed6-3f00-11e8-9189-0cf333c19819.png" style="zoom:60%">

所谓 SBN，我们在上一篇 RBM 简介时已经介绍过，就是一个有向概率图模型，且每层对上层的条件概率依赖是独立的 sigmoid 函数形式。这样的网络对后验($$P(h_0\vert v_0)$$，即已知可见层的构型求上面层的分布)的估计本来是非常困难的，原因在于后验并不能简单的写成各神经元后验乘积这种形式，条件分布不具有独立性。而所谓的 infinite 就是考虑无穷长的 SBN，tied weight 是说这种特殊构造的 SBN，每层的权重矩阵相同（相邻层间相差一个转置）。可见这种 SBN 各层只有两种不同的神经元数量 $$N_v,N_h$$，且这两种层交替出现。该模型取样最后可见层的方式，自然是在无穷远之前取一些随机值作为输入，经过无穷层之后的结果则为符合可见层分布的输出样本。这一过程恰好和 RBM 采样的 Gibbs sampling 一模一样，因此上述的 infinite SBN with tied weights 可以视为 RBM 在 MCMC 这一过程的展开，两者完全等价。这也就意味着以前很困难的后验估计问题消失了，既然这种 SBN 和 RBM 等价，那么 $$P(h_0\vert v_0)=\prod_iP(h_0^i\vert v_0)=\prod_i \sigma(W^T v_0)$$，后验估计变的和 RBM 一样简单，各神经元条件分布统计独立。这个等价关系对于理解后面的 deep belief nets 至关重要，一定要理解。

##### DBN 的渐进式定义

以上构造的 infinite SBN，由于 tied weights 要求，拟合能力一般（由等价性，完全等价于一个 RBM 的拟合效果），试想如果我们放松限制，该网络每层的权重矩阵不再要求相同，自然拟合能力会迅速加强，而深度才真正的发挥出作用。第一步，我们依旧是训练一个infinite SBN with tied weights，也就是训练一个 RBM。之后，我们可以证明，固定第一层训练出的 W，而将之上无穷层重新视为一个 infinite SBN with tied weights $$W'$$，针对通过 $$W^T$$ 倒推的后验样本数据进行训练调优 $$W'$$，可以单调的提升对于可视层的拟合效果。根据等价性，这段话的意思就是说，对于第一个 RBM 产生的隐藏层数据，在上面再训练出一个 RBM 来拟合这些数据。迭代下去，也就是逐层训练 RBM。这样我们就得到了一个 DBN。但是注意，后验独立只在 tied weights 情况下成立，当你开始训练改变第二层的权重（在SBN的语境下，相当于同步改变之上无穷层的权重），第一次的后验推断就失效了！第一层只是一个单纯的单向的有向图 SBN，而无法有效严格的倒出后验条件分布。也就是虽然训练方式是逐层训练 RBM，但每训练更深的一层，上一层的 RBM 假设就遭到破坏，最后的训练结果，就会是顶层依旧是一个 RBM（无向图），而下面各层间都是有向图。等价的说，最后的训练效果得到的模型是一个infinite SBN，除去下面若干层外（DBN 的层数），上面无穷层依旧 tied weights。希望以上渐进式的叙述，可以帮助更好的理解 DBN 的来龙去脉。

通过以上观点，最后的 DBN 的联合分布的定义为

$$
P(\mathbf{v},\mathbf{h^{(1)}},\mathbf{h^{(2)}}...)=P(\mathbf{h^{(N)}},\mathbf{h^{(N-1)}})\prod_iP(\mathbf{h^{(i-1)}}\vert P(\mathbf{h^{(i)}})P(\mathbf{v}\vert \mathbf{h^{(1)}})
$$

其中各条件概率由 SBN 条件概率公式给出，最上层的联合分布由最上层 RBM 的联合分布公示给出，该式也是这个网络本身的定义。

##### 两类基本问题

现在我们把逻辑反过来，从 DBN 的概率分布定义出发，寻找优化模型参数来拟合可见层分布的方法。可以证明逐层训练 RBM 是一种保证拟合效果单调增加的贪心算法。当然贪心算法往往都不是全局最优的，最后还可以通过sleep-wake 算法进行权重进一步调优，该算法本来就是为 SBN 训练设计的，只不过其中对后验独立的假设在普通 SBN里并不成立，而在 DBN 里该近似还可以接受。全局调优的算法细节可以参照参考文献。值得注意的是，在该算法调优里，前向采样和后向推断的权重矩阵被分开优化，这也再次印证了 DBN 下面各层失去了 RBM 的意义。

而对于采样问题，做法也很简单。在 DBN 的语境下，就是先 Gibbs 采样顶层的 RBM，待到平衡之后，在逐层前向计算 sigmoid 概率分布并采样，推进到可见层。等价到 infinite SBN 的语境下，就是先输入一些随机值，经过无穷层 tied weights 的 SBN（等价于 RBM 中的 Gibbs 迭代采样），再继续经过有限层权重不同的 SBN，最后生成符合分布的样本。

![](https://user-images.githubusercontent.com/35157286/38713749-5c137b5a-3f05-11e8-91e3-62460ae5c42c.jpg)

### deep Boltzmann machine

上一节我们看到， DBN 实际上是混合图模型，同时包含了部分有向图和无向图，这一节的 DBM 则完全是 RBM 的扩展，是基于能量模型的典型无向图模型，这将比 DBN 更好理解一些。

DBN 是在 RBM 的基础上假设有多层隐藏层的模型，其被联合分布完全定义，也就是被能量函数完全定义(以如上图的三隐藏层模型为例，忽略了外磁场项 bias)：

$$
E(\mathbf{v},\mathbf{h^{(1,2,3)}})= v_i W^{(1)} h^{(1)}_j+ h^{(1)}_i W^{(2)}h^{(2)}_j+h^{(2)}_i W^{(3)}h^{(3)}_j
$$

通过能量定义，可以导出系统的条件概率依赖：

$$
P(h_j^{(1)}=1\vert \mathbf{v},\mathbf{h^{(2)}})=\sigma(v_iW^{(1)}_{ij}+W^{(2)}_{jk}h^{(2)}_k)
$$

其他各层的条件概率分布同理。其特点是，各层神经元激活的条件概率分布独立，每层条件概率同时依赖相邻的两层，当然边界层的条件概率依旧只依赖一层。我们发现偶数层和奇数层统计独立，相当于可见层和隐藏层在普通 RBM 中的地位。虽然 DBM 是多层结构，但也可以等价的将其视为两层。只不过此时可见层只是其中一层的一部分罢了。

##### 两类基本问题

现在来看给定可见层数据集，DBN 的参数训练问题。训练目标函数依旧是各数据点边际分布的对数和，推导之后和普遍的 BM 的梯度下降更新公式依旧相同，包含正值的数据平均和负值的模型平均两部分（这是自然，毕竟 DBM 只是 BM 的特例）。所以训练算法其他部分都不变，依旧归结为如何求数据和模型平均两部分。只不过比 RBM 复杂的地方在于，数据平均部分也无法直接算出了。两种分布平均我们依旧使用 Gibbs 采样给出，只不过 RBM 的时候我们是根据条件概率交替更新可见层和隐藏层，现在我们则根据条件概率交替更新奇数层和偶数层。对于数据平均部分，我们只需做MCMC update 的时候保持可见层的神经元取值不变，而对于模型平均部分，只需正常的 Gibbs sampling 即可。对于数据平均部分，还有一种更快的迭代方法。即按照神经元的激活概率作为神经元的取值进行迭代，将稳态的取值作为数据平均部分的均值。这种方法的正确性源自后验统计推断的平均场近似，具体细节可见参考文献。

原则上以上的训练可以从随机权重出发，但实践上更好的方法是进行预训练，使得开始时参数就在最优参数附近。预训练过程和 DBN 的逐层训练 RBM 类似，但要注意逐层 RBM 的训练，对于 DBM 只是理论上可有可无的预训练而已，而对于 DBN 则是正式的训练过程，千万不要混淆。

至于采样算法，也和 RBM 类似，可以从随机的奇数层或偶数层出发，依据条件概率迭代进行 Gibbs sampling，直到 Markov chain 达到平衡态。同样的从已知的数据点出发，可能帮助更快的平衡
### deep restricted Boltzmann networks
DRBN 可以想象成是一些堆叠的 RBM，比起 DBN，以堆叠 RBM 的形式训练但最终结果不是 RBM；这里的 DRBN 则是以并非堆叠 RBM 的形式训练，但训练结果是堆叠 RBM。具体的，该模型相关的条件概率被定义为:

$$
\begin{align}
P(x_j^{(l+1)}\vert \mathbf{x^{(l)}})&=\sigma(x_i^{(l)}W_{ij}^{(l)}+b^{(l)}_j)\\
P(x_j^{(l)}\vert \mathbf{x^{(l+1)}})&=\sigma(W_{ij}^{(l)}x_j^{(l+1)}+c^{(l+1)}_j)
\end{align}
$$

注意这样定义的效果是，每层神经元有两组不同的偏置 bias，一组b用于后向推断，另一组c用于前向采样。这一条件概率形式上和 RBM 一致，而不同于 DBM 中，中间层的激活概率由两侧层的取值共同决定。这样灵活的条件概率传递结构定义的 trade off 就是，似乎系统的联合分布概率很难表达，原文作者也没有提及。

##### 两类基本问题

但既然有了条件概率，我们还是可以做 Gibbs sampling。这次的更新策略更加复杂，不同于以前可见层隐藏层或是奇数层偶数层的交替更新，这里必须按照从下到上在从上到下的方式逐层更新取值。根据数据点向上更新一轮的取值被用做求数据平均部分。而连续k轮迭代后的构型，被用来评估模型平均的部分（类似 CD－k 算法）。

取样问题，还是熟悉的配方，随机化初始神经元取值，之后反复的上下上 Gibbs 取样迭代，直到达到平衡态，可见层即符合相应分布。

如果觉得这个模型有些诡异，我们也可以利用和 RBM 与 SBN 间的等价关系类似的等价关系来理解。整个 DRBN 模型对应了一个无穷深的权重有一定规律和联系的 SBN。如果我们直接将这个 DRBN 理解成一个权重成特定结构（等效与 Gibbs 取样时的从下至上在至下的扫描更新过程）的无穷长 SBN，可能很多问题和算法看起来就会更舒服一些。DRBN 的好处是，其最终结构真的是堆叠 RBM，这样我们可以进一步堆叠卷积RBM全连接RBM等不同模块实现特定任务，实现深度 RBM 构建的模块化和多样化。而传统的卷积RBM进行堆叠，如果加了破坏概率解释的池化层则仅可用于判别模型，若全部应用卷积 RBM 由于只学到了局域特征，似乎很难反向生成图像。

关于这些深度的统计生成模型的结构，一篇物理学家的文章说的还不错，感兴趣的可以参见 **arXiv: 1708.04622**， 这是一篇比较不同的深度概率生成模型拟合二维 Ising 模型系综分布效果的文章（结论比较反直觉，这里不剧透了）。

## generalization from physicists

### RBM with complex weight

这种扩展很简单，一句话，就是把 RBM 的参数换成复数即可。这样模型给出的“概率值”（实际上是概率幅）也是一个复数，恰好对应了波函数的不同构型的概率幅。严格的说，此时整个模型已经失去了概率图的意义，不过也无所谓，物理学家用这种模型就是直接来当波函数的假设来做变分蒙卡（VMC）的，有没有概率图的含义已经不再重要。这一工作的滥觞当然是那篇著名的 science 工作，具体细节见参考文献。

### quantum Boltzman machine
一个传统的 Boltzmann machine 对应一个经典的自旋$$1/2$$系统的长程 Ising 模型。该模型的每个格点（取值）是好量子数，因此是一个经典模型。但如果考虑加入和$$\sigma_x$$耦合的横场部分，我们就得到了最简单的量子相变模型——横场 Ising 模型。此时需要引入量子统计相关的 formula 来描述该系统，而由于量子算符没有对易性，模型训练的优化目标无法约化成传统的正负两种平均的形式，由此极大限制了这种类型模型的大小。严格的来说，只能严格按照矩阵运算来训练模型，这种模型可能最多仅包括20个神经元的数量级。

当然，我们可以利用 $$Tr(e^{A+B})\leq Tr(e^{A}e^{B})$$ 的关系，来给出优化目标的下界，此时模型的训练方式可以约化到类似经典 Boltzmann 的形式。这种近似模型被称为 bound based quantum Boltzmann machine。但即使做了这种近似，依旧无法训练横场耦合项的参数值。

稍微讲两句这个量子模型的形式体系和经典模型的联系，我们取$$\sigma_z$$的本征态做基，也就对应了每个神经元的经典取值。而某个经典构型出现的概率，计算方式为 $$P_{s}=Tr(\vert s\rangle\langle s\vert \rho)$$，其中 $$\rho$$ 是系统的密度矩阵，定义为 $$Z=Tr(e^{-H}),\rho=\frac{e^{-H}}{Z}$$，只不过注意这里的能量是一个矩阵，指数函数也是矩阵函数。

一些简单的实验显示，量子波尔兹曼机可以使得拟合的可见层分布与真实分布的 KL 散度更小。作者甚至讨论了如何在量子计算机的硬件上更高效的训练 QBM，细节请参考文末文献。

### generalized Gibbs machine
这个工作将 Boltzmann machine 基于的热力学平衡态统计系综，推广到可积系统非平衡态统计的 GCE （generalized Gibbs ensemble）。可积系统是指存在广延量个物理量算符，其期望值不随时间改变，也即这些算符和哈密顿量对易。对于这样的系统，测量某个态出现的概率计算与上文的 QBM 相同，只不过此时用于定义密度算符的指数形式上，$$-\beta H$$ 变为了 $$-\sum_n \beta_n Q_n$$，其中的 Q 是对应的各个守恒量算符，而各个 $$\beta$$ 则是我们要训练的参数有效温度。

利用标准的横场 Ising 模型，或者其 Jordan－Wigner 变换的费米子对偶模型（自由费米子），我们可以直接给出各个守恒量的定义 $$Q_n$$。我们将输入数据，比如 MNIST 图片，二元化成一个取值为0，1的向量，并将其理解为测量的得到的波函数（0，1表示该格点是否存在费米子占据，数据预处理自动满足了 Pauli 不相容原理）。对于十个不同的数字，我们分别学习出一套有效温度的参数，那么某图片是数字 $$o$$ 的概率则为一个 softmax：

$$
P^o=\frac{Tr(\vert img\rangle \langle img\vert e^{-\beta_n^oQ_n})}{\sum_{o=0}^9  Tr(\vert img\rangle \langle img\vert e^{-\beta_n^oQ_n})}
$$

这种模型由于假定了哈密顿量，其实际需要学习的参数远远小于传统的 Boltzmann machine。事实上，这模型除了借鉴统计系综的分布以外，也不是一个生成模型。相反，其只是借助守恒量构造的知识作为单层 softmax 神经网络的输入而已，模型最后是训练了一个普通的判别模型的神经网络，训练的损失函数是各数字概率和真实的数字概率之间的交叉熵。这样训练好的网络上的权重就对应了不同的有效温度的参数。由于原则上守恒量随着图片像素的增加而增加（广延量个），实践上可以对守恒量的数目做一个截断，这一截断参数也可以用来控制近似的精度。总之，这个工作就是相当于对输入的数据，做普通的神经网络前，搞了个比较物理的预处理而已（将输入类比成波函数，并求出对应的各阶守恒量）。

## other extensions

RBM 还有其他很多扩展，可以想象，其取值情况，能量形式，深度与拓扑结构，蕴含的变种可以相当丰富，下面稍微指出两个例子，不再做具体介绍前因后果。想要了解的可以直接参考文末的文献。

### mean-covariance RBM (mcRBM)

基本哲学是在 RBM 对应的能量函数里添加高阶耦合项比如 $$vvh$$，希望这样可以更好的反应可见层的本来具有的关联情况。

### spike-and-slab RBM (ssRBM)

该模型是对可视层具有高斯 unit 的实数版本RBM的改进。作者认为 Gaussian RBM 之所以表达能力有限，在于隐藏层的二元取值的限制。因此 ssRBM 大幅改进了隐藏层的取值情况，使每个隐藏层神经元都可以取一个二元值 spike 和一个实数值 slab。

## Reference
* [Learning Deep Generative Models](http://www.cs.cmu.edu/~rsalakhu/papers/annrev.pdf)
* [Training Restricted Boltzmann Machines](http://imagine.enpc.fr/~obozinsg/teaching/mva_gm/papers/RBM.pdf)
* [A Practical Guide to Training RBM](http://www.cs.toronto.edu/~hinton/absps/guideTR.pdf)
* [ReLU Improve RBM](http://www.cs.toronto.edu/~hinton/absps/reluICML.pdf)
* [Stacks of Convolutional RBM for shift-invariant feature learning](https://www.cs.sfu.ca/~mori/research/papers/norouzi_cvpr09.pdf)
* [Deep Boltzmann Machines](http://proceedings.mlr.press/v5/salakhutdinov09a/salakhutdinov09a.pdf)
* [An Efficient Learning Procedure for DBM](http://www.utstat.toronto.edu/~rsalakhu/papers/neco_DBM.pdf)
* [A Fast Learning Algorithm for Deep Belief Nets](http://www.cs.toronto.edu/~fritz/absps/ncfast.pdf)
* [Greedy layerwise training of deep networks](http://papers.nips.cc/paper/3048-greedy-layer-wise-training-of-deep-networks.pdf)
* [Deep Restricted Boltzmann Networks](https://arxiv.org/pdf/1611.07917.pdf)
* [Solving the Quantum Many-Body Problem with Aritificial Neural Networks](https://arxiv.org/pdf/1606.02318.pdf)
* [Quantum Boltzmann Machine](https://arxiv.org/pdf/1601.02036.pdf)
* [Supervised machine learning algorithms based on generalized Gibbs ensembles](https://arxiv.org/pdf/1804.03546.pdf)
* [A Spike and Slab RBM](http://proceedings.mlr.press/v15/courville11a/courville11a.pdf)
* [RBM and its higher order extensions](https://www.cs.toronto.edu/~cuty/RBM.pdf)

