---
layout: post
title:  "TensorCircuit-NG"
date:   2020-06-24
excerpt:  "Next-gen AI-native tensor-network-based quantum software framework"
project: true
comments: true
---

**See the repo [here](https://github.com/tensorcircuit/tensorcircuit-ng).**

[![pypi](https://img.shields.io/pypi/v/tensorcircuit-ng.svg?logo=pypi)](https://pypi.org/project/tensorcircuit-ng/)
[![docs](https://img.shields.io/badge/docs-link-green.svg?logo=read-the-docs)](https://tensorcircuit-ng.readthedocs.io/)
[![stars](https://img.shields.io/github/stars/tensorcircuit/tensorcircuit-ng?style=flat)](https://github.com/tensorcircuit/tensorcircuit-ng/stargazers)
[![license](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](https://github.com/tensorcircuit/tensorcircuit-ng/blob/master/LICENSE)

Try the package by ``pip install tensorcircuit-ng``.

## From tensorcircuit to TensorCircuit-NG

tensorcircuit started in 2020 as a simple circuit API on top of Google's TensorNetwork package, with a unified AD interface so that the same code runs on TensorFlow, PyTorch and Jax backends. The [original repo](https://github.com/refraction-ray/tensorcircuit) is now archived.

TensorCircuit-NG is its technical successor, led and maintained by the original development team, and a fully compatible drop-in replacement (``import tensorcircuit as tc`` still works).

## Introduction

TensorCircuit-NG is a high-performance quantum software framework built on tensor network engines and modern ML frameworks (Jax, TensorFlow, PyTorch). It supports automatic differentiation, JIT compilation, hardware acceleration, vectorized parallelism and multi-node multi-GPU distributed simulation, and composes quantum circuits, neural networks and tensor networks seamlessly.

Circuit types cover ideal (``Circuit``), noisy (``DMCircuit``), Clifford (``StabilizerCircuit``), qudit (``QuditCircuit``), approximate MPS (``MPSCircuit``), analog (``AnalogCircuit``), U(1) symmetric (``U1Circuit``) and fermionic Gaussian (``FGSCircuit``) simulation, together with quantum hardware access.

It is also designed as an AI-native research runtime: the repo ships ``AGENTS.md``, 200+ example scripts and built-in agentic skills, so that coding agents such as Claude Code or Codex can turn research questions into runnable, optimized and validated quantum experiments.

```python
import tensorcircuit as tc

tc.set_backend("jax")

def forward(theta):
    c = tc.Circuit(2)
    c.H(0)
    c.CNOT(0, 1)
    c.rx(1, theta=theta)
    return tc.backend.real(c.expectation_ps(z=[0, 1]))

g = tc.backend.jit(tc.backend.grad(forward))
```

## Ecosystem

- [TenCirChem-NG](https://github.com/tensorcircuit/TenCirChem-NG): quantum computational chemistry based on TensorCircuit.
- [TenCirPauli](https://github.com/tensorcircuit/TenCirPauli): Rust-native companion for Pauli propagation and U(1) circuit execution.
- [Quantum computing lectures with TC-NG](https://github.com/sxzgroup/qc_lecture).

## Papers

1. S.-X. Zhang et al., TensorCircuit: a Quantum Software Framework for the NISQ Era, Quantum **7**, 912 (2023), [arXiv:2205.10091](https://arxiv.org/abs/2205.10091).
2. TensorCircuit-NG: A Universal, Composable, and Scalable Platform for Quantum Computing and Quantum Simulation, [arXiv:2602.14167](https://arxiv.org/abs/2602.14167).

## Blogs

- [TensorCircuit-NG：下一代科研基础设施](/TensorCircuit-NG-下一代科研基础设施/)
- [基于 TensorCircuit-NG 的千比特万参数全梯度量子算法训练](/基于-TensorCircuit-NG-的千比特万参数全梯度量子算法训练/)
