---
layout: post
title: "Differences of SVD methods in numpy, tensorflow and pytorch"
date: 2019-03-28
excerpt: "Learn the subtle differences between the similar SVD API"
tags: [python, machine learning]
comments: true
---

* toc
{:toc}

## Introduction

SVD decomposition is frequently used in problems across various disciplines including machine learning, physics and statistics. In this short post, I won't discuss the formulas and backgrounds of SVD. Instead,  this post focuses on the subtle differences of SVD methods in numpy, tensorflow and pytorch, which are all called in python environment.

## Differences

For how to use SVD methods with the three libraries and the subtle differences between them, please refer to the following notebook (the version info of system and libraries are also included in case further changes on corresponding APIs). A very short introduction on SVD formulas is also included in the following notebook, and I recommend you to read the wiki page if you are still unfamiliar with the concept of SVD.

<div class="iframewrapper">
<iframe src="https://nbviewer.jupyter.org/gist/refraction-ray/ec8c633d727744123a56b4c8cf122c75" width="100%" height="2800"></iframe>
</div>

To summarize the difference from the above notebook, please refer the following table.

<div  class="table-wrapper" markdown="block">

|                                   |     numpy     |       tensorflow        |             pytorch              |
| :-------------------------------: | :-----------: | :---------------------: | :------------------------------: |
|       order of return tuple       |    u,s,vh     |          s,u,v          |              u,s,v               |
|     default return as reduced matrix     |      No       |           Yes           |               Yes                |
| whether return the transpose of v |      Yes      |           No            |                No                |
|   what returns if compute_uv is False   |    only s     |         only s          | u, s, v with u and v zero matrix |
|     accepted input data type      |   `np.array`    |   `np.array`, `tf.Tensor`   |        only `torch.tensor`         |
| returned data type | `np.array` | `tf.Tensor`| `torch.tensor` |
|           function name           | `np.linalg.svd` | `tf.svd` or `tf.linalg.svd` |            `torch.svd`            |

</div>

Another side note: in old version of pytorch, SVD API doesn't support broadcasting mechanism, this is fixed in recent version of torch, at least for pytorch 1.3.1.

## Summary

The three methods of course share some similarity. For example, the returned `s` are all represented in a vector form of singular values instead of the `S` matrix itself in SVD decomposition.

It is rather amazing to see that there are so many subtle differences between the similar APIs in these three libraries. Especially when, more or less, tensorflow and pytorch have given some promise on the consitent interface with numpy in terms of linear algebra operations. The fact in this short post (the same API can behave differently in many ways between the three libraries) reminds us to read the corresponding documentations carefully and don't take it for granted that the similar API should behave the same. That is to say, it may raise error when using functions in tensorflow and pytorch just as how they are used in numpy.