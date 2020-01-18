---
layout: post
title: "Automatic Differentiate Computational Physics"
date: 2020-01-14
excerpt: "How can numerical approaches in strongly correlated physics such as exact diagonalization, Monte Carlo and tensor network be integrated into AD infrastructure"
tags: [algebra, algorithm, machine learning, physics]
comments: true
---

* toc
{:toc}

## Introduction

Understanding strongly correlated many-body systems is a very challenging task and there are very few analytic tools solving these problems especially when the strength of interactions goes beyond pertubative region. Therefore, we usually explore properties of these stongly correlated systems relying on computational methods and hopefully these numerical results can verify our theoretical pespectives. Amongst all kinds of computational toolboxes for stongly correlated physics, three of them are the most frequently utilized: they are exact diagonalization (ED), Monte Carlo methods (MC) and tensor network approaches (TN).

As differentiation programming has been envisioned as the new programming paradigm in deep learning community, its value outside ML applications has recently emerged in various fields: computational physics,  in particular. In less than one year, academic works on the combination between automatic differentiation (AD) and computational physics methods have already covered the three most important numerical branches in strongly correlated physics: ED, MC and TN. We will briefly summarize how AD can be naturally encoded in these computational schemes and the possible applications in physics of such combinations. As the prerequisite for this post, the readers must have some basic knowledges on AD and computational methods in physics.



## AD in Tensor Network

### Method

Tensor network is a parameterized representation for wavefunctions or partition functions with high efficiency and accuracy. The core object in tensor network alogrithms such as DMRG, MPS, PEPS, TEBD, MERA and various types of tensor RG is the tensor arising from the contraction between many small tensor components and the core operations in TN algorithms include contraction, transposition of the tensor as well as eigen and SVD decompositions on matrix from reshaped tensors.

### Reference

arXiv:1903.09650

### AD

As described in method section, to make tensor network algorithm AD aware, one need to ensure all involved operations equipped with back propagation formulas. There are already several well known list of AD formulas on linear algebra (see reference [1-5] in [my earlier post](/Gauge-Problem-in-Automatic-Differentiation/)). Specifically, AD for einsum is very elegant with index magic (see [here](https://giggleliu.github.io/2019/04/02/einsumbp.html) for details). With AD-aware linear algebra operations as well as AD approach on fixed point operations (which is useful to differentiate the whole iterative process in tensor RG scheme), one can easily implement AD-aware tensor network algorithms. For tensor RG algorithms, checkpoint technique might be utilized to reduce the memory cost used in back propagation through the long iterative processes.

### Application

1) For some classical model, such as 2D Ising model, one can explicitly write down the partition function exactly as a 2D tensor networks. However, for exact partition function in thermodynamic limit, one need to trace out infinitely many on-site tensors. We can instead achieve this by RG idea: tracking the contraction between tensors for larger size step by step. The fixed point of such tensor RG gives the exact partition function Z of 2D Ising model. We can then evaluate energy (specific heat and relevant susceptility) by automatic differentiate partition functions to the first (second) order. We can also roughly locate the transition point as the peak of specific heat.

2) Tensor networks serve as powerful and efficient expressions for low-enetanglement wavefunctions. Therefore, we can use parameterized tensor network as wavefunction ansatz and evaluate the ground state energy by tracing out MPS-MPO-MPS pair or using some variants of tensor RG algorithm. Then we can minimize the energy against variational parameters in tensor network using gradient-based optimizers such as LBGFS, Adam or plain SGD. And the gradient or higher order of derivatives required in these optimizers are easily obtained from automatic differentiation.

## AD in Monte Carlo

### Method

Monte Carlo method is another big family of numerical approaches to understand strongly correlated systems. It bases on Markov chain Monte Carlo (MCMC) which can simulate probability distributions without explicit normalization factor (partition function). Monte Carlo algorithms are rather versatile, ranging from classical MC,  variational MC, determinant MC, world line MC and so on. The core operation of all these MC alorithms is Monte Carlo sampling, namely estimating expectation values of observables based on probability distributions determined by Markov chains.

### Reference

arXiv:1911.09117

### AD

There are several traditional approaches to apply AD on Monte Carlo estimatos, including score functions (REINFORCE) and path estimators (reparameterization) (see arXiv:1906.10652 for a review). However, these original approaches cannot deal with Monte Carlo estimators with unnormalized probability distributions since the exact value of probability density p is required in score functions. But in almost every MC application in physics, the unnormalized probability distribution is utilized.

Therefore, one need to derive new formula for AD-aware MC estimators with both known and unknown normalization factors. And we call such estimator ADMC in this paper. With the help of detach function, the new infinitely AD-aware MC estimator is quite elegant (see arXiv:1911.09117 for details). Equipped with the new AD-aware MC estimator, one can carry out AD-aware MC algorithms since all other ordinary operations are simple to be automatic differentiated.

### Application

1) For a given model, to locate the peak of some observable/susceptibility (as rough estimation of certain phase transitions), the common approach is to evaluate (using MC algorithms here) the observable with varying parameters; then by drawing the observable curve, one can easily locate the peak. Say we need m MC updates to accurately evaluate the observable for each parameters, and we need n parameter points to draw the curve, the computational cost in O(mn). However, there is no need to get accurate results of observables to obtain the information about the peak. Instead, we utilize noisy stochastic gradient descent and update the varying parameter based on the derivatives obtained from AD in the MC algorithms. In limit case, it is enough to carry out one MC update followed by one parameter update, which is the essence of SGD!  Although the evaluation on the observables is noisy in each step, the updates can still lead to the peak of the observable. And it turns out we only need O(m) time in total to locate such peak. This technique combines state of the art SGD optimization and automatic differentiation infrastructure with AD-aware MC estimators, and it provides impressive speed up to locate phase transitions compared to all the traditional approaches.

2) It is obvious AD-aware Monte Carlo approach can be naturally encoded into VMC framework. The whole setup is very similar to application 2) in tensor network section. One can use neural networks (such as RBM or even autoregressive networks) as parameterized wave function ansatz and evaluate the ground state energy by configuration space Monte Carlo estimation instead of contraction of tensors. Since such MCMC estimation can be differentiated in the context of ADMC, one can carry out gradient based optimization to find optimial ground state energy and corresponding wave functions.

## AD in Exact Diagonalization

### Method

ED is a straightforward numerical approach which directly diagonalize the full quantum Hamiltian of given system and directly evaluate observables from eigenvalues (energy spectrum) and eigen vectors (wave functions) from ED results.

### Reference

arXiv:2001.04121

### AD

As we mentioned in tensor network section, AD formula for eigen operation is well established. However, in most of the physics relevant cases, we only care about very few lowest energy levels and the correspoding wave functions. To effciently do this, we could use spare matrix algorithms to get only few eigenvalues (such as Lanczos method) in the forward pass. It is very inefficient or even intractable to use AD formula for full eigen decomposition in such case, since all eigenvalues and eigen vectors are required.

Previously, people use fixed point relation for eigenvalues as $$x=\frac{Ax}{\vert\vert Ax\vert\vert}$$ (see arXiv:1906.09023) to do back propagations for dominant eigen solver. In this paper, the author instead use some linear system solver in the back propagation pass. This approach is universal which emerges from the general adjoint method and it is more suitable when several rather than one eigenvalues are needed in the forward pass. Besides, this new AD approach is easy to generalize to truncated SVD case as discussed below.

It is worth noting, the same approach can be applied to truncated SVD solver, which is frequently used in tensor RG algorithms to reduce bond dimensions between tensors. AD approach here on truncated SVD is better in speed than AD formula for full SVD decompositions.

### Application

1) Similar to 1) in tensor network, one can apply AD on observables from ED results and get susceptibilities such as specific heat by AD for free.

2) We can compute fidelity susceptibility by using AD-aware fidelity objectives $$\langle \psi\vert \bot{\psi}\rangle$$. Note how [detach function](/Magic-Detach-Function/) is perfectly suited here. The peak of fidelity susceptibility is very useful to locate (dynamic) phase transitions.

3) In some tensor network algorithms in infinite fashion, one need to evaluate the left and right environment tensor as the dominant eigen vector of transfer matrix. Therefore, AD on dominant eigen solver can be integrated in these iDMRG type of algorithms, where further gradient-based optimization is similar to 2) in tensor network case.



## Summary

These works mentioned in this post are not only applying AD techniques to physics problems, but they also develop new ideas for AD itself which are not even previously disscussed in CS community. Physicists' contributions to AD include: infinitely AD on Monte Carlo estimator with unnormalized distributions and efficient and stable infinitely AD on dominant eigen solvers and truncated SVD operations. These new findings of AD have not even adopted in mainstream machine learning libraries for now, but these new AD possibilities are definitely important for machine learning community, too. There is a great possibility that deep learning setup with ingredients such as complex number, SVD, EIG, QR decompositions and MCMC middleware can beat SOTA models now.

As we can see, the application of AD integrated computational methods in condensed matter physics are mainly divided into two categories: automatic differentiate observables to obtain susceptibilities; obtain gradients by AD to carry out optimizations on ground state energy and wavefunctions. It would be exciting to see more AD applications beyond these two families in physics context.

ADTN, ADMC and ADED, what a beautiful world!