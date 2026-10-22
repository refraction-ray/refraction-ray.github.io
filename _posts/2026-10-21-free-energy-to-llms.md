---
layout: post
title:  "Free Energy to LLMs"
date:   2026-10-21
excerpt:  "生成模型与概率统计：从变分自由能到大模型的推理和训练，开源讲义与课件"
project: true
comments: true
tags: [machine learning, physics, statistics]
---

**See the repo [here](https://github.com/refraction-ray/free-energy-to-llms).**

**PDF 下载见 [Releases](https://github.com/refraction-ray/free-energy-to-llms/releases/latest)。**

![](https://raw.githubusercontent.com/refraction-ray/free-energy-to-llms/main/docs/images/hero.png){:width="80%"}

## Introduction

一本从数学推导一路写到工程实现的生成模型教材，以统计物理和概率的视角贯穿始终。

讨论每个模型时都回答四个问题：它表示什么分布，用什么目标学习，怎样采样，真正的瓶颈在哪里。沿着这条线，全书从 VAE、Flow、GAN、扩散和能量模型讲到 Transformer、多模态和强化学习后训练，再讲到训练系统、推理系统、智能体系统，以及 DeepSeek 与 Kimi 的模型架构创新，最后回到生成模型在科学计算中的用例。

- 约 150 页，11 章正文加 2 个附录
- 34 幅自绘矢量插图，全部用 TikZ 写成，源码随书开放
- 134 篇参考文献，覆盖经典论文与 2025–2026 年的技术报告

## Contents

| 章 | 主题 |
|---|---|
| 1 | 从函数拟合到概率分布 |
| 2 | 主流生成模型：表达、训练与采样 |
| 3 | Transformer |
| 4 | 多模态：理解与生成 |
| 5 | 训练方法：从预训练到可验证奖励 |
| 6 | 训练系统 |
| 7 | 推理系统 |
| 8 | 智能体系统 |
| 9 | DeepSeek 与 Kimi 架构案例 |
| 10 | 统计学习与数据模式：常见陷阱 |
| 11 | 把生成模型放回科学计算 |

## Slides

配套课件按每次开课分目录放在 [`slides/`](https://github.com/refraction-ray/free-energy-to-llms/tree/main/slides)

## License

文字、插图、幻灯片及其源码采用 [CC BY-NC-SA 4.0](https://github.com/refraction-ray/free-energy-to-llms/blob/main/LICENSE) 协议。勘误和建议欢迎提 [Issues](https://github.com/refraction-ray/free-energy-to-llms/issues) 或 Pull Request。
