---
layout: post
title: "Gauge Problem in Automatic Differentiation"
date: 2019-11-06
excerpt: "Subtlety of gauge problem in AD and relevant AD formula for SVD and EIG"
tags: [machine learning, algebra]
comments: true
---

* toc
{:toc}

## Introduction

Automatic differentiation infrastructure has been integrated with every machine learning libraries. However, there are still some missing and wrong implementations of AD formula in terms of involved linear algebra operations such as SVD or eigen decompositions. The case becomes even worse when complex numbers play roles in these linear algebra operations let alone some ML libraries don't support complex numbers at all. 

Although there are many famous technical reports with lists of AD formulas for linear algebra operations, they either focus on real case[^lehmann][^walter][^lawrence] or, as I will show below, often **wrong** with missing terms or wrong in the proof (though a wrong proof can still lead to correct AD formula in some cases)[^umbach][^giles]. 

Therefore, in this post, I will explain how to derive the correct AD formula for complex valued SVD and general eigen decompositions (which is complex by nature). I will also explain how the formulas and proofs in literatures are wrong or imperfect. The failure of previous proofs is rooted in the notorious gauge problem in these linear algebra operators, and thus I will demonstrate how to incorporate gauge freedoms in AD context. The basic knowledge on automatic differentiation, complex calculus[^complex] and linear algebra is required to understand this post.

## Hello world example

As every tutorial style post, let's begin with a hello world example for AD with gauge problem. The forward operation $$y, s = f(x)$$ is defined as $$x=ys\frac{1}{y}$$, where $$y^*y=1$$ and $$x,y,s$$ are all scalars. Note this example is very artificial though it actually corresponds to 1D eigen decomposition.

### General framework for AD

In general, both forward mode AD and reverse mode AD formula can be derived from function relations 

$$
dy(dx),~ ds(dx).\label{d}
$$

And theoretically, we believe such relations can always be found, as they describe the change of outputs with the change of inputs which is totally determined by the well-defined function operation. If we find $$\eqref{d}$$, the forward mode AD formula is just:

$$
y'=dy(x'),~s'=ds(x').\label{f}
$$

Note $$dy$$ in $$\eqref{f}$$ stands for the function form defined in $$\eqref{d}$$ instead of the differentials. As for reverse mode AD, we have the full differentiation of loss function $$L$$ as:

$$
dL=\mathrm{Tr} [\overline{x}^Tdx+\overline{x}^\dagger dx^*]=\mathrm{Tr} [\overline{y}^Tdy+\overline{y}^\dagger dy^*
+\overline{s}^Tds+\overline{s}^\dagger ds^*].\label{r}
$$

This formula is general for matrix and complex numbers. The adjoint is defined as $$\overline{x}=\frac{\partial L}{\partial x}$$, and $$\dagger$$ is for Hermitian conjugate. Since we have the knowledge of dependence $$dy(dx)$$ and so on, we could replace all $$dy, ds$$ on RHS with $$dx$$, and the term before $$dx$$ on RHS is just same as $$\overline{x}^T$$. By this, we can derive reverse mode AD formula as function dependence $$\overline{x}(\overline{y}, \overline{s})$$ by comparing coefficients before $$dx$$.

### Failed Attempts

The above is the general framework for finding AD formula for any operators. The core relation is $$\eqref{d}$$, and we must find such function dependence of differentials. Let's now see how we can apply the above procedure to the hello world example here (spoilers: you will fail).

To get $$\eqref{d}$$, we differentiate functions and restrctions as:

$$
dx = dy \,s\frac{1}{y}+y \, ds\, \frac{1}{y}+ys\, (-\frac{dy}{y^2})=ds\\
dy^*\, y +y^*\,dy = 0.
$$

One can simply find $$ds(dx)=dx$$, but it is obvious one can never find $$dy(dx)$$. $$dy$$ and $$dx$$ seems to decouple, namely $$dy$$ has become a free differential that doesn't depend on the differential of the inputs. If we cannot find $$\eqref{d}$$ from the beginning, how can we get the formula for AD in both forward and reverse mode? Where is the problem?

### Gauge problem

The problem is that the function primitive $$y,s = f(x)$$ is somehow ill-posed, i.e. $$y, s$$ is not unique given input $$x$$. Say $$y, s$$ is one output from $$x$$, then $$y\lambda, s$$ must also be an legal output from the same input $$x$$ when $$\lambda=e^{i\theta}$$ is a phase. We can verify this by $$y\lambda s \frac{1}{y\lambda}= x$$ and $$y^*\lambda^*\lambda y=1$$. There are infinite outputs that satisfying the definition of $$f$$. We call such extra freedom of choosing outputs $$\lambda$$ as gauge. Functions with gauge freedom make sense as long as the final objective(loss function) is independent of the gauge. For example, we can define loss function $$L=\vert {y}\vert^2$$ here, and such loss function is independent of $$\lambda$$ (the objective is trivial as our hello world example is over simplified, but you get the idea). 

To summarize, some function or linear algebra operations cannot uniquely determine the outputs based on inputs, the extra freedom is denoted as gauge. The whole setup still make sense as long as the final objective is independent of the gauge.

### Gauge invaraince as a condition

To make the definition of $$f$$ valid, we have another condition here: the loss function must independent of $$\lambda$$. i.e. $$L(y, s)=L(y\lambda, s)$$.

$$
dL = \mathrm{Tr}[\overline{y}^Tdy+\overline{y}^\dagger dy^*+\overline{s}^Tds+\overline{s}^\dagger ds^*]\\
=\mathrm{Tr}[\overline{y\lambda}^T d(y\lambda)+\overline{s}^Td{s}+c.c.].
$$

By taking $$\lambda=1$$ we can have:

$$
\mathrm{Tr}[\overline{y}^Ty\,d\lambda+(\overline{y}^Ty)^*\, d\lambda^*]=0.\label{gi}
$$

It is worthing emphasizing that $$L(y, s)=L(y\lambda, s)$$ is not strong enough to conclude $$\frac{\partial L}{\partial \lambda}=0$$. Instead, gauge invariant loss function only guarantee $$\eqref{gi}$$. Since we know $$d\lambda = e^{i\theta}d\theta$$, when $$\lambda=1$$, $$d\lambda = id\theta$$, $$d\lambda^*=-id\theta=-d\lambda$$. Back to 1D case for our hello world example, we have the following condition from gauge invariance:

$$
\overline{y}y\in R.\label{gic}
$$

At first sight, this relation is too weak to utilize. It just states something must be real. But let's now directly tackle $$\eqref{r}$$ without knowing exactly $$\eqref{d}$$ (which is impossible to uniquely determine due to gauge problem):

$$
dL=\overline{x}dx+c.c.=\overline{s}ds+ c.c.+\overline{y}dy+\overline{y}^*dy^*\\
=\overline{s}ds+ c.c.+\overline{y}yy^{-1}dy+\overline{y}^*y^{*}y^{*-1}dy^*\\
=\overline{s}dx+c.c. +(\overline{y}y)[y^{-1}dy+(y^{-1}dy)^*]\\
=\overline{s}dx+c.c.
$$


In the third line, we have utilized the gauge invariance condition $$\eqref{gic}$$, and in the last line, we have utilized the fact $$y^*y=1$$, which gives $$\frac{dy}{y}+\frac{dy^*}{y*}=0$$. Therefore, the reverse mode AD formula is $$\overline{x}=\overline{s}$$.

### Hindsight and gauge fixing approach

After the derivation, we found that $$\overline{y}$$ indeed has no contribution to $$\overline{x}$$. But this is different from $$dy=0$$. $$dy=0$$ is not true and this is the error various works have made before. Say $$y=e^{i x^*x}$$, which is a legal output according to the definition of $$f$$, the corresponding $$dy$$ is for sure non zero. You may argue that by an appropirate gauge fixing scheme, $$dy$$ could indeed be zero and there is no need to worry about gauge problem. It is true for this simple hello world example, but in more complex case like eig or svd, it is nearly impossible to fix gauge in a natural way. And simply taking some free differentials to be zero is not only wrong in the proof but can also be wrong in the final result. In this hello world example, $$dy=0$$ happen to a legal gauge fixing scheme which is not the case for more general settings. 

More specifically, one can for sure derive correct AD formula by gauge fixing, and such approach is in general easier especially in terms of forward mode AD as we will discuss in the following. In our context, differentials of function definitions for operators cannot fully determine the relation between differentials of outputs and inputs. There must be some differentials of outputs are free from the function definition constraint due to gauge freedom for the outputs. In our toy example, the free differential is just $$dy$$.  But it is not always a correct and safe gauge fixing scheme by simply let such free differentials to be zero. This is because there might be extra restriction conditions on outputs which impose some intrinsic restriction on the free differentials. Such restriction may enforce nonzero nature for free differentials. In other words, it is impossible to make free differentials to be zero no matter which gauge is chosen. That being said, after careful consideration on the restrictions, we could fix the gauge by specify some restriction-consistent form to free differentials. And a correct gauge fixing scheme is in general simpler than doing all stuff in a gauge invariant style. 

In retrospect, since there is extra gauge freedom, we have no way to derive the full formula of differentials in $$\eqref{d}$$ form uniquely as $$dx$$ itself cannot fully determine $$dy, ds$$. Therefore, we must ask help for the condition from gauge invariant loss functions. By integrating this condition into $$\eqref{r}$$, we could hopefully get rid of the direct need of $$dy(dx)$$. 

### Forward mode AD

Even after we have derived the reverse mode AD formula for this toy operation in gauge invariant fashion, we still have no idea about $$dy(dx)$$ which is actually somehow arbitrary due to gauge problem. But we need $$dy(dx)$$ to obtain formula for forward mode AD. There are two approaches to achieve this. The first one is somehow in a gauge invariant fashion by asking help for revese mode formula $$\eqref{r}$$. The philosophy is like do reverse mode twice[^rrf] to get forward mode. After we have obtained $$\overline{x}(\overline{y}, \overline{s})$$, we have:

$$
\mathrm{Tr}[\overline{x}^T(\overline{y}, \overline{s}) dx+c.c.]=\mathrm{Tr}[\overline{y}^Tdy+\overline{s}^Tds+c.c.].
$$

By comparing the coefficients before $$\overline{y}^T$$, we have the effective relation $$dy(dx)$$. Such relation only hold for some fixed gauge.

The second approach is to directly utilize $$\eqref{f}$$ types of equations by gauge fixing. This approach is more straightforward to obtain possible forward mode AD formula. However, as we mentioned in the last subsection, one should by very careful about gauge fixing. Gauge fixing is by no means just taking missing differentials as zero. The restrictions on ouputs shipped with the function definition can enforce nonzero nature of such differentials.

We follow the first approch here. In our toy example, we have:

$$
\overline{x}dx=\overline{s}ds=\overline{y}dy+\overline{s}ds.
$$

By comparing coefficients before $$\overline{y}, \overline{s}$$, we have $$dy=0$$. Such effective $$dy=0$$ is not the same thing of directly dropping $$dy$$ at the stage of proof for reverse mode AD. Again, this over simplified example is not good enough to show this subtle difference. Here, $$dy=0$$ is given by reversing the reverse AD formula. We now have the forward mode AD for this example as: $$y'=0, s'=x'$$. 

### Freedom in AD formula

This is not the end of the story for hello world example. We can actually design many "correct" formulas for reverse mode AD and forward mode AD. For example, the reverse mode AD formula can be written as $$\bar{x}=\bar{s}+y\bar{y}-y^*\bar{y}^*$$. Although the form of the formula can be different, the numerical value is the same for $$\bar{x}$$. Similarly,we can then obtain the forward mode AD formula as $$y'=2y \mathrm{Im} x'$$. Suprisingly, different version of forward AD formula give different $$y'$$ even in numerical value sense. Forward AD still makes sense with gauge freedom since $$l'=\bar{y}y'+\bar{s}s'$$ is the same due to the gague invariance objective $$l$$, no matter how different of $$y'$$ can be.

### Summary on the problem structure

We learn about the general strcture of the problem from this toy example. There are three parts determining a well-defined forward operator: function, restriction and gauge. The function defines how outputs depends on the inputs like $$x=ysy^{-1}$$ here. However, such function alone cannot fully determine the outputs in some cases and we need extra restrictions to describe the operation, i.e. $$y^*y=1$$ here. There could still be some extra freedoms for the outputs after we have included restrictions, these remainning freedoms are thus denoted as gague freedom. In other word, guage freedom and restriction conditions on outputs are two sides of the same coin. All extra freedoms left with function definition should either go to restrictions or gauge freedoms.

In the following, we will discuss two specific AD examples with gauge problem, they are EIG and SVD.

## AD for general eigen decomposition

### Definition

* Input: Matrix A, real or complex valued
* Output: Matrix U, real or complex valued, U could be complex when A is real. Diagonal matrix E, E could also be complex when A is real. And actually, U or E is literally always be complex valued when the dimension of matrix increases.
* Function: $$AU=UE$$
* Restriction: Every column vector of U (right eigen vector of A) is normalized, i.e. $$I\circ (U^\dagger U)=I$$, where $$\circ$$ denotes elements multiplication.
* Gauge: Based on the above condition, the outputs are not unique. The gauge freedom is diagonal matrix $$\Lambda$$ whose elements are all in phase form $$e^{i \theta}$$. If U, E are legal outputs,  $$U\Lambda, E$$ are also satisfying all the above relations.

### Free differential

Let's try to get the AD formula following the philosophy of hello world example. The first step is differentiate the function, and find the so-called free differentials, i.e. differentials of some outputs decoupled from function relation.

The differentiation of  function is given by:

$$
dA\,U+AdU=dU E+UdE.
$$

Multiply $$U^{-1}$$ on the left, we have:

$$
U^{-1}dA \,U+EdC = dC E+dE,\label{dfeig}
$$

where $$dC=U^{-1}dU$$. By $$\eqref{dfeig}$$, we can extract equation from diagonal part as well as off-diagonal part, which are:

$$
dE = I\circ(U^{-1}dA\,U)\\
\bar{I}\circ dC = F\circ (U^{-1} dA\, U) \label{dedc},
$$

where $$\bar{I}$$ is matrix with all elements one but zero diagonal terms. F is defined with zero diagonal elements and off diagonal terms as:

$$
F_{ij}=\frac{1}{E_j-E_i}.
$$

Since $$\eqref{dedc}$$ is all information we have on differentials of outputs, $$I\circ dC$$ term is missing. $$I\circ dC$$ is the free differential in this problem due to gauge freedom $$\Lambda$$. But we cannot simply set it as zero by gauge fixing arguments, since there is still restrictions in the definition of eig operation. Therefore, the value of $$I\circ dC$$ must satisy the following restriction on the inner structure of $$I\circ dC$$:

$$
I\circ(U^\dagger dU+dU^\dagger\, U)=0.\label{dres}
$$


### Gauge fixing route

Let's see how $$\eqref{dres}$$ impose restrictions on $$I\circ dC$$, recall $$dC=U^{-1}dU$$.

$$
0=I\circ(U^\dagger dU+h.c.)=I\circ(U^\dagger U dC+h.c.)\\
=I\circ(U^\dagger U (I\circ dC+\bar{I}\circ dC)+h.c.)\\
=I\circ(U^\dagger U)I\circ dC +I\circ(U^\dagger U \bar{I}\circ dC)+h.c.\\
=I\circ dC+I\circ(U^\dagger U (F\circ(U^{-1}dA U)))+h.c.
\label{idc}
$$

Therefore, we cannot safely set $$I\circ dC$$ zero as $$I\circ(U^\dagger U (F\circ(U^{-1}dA U)))$$ is not zero in this gauge in general. But we can fix the gauge safely by simply let 

$$
I\circ dC = -I\circ (U^\dagger U(F\circ(U^{-1}dAU))).
$$

This gauge fixing is safe as it trivially satisy $$\eqref{idc}$$. And we could continue to derive both forward and reverse mode AD formulas, since we now have relations in the form of $$\eqref{d}$$.

The following is the derivation of backprop formula:

$$
\mathrm{Tr}(\bar{E}^TdE+\bar{U}^TdU+c.c.) \\
=\mathrm{Tr}(\bar{E}^TI\circ(U^{-1}dA\,U)+\bar{U}U (I\circ dC+\bar{I}\circ dC)+c.c.)\\
=\mathrm{Tr}(U\bar{E}^TU^{-1}dA+c.c.)+\mathrm{Tr}(\bar{U}^TU(F\circ (U^{-1}dA\,U)\\-I\circ(U^\dagger U(F\circ (U^{-1}dA\,U))))+c.c.)
\\=\mathrm{Tr}(U\bar{E}^TU^{-1}dA+c.c.)+\mathrm{Tr}((\bar{U}^TU\circ F^{T}) U^{-1}dAU+c.c.)\\-\mathrm{Tr}((\bar{U}^TU\circ I)U^\dagger U (F\circ(U^{-1}dA\,U))+c.c.)\\
=\mathrm{Tr}(U\bar{E}^TU^{-1}dA+c.c.)+\mathrm{Tr}(U(\bar{U}^TU\circ F^{T}) U^{-1}dA+c.c.)\\-\mathrm{Tr}(U((\bar{U}^TU\circ I)U^\dagger U \circ F^T)U^{-1}dA+c.c.).
$$


By comparing this with $$\mathrm{Tr}(\bar{A}^T dA+c.c.)$$, we have the backprop formula for general eig operation:

$$
\bar{A} =U^{-T}(\bar{E}+F\circ U^T\bar{U}-F\circ(U^TU^*(I\circ U^T\bar{U})))U^T.
\label{beig}
$$

This is consistent with 4.77 in [^umbach]. We will show in gauge invariant subsection that $$I\circ U^T\bar{U}$$ is guaranteed to be real and thus the real cast in [^umbach] is redundant.

Note backprop in that paper is on gradients while $$\eqref{beig}$$ is on derivatives. They are different up to complex conjugate. From implementation perspective, autograd backprops derivatives and should utilize exactly $$\eqref{beig}$$ while tensorflow backprops gradients and should use the formula in [^umbach].

The forward mode AD is also straightforward:

$$
E'=I\circ (U^{-1}A'U)\\
U'=UU^{-1}U'=U(I\circ C'+\bar{I}\circ C')=\\
U(-I\circ(U^\dagger U F\circ(U^{-1}A'U))+F\circ(U^{-1}A'U)).
$$


### Gauge invariant route

One can also derive the backprop formula with the knowledge that the objective is gauge invariant as $$L(U,E)=L(U\Lambda, E)$$. In this fashion, we can get rid of gauge fixing which is not so elegant. Though I have to admit gauge fixing is the simplest way to derive AD formula especially for forward mode. Obtainning forward mode AD formula in gauge invariant fashion may be very painful.

Starting from:

$$
dL=\mathrm{Tr}(\bar{U}^TdU+\bar{E}^TdE+c.c.)\\
=\mathrm{Tr}(\overline{U\Lambda}^TdU\Lambda+\bar{E}^TdE+c.c.),
$$

we conclude that gauge invariant loss function gives:

$$
\mathrm{Tr}(\bar{U}^TUd\Lambda +c.c.) = 0.\label{weakgi}
$$

It should be emphasized that gauge invariance cannot be taken as $$\bar{\Lambda}=0$$. In eig case, $$\bar{\Lambda}^T=\bar{U}^TU$$. Supoose an guage invariant $$L=U_{00}^*U_{00}$$, $$\bar{\Lambda}$$ is obvious nonzero in this case.

Let's look in detail on $$\eqref{weakgi}$$. In our case, the diagonal matrix $$d\Lambda$$ is made from elements like $$e^{i\theta}$$. When $$\Lambda = I$$, we can easily show that $$d\Lambda^* = - d\Lambda $$, and thus they are not independent. This is obvious, since each element of $$d\Lambda$$ is controlled by one real parameter $$\theta$$ instead of two. Therefore, we have:

$$
\mathrm{Tr}((\bar{U}^TU-c.c.)d\Lambda)=0
$$


Since each row of $$d\Lambda$$ is independent, the above formula actually indicates:

$$
I\circ \bar{U}^TU \in R.
$$

What if the gauge freedom is enlarged? For example, we can move the normalization condition for U (restrictions) to gauge freedom in eig forward pass settings. In that case, gauge $$\Lambda$$'s diagonal elements are in the form $$re^{i\theta}$$, namely every right eigenvector can be rescaled as normalization condition is gone. Therefore, $$d\Lambda$$ and $$d\Lambda^*$$ are independent and can vary freely. This indicates that 

$$
I\circ \bar{U}^TU=0,
$$

which is a much stronger condition since the gauge in enlarged. Correspondingly, the objective has to be independent on norm of eigenvectors. For example $$L=U^*_{00}U_{00}$$ is gauge invariant when gauge contains only phase, while it is not gauge invariant when norm is also included in gauge.

Now considering the relation in $$dL$$ (we only consider $$I\circ dC$$ part since other parts are trivial and have been shown in the last subsection):

$$
dL=\mathrm{Tr }(\bar{U}^TU I\circ dC+c.c.)\\
=\mathrm{Tr}((I\circ\bar{U}^TU)  2\Re(I\circ dC)),
\label{dlgi}
$$

where we have utilized the fact from gauge invariance: $$I\circ\bar{U}^TU$$ is real. According to $$\eqref{idc}$$, 

$$
2\Re(I\circ dC) = -I\circ (U^\dagger U(F\circ(U^{-1}dAU)))+c.c.
$$

Plug this into $$\eqref{dlgi}$$, we can arrive to the same formula as in the guage fixing case.

### Operation combination route

There is actually another route towards correct AD formula but free from subtlety of restrctions in the operation definition. In this approach, we consider the eig operation as the combination of two: op1. eig without normalization restriction and thus gauge freedom $$\Lambda_k =r_ke^{i\theta_k}$$; op2. normalization function on each columns of U.

As we have shown in the last subsection, for 1. gauge invariance guarantees that $$I\circ \bar{U}^TU=0$$. Therefore, the backprop formula is very simple to obtain for op1 since the most involved part of $$I\circ dC$$ is canceled as:

$$
dL = \mathrm{Tr}(\bar{U}^TUdC)=\mathrm{Tr}(\bar{U}^TU(I\circ dC+\bar{I}\circ dC))\\
=\mathrm{Tr}(I\circ \bar{U}^TU I\circ dC)+\mathrm{Tr}(\bar{U}^TU\bar{I}\circ dC)\\
= \mathrm{Tr}(\bar{U}^TU\bar{I}\circ dC).
$$

From this, we could actually derive the backprop formula in [^giles]. In other words, backprop formula in [^giles] is only valid when objective is irrelavant with the norm of eigenvectors. Such large gauge strongly suppresses the possibility of loss functions. And proof in [^giles] is not rigorous by directly dropping $$I\circ dC$$ terms instead of arguing from gauge invariance like what we have done here. [^umbach] also has the same loophole in the proof for op1.

We can now pipeline op2 following op1, and then consider the derivatives for the combination of operations. There are two approaches: 1. derive the derivatives for the composed operator by chaining the derivatives of individual operators by hand as [^umbach] (see details in [^umbach] P30); 2. utilize AD machinery by smartly defining the operator: include a normalization process in the operator definition. Even though `tf.linalg.eig` already returns normalized eigenvectors, we still pipeline this with a normalization function as the forward pass. By this, the backpropagation is automatically correct without any further derivations. 

In highsight, the difficuly of AD doesn't come from gauge but comes from restrictions. The larger the gauge, the smaller the restrictions, the easier to obtain AD formulas, though the apllicable range of loss functions is also smaller. With operation combination idea, we first derive the AD formula for functions with no restrictions on outputs and thus the largest gauge freedoms. And then by applying restriction equivalent transformations, we could obtain the AD formula for the combination of operators as a whole either by analytically derivation or by AD machinery.

### Side notes on real input case

After obtaining the correct backprop formula for eigen operation, there are still some issues when the input matrix A is real. The backprop formula gives complex results for real inputs and real loss functions(outputs). How could a function with real inputs/outputs give complex derivatives?

Well, the answer is that there are two different scenarios. 1). complex input/output happens to be real at the same time. For example $$y = \Re x+\Im x$$, the input x and output y can be both real. But the derivative is still $$\frac{\partial y}{\partial x}=\frac{1-i}{2}$$, which is a complex number. This is because there is no restriction for x to move toward imaginary axis. This argument applies to eig case. If the input matrix is allowed to vary freely in complex field but happen to be a real one, then it is ok to have complex derivatives. However, 2) if input/output is restricted to real number field then the derivative is the real part of the result from the AD formula $$\eqref{beig}$$.

This fact could also be explained as follows. If A is only allowed to vary in real number field, then $$dA=dA^*$$, $$\bar{A}=\bar{A}^*$$ , $$\eqref{r}$$ is reformulated as 

$$
dL=\mathrm{Tr}(2 \bar{A}^T dA)=\mathrm{Tr}(\bar{U}^T dU+\bar{E}^TdE+c.c.)=\mathrm{Tr}(2\Re (\bar{U}^T dU+\bar{E}^TdE)).
$$

Therefore, we now have $$\bar{A}=\Re[\bar{U}^T]$$. This formula demonstrates why we need to take real part of the derivative for the complex case.

So the better understanding on this issue is as follows: the general eigen operation can be viewed as the combination of `A=tf.cast(A, tf.complex)` follows by usual `E, U = tf.linalg.eig(A)`. From this perspective, the input of `eig` can be thought as always complex. Then it is fine to have complex derivatives $$\bar{A}$$, but such direvatives will be casted to real type when backpropagating through the cast op[^tfcast]. Therefore, whether we should take real part of the derivative results depends on ourselves: it is ultimately connected to whether complex input matrix $$A$$ is allowed.

Operations such as `eig` implies that complex number can naturally emerge from real valued input. In broader context, only complex number is natural for general operations. It is somehow silly to restrict all operations to real numbers in deep learning setups and omit the implementation of complex valued tensors in some machine learning libraries. Complex number is everywhere and this reflects an important math fact that the field of complex numbers is the largest algebraically closed field.

## AD for complex valued SVD

SVD is defined as $$A=USV^\dagger$$, where $$U, V$$ are both unitary and S is diagonal. It is easy to see there is gague freedom $$\Lambda$$ for this operation. Namely, $$U\Lambda, V\Lambda, S$$ are also legal outputs for SVD as long as $$\Lambda$$ is a diagonal matrix with phase elements in the form $$e^{i\theta}$$. Therefore complex valued SVD naturally falls into the category of operation with gauge freedoms. For details on the derivation of backprop formula for SVD, just check our work[^svd]. v3 of this preprint gives the correct proof, and it is also interesting to see how the first two versions of proof are problematic.

## Acknowledgement

Many ideas and derivations in this post are credited to my colleague [Zhou-Quan Wan](https://github.com/Zhouquan-Wan). We also thank [Jin-Guo Liu](https://github.com/GiggleLiu) and [Evgeniy Zheltonozhskiy](https://github.com/Randl) for relevant discussions[^discuss] on AD for SVD and EIG case respectively.


## References

[^svd]: [Automatic Differentiation for Complex Valued SVD](https://arxiv.org/abs/1909.02659).

[^rrf]: [A new trick for calculating Jacobian vector products](https://j-towns.github.io/2017/06/12/A-new-trick.html), also see very informative discussions at <https://github.com/HIPS/autograd/pull/175>.

[^lehmann]: [Algorithmic Differentiation of Linear Algebra Functions with Application in Optimum Experimental Design (Extended Version)](https://arxiv.org/abs/1001.1654v2).

[^walter]: [Structured Higher-Order Algorithmic Differentiation in the Forward and Reverse Mode with Application in Optimum Experimental Design](https://pdfs.semanticscholar.org/e52d/834dac4fdfdc4328105e29ae657c3fa51467.pdf).

[^lawrence]: [Auto-Differentiating Linear Algebra](https://arxiv.org/pdf/1710.08717.pdf).

[^umbach]: [On the Computation of Complex-valued Gradients with Application to Statistically Optimum Beamforming](https://arxiv.org/pdf/1701.00392.pdf).

[^giles]: [An extended collection of matrix derivative results for forward and reverse mode algorithmic differentiation](https://people.maths.ox.ac.uk/gilesm/files/NA-08-01.pdf).

[^complex]: [An Introduction to Complex Differentials and Complex Differentiability](https://mediatum.ub.tum.de/doc/631019/631019.pdf).

[^tfcast]: For the backprop implementation of `tf.cast`, see source [here](https://github.com/tensorflow/tensorflow/blob/9cee786abd939d1f5f495d439d1d0b6c5ea7af33/tensorflow/python/ops/math_grad.py#L1841-L1852).

[^discuss]: See relevant discussions on AD issue in SVD case [tf issue13641](https://github.com/tensorflow/tensorflow/issues/13641) and EIG case [tf PR33808](https://github.com/tensorflow/tensorflow/pull/33808). 


