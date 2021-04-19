---
layout: post
title: "Quantum Chemistry in Second Quantization"
date: 2021-04-19
excerpt: "适合物理学家阅读的量子化学速查手册"
tags: [quantum computation, chemistry, physics]
comments: true
---


* toc
{:toc}


最近做分子问题的 VQE 看了些量子化学的内容。深感其大多数情形下语言之落后繁琐，以及和物理学科的代沟。因此本文用物理学家更熟悉的数值方法语境和二次量子化的语言，来简要概述量子化学的一些基本概念和数值方法，以使物理背景的读者更加快速简洁地了解量子化学的基本内容。

## 二次量子化

我们的出发点和固体物理相同，都是考虑了波恩奥本海默近似，将原子核背景固定的多电子薛定谔方程 (下式我们省略了无关的常数)。

$$
H = \sum_i \nabla^2_{r_i} +v(r_i) + \frac{1}{2}\sum_{r_i, r_j}\frac{1}{\vert r_i-r_j\vert}\label{Sh}
$$

为了引入二次量子化的语言来描述这一多体哈密顿量，我们需要找到一组希尔伯特空间（近似）的单体完备基。和固体物理唯一的区别是量子化学的问题中没有平移对称性，因此不用广延的平面波波函数做单体展开的基。相反我们用分子中原子固有的原子轨道来作为分子哈密顿量希尔伯特空间展开的基矢。这些轨道波函数来自于氢原子中心力场单电子薛定谔方程的解，但是考虑到二次量子化时，我们需要计算在这组基下边的交叠积分，为了可以简化积分计算，在量子化学计算中，通常是用若干个高斯波函数的权重叠加，来做各个原子轨道波函数的近似。也即（忽略归一化因子）

$$
\chi(r) \approx\sum_i c_i e^{-\alpha_i r^2}
$$

也就是说量子化学里的每一种基组名称，描述了两个事情：1.每个原子轨道是怎么被近似的 2. 整组基我们一共都选取了哪些原子轨道。比如 STO-3G，其告诉我们：1. 每个原子轨道都由三个高斯函数的叠加近似 2. 整组基只包括对应原子有填充的主量子数对应的原子轨道集合。关于不同基组对应的 2，也即轨道数目，可以参考讨论[^sobasis]。这里的高斯函数及其叠加权重，一般来讲都是提前和原子波函数对比拟合好的固定基组，当然这些参数也可以放开作为能量优化的变分参数，这是后话。有了单体基组后，我们就可以将$$\eqref{Sh}$$表达成二次量子化的形式：

$$
H = \sum_{ij} t_{ij}a^\dagger_ia_j+\sum_{ijmn}V_{ijmn} a^\dagger_ia^\dagger_j a_ma_n\label{ssh}
$$

其中 $$t_{ij}=\int\int \chi_i^{*(1)} \hat{H}_1\chi_j^{2} d\vec{r}^{1} d\vec{r}^{2} $$， $$V_{ijmn}=\iint \chi_{i}^{*(1)} \chi_{j}^{*(2)} \hat{H}_{2} \chi_{m}^{(2)} \chi_{s}^{(1)} d \vec{r}^{(1)} d \vec{r}^{(2)}$$。关于二次量子化的形式体系，和这里的系数作为交叠积分的推导，可以参考[^secondq]。这里费米子升降算符 $$a^\dagger_i$$ 和原子轨道波函数 $$\chi_i$$ 联系在一起。

## Hartree-Fock

所谓的 LCAO，就是用一组原子轨道的线性组合来近似分子轨道。在二次量子化的语言里，就是找到一个 unitary $$U$$，使得 新算符 $$\vec{c}=U\vec{a}$$ 下对应的 n 电子填充基态 $$\vert\Omega\rangle =\prod_i^n c_i^\dagger\vert 0\rangle$$，给出的能量期望 $$\langle\Omega\vert H\vert \Omega\rangle$$ 最小。变分优化能量期望以得到最优的矩阵 U，这就是 Hartree-Fock 方法。其特点和做的近似其实就是，假设分子系统的基态可以用一个 Slater 行列式 $$\prod_i^n c_i^\dagger\vert 0\rangle$$ 表达，而实际系统里的真实基态，需要多个 Slater 行列式态的叠加才可以表示。这里新的算符 $$c^\dagger$$ 对应的波函数，即是所谓的分子轨道波函数。其对应的变换矩阵 U 的每一列，物理意义则是所谓的轨道杂化。之后所有的更复杂的计算，将基于变换之后的，以分子轨道为基的哈密顿量 $$H(c^\dagger, c)$$ 讨论。这里的分子轨道，在不考虑自旋轨道耦合等相对论效应时，每个对应两个简并的自旋轨道。

HF 方法在二次量子化这一自动考虑了费米子反对称性的语境里，就是如此简单。但其在传统量子力学的语言中则比较复杂，在量子力学的语境中对应的 HF 方法论可以参考[^HF]。值得注意的是，对于变分数值方法，量子化学往往喜欢叫做自洽场方法。传统上其通过每次迭代来实现优化，并以某些量的自洽和收敛作为结束条件。实际上，所有这种自洽场方法，也可以用更加新的可微编程范式所实现，从而直接对大量变分参数进行梯度优化，理解和实现上都更加直接。已经有文献给出了既可以优化 LCAO 参数，又可以同时优化原子轨道近似的基中参数的可微 HF 方案 [^adhf]。这种自洽场方法，也可以出现在之后更复杂的方法里，SCF 实际上暗示着对应算法除了计算能量，还要在相应的框架里变分优化能量，进一步调整 LCAO，即分子轨道的波函数。

## Post Hartree-Fock

Hartree-Fock 用物理的语言理解，就是做了一个平均场，来近似能量$$\eqref{ssh}$$。量子化学计算的现代方法，则是从 Hartree-Fock 的平均场结果出发，也即从 $$c^\dagger$$ 定义的哈密顿量出发，来更好得到分子哈密顿量的能谱，尤其是其基态能量的信息。对于有四费米子相互作用的哈密顿量求解其基态，也是整个凝聚态物理的核心，因此其实量子化学和计算凝聚态物理的方法非常类似。我们对量子化学这些 post Hartree-Fock 的方法，用物理的语言做一简单列举，并给出其在计算物理中的对应。

### Møller–Plesset perturbation theory

该方法就是简单的量子力学微扰论。以 HF 多体基态出发，计算四费米子项的微扰贡献对分子基态能量的修正（本质上修正项是 $$H-H_0$$, $$H_0(a^\dagger, a)=a^\dagger (U^\dagger \Lambda_cU)a$$ 是优化过的给出 HF 基态和分子轨道能谱的含参哈密顿量）。该方案在微扰阶数低时，需要的计算资源较少。不同阶的微扰论分别记作 MP2，MP3，MP4 等等。

### Configuration interaction

现在我们考虑在多体 Fock space 来将二次量子化的哈密顿量对角化。所谓 configuration interaction 就是考虑真实的基态是在 HF 轨道下的不同直积态的叠加。当考虑了所有可能的形如 $$c_i^\dagger ... c_m...\vert \Omega\rangle$$ 填充和激发后，我们就可以得到分子基态的严格能量（忽略原子轨道基的不完备性后），这种方法其实就是物理中的严格对角化。当然其缺点是所需资源随着轨道自由度数目指数发散。

因此很多时候，我们需要限制在一些更重要的子空间来做对角化。其中只包含了单电子激发（singlet）和双电子激发（double）的方法可称为 CISD。截止到四个电子激发的空间可称为 CISDTQ。更进一步，我们可以只考虑较低能的可能态上的激发，这被称为对 active space 的限制，从而有效减少轨道自由度的数目。对于 complete active space，需要我们指定 core，active 和 virtual 三种分子轨道。其中 core 轨道一直全满，active 是可能填充的计算空间，而 virtual 轨道一直是空的。对于 n 个电子填充进 m 个分子轨道（2m 个自旋轨道）的解空间，通常记为 CAS(n, m)。投影掉 virtual 轨道，只需直接舍弃包含相应分子轨道算符的哈密顿量分量，而投影掉 core 轨道，则需考虑包含相应算符对角项给出的能量平移。二者在 openfermion 和 qiskit chemistry 中都有相应的 API 来约化量子比特空间。除了 complete active space，还有诸如 restricted active space 和 generalized active space 的概念，他们允许存在一些只能有特定激发的轨道对应的基。这些 active space 的对角化配合上分子轨道上的变分，组成了著名的 CASSCF，RASSCF 和 GASSCF 方法。

以上我们讨论的空间，都是从 HF 或自洽分子轨道基态填充上完成的激发。我们可以同时取多个多体波函数作为待激发的构型来构造基，这被称为 multireference 方法，也即 MRCI。关于 multiconfigurational 和 multireference 的一个讨论，可以参见[^somm]。

### Coupled cluster

CC 类的方法是指对于变分波函数 $$e^{\theta_{ij} c^\dagger c + \theta_{ijmn}c^\dagger...c...+...}\vert \Omega\rangle$$ 做哈密顿量期望的变分优化。相应的只考虑到四费米子算符的情形，被称为 CCSD。VQE 中最著名的 UCC ansatz 就来自于该方法的 unitary 化 （$$e^T\rightarrow  e^{T-T^\dagger}$$）。数值时间上，需要对该指数算符做泰勒展开和截断来近似。

### DMRG & QMC

对于这里的费米子二次量子化哈密顿量$$\eqref{ssh}$$，我们总是可以通过 Jordan-Wigner 或 Bravi-Kitaev 变换，将其转化为泡利矩阵的量子比特自由度，这一方面是 VQE 的理论基础，另一方面也可以利用一些凝聚态物理中的先进数值方法来进行哈密顿量基态优化。DMRG 就是这其中的典型代表。传统 DMRG 中的实空间自由度被量子化学中的分子轨道自由度替代。关于 DMRG 在量子化学中的应用和其可能的独有困难（比如长程相互作用和数量极大的四费米子项），可以参考[^dmrgtalk]。

除此之外，多体物理中常用的量子蒙卡方案也可以用来计算分子哈密顿量的基态。

### DFT

密度泛函理论方法和物理中的也没太多区别，我们选取电子密度而非电子波函数作为优化的参数。唯一不同的还是展开基是原子轨道而不是平面波基组。DFT 由于对交换作用的泛函做了近似，本质上问题约化到了单体问题，因此可以计算较大的系统，当然在量子化学问题中，可能精度没有上述的方法有优势。


最后，关于这一计算量子化学方向，比较好的一个系列讲解可以参考[^youtube]，不过这一讲解是基于一次量子化语言的。

## References

[^secondq]: [Second Quantization Note](https://www.tkm.kit.edu/downloads/ss2016_tkm2/second_quantization.pdf)

[^HF]: [Note on Hartree-Fock and DFT methods](https://www.itp.tu-berlin.de/fileadmin/a3233/upload/SS12/TheoFest2012/Kapitel/Chapter_3.pdf)

[^adhf]: [Automatic Differentiation in Quantum Chemistry with Applications to Fully Variational Hartree–Fock ](https://pubs.acs.org/doi/10.1021/acscentsci.7b00586)

[^somm]: [What exactly is meant by 'multi-configurational' and 'multireference'?](https://chemistry.stackexchange.com/questions/103387/what-exactly-is-meant-by-multi-configurational-and-multireference)

[^youtube]: [Computational Chemistry Lectures on Youtube](https://www.youtube.com/playlist?list=PLkNVwyLvX_TFBLHCvApmvafqqQUHb6JwF)

[^sobasis]: [How many basis functions used in STO-3G and 6-31+G\*\* for the water molecule?](https://chemistry.stackexchange.com/questions/41163/how-many-basis-functions-used-in-sto-3g-and-6-31g-for-the-water-molecule)

[^dmrgtalk]: [Density matrix renormalization group with long-range interactions and its applications range interactions and its applications](http://cat.sxu.edu.cn/docs/2012-08/20120823233233250891.pdf)

