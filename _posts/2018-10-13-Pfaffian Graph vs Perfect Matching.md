---
layout: post
title: "Pfaffian Graph vs Perfect Matching"
date: 2018-10-13
excerpt: "计算图上 perfect matching 总数的数学工具"
tags: [physics, graph theory, algebra]
comments: true
---


* toc
{:toc}

## 引言

这篇内容作为**物理中的计算理论**的一个间奏，为了之后的 2D spin glass is in P 做铺垫。本文将交代一个重要的数学工具。我们尝试解决图上 perfect matching 数目这一问题，并将这一几何问题转化到代数上 Pfaffian 的计算，从而对某些图能够在多项式时间计算 perfect matching 的数目。

## 形式定义

### 图与邻接矩阵

图 $$G=(V,E)$$ 的相关定义和常用规范相一致。所谓图上的 matching 是指一组边的的集合，使得图上每个顶点在 matching 中的边不超过一条。如果所有顶点都可以保证有一条相连的边在 matching 中，这样的 matching 被称为 perfect matching。事实上 matching 的定义和物理系统中的 dimer model 相一致。关于 matching 的更多特征及算法，可以参考我之前写的[无向图的最大权重匹配算法](/无向图的最大权重匹配算法)，这一算法下篇还会用到。

所谓 planar graph 是指的可以嵌在平面上，并且没有边交叉的图。对于平面图，每个被边包围的区域成为一个面 (face)。我们有 Euler 定理，对于平面图，顶点数－边数＋面数＝2 （这里的面数包括了最外面的开 face）。平面图的对偶图，是指图的顶点对应原图的 face，图的边的有无对应原图相邻 face 之间是否存在边。图的定向是指给无向图的每个边都指定一个方向，使得图成为有向图。

图上的 cycle 是指的一组首尾闭合的边。cycle 的和是两个 cycle 边集合的 symmetry difference，也即从全集里去掉交集部分。所谓的 cycle generating family 是指的一组 cycle 的集合，使得图的每个 cycle 都可以用这些 cycle 的和来表示。对于平面图，cycle generating family，就是全部 face 的 cycle。对于可以嵌入在 torus 上的图，cycle generating family 还需要添加两个 topological non trivial 的 loop。

所谓图的邻接矩阵，是指一个 $$n\times n$$ 的矩阵，且在有边存在的项上非空，比如 $$a_{ij}=1$$，表示对应的图从 i 顶点到 j 顶点有一条边。如果是有向图，还可以使得 $$a_{ij}=1, a_{ji}=-1$$，表示有一条从 i 指向 j 的有向边。进一步，对于有权图，还可以将对应的矩阵元改造成权重值。

### Pfaffian

对于$$2n\times 2n$$反对称的矩阵 $$A=\{a_{ij}\}$$，我们可以定义矩阵的 Pfaffian，

$$
pf(A)=\frac{1}{2^nn!}\sum_{\sigma\in S_{2n}}sgn(\sigma)\prod_{i=1}^n a_{\sigma(2i-1),\sigma(2i)}
$$

其中$$S_{2n}$$是2n个元素的置换群，$$sgn(\sigma)$$ 对于奇偶置换分别等于 $$\mp 1$$。为了简化该定义，我们也可以不对所有置换求和，而只对独立的划分求和。一个划分，是指一种对 2n 个数的配对方式。对于 $$(2n)!$$ 中置换，共有 $$(2n-1)!!$$ 种独立的划分。记 $$\Pi$$ 为划分的集合，我们有化简的 Pfaffian 表达式，

$$
pf(A) = \sum_{\pi\in\Pi}sgn(\pi)\prod_{i=1}^n a_{\pi(2i-1),\pi(2i)}
$$

Pfaffian 最重要的性质，莫过于 $$pf(A)^2=det(A)$$，该证明可以参考 [1] ( 对于熟悉场论的读者，[1] 的第四节的证明，利用 Grassmann 代数，显然更有启发性)。更多关于 Pfaffian 的性质请参考 [wiki](https://en.wikipedia.org/wiki/Pfaffian)。

### Determinant is in P

由于 Pfaffian 是行列式的开方，因此虽然定义是指数级的时间复杂度，但可以在多项式时间计算得到结果。这源于行列式的不平凡之处，行列式定义上的指数多的项数，和计算时的多项式时间复杂度，毫无疑问成为了解决看似 intractable 问题的一座有力的桥梁，大量复杂的组合问题如果归约到和行列式相关，那么也就找到了多项式时间的解决方案。

行列式之所以能在多项式时间求得，源于其性质 $$\det(AB)=\det(A)\det(B)$$。考虑到基于高斯消元法的 LU 分解，我们可以在 $$O(n^3)$$ 的时间里求得矩阵的 LU 分解，将矩阵分解为一个上三角矩阵和一个下三角矩阵的乘积。再根据三角矩阵的行列式等于对角线上项的积，我们立即就得到了矩阵的行列式。因此某种程度上，行列式的积等于积的行列式这一性质 highly nontrivial，它直接导致了看似指数复杂度的行列式是一个 P 问题。

与之对比，所谓 permanent （积和式）是去掉行列式各项的符号得到的式子。也即 

$$
perm(A)=\sum_{\sigma\in S}\prod_{i=1}^na_{i,\sigma(i)}
$$

由于 permanent 不具有积的积和式是积和式的积这样好的性质，使得 permanent 的计算是 #P 困难的。这一对比，更加突出了行列式是 P 问题并不是平凡的。

## 邻接矩阵的 Pfaffian

### Pfaffian 与任意图的 perfect matching

我们考虑上面对于偶数个顶点的有向图定义的邻接矩阵，由于 $$a_{ij}=-a_{ji}$$，可知该矩阵是反对称的，因此可在该矩阵上定义 Pfaffian。将该 Pfaffian 按照以划分求和的定义展开，我们发现 Pfaffian 展开对应的每个非零项的划分恰好和图上可能的 perfect matching 一一对应。现在唯一需要 fix 的问题就是，Pfaffian 还有一个置换的符号，如果能保证所有非零项的符号一致，那么邻接矩阵的 Pfaffian 就对应了相应图上 perfect matching 的总数目。也就是说，求图可以具有的 perfect matching 数目的问题，就归结为了给图 G 的每条边定向的问题：能否找到一种定向的方式，使得这一有向图对应的邻接矩阵，给出的 Pfaffian 各项的符号相同。

### Permanent 与二分图的 perfect matching

其实 permanent 也可以给出类似的联系。考虑一个 bipartite graph，且两边的顶点数相同均为 n，那么可以给出一个类似邻接矩阵的 $$n\times n$$ 的矩阵，行和列分别代表两个部分。对于该矩阵求 permanent 就相当于求出了原 bipartite graph 上 perfect matching 的数目，而且这里也没有符号问题。不过硬伤就是，这种转化看似同样有力，但并没有什么用。只不过把一个指数多项的求和转化到了另一个，问题依旧无法在 P 时间内得到解决。

## Pfaffian Orientation

### 观察

如上节所述，我们已经将图上 perfect matching 数目的 counting 归约到了如何将图上的边定向，从而实现各项在 Pfaffian 中的符号相同，我们将满足条件的定向称为 Pfaffian orientation，对应可以实现这种定向的图称为 Pfaffian 图。[2] 和 [3] 证明了，所有 planar graph 都是 Pfaffian graph。下面我们就来看一下这个证明。以下只叙述一个框架，需要实例和进一步解释的可以参考 [4]。

这个证明的核心得益于这样一个观察，同一图上两个不同的 perfect matching 的 symmetry difference 是一些偶数个边组成的 cycle。这一 cycle 上的边，交替处在两个不同的 matching 里。如下图，a和b代表两种不同的 perfect matching，c 去掉匹配边的交集，剩下的是两个 even cycle。

<img width="70%"  src="https://user-images.githubusercontent.com/35157286/46900451-dc58dd00-ced4-11e8-9f3d-21e35d574560.png">

考虑这样的 even cycle 上，如果顺时针和逆时针的有向边都是奇数条，则称为 oddly oriented。（这一 oddly oriented 的定义也可以扩展到 odd cycle，这时 oddly oriented 可以定义为有奇数条边是逆时针定向的。）对于每一个这样的 even cycle，两个 perfect matching 相差一个奇置换的符号，而 oddly oriented 就保证了两个 matching 在该 cycle 上贡献负号的边数一奇一偶，这又贡献一个符号，从而两者抵消。也就是说，两个 matching 如果 symmetry difference 形成的 even cycle 都是 oddly oriented，那么两个 matching 在 Pfaffian 展开时的符号相同。推而广之，如果我们能找到这样一种定向，使得所有的 even cycle 都是 oddly oriented，那么所有的 perfect matching 的项符号就相同，这样 Pfaffian 就直接给出了所有 perfect matching 的数目。

### 证明

现在我们就要来证明，平面图总能找到这样的定向，使得所有 even cycle 都是 oddly oriented。这一证明基于以下几个事实。

1. 存在这样一种定向，使得平面图的每个 face 都 oddly oriented。
2. 对于 cycle，其 oriented 的奇偶性与其 cycle 内部包含的顶点个数相反。
3. perfect matching 给出的 symmetry difference，这种 cycle 只能包含奇数个顶点。

先看1，一种构造性的证明，就是[FKT algorithm](https://en.wikipedia.org/wiki/FKT_algorithm)。在图G上产生一颗生成树（通过BFS或DFS），对于生成树上的边随意指定定向。利用图G的对偶图，不过只保留那些对应不在树上的边的对偶边。注意到此时对偶图是一颗树，否则对偶图有 loop，说明原图有顶点不在生成树上，和生成树定义矛盾。从对偶图的树的叶子顶点开始删除，每次删除一个顶点，将连接该顶点的边对应的原图的边的定向固定，使得删除顶点所在的 face 形成的 cycle 是 oddly oriented。该构造性证明的一个图示如下（来自 wikipedia）。该证明实际上已经找到了任意平面图的 Pfaffian orientation，剩余的 2，3只是进一步验证正确性而已。找到合适的定向的该算法的复杂度显然是多项式级（线性时间）。

![pfaffian_orientation_via_fkt_algorithm_example](https://user-images.githubusercontent.com/35157286/46900439-9c91f580-ced4-11e8-8a86-592817a48ef1.gif)



现在证明 2。考虑一些 face 的 symmetry difference 组成的 cycle，其中可能包含若干个顶点。对于 cycle 内部的每一条边，必然贡献了原来某个 face 的逆时针定向（因为两相邻 face 的时针定向在该处相反，但边的定向唯一）。本来全部的逆时针定向边数目为 $$odd*f$$。而不在外 cycle 而在内部被抵消掉的逆时针定向边的数目，根据 Euler 定理，为环上顶点数目＋总 face 数－环上边数＋环内顶点数－1。这一数目的奇偶性取决于$$f+v+1$$，v 为环内包含的顶点数目，也即外 cycle 剩余的逆时针定向的边数的奇偶性与$$v+1$$相同，也即 cycle 的定向奇偶性与包含内点数目的奇偶性相反。

最后，3的证明是显然的。由于平面图的边不能交叉，如果存在 perfect matching 的 symmetry difference 包含了奇数个顶点，那么至少有一个顶点无法形成 matching，与 perfect matching 相矛盾。

由1，2，3，就保证了，FKT 算法找到的定向，可以使得图中任意 even loop 是 oddly oriented，也就保证了平面图可以利用 Pfaffian 来计算 perfect matching 的数目。需要注意如果不是 perfect matching （dimer 系统里存在 monomer），以上方法无法有效计算一般 matching 的数目，因为事实3会被破坏，每个 cycle 现在可以包含奇数个顶点。

## Non planar graph 的一些结果

以上的证明只针对平面图的情况。可以证明对于任意图，计算 perfect matching 的数目是 #P 完全的 ([5])，这正源自积和式计算的复杂度。换句话说一般图是无法找到相应的 Pfaffian orientation。事实上，非平面图的 minor $$K_{3,3}$$（每组有3个顶点的全连接 bipartite graph）就无法实现这种定向，这也阻止了一般非平面图的问题的解决。

不过我们可以得到一些在 torus 上嵌入图的结果。由于这对应了物理系统的周期性边界条件，所以值得特别的关注。这一部分，我们只以方格子进行讨论，事实上一般的情形比较复杂。正如 [6] 指出的

> Kasteleyn was able to compute the dimer partition function for the square lattice on the torus using 4 Pfaffians. He also stated **without further detail** that the partition function for a graph of genus g
> requires 4^g Pfaffians. Such a formula was found much later by Tesler and Gallucio-Loebl.

这一个 later 就是四十年的光景，所以一般情况证明可想而知比较麻烦，这里不再赘述，可以参考 [7]，[8]。

对于方格子的情形，可以看出平面图上的方格子，如下图的定向是一个 Pfaffian 定向。

<img width="40%" alt="screen shot 2018-10-13 at 11 31 55" src="https://user-images.githubusercontent.com/35157286/46900825-93f0ed80-cedb-11e8-86ed-5283d49d2a81.png">

如果对该方格子的边界加上周期性边界条件，那么平面图就变为了 torus 上的图。可以证明这种图上找不到Pfaffian orientation，证明和上节的事实2类似，具体细节可以参考 [4] 的定理 4.2。事实上，我们保持上图的定向，记为$$B_1$$，引入周期性边界条件和水平竖直的假想边界线，对于每个 perfect matching，现在会出现四种情况(e,e),(o,e),(e,o),(o,o)之一。这四种情况分别代表有偶数（奇数）个 matching 跨越了水平（竖直）的边界线。那么我们用一个未穿越任何边界线的 matching 作为基准（该基准 matching 属于 (e,e) 类），不同类的 matching 与基准的 symmetry difference 如下图。

<img width="90%" alt="screen shot 2018-10-13 at 11 39 48" src="https://user-images.githubusercontent.com/35157286/46900910-afa8c380-cedc-11e8-9acf-da9aa310e429.png">

对于不跨越边界线的 even cycle，依旧不贡献相对符号。而对于每个跨越了边界线的 cycle，按照以上的定向，事实上是 even oriented 的（其实这一点也很 subtle，到考虑这种 torus 上 nontrivial cycle 的变形时，其实是可以变为 oddly oriented，只不过此时该 cycle 两侧的顶点数都变为奇数个，因此不可避免的引入了另一个方向上的跨越边界线的 cycle，使得总 cycle 数变为了偶数个，最后等效的，每个穿越边界线的 cycle 是 even oriented）。这样每个跨越边界线的 cycle 贡献一个负号到对应的 Pfaffian 的项。因此我们计算按照$$B_1$$定向的邻接矩阵的 Pfaffian，(e,o),(o,e),(o,o) 构型对应的 perfect matching 的符号和 (e,e) 构型对应的符号相反。其中比较巧妙的是 (o,o) 构型实际上也只对应奇数个跨越边界线的 cycle。

下一步，我们要反转定向$$B_1$$中跨越水平边界的定向，用新的邻接矩阵$$B_2$$，计算一个新的 Pfaffian，此时(o,e),(o,o),(e,e) 对应的 matching 计算正确。同理，我们也可以只翻转$$B_1$$中跨越垂直边界的定向形成邻接矩阵$$B_3$$，或者同时反转水平和垂直边界的定向得到$$B_4$$。每一次都有一些构型是计算正确的，而其他的是反号。一共四族构型，现在我们有四个独立的 Pfaffian，这样我们已经可以单独求出每组构型中的 perfect matching 的数目。记这四次的邻接矩阵分别为 $$B_i$$，那么总的 perfect matching 的数目为 $$\frac{1}{2}(-Pf(B_1)+Pf(B_2)+Pf(B_3)+Pf(B_4))$$。同样的我们也可以单独求出不同的构型中 perfect matching 的数目。

上述做法容易推广到亏格为$$g$$的曲面嵌入的图上，此时由于有 $$2g$$ 个 nontrivial 的 cycle，那么根据以上的论述，应该需要计算 $$2^{2g}$$ 个矩阵的 Pfaffian，才可求得总的 perfect matching 的数目。

## 邻接矩阵的一些魔改

我们在以上的讨论中，都默认了邻接矩阵的对应边的矩阵元取值为$$\pm1$$。通过改变这一矩阵元的定义，可以为相应的 Pfaffian 的计算带来更丰富的物理内涵。

可以令不同的边处于不同的边类中，就如 [3] 中的做法。例如，对于方格子，每个竖直边邻接矩阵元定义为$$\pm z_1$$，每个水平边的邻接矩阵元定义为 $$\pm z_2$$，这样对应 Pfaffian 中 $$z_1^{n_1}z_2^{n_2}$$ 的系数，就是 perfect matching 有 $$n_1$$ 个 matching 边竖直，$$n_2$$ 个 matching 边水平的个数。

更进一步的，如同 [9] 中所做的那样，对于每个边对应的邻接矩阵元，可以定义为 $$\pm e^{-\beta E_e}$$，$$E_e$$ 代表了该边对系统总能量的贡献。那么对这样一个改造过的邻接矩阵求 Pfaffian，就可以很容易得到相应 dimer 系统的配分函数。

## Reference

1. [Notes on antisymmetric matrices and the Pfaffian](http://scipp.ucsc.edu/~haber/webpage/pfaffian2.pdf).
2. P. W. Kasteleyn, *The statistics of dimers on a lattice: I. The number of dimer arrangements on a quadratic lattice*, Physica 27, 1209 (1961).
3. P. W. Kasteleyn, *Dimer Statistics and Phase Transitions*, Journal of Mathematical Physics 4, 287 (1963). 
4. [Perfect Matchings and Pfaffian Orientation](https://esc.fnwi.uva.nl/thesis/centraal/files/f887198315.pdf).
5. L. G. Valiant, *The complexity of computing the permanent*, Theoret. Comput. Sci., 8(2), 189 (1979).
6. D. Cimasoni, *The geometry of dimer models*, arXiv: 1409.4631 (2014).
7. G. Tesler, *Matchings in graphs on non-orientable surfaces*, J. Combin. Theory Ser. B,
   78(2), 198 (2000).
8. A. Galluccio and M. Loebl, *On the theory of Pfaffian orientations. I. Perfect matchings and permanents*,  Electron. J. Combin., Research Paper 6, 18 (1999).
9. F. Barahona, *On the computational complexity of Ising spin glass models*,  J. Phys. A: Math. Gen 15, 3241 (1982).

EOF
