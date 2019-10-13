---
layout: post
title: "Reduce VI to VAE"
date: 2019-10-13
excerpt: "VAE 在传统 VI 框架的什么位置，以及其看似自然的不平庸之处"
tags: [machine learning, statistics]
comments: true
---

* toc
{:toc}


## 引言

在之前的[文章](/VAE的逻辑与实践/)里，我从统计推断的视角出发，简要讨论了 VAE 网络结构的由来。但是如何从 general VI 的形式体系，无缝地延拓到 VAE 的形式体系，事实上并没有仔细说明。最近重新查找了一些资料，想不到最后使我想通这件事的不是哪篇 paper，而是 Pyro 的文档[^pyro]，特别是其文档的 SVI 的部分。这篇内容，我将仔细说明一个 general VI formalism 过渡到 VAE 的过程。本文需要读者有一定的统计推断和机器学习的知识，并且了解 VAE。本文不是 VAE 教学，而是解释 VAE 究竟为什么及如何完全内嵌在大的 VI 框架之中。

## Variational Inference

假设我们有一些观测数据，记为 $$\mathbf{x}= (\mathbf{x}_1, …\mathbf{x}_n)$$（注意其中每个数据点都可以是高维向量，整个 x 可以理解为一个矩阵）。从统计角度来看所谓的生成模型，也即找到模型可以 $$\max_\theta P_\theta(\mathbf{x}) $$。当我们假设各个数据点是统计独立的时候，我们的任务是一个优化问题 $$\theta^*=\mathrm{argmax}_\theta \sum_i \log P_\theta(\mathbf{x}_i)$$。通常我们会利用一些隐变量 $$\mathbf{z}$$，也即 

$$
P_\theta(\mathbf{x})=\sum_{\mathbf{z}}P_\theta(\mathbf{x}\vert\mathbf{z})P(\mathbf{z}).
\label{bay}
$$

这样整个问题就落在了 Bayes Inference 的范式里。其中 prior $$P(\mathbf{z})$$ 只需假设为某个简单的分布即可。因为理论上只要数据足够多，先验不影响后验。或者在 $$\eqref{bay}$$ 的语境里，不管先验多离谱，一个足够复杂的 likelihood 还是可以尽量最大化 $$P_\theta(\mathbf{x})$$。当然事实上，先验的选取在不同的问题里还是有些讲究的，包括考虑到 uninformative 或者 conjugate distribution 的性质等等，关于 prior 的细节可以参考我之前关于贝叶斯哲学的[文章](/贝叶斯推断小议/)。如果对于先验特别纠结的话，在先验上也加上 $$\theta$$ 依赖也是可以的。

注意到 $$\eqref{bay}$$ 几乎是没有办法直接求的，因为 trace 掉的 z 的空间是指数大的，也即该式子 intractable。求 $$\theta^*$$ 来最大化 $$P(\mathbf{x})$$，我们称为**第一类问题**。

让我们暂时考虑另一个问题，假设我们找到了 $$\theta^*$$，或者模型 $$\theta$$ 依赖根本不存在。后者情况出现在，可以按照统计模型的事实直接写出 likelihood 的情况，比如扔硬币问题。n 次的结果 1，0 构成了 $$\mathbf{x}$$，硬币的正面朝上概率对应隐变量 $$\mathbf{z}$$ （一维向量），那么我们可以直接写下 $$z\sim Beta(1,1)$$，$$x\vert z\sim B(n,z)$$。这种根据事实和有明确物理意义的隐变量写下的概率模型，具有确定的 prior 和 likelihood，并不需要额外引入参数做优化。此时我们如果想求后验的话有：


$$
P_{\theta^*}(\mathbf{z}\vert\mathbf{x})=\frac{P_{\theta^*}(\mathbf{x}\vert\mathbf{z})P_{\theta^*}(\mathbf{z})}{\sum_{\mathbf{z}}P_{\theta^*}(\mathbf{x}\vert\mathbf{z})P_{\theta^*}(\mathbf{z})}.
\label{post}
$$

注意到 $$\eqref{post}$$ 的分母也是 intractable，原因同理，我们无法在多项式资源里遍历整个隐变量空间。我们把已知确定的先验 $$P_{\theta^*}(\mathbf{z})$$ 和可能性 $$P_{\theta^*}(\mathbf{x}\vert\mathbf{z})$$，求后验 $$P_\theta^*(\mathbf{z}\vert \mathbf{x})$$ 的问题称为**第二类问题**。

解决第二类问题，有两种思路。关键就是当我们讨论求出 $$\mathbf{z}\vert\mathbf{x}$$ 的概率分布时，我们在讨论什么。明显有两种方式可以表达一个概率分布，第一个就是一大堆在这个概率分布下生成的数据点，当这些点足够多时，我们通过点的密度可以对分布的情况有很好的了解，也可以轻松的通过这些数据点来求分布本身的期望和方差等特征。而得到这些符合后验分布的数据点，是很简单的，通过 MCMC 即可。因为 MCMC 利用了 Metropolis 算法，所以实际上只需要利用两个不同点的概率比，而不需要归一化因子（$$\eqref{post}$$ 的分母）。也就是说，直接按照 $$\eqref{post}$$ 式分子跑 MCMC 即可。当然 MCMC 有很多高效的方案，比如 Hamiltonian Monte Carlo[^hmc] 或者其更高级的自适应版本 NUTS[^nuts] 等，这里就不赘述了。

另一种思路，就是先假设一个后验的概率分布函数形式 $$q_\phi(\mathrm{z})$$，这样我们需要做的就是使该具有变分参数 $$\phi$$ 的概率分布尽量接近真实的后验。这就是统计推断（VI）。需要注意 VI 的方案往往更快，但是由于需要概率分布 ansatz，所以是 biased 方法。既然目标是使两个分布相似，这自然使我们想起了 KL 散度，在 VI 的语境中，最常用的是 reverse KL 散度（关于 forward KL divergence 和 reverse KL divergence 的区别和讨论，可以参考[^kl]），也就是前边是假设分布，后边是真实分布。我们希望优化的目标函数即为：


$$
KL(q_\phi (\mathbf{z})\vert P_{\theta^*}(\mathbf{z}\vert\mathbf{x}))=\sum_{\mathbf{z}}q_\phi(\mathbf{z})(\ln q_\phi(\mathbf{z})-\ln P_{\theta^*}(\mathbf{z}\vert\mathbf{x})).
\label{kll}
$$

但是很明显，我们并不知道 $$P(z\vert x)$$，不然也不用假设 q 了。因此我们需要利用贝叶斯公式，将 $$\eqref{kll}$$ 转化为


$$
KL(q_\phi (\mathbf{z})\vert P_{\theta^*}(\mathbf{z}\vert\mathbf{x}))=E_{q_\phi}[\ln q_\phi(\mathbf{z})]-E_{q_\phi}[\ln P_{\theta^*}(\mathbf{x},\mathbf{z})]+P_{\theta^*}(\mathbf{x}).
\label{klt}
$$

注意到最后一项与 $$\mathbf{z}$$ 无关，因此被提出了期望以外。我们想做的是优化参数 $$\phi$$，来找一个最优的近似后验，也即 $$\phi^*=\mathrm{argmax}_\phi (q_\phi (\mathbf{z})\vert P_{\theta^*}(\mathbf{z}\vert\mathbf{x}))$$。考虑到 $$P_{\theta^*}(\mathrm{x})$$ 不含 $$\phi$$，因此虽然我们不会求 $$\eqref{klt}$$ 右边的第三项，但我们实际上也不需要求。至此，通过 VI 的方式，第二类问题被解决。我们要做的，只是通过机器学习的框架工具（自动微分，随机梯度下降优化器，mini-batch 等），来找到 $$\phi$$ 即可。当然这里面还有一个，蒙卡期望值如何求导的问题[^mcgradient]，这个问题是一个很大的话题，本文不讨论。（这里的优化想法基本上来自2010年代后，深化学习流行之后的想法，相关工作包括 SVI[^svi]，BBVI[^bbvi]，ADVI[^advi]，VAE[^vae] 等等。这之前的优化基本上是用了平均场假设的后验加上传统的半解析变分，即所谓的 coordinate-ascent optimization[^mfvi][^vireview]。）

现在问题来了，我们解决第二类问题的路线图，建立在解决第一类问题，得到了最优的 $$\theta^*$$ 的基础上。然而我们并没有通用的方案直接解决第一类问题。因此我们干脆把 $$\eqref{klt}$$ 里的常量 $$\theta^*$$ 改为变量 $$\theta$$。由此我们有


$$
ELBO=E_{q_\phi}[\ln P_\theta(\mathbf{x},\mathbf{z})-\ln q_\phi(\mathbf{z})]= \ln P_\theta (\mathbf{x})-KL(q_\phi (\mathbf{z})\vert P_{\theta}(\mathbf{z}\vert\mathbf{x})).
\label{ELBO}
$$

如果我们最大化这个叫 ELBO 的东西，我们似乎同时在最大化 P(x) 并且最小化 KL 散度，这就实现了一石二鸟的效果（当然也可能都没做到极致，这是不得已的牺牲）。同时优化参数 $$\theta,\phi$$，我们相当于同时解决了第一类和第二类问题。

## VAE

假设我们的问题中观测点 $$\mathbf{x}$$ 有 N 个，每个 D 维；我们假设该问题的隐变量也是 N 个，每个 d 维。一个完整的概率模型的假设，包括了隐变量的个数，隐变量的先验分布和基于隐变量的条件概率（可能性）。在这里我们假设 $$P(\mathbf z)=\Pi_i P(\mathbf{z}_i)$$，也即隐变量每 d 个值相互独立。且 $$\mathbf{z}_i\sim N(\mathbf{0}, I_{d\times d})$$，即每 d 个分量满足标准分布。条件概率部分，我们则假设以下的相互独立性：


$$
P_\theta(\mathbf{x}\vert\mathbf{z})=\prod_i P_{\theta_i}(\mathbf{x}_i\vert \mathbf{z}_i).
$$

根据贝叶斯公式有


$$
P_\theta(\mathbf{z}\vert\mathbf{x})=\frac{\prod_i P_\theta(\mathbf{x}_i\vert \mathbf{z}_i) P(\mathbf{z}_i)}{\sum_z\prod_iP_\theta(\mathbf{x}_i\vert \mathbf{z}_i) P(\mathbf{z}_i)}\\
=\prod_i \frac{P_\theta(\mathbf{x}_i\vert \mathbf{z}_i) P(\mathbf{z}_i)}{\sum_z P_\theta(\mathbf{x}_i\vert \mathbf{z}_i) P(\mathbf{z}_i)}=\prod_i P_\theta(\mathbf{z}_i\vert \mathbf{x}_i).
$$


也即可能性的独立性，保证了后验的独立性。由此我们假设的后验分布形式为


$$
q_\phi(\mathbf{z})=\prod_i q_{\phi_i}(\mathbf{z}_i).
$$

当数据点很多，也即 N 很大时，待优化参数 $$\phi_i, \theta_i$$ 都 scale with N，这是不可接受的。因此我们利用 armotized VI，也即选择拟合函数 f，g 使得 $$\phi_i\approx f(\mathbf{x}_i)$$， $$\theta_i\approx g(\mathbf{z}_i)$$。两个函数，当然可以用两个神经网络来作为通用函数拟合器，这样我们需要优化的参数从 N 的量级就降低到了 D 的量级（取决于神经网络的细节）。当后验假设 $$q( \mathbf{z}_i)$$ 也取由 $$\phi_i$$ 控制期望和方差的高斯分布时，整个问题就完全 reduce 到了 vanilla VAE。由此我们完成了从 VI 问题到 VAE 的联系的推导，严格表明了 VAE 只是先验和可能性满足一定独立性假设下的 VI 的特例。这其中， VI 能做是因为改为优化 ELBO，实际上牺牲了严格最大化 P(x) 或者严格使得后验分布和假设分布 q 相似。而在此基础上，VAE 利用了 armotized VI，又进一步牺牲了结果的严格性。

### mini-batch

最后我们来看一下，什么情况下，可以通过 mini-batch 的方式优化 ELBO，由此说明为什么 VAE 可以进行 mini-batch update（这不是 trivial 的）。观察 $$\eqref{ELBO}$$，

$$
ELBO=E_{q_\phi}[\ln P_\theta(\mathbf{x}\vert\mathbf{z})+\ln P_\theta(\mathbf{z})-\ln q_\phi(\mathbf{z})],
$$

这里我们暂时只考虑对 $$\mathbf{x}$$ 做 mini-batch，可以看到只有 $$P_\theta(\mathbf{x}\vert\mathbf{z})=\prod P_\theta(\mathbf{x}_i\vert\mathbf{z})$$，才能保证 mini-batch 更新有定义。也即不同的 x 样本，需要是被隐变量独立产生的。

此时的 $$\mathbf{z}$$ 还做不了 mini-batch，假设 $$\mathbf{z}$$ 包含的隐变量也是 N 的量级，我们当然也希望 z 也可以 mini-batch 来 evaluate  ELBO。如果 z 的先验，和后验假设都是关于不同 $$z_i$$ 独立的，则我们回到了 VAE 的无脑的 mini-batch 模式。如果我们假设只有 N/2 个 $$\mathbf{z}_i$$ 且有 $$P(\mathbf{x},\mathbf{z})=\prod_i^{N/2}P(x_i\vert z_i)P(x_{2i}\vert z_i)P(z_i)$$。此时 ELBO 可以表示为

$$
ELBO=\frac{1}{N_s}\sum_i^{N/2}\sum_{z_i\in q_\phi(z_i)}^{N_s} \ln P(x_i\vert z_i)+\ln P(x_{2i}\vert z_i)+\ln P(z_i) - \ln q_{\phi_i}(z_i),
$$

此时，选取一个 $$x_i$$ 对应一个 $$z_i$$ sample，其需要 evaluate 的 loss 实际上是

$$
ELBO(x_i,z_i)\propto \ln P(x_i\vert z_i)+1/2 \ln P(z_i)-1/2 \ln q_{\phi_i}(z_i).
$$

综上，VAE 可以做 mini-batch 优化参数，以及 mini-batch 对应的 local 目标函数和总体 ELBO 函数形式相同，虽然从神经网络出发觉得都是自然的，但其实都是统计推断的非平庸的结果。



## Reference

[^pyro]: [Documentation for Pyro](http://pyro.ai/examples)

[^kl]: [KL Divergence: Forward vs Reverse?](https://wiseodd.github.io/techblog/2016/12/21/forward-reverse-kl/)

[^mcgradient]: [Monte Carlo Gradient Estimation in Machine Learning](https://arxiv.org/abs/1906.10652)

[^hmc]: [A Conceptual Introduction to Hamiltonian Monte Carlo](https://arxiv.org/abs/1701.02434)

[^nuts]: [The No-U-Turn Sampler: Adaptively Setting Path Lengths in Hamiltonian Monte Carlo](https://arxiv.org/abs/1111.4246) 

[^vireview]: [Variational Inference: A Review for Statisticians](https://arxiv.org/pdf/1601.00670.pdf)

[^mfvi]: [Variational Bayes and The Mean-Field Approximation](https://bjlkeng.github.io/posts/variational-bayes-and-the-mean-field-approximation/)

[^svi]: [Stochastic Variational Inference](https://arxiv.org/abs/1206.7051)

[^bbvi]: [Black Box Variational Inference](https://arxiv.org/pdf/1401.0118.pdf)

[^advi]: [Automatic Differentiation Variational Inference](https://arxiv.org/pdf/1603.00788.pdf)

[^vae]: [Auto-Encoding Variational Bayes](https://arxiv.org/pdf/1312.6114.pdf)