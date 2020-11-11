---
layout: post
title: "Nielsen Chuang 教材关于 Shor 算法讲解的三个错误"
date: 2020-11-11
excerpt: "兼谈 Shor 算法经典部分的各种物理教材忽视的微妙之处"
tags: [physics, quantum computation, complexity theory, algorithm]
comments: true
---

* toc
{:toc}

## 引言

本文将围绕 *Quantum Computation and Quantum Information* by Michael Nielsen and Issac Chuang  [^Nielsen] （以下简称教材）关于 Shor 算法讲述里出现的三个错误，对这些错误的地方进行订正，并发散到 Shor 算法里的其他一些细节的分析。我们将会看到，Shor 算法的难度完全在于经典部分预处理后处理的各种细节和其复杂度和正确性的严格证明，反而其量子核心部分的 phase estimation 是非常简单的。因此很多物理教材舍本逐末，其实完全没有讲清楚 Shor 算法的复杂性，那些一带而过的话其实并不平庸。如此一本圣经级别的教材在这么短短一小节，就出现了至少三处错误，这也侧面反应了 Shor 算法经典部分的困难。本文默认读者对 Shor 算法，计算复杂度理论和初等数论[^numbertheory]有一定的了解。

##  Reduction of factoring to order finding

在这一部分，教材给出了错误的定理 5.3， 任取与 N 互质的 $$x$$，成功概率 $$P(2\vert r \text{ and } x^{r/2}\neq -1(\mod N))\geq 1-\frac{1}{2^{m-1}}$$, 而非教材中的 $$1/2^m$$。这也能说明我们为什么要首先排除 N 是 perfect power 也即 $$N=a^b$$ 的情形。此时若 a 为素数，则 $$m=1$$，那么这样的 N 对应的成功概率下界降到了 0，由此无法有效保证选出合适的 x ，使得定理 5.2 可以应用。而 $$m=2$$ 时，成功概率至少一半。注意到这里的概率，根据每次输出的结果可以简单验证是否成功，因此不需要概率严格大于 $$1/2$$ 即可（因为不需要重数投票确定正确答案）。常数 $$k$$ 次取样后，取到满足条件 $$x$$ 的概率将会变为 $$1-\frac{1}{2^{k(m-1)}}$$。

这一证明在 Shor 论文原文中有一证明梗概，下面将作一简要的分析。值得指出的是，factoring 到 order finding 的 reduction 是 Shor 独立作出的，详情可参考 stackoverflow 上[^so] Shor 本人的回答。

首先我们需要如下引理：p 是一个奇素数，$$\phi(p^\alpha)$$ 的质数分解中包含 $$d$$ 个 2，那么对于和 $$p^\alpha$$ 互质的全体元素 $$Z^*_{p^\alpha}$$ , 恰好有一半可以被 $$2^d$$ 整除。

证明概述：考虑到 $$Z^*_{p^\alpha}$$ 是循环群[^cylic], 所有元素可以表示成 $$g^k (\mod p^\alpha)$$ 的形式，对于奇数 k， $$g^{kr}=1\mod p^\alpha$$，考虑到 $$\phi(p^\alpha)$$ 是 g 的 order，那么 $$\phi(p^\alpha)\vert kr$$，因此 $$2^d\vert r$$. 若 k 为偶数，$$g^{k\phi(p^\alpha)/2}=1\mod p^\alpha$$，因此 $$r\vert (\phi(p^\alpha)/2)$$，那么 r 里的含 2 量要比 $$\phi$$ 少，所以 $$2^d$$ 无法整除奇数 k 元素对应的 r。（这里的含 2 量是为了直观，指的是对应数字质数分解中 2 的个数）。

这一引理我们只需要用一个基于其的非常松的推论：对于 $$Z^*_{p^\alpha}$$ 的所有元素，至多有一半元素 order r 的含 2 量相同。证明：对于群里一半元素，对应的 r 含 2 量大于等于 $$\phi(p^\alpha)$$ 的含 2 量 $$d$$；而另一半元素，其 r 无法被 $$2^d$$ 整除，也即他们 order 的含 2 量小于 $$d$$。

基于这一推论，我们给出定理 5.3 的证明概要。$$N=p_1^{\alpha_1}...p_m^{\alpha_m}$$，根据剩余定理，随机取 $$x$$ 和 $$N$$ 互质，等价于分别随机取 $$x_j$$ 与 $$p_j^{\alpha_j}$$ 互质，$$x=x_1..x_m$$，他们对应的 order 分别为 $$r_j$$。首先我们需要展示，当 r 是奇数或者 $$x^{r/2}$$ 是平凡解的时候，对应的全部 $$r_j$$ 含 2 量相同。那么根据上面的推论，每一个任取的元素 $$x_j$$，其 order $$r_j$$ 要想和 $$r_1$$ 含 2 量相同，概率至多为 $$1/2$$，因为至多一半元素 r 含 2 量一样。由此失败概率上界为 $$\frac{1}{2^{m-1}}$$，因为对于取出的 $$x_1$$ 的 order $$r_1$$ 并没有限制。

那么 r 是奇数时，各个 $$r_i$$ 均为奇数，全部 order 含 2 量为 0，相同。当 $$x^{r/2}=-1\mod N$$, 则 $$x^{r/2}=-1\mod p_j^{\alpha_j}$$, 也就是说 $$r$$ 是 $$r_j$$ 的奇数倍，那么所有的 $$r_j$$ 都和 $$r$$ 含 2 量相同。证毕。

## Perfect Power Test

正如我们上面所说，要想保证 Shor 算法正确性，前处理需要用经典算法排除掉 $$N=a^b$$ 的情形。这一排除同样是很多素性测试方案所必需的前处理。这一方法，教材以练习 5.17 的形式呈现，但很明显，这个练习所给出的证明方式和最后复杂度结论应该都是错误的。下面我们先展示一个最基本的解法，其复杂度要高于教材暗示的 $$O(LM(L))$$，因为该教材默认了 $$M(L)=L^2$$。我们最后会看到，这一问题虽然可以在 $$O(LM(L))$$ 甚至更低的复杂度里解决，但需要的算法远不是一道教材习题的难度，因此在这一问题的分析上，教材作者理解是错误的。

对于不同的 b，从 $$2$$ 增加，直到 $$2^b>N$$，共 $$O(L)$$ 个不同的 $$b$$, $$L$$ 是 $$N$$ 的位数。每次通过评估 $$a^b$$ 和 $$N$$ 的大小关系二分，来搜索是否有合适的 $$a$$，使得 $$a^b=N$$。$$b=2$$ 时先将二分起始右界移动到了 $$N^{1/2}$$ 处，之后每一轮 $$b$$ 对应的左右界初始值分别为 $$1,N^{1/(b-1)}$$， 由此可得所需要的二分评估指数 $$a^b$$ 的总次数为：(注意该推导里的等号其实都是 scaling 意义下的放缩)

$$
\sum_{b=2}^L \log(N^{1/b})= L\sum_{b=2}^L \frac{1}{b} = L \log b = L\log L
$$

而每个快速指数评估需要的操作数目是 $$\log b M(L)=\log L M(L)$$, 因此 Power Test 不太 naive 的复杂度其实为 $$M(L)L\log L^2$$.

该问题更精妙的复杂度结果是一些90年代发表的文章[^sieve][^linear]，前者将复杂度降到了 $$O(L^3)$$，后者甚至该问题的复杂度可以降低到接近线性。但这些算法本身是不可能作为同为 90 年代的教材的一道课后习题的，因此教材作者对这一问题的复杂性的理解完全有误。

## 保证分母 r 不被约分

这一错误是作者的理解问题，其错误的理解了 $$s_1', s_2'$$ 的含义。

正解：要想保证求出正确的 $$r$$，要保证每次测量均匀得出的 $$0\leq s\leq r-1$$ 不能和 $$r$$ 约分（互质）。这样的 s 的数目即位 $$\phi(r)$$。考虑到在 $$r\rightarrow \infty$$ 的极限下，我们有 $$\frac{\phi(r)}{r\log\log r} -> \delta$$，因此我们只需要运行 phase estimation $$O(\log\log r)=O(\log L)$$ 次，从每次提取的分数中选取最大的分母即为真实的 $$r$$。这也是 Shor 原始文章[^Shor]给出的方案。考虑到 phase estimation 中复杂度有 $$O(L^3)$$，这将使得整个复杂度略高于该值。

而教材中，第一种方案给出的分析依赖于素数个数而不是和 $$r$$ 互质的数的个数，因此复杂度放大到了 $$O(L)$$ 次分数 $$s/r$$ 估计。而在第三种方案中，作者给出了一种所谓常数次计算来估计真实分母的方案，并计算了概率，其假设是说，如果找到两组 $$s', r'$$, 且分子互质，那么分母的最小公倍数即为 r。然而 $$r=16，s_1=6, s_2=10$$，对应的 $$s', r'$$ 分别为 $$3,8$$ 和 $$5,8$$，按照作者的方案，得出的 $$r=8$$，显然是错误的。其实这里作者给出的方案是有问题的，正确的讲述方式是考虑两次取得的 $$s$$ 互素，而非约分后才互素，这种情况对应的约分后分母的最小公倍数才是真实的 $$r$$。这一概率对应的证明不变，只是没有了作者错误并画蛇添足的解释：“so to upper bound $$p(q\vert s_1')$$ it suffices to upper bound $$p(q\vert s_1)$$”。事实上，公式 5.56 中的 $$s$$ 本来就没有 prime。

## 其他备注

### Reduction from order finding to factoring

事实上 factoring 和 order finding 是多项式等价的。我们在 Shor 算法中看到了一个将 factoring 多项式约化到 order finding 的随机算法，这小节将给出反向约化的多项式级别算法，注意到反向的约化是一个确定性的算法。

1. 计算 $$\phi(N)$$ 的质因数分解
2. 将 $$n=\phi(N)$$ 每次除以一个质因子，判断得到的数是否保持 $$a^n=1\mod N$$, 若是，更新 n，继续这一循环。

正确性分析：order r 一定可以整除 $$\phi(N)$$。

复杂度分析：忽略 factoring 所需的复杂度，计算 $$\phi(N)$$ 可通过对 N 质因数分解再带入欧拉公式完成，O(1)。循环每次成功除以一个因子为一轮，最坏情况需要 $$\sum_i \alpha_i<\log N=L$$ 轮。每轮最多尝试全部不同的因子共 $$m<L$$ 个。每次的指数可以用 divide conquer 方法进行快速指数计算，最多需要 $$L$$ 次乘法。因此这一反向的归约也是多项式级的。

### 倒推连分数的唯一性

对于所有分母不大于 $$r$$ 的分数，最小的距离就是 $$1/r-1/(r-1)=1/r(r-1)$$，因为该距离的分母已经最大，分子已经最小。那么如果选取足够多的比特数，使得准确值的位数达到 $$2L+1$$ 的情形下，我们测量出的值和真实的 $$s/r$$ 的距离小于 $$\frac{1}{2^{2L+1}}<\frac{1}{r^2}<\frac{1}{r(r-1)}$$ ($$r<N<2^L$$)，也就是说对于所有分母不大于 $$r$$ 的分数里，最多只存在一个与测量值的距离满足理论条件，这是唯一性的一边，这部分证明很少有人提到。大家都会提到的是如何求这个数，也即连分数算法，该待求分数必定是测量小数的某阶连分数展开近似。我们依据连分数的算法将测量值逐阶展开，并根据递推关系计算相应的对应的分数 $$p/q$$ (教材附录定理 A4.15)。每计算出一阶的该分数，就计算该分数和我们的测量小数的距离是否小于 $$1/2^{L+1}$$，直到第一个满足条件的阶数成立，对应分数即位待求的 $$s/r$$ 或其约分式。满足条件分数的存在性，是 phase estimation 取的精度位数所保证的。因此这一从有限精度小数到求出分数的过程，需要三个定理来保证：分别是 phase estimation 精度估计对应的存在性，这里定理提到了唯一性，和教材定理 5.1 保证的求解方法。

### 其他经典算法的多项式复杂度

以下是其它一些在 Shor 算法过程中会利用到的经典算法。

* 整数乘法

  小学数学 $$M(L) = O(L^2) $$

  Schonhage-Strassen 算法 $$M(L)=O(L\log L\log\log L)$$，利用了快速傅立叶变换

* 整数除法

  小学数学 $$O(L^2)$$, 注意到商的每一位只能取 0 到 b-1，b 是进制位数，因此每一位的暴力猜解是常数复杂度。

  更精巧的 Newton–Raphson division，可以将除法复杂度约化成乘法复杂度。也即 $$O(M^n)$$。

  更多大整数算术运算的复杂度可以参见[^wiki]。

* 辗转相除寻找最大公约数和判断是否互质

  每两轮对应数字至少折半，因此只需要 $$O(L)$$ 步就可完成 gcd 计算。每步的除法是 $$O(L^2)$$，因此整个辗转相除法的复杂度是 $$O(L^3)$$。实际上如果仔细考察的话，会发现每一步除法的复杂度等于除数的位数乘以商的位数，而每轮商的位数是相邻两轮除数位数之差，考虑到该效应求和，则辗转相除的总复杂度仅为 $$O(L^2)$$。

* 随机素性测试

  随机可直接利用费马小定理检验素性，只需要 $$O(1)$$ 次，每次一个快速指数计算 O(LM(L))

  事实上确定性的素数测试也已经被证明是一个 P 问题[^AKM]。

* 快速指数计算 $$a^b$$

  divide and conquer $$O(\log b\, M(L))$$，这里考虑到了 $$a,b$$ 位数不同。

* 连分数的收敛性

  连分数各阶近似对应的分数，每两次分母至少翻倍，因此可以保证展开 $$O(L)$$ 阶即可完全表达一个 L bit 分母的分数。


## References

[^so]: [Was the reduction in Shor's algorithm originally discovered by Shor?](https://cstheory.stackexchange.com/questions/25512/was-the-reduction-in-shors-algorithm-originally-discovered-by-shor)

[^cylic]: [When Is the Multiplicative Group Modulo n Cyclic?](https://pi.math.cornell.edu/~mathclub/Media/mult-grp-cyclic-az.pdf)

[^sieve]: [Sieve algorithms for perfect power testing](https://link.springer.com/article/10.1007/BF01228507)

[^linear]: [DETECTING PERFECT POWERS IN ESSENTIALLY LINEAR TIME](https://www.ams.org/journals/mcom/1998-67-223/S0025-5718-98-00952-1/S0025-5718-98-00952-1.pdf)

[^AKM]: [AKM primality test](https://en.wikipedia.org/wiki/AKS_primality_test)

[^wiki]: [Computational complexity of mathematical operations](https://handwiki.org/wiki/Computational_complexity_of_mathematical_operations)

[^numbertheory]: [My notes on primary number theory](https://re-ra.xyz/misc/numbertheory.html)

[^Shor]: [Polynomial-Time Algorithms for Prime Factorization and Discrete Logarithms on a Quantum Computer](https://arxiv.org/pdf/quant-ph/9508027.pdf)

[^Nielsen]: [Quantum Computation and Quantum Information](http://mmrc.amss.cas.cn/tlb/201702/W020170224608149940643.pdf)