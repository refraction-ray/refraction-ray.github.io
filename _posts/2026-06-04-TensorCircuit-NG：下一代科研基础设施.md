---
layout: post
title: "TensorCircuit-NG：下一代科研基础设施"
date: 2026-06-04
excerpt: "当宣传趋同时，如何判断一个量超智平台是否真正成熟可用？"
tags: [python, machine learning, quantum computation]
comments: true
---


* toc
{:toc}

## 引言

近年来，量子计算、人工智能（AI）与高性能计算（HPC）的融合正在成为全球科研基础设施演进的重要方向。从 AI4Science 到量子机器学习，从超算中心建设到异构算力协同，“量子×AI×HPC”“量超智融合”“下一代科研基础设施”等概念频繁出现在学术会议、产业论坛和企业宣传材料中。

然而，一个越来越明显的现象也正在出现：

**宣传材料趋同，而实际产品的技术深度差异巨大。**

无论是量子软件平台、AI4Science 基础设施，还是异构计算框架，越来越多项目开始使用类似的叙事方式：

- 统一量子与人工智能；
- 支持异构算力调度；
- 面向未来科研基础设施；
- 服务材料、化学、生物医药等产业场景；
- 构建开放生态与开发者社区。

这些方向本身没有问题。事实上，它们已经逐渐成为行业共识。

真正的问题在于：

**当大家都在讲相似的故事时，如何判断一个平台是否真正完成了技术落地，而不仅仅停留在概念包装和 PPT 展示阶段？**

对于科研基础设施而言，与其关注宣传口号，不如关注五个更容易验证的问题：

1. 是否开源？
2. 是否存在公开 Benchmark？
3. 是否被高质量科研社区持续使用？
4. 是否形成产业和行业应用案例？
5. 是否持续进行版本迭代和更新？

对于任何声称自己是“下一代科研基础设施”的平台而言，这五个问题都应该能够被客观回答。

[TensorCircuit-NG](https://github.com/tensorcircuit/tensorcircuit-ng) 平台的发展历程，提供了一个典型案例。它的价值不只在于提出“量子×AI×HPC”的愿景，更在于已经形成了可以被检查的代码、可以复现的性能测试、可以追踪的学术使用、可以观察的产业外溢，以及持续六年的工程迭代。


## 第一项标准：是否开源？

对于科研软件而言，开源并不仅仅意味着代码公开。

更重要的是，它意味着：

- 技术可以被验证；
- 性能可以被复现；
- 算法可以被检查；
- 用户可以独立部署；
- 第三方研究者可以在不依赖封闭服务的情况下重复实验结果。

科研领域从来不缺少优秀的宣传材料。真正稀缺的是能够被同行重复验证的技术体系。

TensorCircuit 的开放过程并不是一次性的代码发布，而是一条可以持续追踪的工程演进线索：从最初的个人开源版本，到腾讯量子实验室期间的开发版本，再到当前持续维护的 TensorCircuit-NG 版本，项目的核心代码、文档、测试和示例始终保持开放。GitHub 记录保存了完整开发历史，累计 star 和 fork 数超过 500，代码提交超过 2700 次，发行版本超过 30 个，并有来自世界各地的 30 余位开发者参与贡献。

从工程体量看，TensorCircuit-NG 已经不是一个短期概念验证项目，而是一个平台级科学计算软件。项目累计包括约 7 万行代码，配套类型注释、单元测试、持续集成、文档和教程体系。目前仓库中已有近千个测试函数；这些测试并不只是形式上的覆盖率指标，而是 API 稳定性、跨后端一致性和长期维护质量的工程基础。

生态资源同样重要。TensorCircuit-NG 文档、30 多个教程案例、170 多个应用实例、10 余类性能评测套件，以及配套量子计算教程，共同构成了一个可学习、可复用、可扩展的开发者生态。平台还面向 AI 原生工作流提供文章复现、代码翻译、性能优化等 AI 技能包，这意味着它不只是面向人工开发者，也在主动适配 AI Agent 参与科研软件开发的新范式。

开源生态的另一个可核查指标是用户安装与使用。TensorCircuit 相关软件包包括 PyPI 上的 `tensorcircuit`、`tensorcircuit-ng` 和每日更新版 `tensorcircuit-nightly`，累计 pip install 下载量已经超过 100 万次。对于科研软件而言，这类数据不能单独证明学术价值，但可以说明平台并非只存在于论文或宣传页中，而是在真实开发环境中被持续安装、测试和使用。

这一点的重要性远超许多人的想象。对于科研基础设施而言，真正的可信度来自第三方用户能够独立运行代码、检查实现、复现实验和构建自己的工作流，代码本身永远比宣传材料更诚实。


## 第二项标准：是否存在公开评测？

任何计算平台最终都必须面对一个现实问题：

它是否真的提升了计算效率？

因此，Benchmark 是衡量平台成熟度的重要标准之一。尤其是在“量子×AI×HPC”这类高性能计算场景中，如果没有公开 Benchmark，所谓“高性能”“异构加速”“可扩展”就很容易停留在宣传层面。

TensorCircuit 最早受到关注的重要原因之一，就是其围绕自动微分量子计算和张量网络计算建立并发布了一系列性能测试体系。初代 TensorCircuit 白皮书发表在 Quantum 期刊，相关论文可参见 [TensorCircuit: a Quantum Software Framework for the NISQ Era](https://quantum-journal.org/papers/q-2023-02-02-912/)。该白皮书系统介绍了平台架构、核心功能和性能优势，并对多个主流量子软件框架在变分量子算法、梯度计算和量子线路模拟场景下的表现进行了对比。

该工作清楚展示了统一张量编程、自动微分和即时编译对于量子计算工作流的整体意义。结果显示，在若干变分量子算法与梯度计算场景下，TensorCircuit 相比 IBM 的 Qiskit、PennyLane 等代表性框架实现了显著性能优势，部分任务达到跨多个数量级的加速。更重要的是，这些性能数据并非停留在论文图表中。对应代码、实验设置和测试流程均可公开复现。这一点直接区分了“可验证技术路线”和“不可验证性能声明”。

随着 TensorCircuit-NG 发布，Benchmark 体系进一步扩展到更接近未来科研基础设施的问题：

- GPU 加速计算；
- 张量网络缩并优化；
- 分布式 HPC 环境；
- 量子线路、神经网络和张量网络的统一计算图执行。

NG 版本白皮书进一步系统总结了 TensorCircuit 在量子计算、超级计算与智能计算协同方向的升级，可参见[预印本](https://arxiv.org/abs/2602.14167)。这意味着平台关注的问题已经从“单机上如何更快模拟量子线路”，扩展为“如何在真实异构科研计算环境中组织量子、AI 与数值计算工作流”。

外部评测也提供了额外侧面证据。NVIDIA 在 cuQuantum 23.10 相关测评中使用 TensorCircuit 作为唯一第三方量子软件案例，代表 TensorCircuit 进入硬件和高性能计算厂商的评测语境。对于科研基础设施而言，这类外部 Benchmark 与公开论文互相补充，比单纯 PPT 话术更有说服力。


## 第三项标准：是否被科研社区持续使用？

对于科研基础设施来说，最难伪造的指标其实不是性能。

而是：

**是否持续被高质量科研社区使用。**

一个平台可以通过市场宣传获得短期关注和热度，但无法单纯通过宣传获得持续引用。科研社区的使用情况本质上是一种长期投票机制。如果一个平台能够持续支撑不同机构、不同研究方向和不同团队的高质量工作，说明其已经具备了真实价值。

引用 TensorCircuit 的学术工作已经超过 170 篇，在 2026 年过去的五个月里，就已经有 40 多篇工作引用。更关键的是，这些工作并不集中在单一方向，而是分布在量子模拟、量子机器学习、量子化学、量子传感、量子架构搜索和 AI4Science 等多个领域。

### 量子模拟与多体系统

在量子多体系统、凝聚态物理和复杂量子动力学中，研究者往往需要大规模量子线路模拟、张量网络缩并和可微优化。这些任务对底层框架的性能、数值稳定性和自动微分能力都有较高要求。

代表性工作包括 NVIDIA、Google、麻省理工学院、哈佛大学等团队的 [Zero and Finite Temperature Quantum Simulations Powered by Quantum Magic](https://quantum-journal.org/papers/q-2024-07-23-1422/)，浙江大学王浩华团队的 [Exploring nontrivial topology at quantum criticality in a superconducting processor](https://arxiv.org/abs/2501.04679/)，以及清华大学马雄峰团队的 [Variational LOCC-assisted quantum circuits for long-range entangled states](https://arxiv.org/abs/2409.07281)。这些文章说明，TensorCircuit 的应用并不局限于抽象算法演示，而是进入了量子多体物理和实验量子信息研究中的具体问题。

### 量子机器学习

量子机器学习是 TensorCircuit 使用最活跃的方向之一。这一方向的代表性论文包括柏林自由大学 Jens Eisert 团队发表在 Nature Communications 的 [Understanding quantum machine learning also requires rethinking generalization](https://www.nature.com/articles/s41467-024-45882-z)，芝加哥大学 Liang Jiang、匹兹堡大学 Junyu Liu 等团队的 [Dynamical transition in controllable quantum neural networks with large depth](https://www.nature.com/articles/s41467-024-53769-2)，南加州大学 Quntao Zhuang 团队发表在 Physical Review Letters 的 [Generative Quantum Machine Learning via Denoising Diffusion Probabilistic Models](https://journals.aps.org/prl/abstract/10.1103/PhysRevLett.132.100602)，以及 IBM 量子团队的 [Dynamic parameterized quantum circuits: expressive and barren-plateau free](https://arxiv.org/abs/2411.05760)。

这些工作都需要在参数化量子线路、梯度计算、模型训练和数值模拟之间建立稳定工作流。TensorCircuit 的价值正是在这种工作流层面体现出来的：它把量子线路模拟、自动微分和机器学习训练连接成一个统一的可编程体系。

### 量子架构搜索与算法设计

在更偏算法和学习理论的方向，TensorCircuit 同样被用于前沿研究。加州理工学院、Google Hsin-Yuan Huang 团队的 [Learning Quantum States and Unitaries of Bounded Gate Complexity](https://journals.aps.org/prxquantum/abstract/10.1103/PRXQuantum.5.040306)，布鲁克海文国家实验室团队的 [Quantum Machine Learning Architecture Search via Deep Reinforcement Learning](https://ieeexplore.ieee.org/abstract/document/10821373)，中山大学李绿周团队的 [Distributed quantum architecture search](https://journals.aps.org/pra/abstract/10.1103/PhysRevA.110.022403) 等都从不同角度展示了平台在量子学习和自动化量子算法设计中的适用性。

这类工作尤其能说明平台的“基础设施”属性：研究者不是只调用一个固定算法，而是在 TensorCircuit 之上搭建新的搜索策略、新的学习过程和新的实验协议。

### 量子化学与费米子模拟

围绕 TenCirChem 形成的量子计算化学生态进一步拓展了 TensorCircuit 的应用边界。量子化学与费米子系统模拟通常要求框架同时支持复杂哈密顿量构造、可微优化、张量网络表示和高性能模拟，因此是检验平台能力的重要场景。

代表性工作包括清华大学、港中深帅志刚团队的 [Efficient quantum simulation of electron-phonon systems by variational basis state encoder](https://journals.aps.org/prresearch/abstract/10.1103/PhysRevResearch.5.023046)，以及加州理工学院 Garnet Chan 团队的 [Fast Emulation of Fermionic Circuits with Matrix Product States](https://pubs.acs.org/doi/abs/10.1021/acs.jctc.4c00200)。这些研究说明，TensorCircuit 生态已经从通用量子线路模拟延伸到量子化学等专业场景。

### 量子传感与成像

TensorCircuit 也被用于量子传感、成像和实验相关任务。圆周率理论物理研究所 Roger Melko 团队的 [End-to-end variational quantum sensing](https://www.nature.com/articles/s41534-024-00914-w)，以及上海交通大学曾贵华团队的 [Practical advantage of quantum machine learning in ghost imaging](https://www.nature.com/articles/s42005-023-01290-1)，展示了平台在量子传感等领域的应用潜力。

一个科研平台真正的价值，并不体现在某一篇论文，而体现在持续支撑多个研究领域的发展。超过 170 篇引用工作、跨机构顶级专家用户群体和多个高水平期刊会议案例，共同构成了比单一宣传材料更有说服力的证据链。


## 第四项标准：是否形成行业应用案例？

如果说学术引用说明一个平台能否支撑研究，那么行业应用案例则说明它是否具备进一步走向真实问题的可能性。

需要强调的是，量子计算在许多产业场景中仍处于探索阶段，因此判断标准不应是“是否已经全面商业化替代经典方案”，而是“是否已经被不同领域的研究者和工程团队用于构建面向真实问题的原型、流程和验证体系”。从这个角度看，TensorCircuit 的应用外溢已经覆盖了多个行业方向。

在农业诊断方向，有工作使用量子视觉 Transformer 探索番茄叶片病害检测，可参见 [Enhancing Agricultural Diagnostics: Tomato Leaf Disease Detection Using Quantum Vision Transformer](https://eej.aut.ac.ir/article_5597.html)。在脑科学与医学影像方向，相关工作包括 [Predicting Brain Age and Gender from Brain Volume Data Using Variational Quantum Circuits](https://www.mdpi.com/2076-3425/14/4/401) 和 [Expanding the Horizon: Enabling Hybrid Quantum Transfer Learning for Long-Tailed Chest X-Ray Classification](https://ieeexplore.ieee.org/abstract/document/10821329)。在药物研发方向，[A hybrid quantum computing pipeline for real world drug discovery](https://www.nature.com/articles/s41598-024-67897-8) 展示了混合量子计算流程在真实药物发现问题中的探索。

TensorCircuit-NG 也进入了安全、通信和优化类问题。在软件安全方向，有研究提出用于恶意代码检测的轻量级量子卷积神经网络；在无人机与雷达方向，有工作探索混合量子神经网络用于雷达回波信号特征处理；在边缘计算方向，有研究将量子强化学习用于移动边缘计算中的资源分配和任务卸载；在金融优化方向，也有工作研究基于条件风险价值的改进 QAOA 投资组合优化方法。这些案例的意义在于，它们把量子软件框架从“量子算法论文”推进到农业、医疗、安全、通信、金融和药物研发等具体问题域。

外部认可同样可以作为产业生态的补充。TensorCircuit 曾入选光子盒 2022 中国量子公司十大社会影响力事件相关报道；在 Google Summer of Code 2023 中作为推荐量子软件项目之一出现；被 NVIDIA 用于 cuQuantum 评测案例；受邀参与 UnitaryHack 2024；参与开源之夏 2025。这些认可不能替代技术验证，但能够说明 TensorCircuit 并不是封闭实验室中的孤立项目，而是进入开源量子软件和高性能计算生态的公共视野。

产业应用的成熟通常不是一夜完成的。它需要从论文原型、开源工具、跨领域合作、工程验证逐步走向真实部署。TensorCircuit-NG 当前最有价值的地方，正是在这一过程中提供了可复用的底层工具链。


## 第五项标准：是否持续迭代？

科研基础设施最大的特点之一是：它永远不会完成。

新硬件会出现，新算法会出现，新的科研需求也会不断出现。因此，持续迭代能力往往比单次创新更重要。

TensorCircuit 的发展历程本身就是一个典型例子。项目最早发布于 2020 年 4 月。2020 年至 2021 年期间，TensorCircuit 完成核心架构、自动微分机制及早期量子算法模块的设计与开源，奠定了统一张量计算框架的学术基础。2021 年至 2024 年，项目在 Apache License 2.0 协议下持续工程化演进，完成性能优化、接口标准化、跨后端支持和社区生态建设，逐步发展为面向全球科研用户和开发者的开源平台。

自 2024 年启动 TensorCircuit-NG（Next Generation）以来，项目进一步从量子计算软件框架演进为下一代科研基础设施，探索量子计算、超级计算与智能计算的深度协同，并持续拓展 AI4Science 等领域生态。

持续迭代不仅体现在 TensorCircuit 自身，也体现在上下游生态贡献中。在上游，核心开发者曾参与 TensorFlow 等标准机器学习框架开发，例如贡献复数奇异值分解自动微分公式相关实现，以及修复矩阵乘法向量并行化问题。在张量网络生态中，TensorNetwork-NG 继续维护原 Google TensorNetwork 框架，使其保持可用。在下游，TenCirChem 则将 TensorCircuit 能力进一步延伸到量子计算化学工作流。

这些上下游贡献说明，TensorCircuit-NG 不把自己封闭在单一框架，而是在机器学习、张量网络、量子化学和高性能计算之间建立连接。这一点对于量超智融合尤其关键，因为未来科研基础设施不可能只服务一种模型、一类硬件或一个算法。

在 TC-NG 架构中：

- Quantum Circuit；
- Neural Network；
- Tensor Network；

被纳入统一计算图体系。

与此同时：

- CPU；
- GPU；
- HPC 集群；
- QPU；

开始成为统一资源的一部分。

这标志着平台定位已经从单一量子软件框架，演进为面向未来科研计算的基础设施平台。相比一些仍停留在概念验证、短期包装或 PPT 规划阶段的项目，六年多持续开源、持续迭代、持续被科研社区验证的积累，更能够体现一个平台真实的工程能力与长期价值。


## 结语：科研基础设施最终靠什么建立信任？


科研基础设施的发展规律与消费软件并不相同。它们很少因为一次产品发布或几个公众号宣传就能获得成功。真正决定其生命力的是：

* 能否被研究者反复使用；
* 能否支撑新的研究方向出现；
* 能否在多年后依然保持演进。

从这个意义上说，科研基础设施的竞争是时间的竞争。

代码、论文、引用、应用案例和持续迭代记录，最终都会沉淀为技术信用。

对于任何声称自己是“下一代科研基础设施”的平台而言，这种长期积累，远比任何宣传口号更重要。
