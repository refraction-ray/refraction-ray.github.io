---
layout: post
title: "Math behind REINFORCE"
date: 2020-08-12
excerpt: "Something beyond simple score function and baseline arguments. No cheap intuitions like why they should work but just rigorous math as they must work. "
tags: [statistics, machine learning]
comments: true
---

* toc
{:toc}


## Introduction

In this post, I'll present the (not so) rigorous math and the logic flow behind REINFORCE estimator: which is widely utilized in reinforcement learning setup, such as plain policy gradient or actor-critic frameworks. The readers should be familiar with RL since I won't talk about inspiration or details of these RL algorithms here, and you can easily find great materials on these topic elsewhere. 

The math of REINFORCE is elaborated in [^pgpost] with great detail, we here instead try to use more elegant notations and utilize ideas from [^gae] to futher bridge the missing gap in [^pgpost] when advantage baseline is involved.

## General Setup

The original objective to be optimized is the total reward:

$$
\mathcal{L} = E_{s[0:T], a[0:T-1]} [R].
$$

We only consider typical Markov decision process for now, which indicates that:

1. (Enviroment aspect) The next state is only determined by the last state and the action: $$P(s_{t+1}\vert s_t, a_t)$$. 
2. (Agent aspect) The action is only determined by current state: $$\pi_\theta(a_t\vert s_t)$$.
3. (Enviroment aspect) The reward in each step is only determined by the current state, the current action, or, at most the next state: $$r_t = r(s_t, a_t, s_{t+1})$$. (This requirement is the key for everything in the way of this post, otherwise non-Markovian reward makes every line of this post invalid.)

The total reward is defined as $$R=\sum_{t=0}^{T-1} r_t$$. The formula in this post has finite time cutoff  $$T$$, but all formulas are easily generalized to $$T\rightarrow\infty$$ case.

## Expectation Operators

To present the math behind REINFORCE, we should figure out what on earth these expectation $$E$$ stands for and what properties do these expectation operators have. These facts are vitally important and often omitted in other introductory posts.

Suppose we always start from a fixed state $$s_0$$, (a probabilistic starting state can be easily generalized), then $$E$$ actually stands for 

$$
E_{s[0:T], a[0:T-1]}(\cdot) = \sum_{a_0} \pi_\theta(a_0\vert s_0)\sum_{s_1}P(s_1\vert s_0, a_0)\sum_{a_1}\pi_\theta(a_1\vert s_1)...\sum_{s_T} P(s_T\vert s_{T-1}, a_{T-1}) (\cdot). \label{def}
$$

We have several important properties of such operator:

$$
E_{s[0:T], a[0:T-1]} = E_{s[0:t], a[0:t-1]}E_{s[t+1:T]a[t:T]} = E_{s[0:t], a[0:t]}E_{s[t+1:T]a[t+1:T]}, ~~~\forall \; 0<t<T
$$

Note this decomposition must be in time sequence, i.e. the following is not allowed:

$$
E_{s[0:T], a[0:T-1]} \neq E_{s[t+1:T]a[t:T]} E_{s[0:t], a[0:t-1]} 
$$

One can justify this by considering the definition in $$\eqref{def}$$ directly. Namely one cannot first trace out things of early time as objects in later time depends on them. Or more percisely, the notation should be something like $$E_{(s[t:T], a[t:T])\vert (s[t-1], a[t-1])}$$ which is actually a conditioned expectation. That is why it cannot be inverted, since the condition is always in the direction of time.

The second propety is the generalization of score function, namely we have the following gradient exchange:

$$
\nabla_\theta E_{s[0:T], a[0:T-1]}( O(s[0:T], a[0:T-1])) = E_{s[0:T], a[0:T-1]}(O(\cdot)\sum_{t=0}^{T-1} \nabla_\theta\ln \pi_\theta(a_t\vert s_t)).
$$

Together with the zero property from score function ($$C$$ is a constant):

$$
\nabla_\theta E_{s[0:T], a[0:T-1]}( C) = E_{s[0:T], a[0:T-1]}(C\sum_{t=0}^{T-1} \nabla_\theta\ln \pi_\theta(a_t\vert s_t))=0.
$$

Also, we have:

$$
E_{s[0:T], a[0:T-1]}[O(s_t, a_t)] = E_{s[0:t-1], a[0:t-1]} E_{a_t, s_t}[ O(s_t, a_t)].\label{prop2}
$$

And if $$O = \nabla\ln \pi_\theta(a_t\vert s_t)$$, we are left with

$$
E[\nabla_\theta\ln \pi_\theta(a_t\vert s_t)] = 0.\label{prop3}
$$

This relation can be taken as the outcome of either $$\eqref{prop2}$$ or $$\eqref{prop3}$$.

## Policy Gradient

We now have the gradient of the total reward as:

$$
\nabla_\theta \mathcal{L} = E_{s[0:T], a[0:T-1]}(\sum_{t=0}^{T-1} r_t\sum_{t=0}^{T-1} \nabla_\theta\ln \pi_\theta(a_t\vert s_t)) \label{org}
$$

Note from now on, all simplification and generalization are directly manipulated on gradient level, this indicates that no corresponding loss function can be found in the following in general. One should directly utilized gradient formula in the implementation. (This is where both TensorFlow and Keras did the things WRONG, see [^issue].)

To simplify $$\eqref{org}$$, we notice that:

$$
E[r_t \nabla_\theta \ln \pi_\theta(a_{t'}\vert s_{t'})] = E_{s[0:t+1], a[0:t]} [r_t E_{s[t+2:T], a[t+1:T-1]} \nabla_\theta \ln \pi_\theta(a_{t'}\vert s_{t'})] =0~~~(t+1<t').
$$

And when $$t+1=t'$$, one can also show the expectation is zero (one need to expand relevant expectation to original sum expressions to see that.) Therefore, we have the simplification of $$\eqref{org}$$:

$$
\nabla_\theta \mathcal{L} =  E[\sum_{t=0}^{T-1} (\nabla_\theta \pi_\theta(a_t\vert s_t)\sum_{t'=t}^{T-1}r_{t'})].\label{pg}
$$

This is the familiar expression as in policy gradient.

## Baseline

$$\eqref{pg}$$ shows higher variance and we need to include some baseline to reduce the variance. In other words, we need the formula looks like:

$$
\nabla_\theta \mathcal{L} =  E[\sum_{t=0}^{T-1} (\nabla_\theta \pi_\theta(a_t\vert s_t)(\sum_{t'=t}^{T-1}r_{t'}-b_{t'}(s[0:T],a[0:T-1])))].
$$

Of course the most general form of b as the above introduces bias in the estimation, what we need is to find some specific forms of b, such that 

$$
E[ (\nabla_\theta \pi_\theta(a_t\vert s_t)b_{t'}(s[0:T],a[0:T-1])] = 0.
$$

This can be done as long as b has dependence only on early times, namely $$b_t(s[0:t], a[0:t-1])$$. This automatically include the trivial case, where the baseline is a function of the current state $s_t$ such as another value network $$V(s_t)$$. This zero propety is proved in very similar fashion as the above derivation and you can see them in [^gae] Appendix B.

We can also change the reward sum part using the same philosophy. The reward part can be any $$A_t$$ as long as 

$$
E_{s[t+1:T], a[t+1, T-1]| a_t, s_t } [A_t(s[t:T], a[t:T-1])] = Q(s_t, a_t) , ~~~ \forall s_t\in S, a_t \in A \label{cond}
$$

where $$Q(s_t, a_t)$$ is the expectation value of total reward after that time (also see proof in [^gae] Appendix B which is very straightforward with expectation decomposing notation). Therefore, the sum of later reward trivially falls into this category. The possibilities for $$A_t$$ include:

* directly call on Q value if there is a value-state network: $$Q_\theta(s_t, a_t)$$,
* sum of samples of later reward $$\sum_{t'=t}^{T-1}r_{t'}$$,
* value network output with immdiate reward: $$r_t + V(s_{t+1})$$, note the common $$-V(s_{t})$$ part is classified into baseline part.

All of these are justified with rigorous math since they all follow the condition $$\eqref{cond}$$. They are established by solid math instead of some random arguments or so called intuitions as other posts presented.



## Decay Factor

All discussion above are exact, the deviation is from fluctuation of Monte Carlo estimation. We can in princinple add decay factor $$\gamma$$ on the reward such that $$A_t = \sum_{t'=t}^{T-1}\gamma^{t'-t} r_{t'}$$. And we can pull this into $$\eqref{pg}$$ which is in general better in practice. But this totally change the formula and has no corresponding loss function origins. Namely, it is hard to come up with a natural loss function definition, whose gradient actually gives such formula with discounted rewards. All decay factor stuff is ad-hoc and is forced directly into the gradient formula with no reason. It just works in practice and that's all. Theoretically, I cannot see any benefits from such factor which deviates the correct formula.  (I really dislike the decay factor idea, but it just works in experiments. One has to admit dirty tricks for no reason often outperforms perfect math.)


## References

[^gae]: [High-Dimensional Continuous Control Using Generalized Advantage Estimation](https://arxiv.org/pdf/1506.02438.pdf)

[^pgpost]: [Going Deeper Into Reinforcement Learning: Fundamentals of Policy Gradients](https://danieltakeshi.github.io/2017/03/28/going-deeper-into-reinforcement-learning-fundamentals-of-policy-gradients/)

[^issue]: <https://github.com/keras-team/keras-io/issues/194>