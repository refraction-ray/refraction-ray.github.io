---
layout: post
title: "cubic graph 上的 NP 完全问题"
date: 2018-10-09
excerpt: "关于 cubic graph 上两个问题 NP 完全性的讨论和独立证明"
tags: [graph theory, complexity theory]
comments: true
---


* toc
{:toc}
## 引言

这篇算是**物理学中的计算理论**正式的序章，我将从 3SAT 出发，给出一个约化到 cubic graph 上的 MAX CUT 问题的证明。本来这篇是要讲两个NP完全问题的证明，以用来证明物理系统中的 NP 困难问题。但第二个问题：Maximum independent set in a planar cubic graph 的完整证明，我并没有找到，因为所有的参考文献都指向了一个找不到的引用：Maire, D., and Storer, J. A. 1977, *A note on the complexity of the superstring problem*, Tech Report N. 233, Princeton University. 而且我不是一个人，[2] 引用这篇时，还特意加上了句 as cited in [3]，说明 [2] 作者也没找到这篇文献。不过 [4] 有一个该问题弱化版的证明，给出了 at most cubic graph 和 planar at most degree-six graph 的肯定结果，我们在证明我们的主要结论，第一个NP完全问题后，会稍微跑个题证一下第二个问题的弱化版。

另外关于 MAX CUT 在 cubic graph 上的证明，虽然 [1] 号称证明了，但没给出任何证明的细节，甚至框架都不怎么有，因此本文完全是我自己参考 MAX CUT 在一般图上的证明做的改造 (参考 [5] 的习题 7.24，7.25)，[1] 的 Lemma 5 也给了我一些灵感。当然证明和程序一样，很有可能有 bug 甚至完全是错误的，鉴于是我自己思考的证明，错误的可能性更大。如果读者发现了证明中的漏洞，希望可以指出。

最后，引言没怎么看懂没关系，从下面的形式定义开始读起即可，虽然也可能不太好看懂。

## 一些定义

### 复杂度

关于复杂度 NP，和 NP-Complete 的定义，以及NP完全证明的逻辑，这里不再赘述。至于3SAT是NP完全问题，可以参考任何一本复杂度入门教材关于 COOK-LEVIN THEOREM 的证明，比如 [5] 的 THEOREM 7.37，COROLLARY 7.42。关于 3SAT 的内涵，还需要强调一下。一般地，我们要求3SAT是一些分句(clause)的 AND 构成，而每个分句由3个变量(variblae)的 OR 构成。这也是3SAT里3的来源。但为了后文需要，我们把每个分句的变量数目放宽到2到3个，我们将这种问题记为2-3SAT，以与3SAT区别，注意到某些文献中，3SAT默认就是2-3SAT。显然2-3SAT也是NP完全问题。

之所以做出这种仔细的区分，是因为两者有个本质的不同。当我们限制每个变量在SAT逻辑表达式中只出现3次时（$$x,\bar x$$出现总次数），对于3SAT，satisfying assignment 变得显然。这源于[Hall's marriage theorem](https://en.wikipedia.org/wiki/Hall%27s_marriage_theorem)，具体的构造证明可以参考 [6]。而对于2-3SAT，问题仍处在NP完全，具体证明下面会讲到。

### 图论

关于图G=(V,E)，我们都采用最一般接受的定义方式。比如度数(degree)表示从该顶点出发的边的数目。如果所有顶点的度数都为3，这样的图称为 cubic graph。所谓 planar 图，是指给定图可以以一种边不交叉的方式嵌入到平面上。

所谓 cut，是指从图中选取一些顶点 $$S\subset V$$，对应的边的集合 $$CUT=\{e\vert e\in E, e[0]\in S, e[1]\in V-S\}$$，该集合的元素数被称为 cut 的大小。用人话说，就是选取一些顶点，那么连接这些顶点和其他顶点的边就叫做一个 cut，直观上这些边被理解为跨越了选取区域的边界。我们这里只考虑这种边的数目，而不考虑边的总权重，这也被称为 simple max cut。对应的 MAX CUT 判别问题为：

> 给定图 G，整数 k，是否存在顶点集合 S，使得对应的 cut 边数大于等于 k。

容易证明，该问题对于一般图是NP完全（[7] 给出了一个权重版本的 MAX CUT NP完全证明，很容易稍加改造来证明这里的 simple max cut）。由于我们本文最后可以证明出该问题对于 cubic 图都是NP完全，那么很容易归约到一般图，因此不再重复一般图 NP 完全的证明。

对于NP完全的问题，我们总是想尝试把问题限制的更小，以判断对应问题是否还是NP完全问题。比如对于一般图， MAX CUT 是NP完全的，那对于顶点度数均为 m 的图，该问题是否还是NP完全呢。毫无疑问，当 m＝2 时，该问题变得平庸，整个图退化成一个环，因此不在NP完全。因此第一个需要分析的非平庸结果，就是m=3。这也将是本文的主要工作：证明 cubic 图上的 MAX CUT 是NP完全的。

继续之前的另一个提醒，maximum cut 和 maximal cut 完全不是一个问题，后者是P，MAX CUT 指的是前者，要小心踩中这些莫名其妙的坑。

此外，给出 maximum independent set 问题的定义。对于给定图$$G=(V,E)$$，所谓 independent set 是指一组顶点 $$S\subset V$$，使得 $$\forall e\in E, e[0]\notin S \vee e[1]\notin S$$。说人话就是，找到一组顶点，使得没有原图的边的两个端点都在这组顶点内。很容易看出 independent set 是 vertex cover 的补集。后者的人话定义是：找到的一组顶点，使得每个边都至少有一个端点在这组顶点内。对应的 maximum independent set 问题是指：

> 给定图 G，整数 k，是否存在 independent set，使得其中的顶点数大于等于 k。

该问题对于一般图是NP完全，证明可以参考 [8]。同样的我们也可以继续限制图 G 的复杂情况，来分析对应问题是否仍在NP完全。这方面最强的结果引言部分已经叙述了，对于planar cubic graph，该问题依旧NP完全。但证明暂时不详。其弱化版本 cubic graph 的结果我也将在本文给出。

以上这两个NP完全问题，将是之后处理物理系统问题NP困难证明的基础。

## 3SAT 到 NAE 3 SAT

从现在开始，我将通过一步步的多项式时间归约，来将3SAT问题转化为 MAX CUT in cubic graphs 的问题，从而表明该问题也是NP完全。

所谓 nae 3SAT，是指一个满足条件的 assignment 不仅要使得表达式为真，还需要对于SAT表达式中的每个分句，为真的变量不超过2个。也就是说nae 3SAT是在3SAT上加上了，每个分句条件不能都为真的额外限制。

现在我们构造从3SAT问题到nae 3SAT的映射。将3SAT的每一个分句$$(y_1\vee y_2\vee y_3)$$，转化为nae 3SAT的两个分句$$(y_1\vee y_2\vee z_i)\wedge(\bar z_i\vee y_3\vee b)$$。其中$$z_i$$是对应原来每个分句都有一个新变量，而$$b$$则是全局变量。这一转换很明显多项式时间就可以做到。

下面证明，若3SAT为真则nae 3SAT为真。若3SAT为真，要么$$y_1,y_2$$中一个为真，此时令$$z_i=0$$，若$$y_3$$为真，则令$$z_i=1$$，令$$b=0$$。如此既保证了每个nae 3SAT的分句为真，也保证了没有一个分句的三个变量全为真。

再证，若nae 3SAT为真则3SAT为真。若nae 3SAT为真，对应 assignment 中若$$b=1$$，则取该 assigment 的否定（显然对于nae 3SAT，解 assignment 的否定也是一个解）。该解中每两个分句，选取 z 变量为0的一个分句，其中必存在一个 y=1，满足原 SAT，证毕。

由此可得 nae 3SAT 是NP完全问题。

## NAE 3SAT 到 NAE 2-3SAT

现在我们将刚刚归约到的nae 3SAT，进一步归约到nae 2-3SAT。如果没有其他要求，这自然是平凡的。但我们要求得到的nae 2-3SAT中，每个变量出现的总次数恰好是3次，同时依旧每个分句变量都不全对。我们将证明，这依然能保证NP完全。

构造该映射，对于原 nae 3SAT中的变量，若该变量只出现过一次，则该变量可以无约束的设为真，对应的分句平凡，可以消去，该变量消失。若该变量出现两次以上，做如下操作：假设变量x出现了n次，我们引入n个新变量$$x_i​$$，依次替换原SAT中每次出现的x，同时补加如下分句$$(x_1\vee \bar x_2)\wedge (x_2\vee \bar x_3)…(x_{n-1}\vee\bar x_{n})\wedge(x_n\vee \bar x_1)​$$。由此，每个变量前边分句出现一次，补加分句会出现两次，恰好出现三次。这一归约，独立变量数恰好变为原变量数的三倍，分句数也比以前多了3倍，因此仍只需多项式时间。

下面证明，若nae 3SAT为真，变量均出现3次的nae 2-3SAT也为真。对应一组assignment，我们将$$x_i$$均设为和x的真值相同，显然nae 2-3SAT为真，且每个分句的变量都不全对。

反之，若nae 2-3SAT为真，由于后续的两变量分句的约束，所有的$$x_i$$只能真值相同。将这样一组assigment，可以无歧义的替换为原来3SAT变量的一组assignment，显然这组赋值使得原nae 3SAT成立。

由此可得，每个变量恰好出现3次，每个分句变量都不全对，分句可包含2到3个变量的 nae 2-3SAT，也即按上述构造形成的 SAT 是NP完全问题。

## NAE 2-3SAT 到 MAX CUT on graph with at most degree 3

得到上面的SAT表达形式后，我们开始尝试构造从该表达式到图的映射。具体的图由如下几部分组成。我们将nae 3-SAT 表达式的分句数记为$$c_1$$，根据上节的构造过程，易知新添加的2变量分句总数为$$c_2=3c_1$$，对应的新2-3SAT表达式中的变量总数为$$m=3c_1$$。

$$V$$ = {上述表达式出现的所有变量（每次出现就计作一个顶点）}，$$\vert V\vert ＝9c_1$$。

$$E_1$$ = {对于每一个变量的三个顶点，连接形如$$x \bar x$$这样真值互斥的边}，$$\vert E_1\vert =6c_1$$。

$$E_2$$ = {对于所有的2变量分句对应的边进行连接}，$$\vert E_2\vert=3c_1$$。

$$E_3$$ = {对于每个3变量分句出现的3个变量的相互三条边都连接}，$$\vert E_3\vert = 3c_1$$。

该映射将表达式转化为图 $$G=(V,E_1+E_2+E_3)$$。根据构造易知该图的顶点度数最多为3。该转化显然可在多项式时间完成。

该图的精神是每个nae 3SAT中的原始变量，经过nae 2-3SAT和图的两手转化，形成了一个大环，这个大环上的边正是来自2变量分句对于新变量真值相同的锁定环，以及$$E_1$$要求的变量自身互斥真值顶点连接的边所交替构成的。环上每两个顶点，就有一个伸展出一边，对应的伸展出的端点，就是该新变量在前部3变量表达式出现的部分。而该伸展边则来自$$E_1$$。因此同一老变量大环上的顶点，有一半度数为2，这也是该图中唯一度数不为3的一族顶点。那些伸展出的顶点，恰好按照$$E_3$$，组成一些三角形。这就是整个图构造的结构概况，高视角来看就是一些大环被三角连接，恰好这些高视角的结构都是原始3SAT中的内容。

现在我们要证明指定的nae 2-3SAT为真等价于该图存在cut边数大于等于$$11c_1$$。

当表达式为真，我们选出赋值为1的变量对应的顶点，此时对应的 cut 恰好为$$11c_1$$。其中$$6c_1$$来自$$E_1$$，每个变量互斥边都在cut中。$$3c_1$$来自$$E_2$$，所有的2变量分句，都只能有一个变量为真，因此这些边都在cut上，连接了真值不同的顶点。还有$$2c_1$$的边来自$$E_3$$，由于每个3变量分句，都要求不全为真，因此总是有两个边处在cut上。

当该图存在cut边数等于$$11c_1$$时，注意到该图中的存在$$c_1$$个三角形（来自于$$E_3$$），而三角形无论如何取顶点，最多有两个边能在 cut 上，因此总共有$$12c_1$$条边的该图，可能存在的cut上限就是$$11c_1$$。也即每个三角形两个边在cut上，其他所有边都在cut上。如果能找到这样一族顶点，实现该cut边情况，可以证明该组顶点对应的变量赋值为真即可。当然这里要格外注意，顶点选取的一致性，防止出现形如对应$$x,\bar x$$的顶点可以同时被选取的情况。$$E_1$$保证了这一点。要想使$$E_1$$的边都在cut上，对于每个变量，那么要么选取所有的$$x$$顶点，要么选取所有的$$\bar x$$顶点。

由此我们证明了 MAX CUT 对于 degree at most 3 的图依旧NP完全。

## at most cubic graph 到 cubic graph

最后我们需要将顶点度数不大于3的图上的 MAX CUT 问题归约到顶点度数全部为3的图上去。

这里我们借助以下的图结构。

<img width="30%" src="https://user-images.githubusercontent.com/35157286/46659852-52fd8e00-cbe8-11e8-9162-f30c5384ef10.png">

考虑上图顶点3是本来度数为2的顶点，为了改造，我们在3上补一条边(31)使得顶点3的度数变为3，同时再补充图示的图结构，那么补充之后整张图变为 cubic graph。事实上对于本来度数为1的顶点，可以将其视为上图的1，也可以补充类似的从1出发的图结构，来使得全图变为 cubic graph。但我们上节构造的图中，并没有度数为1的顶点，因此不再考虑。

现在我们来证明，如果原来图存在不小于$$11c_1$$的cut等价于新的 cubic graph 存在不小于$$32c_1$$的cut。

若原图存在$$11c_1$$的cut，那么新的图中，顶点取法相同，同时对于每一个新添的图结构，若顶点3在原图被选取，则新图中同时选取顶点0和顶点4，反之亦然。这样对于原来存在的$$3c_1$$个度数为2的顶点，新图上每个又多出了7条cut边，证毕。

反之，若新图存在$$32c_1$$的cut，对于该图的$$3c_1$$个上述结构，由于三角形的存在，每个结构都至少有一条边不能在 cut。再加上以前的$$E_3$$中的三角形，共有$$4c_1$$条边无法进入cut。而新图的总边数为$$36c_1$$，那么理论上最大cut就是$$32c_1$$。其唯一取法就是取到所有边，除了三角形必舍掉一边以外。那么这样一种取法，盖住这些新增的结构，剩余的取法对于原图还是适用，恰好为$$11c_1$$的cut。证毕。

综合以上全部内容，可以得出 MAX CUT in cubic graph 是NP完全问题。

## 关于 independent set 问题的补充

虽然我暂时没想到，也没找到 max independent set 在 cubic planar graph 上是NP完全的证明。但还是延续上述证明 MAX CUT 的思路和方法，证明一些更弱一些的结论，作为补充。

证明的起点是从每个分句不全对的nae 2-3SAT开始（事实上，这里我们不需要 not all equal 的限制即可完成证明）。尝试用这种结构的表达式构造等价的图。事实上，该图的构造和证明 MAX CUT 时所用的图完全一样。（神奇的是，我们使用 nae 2-3SAT，仿照一般的 independent set 证明的图构造，见 [8]，但这一构造由于使用了nae 2-3SAT 自动退化为了 [1] 中证明所用的构造，而此构造又恰好和前面我们所叙述的 MAX CUT 的图构造一致，这反应了nae 2-3SAT 方法的普适性）。

下面来证明nae 2-3SAT可以为真等价于该图存在 independent set 元素数大于等于$$4c_1$$。

若存在一组变量的 assigment 使得表达式为真，那么将对应为真的顶点选为 independent set。注意若一个分句中，有两个变量为真，那么随机选一个放入 independent set。根据边的构成，可知这些顶点不会有边相连，而顶点的总数目为$$4c_i$$。

反之，注意到该图中的环结构和三角形，使得该图理论上的最大 independent set 就是 $$4c_i$$，如果取到了该值，则说明恰好所有的环中有一半交替顶点和所有三角形中各有一个顶点在该 set 中。如果我们将 set 中的顶点代表的变量赋值为真，可以根据边的定义验证其相容性。因此每个三角形，也即分句中，都有一个变量为真，原3SAT表达式满足。证毕

事实上，上面证明，不只是图的构造，连选取的顶点集合都和 MAX CUT 情形一样。这应该理解为 nae 2-3SAT 对应图的特殊性，一般图上 MAX CUT 对应的顶点集合和 max independent set 对应的集合不一定相同。

而对于 max independent set 在 cubic 图上的证明，也可以采用相同的构件添加方案。此时无论原图中度数为2的点是否在 set 中，新加的部分，每个都只能在 set 中增加两个点。也即原图存在不小于$$4c_i$$的 independent set 等价于新的 cubic graph 存在不小于 $$10c_i$$ 的 set。

由此我们完成了 max independent set 在 cubic 图上依旧NP完全的证明。但事实上，后面的物理部分需要更强的结论，也即 cubic planar 图上的NP完全。我能找到的证明中最强的结论则是 [4] 中的 Theorem 2.7，作者证明了顶点至多度数为6的 planar graph 上，max independent set 问题NP完全。

*更新*：(以下内容的正确性更加不确定，只是我现时的一个理解)

考虑到 planar 3SAT 也在NP完全之中 (见 [9]，[10])，那么上面证明时用到的 cubic 图自然是 planar graph。由此最初证明未找到的 cubic planar graph 上的 max independent set 是NP完全得证。

进一步的提醒：注意到 MAX CUT 问题在 planar graph 上可以转化到最大匹配问题，从而是在 P (见 [11])，两问题在平面图上复杂度不同，而证明高度相似可能的解释是，只有3SAT的平面图版本是NP完全的，而 nae 3SAT 平面图版本不是。原因在于，MAX CUT 证明严格依赖于3变量分句的变量不能都为真，而 independent set 的分析中对分句变量是否都真不敏感。这一理解很可能是对的，因为我又找到了 [12]。其答案指出

> PLANAR NAE k-SAT is in P for all values of k.

关于 nae SAT 的平面图版本复杂度分析，可以进一步参照 [13]。

## Reference

1. Yannakakis, *Node- and Edge-Deletion NP-Complete Problems* in Proc. 10th Annual ACM Symp. on Theory of Computing (1978).
2. R. Greenlaw, and R. Petreschi, *Cubic graphs*, ACM Comput. Surveys, 27 (1995), pp 471-495.
3. M. R. Garey, and D. S. Johnson, *Computers and Intractiability: A Guide to the Theory of NP-Completeness*, W.H. Freeman and Company, San Francisco (1979).
4. M. R. Garey, D. S. Johnson, and L. Stockmeyer, *Some simplified NP-Complete Graph Problems*, Theoretical Computer Science (1976), 237-267. 
5. M. Sipser, *Introduction to the Theory of Computation*, 2ed.
6. [How to prove that 3-CNF is satisfiable using Hall's marriage theorem?](https://math.stackexchange.com/questions/2606499/how-to-prove-that-3-cnf-is-satisfiable-using-halls-marriage-theorem)
7. [Reduction from 3SAT to MAX CUT](http://www.cs.cornell.edu/courses/cs4820/2014sp/notes/reduction-maxcut.pdf).
8. [NP-Completeness of Independent Set](https://www.nitt.edu/home/academics/departments/cse/faculty/kvi/NPC%20INDEPENDENT%20SET-CLIQUE-VERTEX%20COVER.pdf).
9. David Lichtenstein, *Planar Formulae and Their Uses*, SIAM J. Comput., 11(2), 329-343 (1982).
10. [Lecture note on planar 3SAT](http://courses.csail.mit.edu/6.890/fall14/scribe/lec7.pdf).
11. F. Hadlock, *Finding a Maximum Cut of a Planar Graph in Polynomial Time*, SIAM  J. Comput. , 4(3), 221-225 (1975).
12. [For which k is PLANAR NAE k-SAT in P?](https://cstheory.stackexchange.com/questions/5994/for-which-k-is-planar-nae-k-sat-in-p)
13.  B. Moret, *Planar NAE3SAT is in P*, ACM SIGACT News, 19, 2  (1988).


EOF