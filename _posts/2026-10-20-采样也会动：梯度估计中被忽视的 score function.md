---
layout: post
title: "采样也会动：梯度估计中被忽视的 score function"
date: 2026-10-20
excerpt: "采样分布本身随参数移动，而反向传播看不见它。从 Keras 的 actor-critic 示例到 DeepSeek 的 GRPO 算法，两种无视 score function 造成的误用"
tags: [statistics, machine learning, llm]
comments: true
---


* toc
{:toc}

## 引言

2020 年我写过一篇博客 Math behind REINFORCE[^reinforce]，核心观点只有一句：在 REINFORCE 里，梯度公式才是第一性的对象，所谓的损失函数只是让自动微分恰好吐出这个梯度的一个捷径；一旦公式在梯度层面做过化简和改写，一般就不存在一个对应的损失函数。当时的直接起因是 Keras 和 TensorFlow 官方 actor-critic 示例里的一个错误[^keras]。

最近在整理大模型后训练的讲义时，我在 GRPO 的 KL 正则项里看到了同一类错误。两个例子放在同一个框架下看，错在同一个地方：都把样本上算出的损失当成了目标本身，默认对它直接求导就是对目标的期望求导，而忽视了采样分布也随参数移动，这一部分梯度只能由 score function 显式写进去。下文借用我们 ADMC 工作[^admc]里的 detach 记号，把这件事写成一条统一的规则，再用它检查两个例子，并给出 GRPO 的修正。

## 一个公式，两条通道

设要优化的量是某个函数在策略采样下的期望：

$$
J(\theta)=\mathbb E_{y\sim\pi_\theta}\bigl[f_\theta(y)\bigr]=\sum_y \pi_\theta(y)\,f_\theta(y).
$$

$$\theta$$ 通过两条通道影响 $$J$$：一是固定 $$y$$ 时被平均的数值 $$f_\theta(y)$$ 本身会变；二是每个 $$y$$ 被采到的概率 $$\pi_\theta(y)$$ 会变。直接求导，

$$
\nabla_\theta J
=\underbrace{\mathbb E_{\pi_\theta}\bigl[f\,\nabla_\theta\log\pi_\theta\bigr]}_{\text{采样分布变化}}
+\underbrace{\mathbb E_{\pi_\theta}\bigl[\nabla_\theta f_\theta\bigr]}_{\text{被平均的数值变化}} ,
$$

其中第一项用了 $$\nabla_\theta\pi_\theta=\pi_\theta\nabla_\theta\log\pi_\theta$$。下面记 $$g(y)=\nabla_\theta\log\pi_\theta(y)$$，这就是 score function。

score function 描述的是第一条通道：参数变了以后，原来常采到的样本变得少见，原来少见的样本变得常见。问题在于自动微分只认识计算图。样本一旦采好，它就是一个固定的整数序列，计算图里根本没有换一个 $$\theta$$ 会采到别的 $$y$$ 这件事。所以在样本上算一个量再反传，框架只能看到第二条通道。

## 用 detach 把采样写进计算图

要让自动微分看见第一条通道，就得在计算图里人为放进一个东西来代表“采样分布”。ADMC 里的做法是引入 detach 函数 $$\bot$$：

$$
\bot(x)=x,\qquad \frac{\partial\bot(x)}{\partial x}=0,
$$

它只向前传值、不向后传梯度，对应 PyTorch 的 `detach` 和 TensorFlow 的 `stop_gradient`。用它可以把任意期望写成

$$
J=\Bigl\langle \rho\,f_\theta\Bigr\rangle_{\bot(\pi_\theta)},\qquad \rho=\frac{\pi_\theta(y)}{\bot\bigl(\pi_\theta(y)\bigr)} .
$$

$$\rho$$ 的数值恒为 $$1$$，所以这个式子的数值就是普通的样本平均；求导时 $$\nabla_\theta\rho=\nabla_\theta\pi_\theta/\pi_\theta=g$$，于是

$$
\nabla_\theta(\rho f)=f\,g+\nabla_\theta f ,
$$

两条通道都在。这个形式的直观意义是重要性采样：把采样时的参数冻结为 $$\theta_0$$，用比值 $$\pi_\theta/\pi_{\theta_0}$$ 复用同一批样本去估计邻近 $$\theta$$ 处的期望，再让 $$\theta\to\theta_0$$。$$\bot$$ 所做的，就是在计算图里标出“哪个 $$\theta$$ 是采样时的 $$\theta_0$$”。ADMC 证明了这个写法对任意阶导数都正确，并推广到了非归一化分布（这时要再除以 $$\langle\rho\rangle_{\bot(\pi)}$$）；对归一化分布，它与 DiCE 的 magic box[^dice] 等价。常见的 REINFORCE 替代损失 $$\bot(f)\log\pi_\theta+f$$[^scg] 是它的一阶版本：梯度相同，但数值没有意义，高阶导数也不对。

ADMC 文中还指出，引入 $$\bot$$ 以后，等号的含义被扩大了。$$F=G$$ 不再只表示数值相等，而是表示数值和各阶导数分别相等，$$\bot(F^{(n)})=\bot(G^{(n)})$$，$$n=0,1,2,\dots$$。把一个样本损失 $$L(\theta;y)$$ 交给优化器，实际要求的是

$$
\mathbb E_{\bot(\pi_\theta)}\bigl[\bot(\nabla_\theta L)\bigr]=\bot(\nabla_\theta J),
$$

即 $$n=1$$ 的那个等号。通常说的“无偏估计”只检查了 $$n=0$$。

下面两个例子都违反了这条规则，原因也相同：把样本损失的数值当成了目标本身，于是默认把它整个交给了样本上的自动微分。

## 例一：actor-critic 的基线

经典 actor-critic 中，回报 $$R$$ 与 $$\theta$$ 无关，第二条通道为零，全部梯度都来自 score function：

$$
\nabla_\theta J=\mathbb E\bigl[(R-V(s))\,\nabla_\theta\log\pi_\theta(a\vert s)\bigr].
$$

$$V(s)$$ 在这里只是基线，是梯度公式里的一个系数，按规则应当写成 $$\bot(V)$$。但当时的示例代码写的是

```python
diff = ret - value          # value 来自与 actor 共享参数的网络
actor_loss = -log_prob * diff
```

`value` 没有截断梯度。对 $$L=-(R-V_\theta)\log\pi_\theta$$ 反传，得到

$$
\nabla_\theta L=-(R-V_\theta)\,g+\log\pi_\theta(a\vert s)\,\nabla_\theta V_\theta .
$$

第二项来自系数 $$R-V_\theta$$ 没有套上 $$\bot$$：score function 的载体 $$\log\pi_\theta$$ 被当成了目标的一部分，连同系数一起求导。由于 $$V_\theta(s)$$ 不依赖动作，固定 $$s$$ 对 $$a\sim\pi_\theta(\cdot\vert s)$$ 取期望时可以把 $$\nabla_\theta V_\theta$$ 提出来：

$$
\mathbb E_{a\sim\pi_\theta(\cdot\vert s)}\bigl[\log\pi_\theta(a\vert s)\bigr]\,\nabla_\theta V_\theta(s)
=\Bigl(\sum_a\pi_\theta(a\vert s)\log\pi_\theta(a\vert s)\Bigr)\nabla_\theta V_\theta(s).
$$

括号里是策略在 $$s$$ 处熵的相反数，只要策略不是确定性的，它就严格小于零，这一项的期望不为零，所以梯度是有偏的：梯度下降时它会额外把 $$V_\theta(s)$$ 往大里推。修正就是一行 `tf.stop_gradient(value)`。

这个错误不会让训练崩掉，但会让它变差。维护者后来跑了 50 次 CartPole，解出任务所需步数从 $$471.7\pm206.1$$ 降到 $$298.1\pm93.1$$。

## 例二：GRPO 的 KL 项

GRPO 在损失里对每个 token 单独减去一项 $$\beta\,\widehat{\mathrm{KL}}$$。DeepSeekMath 原文[^grpo]的估计量是

$$
\widehat{\mathrm{KL}}=k_3=u-\log u-1,\qquad u=\frac{\pi_{\mathrm{ref}}(y\vert s)}{\pi_\theta(y\vert s)},
$$

给出的理由是两条：它是 KL 的“unbiased estimator”，而且“guaranteed to be positive”，出处是 Schulman 的博客 Approximating KL Divergence[^schulman]。

这两条都对，但都是数值性质，也就是只检查了 $$n=0$$ 的等号。记 $$k_1=-\log u=\log(\pi_\theta/\pi_{\mathrm{ref}})$$，在 $$\pi_\theta$$ 下采样时

$$
\mathbb E_{\pi_\theta}[u]=\sum_y\pi_\theta\frac{\pi_{\mathrm{ref}}}{\pi_\theta}=1
\quad\Longrightarrow\quad
\mathbb E_{\pi_\theta}[k_3]=\mathbb E_{\pi_\theta}[k_1]=\mathrm{KL}(\pi_\theta\Vert\pi_{\mathrm{ref}}).
$$

$$u-1$$ 期望为零，是一个控制变量：在 $$u=1$$ 附近 $$k_1\approx-(u-1)+\tfrac12(u-1)^2$$，加上 $$u-1$$ 正好消掉可正可负的一阶项，剩下 $$\tfrac12(u-1)^2\ge0$$，单样本方差更小。

但 GRPO 把它直接写进了损失，对固定的 rollout 样本反传。这时 $$f_\theta=k_3$$ 显式依赖 $$\theta$$，两条通道都不为零，而损失里没有 $$\rho$$，自动微分只走了第二条：

$$
\nabla_\theta k_3=\nabla_\theta u-\nabla_\theta\log u=-u\,g+g=(1-u)\,g .
$$

取期望，

$$
\mathbb E_{\pi_\theta}\bigl[(1-u)g\bigr]
=\sum_y\pi_\theta\,g-\sum_y\pi_{\mathrm{ref}}\,g
=-\mathbb E_{\pi_{\mathrm{ref}}}\bigl[\nabla_\theta\log\pi_\theta\bigr]
=\nabla_\theta\mathrm{KL}(\pi_{\mathrm{ref}}\Vert\pi_\theta).
$$

于是：

* 损失的数值是 $$\mathrm{KL}(\pi_\theta\Vert\pi_{\mathrm{ref}})$$ 的无偏估计；
* 这个量的真梯度是 $$\mathbb E[k_3 g]+\mathbb E[(1-u)g]=\mathbb E[k_1 g]=\nabla_\theta\mathrm{KL}(\pi_\theta\Vert\pi_{\mathrm{ref}})$$；
* 实际反传算出的只有后一半，而这一半的期望恰好是参数顺序交换后 $$\mathrm{KL}(\pi_{\mathrm{ref}}\Vert\pi_\theta)$$ 的梯度，前向和后向 KL 互换了。

漏掉的 $$\mathbb E[k_3\,g]$$，正是 $$\rho$$ 本该贡献的 score function 那一项。

两项的角色在估值和求导时还互换了。估值时，$$-\log u$$ 是主项，$$u-1$$ 是控制变量；求导时，期望梯度全部来自 $$u$$ 项（$$u$$ 在这里像一个重要性权重，把 $$\pi_\theta$$ 下的样本换算成 $$\pi_{\mathrm{ref}}$$ 下的平均），而 $$-\log u$$ 贡献的 $$g$$ 期望为零，退化成了一个逐样本基线，让权重 $$1-u$$ 在 $$u=1$$ 处恰为零。如果只把 $$k_1$$ 写进损失，反传得到 $$g$$，期望为零，KL 项完全没有约束作用。

把几种写法放在一起：

| 实现 | 单样本梯度 | 期望梯度 |
|---|---|---|
| $$k_1$$ 写进损失 | $$g$$ | $$0$$ |
| $$k_3$$ 写进损失（GRPO） | $$(1-u)\,g$$ | $$\nabla\mathrm{KL}(\pi_{\mathrm{ref}}\Vert\pi_\theta)$$ |
| $$\bot(k_1)\log\pi_\theta$$ | $$k_1\,g$$ | $$\nabla\mathrm{KL}(\pi_\theta\Vert\pi_{\mathrm{ref}})$$ |
| $$\tfrac12k_1^2$$ 写进损失 | $$k_1\,g$$ | $$\nabla\mathrm{KL}(\pi_\theta\Vert\pi_{\mathrm{ref}})$$ |
| $$k_1$$ 并入奖励（PPO） | $$k_1\,g$$（经优势） | $$\nabla\mathrm{KL}(\pi_\theta\Vert\pi_{\mathrm{ref}})$$ |
| $$\rho\,k_3$$ 写进损失 | $$(k_3+1-u)\,g$$ | $$\nabla\mathrm{KL}(\pi_\theta\Vert\pi_{\mathrm{ref}})$$ |

第四行 $$\tfrac12k_1^2$$ 的数值并不是 KL 的无偏估计，求导却给出了正确的梯度；第二行正好相反。用扩大的等号来说，前者只满足 $$n=1$$，后者只满足 $$n=0$$。最后一行两者都满足，下文再展开。

## 序列层面：还漏了一层

上面都固定了前缀，只看单个位置。语言模型的 KL 是整条回答的，而后面的前缀本身也由 $$\pi_\theta$$ 生成，所以序列层面还有一条“前缀分布变化”的通道。仍用 score function 展开，记 $$g_k=\nabla_\theta\log\pi_\theta(y_k\vert s_k)$$，$$k_{1,t}=\log[\pi_\theta(y_t\vert s_t)/\pi_{\mathrm{ref}}(y_t\vert s_t)]$$，

$$
\nabla_\theta\mathrm{KL}\bigl(\pi_\theta(\cdot\vert c)\Vert\pi_{\mathrm{ref}}(\cdot\vert c)\bigr)
=\mathbb E_{y\sim\pi_\theta}\Bigl[\sum_{k=1}^{T}g_k\sum_{t=k}^{T}k_{1,t}\Bigr].
$$

位置 $$t$$ 的对数比不只乘在本位置的 $$g_t$$ 上，也乘在之前各位置的 $$g_k$$ 上，因为前面的 token 决定了后面在什么前缀上做选择。这和 REINFORCE 里 reward-to-go 的结构完全一样，推导也就是那篇旧博客里因果性论证的翻版：$$k<t$$ 时，$$\mathbb E[k_{1,k}\,g_t]$$ 经条件期望为零，只有 $$t\ge k$$ 的项留下。

PPO 把 $$-\beta k_{1,t}$$ 并入逐位置的奖励，再经优势累加，在不打折扣（$$\gamma=\lambda=1$$）时算的正是这个式子，所以 PPO 的 KL 项对应整条回答的 $$\mathrm{KL}(\pi_\theta\Vert\pi_{\mathrm{ref}})$$。GRPO 的 KL 项只作用于本位置，即使把 $$k_3$$ 换成正确的单位置写法，也只剩对角项 $$g_tk_{1,t}$$，丢了前缀那一层。用 $$k_3$$ 时更彻底：它的期望梯度是

$$
\mathbb E_{y\sim\pi_\theta}\Bigl[\sum_t\nabla_\theta\mathrm{KL}\bigl(\pi_{\mathrm{ref}}(\cdot\vert s_t)\Vert\pi_\theta(\cdot\vert s_t)\bigr)\Bigr],
$$

前缀来自 $$\pi_\theta$$；而整条回答 $$\mathrm{KL}(\pi_{\mathrm{ref}}\Vert\pi_\theta)$$ 的梯度是 $$-\mathbb E_{y\sim\pi_{\mathrm{ref}}}[\sum_t g_t]$$，前缀来自 $$\pi_{\mathrm{ref}}$$。两者一般不相等：前缀来自不同的分布，GRPO 也没有计入前缀分布随 $$\theta$$ 的变化。所以在整条回答的层面，它不是任何一种参数顺序的序列 KL 的梯度，只是各位置局部约束的和。

## 同一框架下的两个错误

| | actor-critic 示例 | GRPO 的 KL 项 |
|---|---|---|
| 目标 $$J$$ | $$\mathbb E[R]$$ | $$\mathbb E[k_3]=\mathrm{KL}(\pi_\theta\Vert\pi_{\mathrm{ref}})$$ |
| 按规则的写法 | $$\bot(R-V)\log\pi_\theta$$ | $$\rho\,k_3$$ |
| 实际写法 | $$(R-V_\theta)\log\pi_\theta$$，系数没有套 $$\bot$$ | 只有 $$k_3$$，没有 $$\rho$$ |
| 与正确梯度的差 | 多出 $$\log\pi_\theta\,\nabla_\theta V$$，期望 $$\bigl(\sum_a\pi_\theta\log\pi_\theta\bigr)\nabla_\theta V\neq0$$ | 缺少 $$k_3\,g$$，期望 $$\mathbb E[k_3\,g]\neq0$$ |
| 修正 | `stop_gradient(value)` | 乘上 $$\rho$$（见下节），或 $$\bot(k_1)\log\pi_\theta$$，或像 PPO 那样并入奖励 |

放在同一框架下，两者是同一个错误：采样分布没有在计算图里显式表示出来，样本损失的数值被当成了目标，整个交给自动微分。actor-critic 示例把 $$(R-V)\log\pi$$ 看成了一个要优化的量，于是连系数一起求导；GRPO 把 $$k_3$$ 看成了 KL 本身，于是认为对它求导就是对 KL 求导。一个多出了一项，一个少了一项，原因都是忽视了采样也随参数移动。score function 描述的就是这种移动，它在计算图里看不见，只能由人写进去。

## 按 ADMC 的写法修正：只差一个 $$\rho$$

按上面的规则，GRPO 的 KL 项不必换掉 $$k_3$$，只要乘上 $$\rho$$：

$$
L_{\mathrm{KL}}=\rho\,k_3,\qquad \rho=\frac{\pi_\theta(y\vert s)}{\bot\bigl(\pi_\theta(y\vert s)\bigr)} .
$$

数值上 $$\rho=1$$，损失的数值还是原来的 $$k_3$$，监控曲线不变，低方差、非负的好处都保留；求导得到 $$(k_3+1-u)\,g$$，期望正是 $$\mathbb E[k_1 g]=\nabla_\theta\mathrm{KL}(\pi_\theta\Vert\pi_{\mathrm{ref}})$$，即上表最后一行。到了序列层面，$$k_{3,t}$$ 所在前缀的采样概率按链式法则是第 $$1$$ 到第 $$t$$ 步条件概率的乘积，所以 $$\rho$$ 的分子分母都换成这些逐 token 概率的连乘：

$$
\rho_t=\prod_{k\le t}\frac{\pi_\theta(y_k\vert s_k)}{\bot\bigl(\pi_\theta(y_k\vert s_k)\bigr)},\qquad
L_{\mathrm{KL}}=\sum_t\rho_t\,k_{3,t},
$$

就得到整条回答 $$\mathrm{KL}(\pi_\theta\Vert\pi_{\mathrm{ref}})$$ 的精确梯度。这里 $$\rho_t$$ 是整个前缀的乘积，求导时每个因子都贡献一个 score function，$$\nabla_\theta\rho_t=\rho_t\sum_{k\le t}g_k$$，于是 $$\nabla_\theta(\rho_t k_{3,t})=\bigl(\sum_{k\le t}g_k\bigr)k_{3,t}+(1-u_t)\,g_t$$。前面各步的 score function 乘上当前步的 $$k_3$$，这些交叉项就是这样来的。代入 $$k_{3,t}=k_{1,t}+(u_t-1)$$ 展开：$$k=t$$ 时，$$(u_t-1)\,g_t$$ 与 $$(1-u_t)\,g_t$$ 逐样本抵消；$$k<t$$ 时，$$g_k$$ 由前缀决定，而给定前缀 $$u_t-1$$ 的期望为零，所以 $$g_k(u_t-1)$$ 的期望为零。剩下的 $$\sum_{k\le t}g_k\,k_{1,t}$$ 对 $$t$$ 求和，就是上一节的序列 KL 梯度式。只乘本位置的 $$\rho$$ 时，缺的是 $$k<t$$ 的 $$g_k\,k_{1,t}$$，也就是前缀分布的变化。写成代码只多几行（on-policy 情形，省略 padding mask）：

```python
logp_cum = logp.cumsum(-1)                          # 逐 token 对数概率的前缀和，形状 [B, T]
rho = torch.exp(logp_cum - logp_cum.detach())       # 数值恒为 1，梯度是前缀的 score function
k3 = torch.exp(ref_logp - logp) - (ref_logp - logp) - 1
kl_loss = (rho * k3).sum(-1).mean()                 # 数值 = 序列 KL 的 k3 估计，梯度 = ∇ 序列 KL(π_θ‖π_ref)
```

从这个写法还能看出两点。

第一，PPO 和 GRPO 里本来就有 $$\rho$$。它们的主目标里那个概率比 $$\pi_\theta/\pi_{\mathrm{old}}$$，正是 $$\pi_\theta/\bot(\pi_\theta)$$ 的“过期”版本：样本来自上一轮参数，$$\theta_{\mathrm{old}}$$ 扮演的就是 $$\bot(\theta)$$。在 $$\theta=\theta_{\mathrm{old}}$$ 处求导，它给出的就是 score function。GRPO 的优势由组内归一化得到，但进入损失时沿用了 PPO 的裁剪目标，仍要乘上这个比值；同一个目标里的 KL 项却直接减去 $$\beta k_3$$，没有乘。多轮 off-policy 更新时，KL 项也应当乘上同一个比值，$$\mathbb E_{\pi_{\mathrm{old}}}[(\pi_\theta/\pi_{\mathrm{old}})\,k_3]=\mathrm{KL}(\pi_\theta\Vert\pi_{\mathrm{ref}})$$，数值和梯度才都对。这与 Zhang 等人指出的 GRPO 重要性权重错配是同一件事[^zhang]。

第二，ADMC 文中给过一个对任意阶导数都正确的 KL 估计量。样本来自 $$p$$ 时，

$$
\mathrm{KL}(p\Vert q)=\ln\frac{\langle\rho\,q/p\rangle_{\bot(p)}}{\langle\rho\rangle_{\bot(p)}}-\frac{\langle\rho\ln(q/p)\rangle_{\bot(p)}}{\langle\rho\rangle_{\bot(p)}},\qquad \rho=\frac{p}{\bot(p)} .
$$

取 $$p=\pi_\theta$$、$$q=\pi_{\mathrm{ref}}$$，则 $$q/p=u$$。把第一项的 $$\ln x$$ 在 $$x=1$$ 处线性化为 $$x-1$$，整个式子就变成 $$\langle\rho\,(u-\ln u-1)\rangle/\langle\rho\rangle=\langle\rho\,k_3\rangle/\langle\rho\rangle$$。而且第一项的自变量 $$x=\langle\rho\,u\rangle/\langle\rho\rangle$$ 在大样本下等于 $$1$$；其中 $$\rho\,u=\pi_{\mathrm{ref}}/\bot(\pi_\theta)$$ 本身没有梯度，$$x$$ 的梯度只来自分母，期望为零。$$\ln x$$ 与 $$x-1$$ 在 $$x=1$$ 处的值和一阶导数都相同，所以这一步线性化不改变数值和梯度，只影响二阶及以上的导数。也就是说，$$k_3$$ 的数值形式和保证梯度正确的 $$\rho$$，在那里本来是一起出现的；GRPO 保留了前者，丢掉了后者。

## 一个理论上错误的跳跃

回头看 DeepSeekMath 对 $$k_3$$ 的论证，它只给了数值层面的两条理由，没有讨论这一项进入损失后梯度是什么。从“这是 KL 的一个好估计量”到“把它放进损失就是在做 KL 正则”，中间缺了 $$n=1$$ 那个等号。这个跳跃在数学上不成立，而且随着 GRPO 成为开源后训练的默认配方，同样的错误写法也被大量复制。

Tang 和 Munos[^tang] 分析了“对 KL 估计量直接反传”和“只做逐 token 而忽略序列结构”这两类问题，涉及 GRPO 的伪代码以及 TRL、Open Instruct 等开源实现。他们认为 $$k_3$$ 损失的正则效果是附带的（“incidental”），并把这类实现归为 bug 而非设计。Zhang 等人[^zhang]同样指出它实际约束的是 $$\mathrm{KL}(\pi_{\mathrm{ref}}\Vert\pi_\theta)$$，并且多轮 off-policy 更新时缺少对应的重要性权重；Liu 等人[^liu]则把 $$1-u$$ 看作 $$-\log u$$ 的有偏一阶近似。几种分析的出发点不同，但都表明 GRPO 的 KL 项优化的不是它写下的那个目标。

说它在理论上错了，不等于说它在实践中更差。

第一，两种梯度的权重分别是 $$k_1=\log(1/u)$$ 和 $$1-u$$，在 $$u=1$$ 处都为零、一阶相同。$$\beta$$ 小、策略离参考模型不远时，两者几乎没有差别。Tang 和 Munos 的实验也是这个结论：在低 KL 的奖励最大化场景里，各种实现的表现差不多；只有在高 KL 的场景，比如 70B 到 8B 的 on-policy 蒸馏，正确实现才明显更好。

第二，两种参数顺序的 KL 只在 $$\pi_\theta=\pi_{\mathrm{ref}}$$ 时为零，所以 GRPO 的 KL 项仍然是一个有效的“别跑太远”的约束，只是形状不同：对策略过度看好的 token（$$u<1$$），$$1-u$$ 的压制不超过 $$1$$，比 $$k_1$$ 温和；对策略已经放弃、参考模型仍然看好的 token（$$u>1$$），$$1-u$$ 线性增长，比 $$k_1$$ 更强地把它拉回来。也就是说，它对“策略变尖锐”更宽容，对“丢掉参考模型覆盖的模式”更严格。在 RLVR 这种希望策略集中到正确推理路径、又不希望多样性彻底坍缩的场景里，这种形状到底是好是坏，没有先验答案。

第三，后来一些长链推理的 RL 配方干脆把 KL 项整个去掉了。例如 DAPO[^dapo] 的理由是，长思维链训练中策略本来就会远离初始模型，这个约束没有必要。这说明在这类任务里，KL 项的精确形式可能本来就不是决定性因素。

结论是，GRPO 的 KL 项在理论上是错误的，这一点应该讲清楚，至少使用者要知道自己实际在优化什么；但它在实践中是更好、更差还是无所谓，只能靠实验回答。机器学习终究有炼丹的一面，数学全对的写法不一定效果最好。我在旧博客结尾抱怨过折扣因子：“It just works in practice and that's all... One has to admit dirty tricks for no reason often outperforms perfect math.” 六年后看，这句话对 $$k_3$$ 也许同样适用。区别在于，用 dirty trick 应该是明知故犯，而不是没有意识到。

## 附：一个自查清单

每次在样本上写一个损失之前，可以先问三个问题：

1. 我想优化的期望 $$\mathbb E_{\pi_\theta}[f_\theta]$$ 是什么？$$f$$ 显式依赖 $$\theta$$ 吗？
2. 采样分布在计算图里由谁代表？有没有 $$\rho=\pi_\theta/\bot(\pi_\theta)$$，或者等价的 $$\bot(\cdot)\log\pi_\theta$$？序列问题里，它是否覆盖了整个前缀？
3. 被当作系数、基线或优势的量，是否都套上了 $$\bot$$？如果省掉了某一项梯度，是否证明过它的期望为零？

值和求梯度是两个等号，数值能对上只说明第一个成立，不自动保证第二个成立。

## References

[^reinforce]: [Math behind REINFORCE](https://re-ra.xyz/Math-behind-REINFORCE/)

[^keras]: [Possible issue of gradients calculation in actor_critic_cartpole example](https://github.com/keras-team/keras-io/issues/194)

[^admc]: [Automatic differentiable Monte Carlo: Theory and application](https://journals.aps.org/prresearch/abstract/10.1103/PhysRevResearch.5.033041)

[^dice]: [DiCE: The Infinitely Differentiable Monte-Carlo Estimator](https://arxiv.org/abs/1802.05098)

[^scg]: [Gradient Estimation Using Stochastic Computation Graphs](https://arxiv.org/abs/1506.05254)

[^grpo]: [DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models](https://arxiv.org/abs/2402.03300)

[^schulman]: [Approximating KL Divergence](http://joschu.net/blog/kl-approx.html)

[^tang]: [On a few pitfalls in KL divergence gradient estimation for RL](https://arxiv.org/abs/2506.09477)

[^zhang]: [On the Design of KL-Regularized Policy Gradient Algorithms for LLM Reasoning](https://arxiv.org/abs/2505.17508)

[^liu]: [Rethinking KL Regularization in RLHF: From Value Estimation to Gradient Optimization](https://arxiv.org/abs/2510.01555)

[^dapo]: [DAPO: An Open-Source LLM Reinforcement Learning System at Scale](https://arxiv.org/abs/2503.14476)
