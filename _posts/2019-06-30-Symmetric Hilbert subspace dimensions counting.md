---
layout: post
title: "Symmetric Hilbert subspace dimensions counting"
date: 2019-06-30
excerpt: "严格对角化，对称性加速哪家强"
tags: [physics, mathematica, algebra]
comments: true
---

* toc
{:toc}


## 引言

本文将记录一些严格对角化(Exact Diagonalization)时，通过对称性降低 Hibert space 维度，分块对角化的方式，其重点放在对于 symmetry reduced block 的维度的计算。

如无特别说明，以下的讨论，都假设系统有 $$N$$ 个格点，每个格点有 $$d$$ 个自由度。对于具体的数值比较，数据则都来自 $$d=2$$ 的情形。

## Z2 对称性

先来一个最简单的例子，如果模型具有某种 $$Z_2$$ 对称性，操作记为 P。则我们需要将直积 basis 重新组合为

$$
\vert i'_{\pm}\rangle=\frac{1}{\sqrt{2}}(\vert i\rangle \pm P\vert i\rangle).
$$

对于 $$P\vert i\rangle=\pm\vert i\rangle$$ 的态，其本身就具有 $$Z_2$$ 对称性，分别是 $$Z_2$$ 的一维奇表示和偶表示的基。我们记本来 $$d^N$$ 个直积态中，满足 $$P\vert i\rangle =\pm\vert i\rangle$$ 的态的数目为 $$F_e, F_o$$，则系统 Hamiltonian 最后对角化分块的 Hilbert space 维度分别为 $$F_e+(d^N-F_e-F_o)/2, F_o+(d^N-F_e-F_o)/2$$，当本来系统 $$F_e\neq F_o$$时，两个子块大小并不相同。

考虑一个具体例子，假设 spin 1/2 的系统具有 inversion 对称性，这是典型的 $$Z_2$$ 对称性，我们这里只考虑格点为偶数个的一维情形。此时 $$F_e=2^{N/2}, F_o=0$$，对应的两个子空间的维度为 $$2^{N-1}\pm2^{N/2-1}$$。衡量一个对称性对于加速 ED 计算的贡献，就要看 effectively，symmetry reduce 之后的 block 的 dimension 相当于“减掉”了几个格点。很容易看出 $$Z_2$$ 可以减少 **1** 个格点，而且这种 reduce 不随 size 增加而变强。

## 平移对称性

有限格点的平移对称性和 $$Z_2$$ 其实类似，对应的一维不可约表示的基态为：

$$
\vert a(k)\rangle = \frac{1}{\sqrt{N_a}}\sum_{r=0}^{N-1} e^{-i k r} T^r\vert a\rangle.
$$

其中 $$N_a$$ 是归一化因子，$$T$$ 是平移操作算符。注意只有周期性边界条件，这一平移对称性才 well defined。k 是该平移对称性的不同不可约表示的标记，$$k=n\frac{2\pi}{N}$$，$$n=0,1…N-1$$。理想情况下（粗略估计），每 N 个不同的 $$\vert a\rangle$$ 态一组，线性组合贡献到不同的 N 个 k subspace 里。Hamiltonian 具有 $$d^N/N$$ 维度的 subblock。但事实上，和 $$Z_2$$ 一样，有些态 $$\vert a\rangle$$ 本来就具有比 N 小的周期 x，也即 $$T^x\vert a\rangle =\vert a\rangle$$， $$x<N$$。此时，将只有 $$x$$ 个 state 的线性组合分别贡献到 $$k=q\frac{2\pi}{x}, q\in Z$$ 的 sector 里，这种 sector 共有 $$x$$ 个。

现在我们回到 1D spin half chain，来讨论直积态中不同周期态的数目。我们定义 $$F(N)$$ 为 N 个格点的，周期为 N，也即没有更小的周期结构的 0，1 序列。很容易证明，这一序列满足以下递推关系：$$F(N)=2^N-\sum_{T\vert N}F(T)$$，也即要减去可能更小周期的序列数目 $$F(T)$$。T 可能的取值，则是 N 的一切因数。通过该递推关系，我们可以写出该数列的前几项为 2，2，6，12，30，54，126。通过整数序列数据库[^oeis]，可知该数列为 [A027375](https://oeis.org/A027375)。

相应的，根据上述分析，最大的 block 就是 $$k=0$$ 的 block，该 block 的态有来自所有周期的态的贡献，其维数为 $$D(k=0)=\sum_{T\vert N}\frac{F(T)}{T}$$ ，相应的前几项为 2, 3, 4, 6, 8, 14, 20, 36, 60, 108，其对应的数列为 [A000031](https://oeis.org/A000031)。根据此，可分析 $$k=0$$ block 对于 ED 的加速，其在常见的 $$N=20$$ 这一数量级，大概可以减少 **4.3** 个格点的运算量。值得注意的是，同 $$Z_2$$ 对称性一样，比粗略的维度计算更精细的分析，在 N 较大时的渐进行为是一致的，从 $$N=20$$ 开始，粗略估计和精细估计给出的维度对应的等效粒子数差已经小于千分之一。

## U(1) 对称性

U(1) 对称性导致了电荷守恒。因此一般处理起来都是直接按照直积态投影到相应的 charge sector 即可，甚至都不需要任何直积态的线性组合。对于 charge 为 Q 的 sector，或者在 1D spin chain 的语言中，也即 $$J^z_{tot}=Q$$，其 Hilbert space dimension $$C_N^Q$$。其中 half filling 的 sector 是最大的，dimension 为 $$C_{N}^{N/2}$$。在格点数为 20 时，其可等效减少 **2.5** 个格点。

## charge 和 dipole 守恒

对于 fracton 系统，除了电荷守恒外，电荷的极距 $$\sum_i r_i\rho(r_i)=\text{const}$$，也即是 dipole 守恒。下面是利用 mathematica 数清对应 dipole sector (D) 和对应 charge，dipole sector (C,D) 的代码。

```mathematica
dipole[list_] := Total[Table[i, {i, 1, Length[list]}]*list]; 
charge[list_] := Total[list];
Table[d = dipole /@ Tuples[{0, 1}, n]; 
 count = ConstantArray[0, n (n + 1)/2 + 1]; 
 Table[count[[d[[i]] + 1]] += 1, {i, 1, Length[d]}]; 
 Max[count], {n, 1, 12, 1}]
(* {1, 1, 2, 2, 3, 5, 8, 14, 23, 40, 70, 124} *)
Table[d = dipole /@ Tuples[{0, 1}, n]; 
 c = charge /@ Tuples[{0, 1}, n]; 
 count2 = ConstantArray[0, {n + 1, n (n + 1)/2 + 1}]; 
 Table[count2[[c[[i]] + 1, d[[i]] + 1]] += 1, {i, 1, Length[d]}]; 
 Max[count2], {n, 3, 12}]
(* {1, 2, 2, 3, 5, 8, 12, 20, 32, 58} *)
```

上述第一个 `Table` 对应了只考虑 dipole 守恒的分块对角化中最大块的维度，其对应的整数数列为 [A025591](https://oeis.org/A025591)；而第二个 `Table` 同时考虑了 charge 和 dipole 守恒进行的维度计数。其最大子块的维度满足数列 [A277218](https://oeis.org/A277218)。更进一步地我们可以 `MatrixPlot[count2]` 来观察不同 (C,D) sector 的维数情况。

![](/images/dipolesector.svg)

该图横轴为 dipole sector，纵轴为 charge sector。上图颜色越深，对应的 sector Hilbert space dimension 越大，可以看到对于 charge 和 dipole 都处于中间的 sector，subspace 维度最大。对于 $$N=20$$，最大的 subspace dimension 是 5448，相当于等效减去了 **7.6** 个格点！相似的研究方法可以用来进一步研究高自旋 fracton 系统 Hamiltonian 分块对角的情况。

## SU(n) 对称性

本部分内容主要参考了[^SUN]。事实上，在 ED 上施加 SU(2) 对称性很是麻烦，甚至大多数学术工作也都不会去加。因为毕竟加对称性对于对角化方法而言，只是锦上添花，不加一切还是照做，只不过可以计算的 size 变小了而已。

其主要难度在于，对于直积态基的线性组合来构成 non-abelian group 的不可约表示的承载基是 highly nontrivial 的，需要大量的 CG 系数计算以及杨图辅助等等。这里不会讲述考虑 SU(n) 对称性的细节，其形式理论可以对照下文的形式体系部分理解，其实际操作可以参考文献。因为本文的重心是相关对称性对于 ED 分块的 speed up，也即 subspace 的 dimension counting。因此这部分只关注怎么计算 SU(n) 对称性下各 block 的维度大小。首先是各 block 的构成，每个 block 由**同一种不同个的不可约表示的同一个基**组成，也即每个 block 的 dimension 大小等于该不可约表示出现的次数，而非该表示的维度。至于 block 为何是这样组成的，可以参照形式体系部分的证明。

因此，如果对于 SU(n) 每个格点上都是一个基本表示的话，我们需要关注直积分解的系数，或者说重数 $$f_i$$，也即

$$
\otimes^N V_b(G)=\oplus_i f_i V_i(G).
\label{sude}
$$

至于该直积分解的重数，也即对应的 Hilbert space 的维数如何计算，有几种不同的方式。

1. 可以利用特征标的正交关系，也即 $$\int dg\;\chi_i(g)\chi_j(g)=\delta_{ij}$$。我们对 $$(\ref{sude})$$ 两边取 trace，同时乘以 $$\chi_i(g)$$，再在 haar measure 上积分，即可得到 $$f_i$$ 的值。

   作为一个具体的例子，考虑 $$\mathrm{SU(2)}$$ 对称性的 J=0 的子空间维度计算。SU(2) 群的 Haar measure 为 $$\frac{1}{\pi}\int_0^{2\pi} d\theta\; \sin^2\theta$$ [^haar] （特征标总是在其他角度上 isotropic，因此该问题的 measure 中忽略）。同时又有自旋为 j 的不可约表示特征标为 $$\sin((2j+1)\theta)/\sin\theta$$。对于 j=0 block，我们有:
   
   $$
   \frac{1}{\pi}\int_0^{2\pi}d\theta \; (2\cos\theta)^N\sin^2\theta= \frac{C_{n}^{n/2}}{\frac{n}{2}+1}.
   $$
   
   上面只考虑了 N 为偶数，因为奇数格点不会给出 spin singlet 的 sector。

2. 利用杨图的规则，可以直接快速的求出 $$\rm{SU}(n)^N$$ 的任意不可约表示重数。考虑到每个不可约表示都对应一个杨图，那么该不可约表示的重数可由该杨图直接计算 $$f_i=\frac{n!}{\Pi_{j=1}^n l_j}$$，其中 $$l_j$$ 对应杨图每一块上的钩子数，其值为相应方块正下方和正右方的方块数目加 1。

   同样对于 SU(2) 的 J=0 子空间的分析例子，其 1 维不可约表示对应的杨图，为 2 行每行 n/2 列的方块。容易计算得到其重数为 

   $$
   f_1=\frac{n!}{(n/2)!^2/(n/2+1)}= \frac{C_{n}^{n/2}}{\frac{n}{2}+1}.
   $$





根据以上计算，对于 $$N=20$$，spin singlet block 等效于减少了 **6** 个格点。当然 spin singlet 并不是最大的子空间，还是对于 $$N=20$$，重数最大的不可约表示是 $$j=2$$，其相当于减少了 **4.4** 个格点。

## 广义形式体系

现在我们给出 Hamiltonian 算符根据对称性组合为分块对角形式的普遍理论。该理论可以作为前边这些例子的一个参考和注脚。

### 内部连续对称性

先考虑体系格点内部的 d 维自由度的对称性。假设其内部自由度的对称群为 G，此处 G 为全局对称性，即所有格点上的场算符的变换是一致的，而非局域的规范对称性。比如 G 可以是 SU(2)，刻画了格点上自旋的旋转对称性。对于 N 个格点的系统，其 $$d^N$$ 个直积态可以通过线性组合给出若干正交的子空间用来承载对称群 G 的不可约表示。也即：

$$
\otimes^N V(G) =\oplus_{i} f_iV_i(G). \label{decomp}
$$

其中，$$V(G)$$ 是 G 在一个格点自由度上作用的表示（通常是群 G 的基本表示），直积分解成右侧的若干不可约表示 $$V_i(G)$$，每一个不可约表示出现的重数为 $$f_i$$，也即有 $$\sum_i f_i d(V_i)=d(V)^N$$。其中对于 SU(N) 群的情况，以上直积分解的结果，可以利用杨图的相关规则，格外容易计算。

我们现在考虑按 $$(\ref{decomp})$$ 的方式重新组合后的基，对于希尔伯特空间分块的影响。考虑在现在的 basis 上作用群 G，则G 在该空间的表示形式如下：

$$
U(g)=\begin{pmatrix}
V_1(g)&&&\\
&V_2(g)&&\\
&&V_2(g)&\\
&&&...
\end{pmatrix}, g\in G.
\label{sym}
$$

其中 $$V_i(g)$$ 在对角块出现的次数为 $$f_i$$。由于 Hamiltonian 具有 G 对称性，也即在表示空间我们有：

$$
U(g)^\dagger HU(g)=H, \forall g\in G.
$$

我们将直积分解，基重新组合后的 H 的矩阵形式记为：

$$
H=
\begin{pmatrix}
H_{11}&H_{12}&H_{13}&\\
H_{21}&H_{22}&H_{23}&\\
H_{31}&H_{32}&H_{33}&\\
&&&...
\end{pmatrix}.
$$

则根据 $$(\ref{sym})$$，我们有诸如 $$V_i(g)H_{ij}V_{j}^\dagger(g)=H_{ij}$$，也即 $$V_i(g)H_{ij}=H_{ij}V_j(g)$$ 对于任意群元素 g 成立。由于 $$V_i, V_j$$ 都是不可约表示，这正是 Schur lemma 的内容。此式恒成立，要么矩阵 $$H_{ij}=0$$ 是全零矩阵，要么 $$V_i=V_j$$ 是同一个不可约表示。把 Schur lemma 翻译回 ED 的语言，对于按照不可约表示排列的 basis $$\vert j, f, m\rangle$$，其中 j 标记了是哪种不可约表示，f 标记了是该不可约表示中的第几次出现（重数信息），m 标记了该态在不可约表示中的排序。那么对于 Hamiltonian 的矩阵元我们有以下规律：

$$
\langle j,f,m\vert H\vert j',f',m'\rangle=\delta_{jj'}\delta_{mm'}H_{jff'}.
$$

该非零矩阵元出现的条件是如何和 Schur lemma 的两种可能一一对应的。上式说人话就是来自不同种不可约表示的基夹的矩阵元是 0 ($$j\neq j'$$)，来自同一种表示，不同位置态夹的矩阵元也是 0 ($$m\neq m'$$)。

进一步地说，通过直积分解对应的基变换之后，Hamiltonian 分成了不同的子块，一组 (j,m) 就决定了一个子块，每个子块的维度（边长）大小和重数相等 $$d_i=f_i$$。 需要注意的是，对于同一个 j 的不同 m 的子块，能谱将完全相同，这是由高维不可约表示带来的简并。（事实上，能级简并精确的说不是对称性带来的，而是对称性的非 1 维不可约表示带来的。）因此，求得具有全局 G 对称性的 Hamiltonian 的能谱，只需要分别对角化 J 个矩阵，J 是 N 个 V(G) 直积时，分解出的所有可能的不可约表示的数目。若 $$G=\mathrm{SU}(2)$$，则 $$J=(N+1)/2$$。其中每个矩阵的维度是 $$f_i$$，等价于该不可约表示出现的重数。

也就是说 ED 最广泛的情形，不是每种不可约表示分成一个子块，而是**每种不可约表示的某一个基**构成一个子块。这点经常被忽略或误解，原因在于大部分 ED 中处理的对称性是可交换的，没有高维不可约表示，上述区别体现不出来。

### 外部离散对称性

考虑完内部自由度带来的对称性后，再来考虑整个格点结构带来的各种离散的空间对称性 $$K$$，比如平移，空间群操作等等。由于我们已经考虑过了内部自由度，那么以下的操作都在相应的已经约化过的子空间和基上进行。若子空间中本来的 basis 为 $$\vert r\rangle$$，则考虑了离散对称性后的线性组合的新基为[^EDlecture]：

$$
\vert \tilde{r}\rangle=\frac{1}{\mathcal{N}\sqrt{\vert K\vert}}\sum_{k\in K} \chi(k)\vert k(r)\rangle,\label{state}
$$

其中 $$\mathcal{N}$$ 是归一化因子。$$\chi$$ 是相应不可约表示的特征标，不同的不可约表示对应了不同的 $$\vert \tilde{r}\rangle$$。对于所有的不同的 $$\vert r \rangle$$ 同一不可约表示特征标，可以给出一些态 $$\vert \tilde{r}\rangle$$，这些态就构成了原来 block 的更小的 subblock，可以用来进行对角化计算。$$(\ref{state})$$ 在群元素下作用的变换为：

$$
\vert k'(\tilde{r})\rangle =\frac{1}{\mathcal{N}\sqrt{\vert K\vert}}\sum_{k\in K} \chi(k)\vert k'k(r)\rangle\\
=\frac{1}{\mathcal{N}\sqrt{\vert K\vert}}\sum_{k'k\in K}\chi^{-1}(k')\chi(kk')\vert k'k(r)\rangle\\
=\chi^{-1}(k')\vert \tilde{r}\rangle.
$$

当然空间对称性也可能是 non-abelian 群，此时 K 具有高维的不可约表示。这时新基的构造需要按照高维不可约表示的基来组合，并且分块，过程类似上一部分的内部连续对称性。只不过离散群不是李群，不可约表示只有有限多个。这一子空间套子空间的缩减总是可行的，因为模型的所有对称性操作对易，都可以按照 Schur lemma 进行分析。后分析的对称性操作在先分析的对称群的不可约表示的基下，总是只沟通 (j,m) 都相同的子空间，也即保持了对角块的结构。

## References

[^SUN]: Exact Diagonalization of Heisenberg SU(N) Models, PRL **113**, 127204 (2014).

[^EDlecture]: [Introduction to Exact Diagonalization](https://www.pks.mpg.de/~aml/LesHouches/LesHouches_ED_Lecture.pdf).

[^oeis]: [The OnLine Encyclopedia of Interger Sequences](https://oeis.org).

[^haar]: [Notes for the course on Representations of Lie Groups](http://web.math.ku.dk/~jakobsen/replie/compact-work.pdf).