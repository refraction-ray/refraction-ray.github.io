---
layout: post
title: "Magic Detach Function"
date: 2019-09-27
excerpt: "My thoughs on why detach function may be as important as i in complex number"
tags: [algebra, machine learning, statistics]
comments: true
---

* toc
{:toc}

## Introduction

In this post, I won't talk about how to use detach method in pytorch. Instead, I will discuss the function inspired by pytorch's detach method (or stop_gradient method in tensorflow) from mathematical perspective. IMHO, this function is more than a small helpful tool in machine learning libraries, but it provides infinite possibilities to the world of new math and physics.

## Definition

Following the convention of pytorch, we also call our function detach function $$\perp$$, it is defined as 

$$
\perp\!(x) = x~~~~~~~~~~~~\frac{\partial\!\perp\!(x)}{\partial x}=0.
$$

You may feel ok with such a werid function if you are from ML community as it is nothing but stop_gradient in tf or detach in torch. From implementation perspective, such a function is not suprising at all, either. It means that the corresponding op node doesn't further back propagate the gradients, which is a common need in ML practice. 

However, if you are from math or physics community, you might feel uncomfortable about this function. How could $$\partial_x x=0$$ instead of 1, there is no function satisfing the definition (1). Yes, and you also cannot write down a number $$i$$, where $$i^2=-1$$. You may argue that $$i$$ has its own backgound and rationale, you will see later that our detach function has these arguments, too. 

You may also be curious why we define a function exactly like (1) instead of letting the forward part as $$x^2$$ or anything you like. This is in the same spirit of why you don't define some number $$j$$ as $$j^4=-1$$: every number defined in this way can be reexpressed by the combination of real number and $$i$$. Similary, as we can see below, every "weird" function (whose original value and derivatives don't match in common sense) can be expressed by normal functions and $$\perp$$. Namely, this function is a unit to greatly enlarge the function domain; just like $$i$$ enlarge the real number domain to the complex one. You might find the analogy between $$i$$ and $$\perp$$ is very helpful to understand the story here. (Yes, I think I may happen to find something as important as $$i$$ !)

## Completeness Theorem

**Thm1:** For a "weird" function, whose value and every order of derivatives is defined alone, it can always be expressed by normal functions together with detach function $$\perp$$.

Let's firstly see some examples and get a taste on how "weird" can these "weird" functions be. 

a.  $$\mathcal{O}(x)=x-\!\perp\!(x)$$, the value of it is always 0, but it still has derivatives in terms of x, which is $$\mathcal{O'} = 1$$. 

b. $$\mathcal{P}(x)=x\!\perp\!(x)$$, this function is equivalent to $$x^2$$ in values, but its derivative is $$x$$ instead of $$2x$$!

c. $$\mathcal{Q}(x) = \frac{x^2}{\!\perp\!(x^2)} $$, this function is always 1, but $$\mathcal{Q'}(x)=\frac{2}{x}$$!

In a word, weird thing can generate much more weird stuff!

Now, back to the proof of Thm1. Suppose we define any "weird" function $$\mathcal{W}$$ in each order of derivatives as 

$$
\mathcal{W}^{(n)}(x)=h^{(n)}_n(x).
$$

In our context, all normal letters are reserved for "normal" functions, the above equation means that the n-th order derivatives of $$\mathcal{W}$$ is the same as the n-th order derivatives of some normal function $$h_n(x)$$ ($$n=0$$ order is for the original value of the function).

If we have the ability to construct function $$\mathcal{H}_n$$, whose value of each order of derivatives are all zero, except that $$\mathcal{H}^{(n)}_n=h_n^{(n)}$$, then the final construction for $$\mathcal{W}$$ is just $$\mathcal{W} =\sum_n \mathcal {H}_n$$. Therefore, we only need to show that $$\mathcal{H}_n$$ type function can be constructed from normal functions and $$\perp$$.

The strategy is to write down terms in $$\mathcal{H}_n$$ one by one. Starting from the analysis of the n-th order derivative, to make sure there is a $$h_n^{(n)}$$ in this order, we should write down $$h_n$$ for $$\mathcal{H}_n$$. Then, check (n-1)-th order derivative. We want to keep the result to zero in this order, but we are leaving with $$h_n^{(n-1)}$$ from current $$\mathcal{H}_n$$. So we need to add $$-\!\perp\!(h_n^{(n-1)})$$ in this order, which can be integrated back as a new term $$-x^{n-1}/(n-1)! \!\perp\!(h^{(n-1)}_n(x))$$ in $$\mathcal{H}_n$$. Now, check equality in (n-2)-th order derivative. We are again expecting zero, but there are now $$h_n^{(n-2)}-x\!\perp\!(h_n^{(n-1)}(x))$$ left. Again, we add terms to cancel them in this order as $$-\!\perp\!(h_n^{(n-2)}-x\!\perp\!(h_n^{(n-1)}(x)))$$. We can integrate this term back and add it to $$\mathcal{H}_n$$ as $$-x^{n-2}/(n-2)!\!\perp\!(h_n^{(n-2)}-x\!\perp\!(h_n^{(n-1)}(x)))$$. Following this strategy here, the construction of $$\mathcal{H}_n$$ can be done. QED

## Implication

When $$i$$ is introduced as a number, the meaning of $$=$$ has been changed. It is now two equations for $$a=b$$, i.e. $$\Re (a)=\Re(b), \Im(a)=\Im(b)$$. Similary, when $$\perp$$ is introduced as a function, the meaning of $$=$$ has also been enriched. For $$f(x)=g(x)$$, it now means that $$f^{(n)}(x) =_{value} g^{n}(x)$$. In other words, there are two types of equations now: one only holds for the value, and the other one works for each order of derivatives.

For example, $$x-\!\perp\!(x)\neq 0$$, but $$x-\!\perp\!(x)=_{value} 0$$. Therefore, one should be more careful when calculating or proving something with $$\perp$$ on the stage.

## Applications

Find a function $$f(x)$$ such that 

$$
\frac{\partial^n f(y(z))}{\partial z^n}= \frac{\frac{\partial^n y}{\partial z^n}}{y(z)}.
$$

It would be very hard to find a normal function satisfying (2), but if $$\perp$$ is allowed, f is just $$f(x)=\frac{x}{\perp\!(x)}$$.  This stuff is not just playing with symbols and making no sense. On the contrary, such detach function is very easy to implement in computer program, rendering the power of such formalism to all practical numerical methods! For a function that simplifies things a lot analytically and can be utilized in computational program, it is just like a free lunch!

The above question (3) has its background, which originates from gradient estimation on Monte Carlo expectation values (see [this paper](https://arxiv.org/abs/1802.05098) if you want to learn more).

We can also get some insights on why detach function can emerge in realistic problems. We again use Monte Carlo for an example. To measure some expectation value, we have (1/N or 1/Ns is omitted in all sum notation)

$$
\langle O\rangle = \sum_{s_{all}} P(\beta, s)O(s) = \sum_{s\in P(\beta,s) }O(s).
$$

The above value cannot be differentiated directly. The corrected object function which can be automatically differentiated is $$\langle \frac{P}{\!\perp\!(P)}O\rangle$$. How could we understand this without the derivation from (3)? Actually, the object function is very natural. Say we try to use samples from $$\beta_0$$ to estimate the expectation on $$\beta$$, we do something like:

$$
\langle O\rangle_\beta = \sum_{s_{all}}P(\beta,s)O(s) = \sum_{s_{all}} P(\beta_0, s)[\frac{P(\beta, s)}{P(\beta_0, s)}O(s)]\\=\sum_{s\in P(\beta_0, s)}\frac{P(\beta, s)}{P(\beta_0, s)}O(s) = \langle \frac{P(\beta, s)}{P(\beta_0, s)}O(s)\rangle_{\beta_0}.
$$

The same object function as the one ready for AD! If we approach to the derivative limit, $$\beta$$ is then approaching $$\beta_0$$, and the derivative only applies to the numerator (that's the only place for $$\beta$$). rhs in (5) is just the origin of $$\frac{P}{\!\perp\!(P)}O$$. And this also justify why we need "weird" function as detach.

## Future work

The first thing comes to my mind is to generalize the whole formalism to multivariate functions. What does the unit function looks like in that case? Or is our detach function here also enough for the construction of "weird" multivariate functions?

Besides, how can this detach function be utilized in more broad fields of modern math and science? Can such a function drastically simplify some involved formalisms, such as higher order Feynman diagrams in quantum field theory? I believe there are many exciting directions to explore there.