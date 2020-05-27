---
layout: post
title: "Proof on one formula in DQMC"
date: 2020-05-27
excerpt: "Demystify the trace to det magic"
tags: [physics]
comments: true
---

* toc
{:toc}

In this post, we give the proof on the key formula in DQMC computation:

$$
\mathrm{Tr}(\prod_i e^{\hat{h}_i}) = \det(I+\prod_i  e^{h_i}), \label{main}
$$

where $$\hat{h} = c^\dagger h c$$ , $$h$$ is N by N Hermtian matrix and c here is fermion operator. This relation is exact no matter $$h$$ is small matrix or not. It is worth noting although the det is applied on N by N matrix, the trace is instead applied on the $$2^N$$ Hilbert space for quantum states. More specifically, 

$$
\mathrm{Tr}(\prod_i e^{\hat{h}_i}) = \sum_n\langle n\vert \prod_i e^{\hat{h}_i}\vert n\rangle ,
$$

where $$\vert n\rangle$$ is quantum many-body state. 

We presented the proof in 2 by 2 by 2 fashion: the first 2 is that we first show the formula holds when the product involves only one term, i.e.

$$
\mathrm{Tr e^\hat{h}} = \det (1+e^h). \label{1term}
$$

The second 2 is that we give proofs based on both purely second quantization language and path integral like language.

The last 2 is that, we can also show the bosonic version of $$\eqref{main}$$:

$$
\mathrm{Tr}(\prod_i e^{\hat{h}_i}) = 1/\det(I-\prod_i  e^{h_i}), \label{boson}
$$

## Some lemmas

* Lemma 1: For fermion $$c$$, $$\langle n\vert e^{c^\dagger \Lambda c}\vert n\rangle=\prod_i (1-n_i+e^{\lambda_i} n_i)$$, for boson $$b$$, $$\langle n\vert e^{b^\dagger \Lambda b} \vert n\rangle=\prod_i (e^{\lambda_i n_i}) $$, where $$\Lambda$$ is diagonal matrix with diagonal elements $$\lambda_i$$, and $$\vert n\rangle$$ is the direct product state in particle number space. The two relations can be easily shown by Taylor expansion. It is worth noting that $$(\hat n)^2 = \hat{n}$$ for fermion, while it is not true for boson since no Pauli exclusion princinple for bosons.

* Lemma 2: For fermion $$c$$, and coherent state $$\vert \psi\rangle, \vert\phi\rangle$$:

  $$
  \langle \phi\vert e^{c^\dagger \Lambda c}\vert \psi\rangle=\langle \phi\vert\prod_i (1-n_i+e^{\lambda_i} n_i)\vert\psi\rangle = \langle \phi\vert \psi\rangle\prod_i ( (1+(e^{\lambda_i}-1)\bar{\phi}\psi)\\
  =\prod_i e^{\bar{\phi_i}\psi_i}e^{(e^{\lambda_i}-1)\bar{\phi}_i\psi_i} =\prod_i e^{e^{\lambda_i}\bar{\phi_i}\psi_i}  = e^{\bar{\phi}e^\Lambda\psi}.
  $$

  There are so many subtle points in the above proof, though it is obvious at first glance. 1) for 1st equal sign, we utilize Taylor expansion of exponential and the fact that  $$(\hat n)^2 = \hat{n}$$ for fermion. 2) for 2nd equal sign, we need a normal order of $$c^\dagger_i, c_i$$ to evaluate them at $$\psi$$ or $$\phi$$, we are lucky enough as the product is between different $$c_i$$ and thus no delta emerges for normal order communtator. After we replace all operators into Grassmann numbers, we reorder them back, and the sign canceled, and we are back with the factor form with grassmann numbers replacing the operators. 3) We utilize the fact that, for Grassman number $$\eta_i$$,  $$e^{a \eta_1\eta_2}=1+a\eta_1\eta_2$$.

   It is easy to show that, this lemma holds for general Hamiltonian instead of diagonal matrix, i.e.

  $$
  \langle \phi\vert e^{c^\dagger H c}\vert \psi\rangle=e^{\bar{\phi}e^H\psi}.
  $$

  The same formula also holds for bosons. From the above arguments, we only need to show $$\langle\phi\vert e^{\lambda b^\dagger b}\vert\psi\rangle = e^{\bar{\phi}e^\lambda\psi}$$, and the general formula is easy to generalize for Hermitian matrix H and multiple species of boson b. We can show it is true by brute force together with lemma 1.

  $$
  \langle\phi\vert e^{\lambda b^\dagger b}\vert\psi\rangle = \langle 0\vert e^{\bar{\phi}b}e^{\lambda b^\dagger b}e^{\psi b^\dagger}\vert 0\rangle\\
  =\sum_{m=0}^\infty\sum_{n=0}^\infty \langle m\vert \sqrt{m!}\frac{\bar{\phi}^m}{m!}e^{\lambda b^\dagger b}\frac{\psi^n}{n!}\sqrt{n!}\vert n\rangle \\
  =\sum_{m=0}^\infty e^{\lambda m}\frac{ (\bar{\phi}\psi)^m}{m!} = e^{\bar{\phi}e^\lambda\psi}.
  $$



## proof for $$\eqref{1term}$$

### second quantization language

Since h is Hermitian which can be diagonalized with real diagonal terms, we have:

$$
U^\dagger\Lambda U = h,
$$

where $$U$$ is unitary matrix. Therefore, 

$$
\hat{h} = c^\dagger U^\dagger \Lambda Uc = d^\dagger \Lambda d,
$$

where we let new "particles" $$d=Uc$$, this works since $$[d_i^\dagger, d_j]_\zeta = [c^\dagger U^\dagger, Uc]_\zeta = \delta_{ij}$$. We trace the system using direct product state $$\vert n\rangle$$ in $$d$$ basis, we have (with the help of lemma 1):

$$
\mathrm{Tr}(e^{\hat{h}}) = \sum_n \langle n\vert e^\hat{h}\vert n\rangle =\sum_{\{n\}} \prod_i (1-n_i+e^{\lambda_i} n_i)\\
=\prod_i \sum_{n_i=0,1}(1-n_i+e^{\lambda_i} n_i) = \prod_i (1+e^{\lambda_i}) \\= \det (I+e^\Lambda) = \det(U^\dagger)\det(I+e^\Lambda)\det(U) = \det(I+e^H).
$$

Similarly for boson system, we have:

$$
\mathrm{Tr}(e^\hat{h}) = \prod_i \sum_{n_i=0}^{\infty} e^{\lambda_i n_i}=\prod_i \frac{1}{1-e^{\lambda_i}}=\\
1/\det(1-e^\Lambda) = 1/\det(1-e^H).
$$

### path Integral like language

For fermions (where we omit irrelavant measures for complete coherent states and also note the anti PBC for fermions):

$$
Tr(e^\hat{h}) = \int d\bar{\phi}d\phi  e^{-\bar{\phi}\phi}\langle-\phi\vert  e^\hat{h}\vert\phi\rangle \\
= \int d\bar{\phi}d\phi e^{-\bar{\phi}\phi}e^{-\bar{\phi}e^h\phi}=\det(I+e^h).
$$

It is straightfoward to generalize to boson, there is PBC and the Gaussian integral gives $$1/\det$$, which together exactly gives $$1/\det(I-e^h)$$.

## proof for $$\eqref{main}$$

To show the general case, we use induction. If we can show the following:

$$
e^h=e^{h_1}e^{h_2} \rightarrow e^{\hat{h}} = e^{\hat{h_1}}e^{\hat{h_2}}.
$$

Then it is obvious that $$e^h=\prod_ie^{h_i}\rightarrow e^{\hat{h}} = \prod_i e^{\hat{h}_i}$$. Then we have:

$$
Tr(\prod_i e^{\hat{h}_i}) = Tr(e^{\hat{h}}) = \det(I+e^h) = \det(I+\prod_i e^{h_i}).
$$

### second quantization language

We directly utilize Hausdorff formula, if $$e^Z=e^Xe^Y$$, we have 

$$
Z = X+Y+\frac{1}{2}[X,Y]+\frac{1}{12}[X, [X, Y]]+...
$$

So basically $$h=h_1+h_2+\frac{1}{2}[h_1, h_2]+\frac{1}{12}[h_1, [h_1, h_2]]+…$$  Using the same formula, we have $$\hat{h}  = c^\dagger h_1c +c^\dagger h_2 c+\frac{1}{2}[c^\dagger h_1 c, c^\dagger h_2 c]+\frac{1}{12}[c^\dagger h_1 c, [c^\dagger h_1 c, c^\dagger h_2c]]+…$$

Again, we ask help for induction. If we can show that $$[c^\dagger h_1 c, c^\dagger h_2 c] = c^\dagger [h_1, h_2] c$$, then the higher order nested commutator with operators can be flattened by induction as $$c^\dagger […[…]] c$$. And we thus complete the proof. We now show the commutator relation is true:

$$
[c_i^\dagger h^1_{ij}c_j, c^\dagger_k h^2_{km}c_m]= h^1_{ij}h^2_{km}[c_i^\dagger c_j, c^\dagger_k c_m ]\\
=h^1_{ij}h^2_{km}(c^\dagger_i c_m \delta_{jk}+\zeta c_k^\dagger c_j\delta_{im})=c^\dagger[h_1, h_2]c.
$$

Note the above derivation and argument hold true for both boson and fermion systems.

### path integral like language

In fact, we needn't prove such a strong statement, the following is enough:

$$
e^h=e^{h_1}e^{h_2} \rightarrow Tr(e^{\hat{h}}) = Tr(e^{\hat{h}_1}e^{\hat{h}_2}).
$$

We tackle this again by path integral and lemma 2.

$$
Tr (e^{\hat{h}_1}e^{\hat{h}_2})=\int d\bar{\phi}_1d\phi_1d\bar{\phi}_2d\phi_2 e^{-\bar{\phi}_1\phi_1-\bar{\phi_2}\phi_2}\langle \zeta\phi_1\vert e^{\hat{h_1}}\vert \phi_2\rangle\langle\phi_2\vert e^{\hat{h}_2}\vert \phi_1\rangle\\
=\int d\bar{\phi}_1d\phi_1d\bar{\phi}_2d\phi_2e^{-\bar{\phi}_1\phi_1-\bar{\phi_2}\phi_2+\zeta\bar{\phi}_1e^{h_1}\phi_2+\bar{\phi}_2e^{h_2}\phi_1}. \label{half}
$$

If we now do the Gaussian integral with both $$\phi_1, \phi_2$$, we are back to $$\det(I+e^{h_1}e^{h_2})$$, which shows n=2 is true (we utilize the fact on determinant of block matrix, for details, see [^det]). Actually, we can use the same approach to directly prove the main theorem of this note explicitly for n>2 without induction. 

Here I instead show the approach that reduce $$\eqref{half}$$ to induction route we have discussed. To show this, we only integrate $$\phi_2$$ out and keep $$\phi_1$$. This is just another Gaussian integral with source field, we have:

$$
Tr (e^{\hat{h}_1}e^{\hat{h}_2})=\int d\bar{\phi}_1d\phi_1e^{-\bar{\phi}_1\phi_1+\zeta \bar{\phi_1}e^{h_1}e^{h_2}\phi_1}
\\=\int d\bar{\phi}_1d\phi_1e^{-\bar{\phi}_1\phi_1+\zeta \bar{\phi_1}e^{h}\phi_1} = Tr(e^\hat{h}).
$$

## After thought

What's the difference between conventional path integral approach in every textbook and the so called path integral here? It seems that there are some differences and they are not exactly the same thing.

For example, in our formalism, no $$\bar\psi \partial_\tau\psi$$ terms or its discretazation like $$\bar\psi_1(\psi_0-\psi_1)$$ occurs. Since the fomalism here is also introduced by insert coherent states into the quantum trace, why the final formalism is deviating from convential path integral?

Firstly, the formalism in this note is EXACT, it doesn't only hold when $$h_i$$ is small, i.e. by time slicing. It is always true no matter what the h matrix is. All the issues for h getting bigger like commutations are carefully taken care of. On the contrast, path integral formalism only holds when $$\Delta\tau\rightarrow 0$$, i.e. $$h_i$$ is very small that commutators as higher order of $$\Delta \tau$$ are omitted. 

The key here is actually Lemma 2. $$\langle \phi\vert e^{c^\dagger H c}\vert \psi\rangle=e^{\bar{\phi}e^H\psi}$$ is exactly true. But in path integral formalism, we approximate this as $$e^{\bar{\phi}\psi+\bar{\phi}H\psi}$$. And the first term together with the measure term $$e^{-\bar{\phi}\phi}$$ gives the difference $$\partial_\tau$$ term. The remaining one is the familiar $$e^{\hat{H}}$$ term in path integral. So our formalism here is more "correct" than the conventional path integral due to lemma 2 instead of approximating it. But then why not path integral formalism utilize lemma 2 directly from the beginning? You may ask. The reasons are 1) lemma 2 cannot handle interacting Hamiltonian with four fermion terms, it only work for bilinears 2) after lemma2, the path integral formalism is hard to take continuous time limit, since now direvatives $$\partial_\tau$$ is somehow missing and some terms in the new formalism will have no good continuous limit. If you carry out the limit carefully, you are back to the conventional path integral formalism: which is not exact like lemma 2 but is actually exact in the continuous time limit.

So to combine path integral and things here and to confuse everyone, we may have some mixing like:

$$
\int d\bar{\phi}_\tau d\phi_\tau e^{\int_0^\beta d\tau ( -\bar{\phi}_\tau\partial_\tau\phi_\tau+\bar{\phi}_\tau H_\tau\phi_\tau)}=\det(I+\zeta\mathcal{T}e^{\int^\beta_0 d\tau H_\tau})^\zeta.
$$


## References

[^det]: [Proof on block matrix determinant](http://www.ee.iisc.ac.in/people/faculty/prasantg/downloads/blocks.pdf)