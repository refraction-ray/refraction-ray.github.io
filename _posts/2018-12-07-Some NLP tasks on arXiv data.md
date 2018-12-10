---
layout: post
title: "Some NLP tasks on arXiv data"
date: 2018-11-21
excerpt: "利用一些 NLP 的工具处理 arxiv 元数据的尝试"
tags: [python, machine learning, nlp]
comments: true
---

* toc
{:toc}

## 引言

最近在折腾一个自动分析 arxiv 文章的项目，本来的主要目的是为了可以每天过滤出自己感兴趣的 arxiv 文章，并自动发送汇总内容的邮件。不过对于拿到的文章题目摘要等文本数据，难免技痒，自然少不了自然语言处理工具的帮助和尝试，因此本文简要叙述一些最近尝试的 NLP 方法和工具，以做备忘。以下代码的实验环境均为 Python3.6。本文的重点放在方法的基本理解和相应封装工具的简单实践上，至于各个算法背后相应的具体数学原理与实现细节，不在本篇所涉范围内。

## 模糊匹配

模糊匹配是本来过滤 arxiv 每日更新就需要的功能。想要通过关键词列表来判别对应文章是否包含关键词，使用标准的正则匹配是不太理想的。因为关键词的词形，单复数，大小写，空格或是连字符的有无，都会直接影响匹配是否成功。因此基于文本编辑距离的模糊匹配方案就成为了解决问题的选项。关于广义的文本相似度和“距离”的衡量的总结，可以参考[^distance]。编辑距离也被称为 Levenshtein distance，直观理解就是字符串 A 通过多少次插入，删除和替换的操作，可以变为字符串 B，很明显，操作次数就对应了距离，操作次数越多，两个字符串就越不像。实践上，可以较容易得出编辑距离的递归定义，从而通过动态规划来实现。

基于编辑距离的模糊匹配，对应的 python package 是 [fuzzywuzzy](https://github.com/seatgeek/fuzzywuzzy)。除了最一般的 `ratio` 实现了编辑距离之外，还有其他一些匹配函数适合更具体的情景。比如 `partial_ratio` 可以实现部分匹配，只要字符串 A 可以完美匹配字符串 B 的一小部分，也会返回 100 作为分数。需要注意该库中的各种匹配 `ratio` 函数，两个输入字符串都是对易的，不需要特意将短的那部分放到第一个参数，其函数内部已自动进行了调换。该库事实上也只是个 wrapper，如果不额外安装 C 版本的算法实现，调用的是 Python 标准库的 difflib。不过封装时， fuzzywuzzy 漏了一个参数 `autojunk`，使得长字符串的 `partial` 匹配行为并不可靠，具体的来龙去脉，可以参考我的这个 [issue](https://github.com/seatgeek/fuzzywuzzy/issues/224)。其 `partial_ratio` 的实现原理，就是先找到短字符串和长字符串的共同子串，在对这些子串扩展到短串长度来分析编辑距离，匹配分数最大的子串对应的分数，就是 `partial_ratio`。

## 关键词提取

自动关键词提取，是对于 arxiv 摘要处理的另一个尝试。这方面，我使用的 RAKE 算法 [^rake]。基本想法就是首先通过 stop words 比如说介词之类的来分词，分开的各个词组就是关键词的候选。然后开始给各个单词打分。打分的依据是出现在的词组的平均长度。最后每个词组的分数等于构成的各单词的分数之和。这样分数最高的一些词组就成为了关键词。这样的算法特别倾向于选出很长的关键词，这一算法的一个 python 实现可以参考 [RAKE](https://github.com/aneesha/RAKE)。不过为了更好的匹配科研文章的关键词提取，我在这个算法上魔改了一些，包括抑制超长词组的权重，增加了更丰富的科研文章常用动词作为 stop word，当然还有 python3 兼容性，这一魔改版本参见 [rake.py](https://github.com/refraction-ray/arxiv-analysis/blob/master/arxivanalysis/rake.py)，本身也有很高价值的 stop word list 参见 [SamrtStopList](https://github.com/refraction-ray/arxiv-analysis/blob/master/arxivanalysis/SmartStopList.txt)，我之后会对该 stop word list 保持必要的更新。需要注意这些停词都是适用于英文学术文献的 NLP 分析的，对于一般文本的分析，该停词库可能过大，不太合适。模糊匹配和关键词提取，这两部分功能，都已应用整合到了项目代码里，具体的实践可以参考我写的 arxiv 分析项目 [arxiv-analysis](https://github.com/refraction-ray/arxiv-analysis)。以下的部分，都是基于 arxiv 数据更多的尝试，至于怎么结合到项目里，暂时还没想好。

## 词汇关联程度

从语义的角度，来判断词汇关联程度，靠比较两个字符串有多像显然是不行的，一个可行的方案是 word2vec。在 NLP 处理里，每个单词都会被编码为一个 1-hot 向量。也就是第 i 位是 1，其他分量是 0 的向量对应了字典的第 i 个单词。但这样做每个单词对应的向量都是正交的，各单词之间没有联系，特别是当词典中词汇很多时，空间过大，计算起来也很没效率。而 word2vec 就是对 1-hot 向量空间的一个压缩降维，把所有单词对应的向量都压缩到比如100维。此时向量之间不再正交，内积不为 0，而内积的大小就表示了两个单词的关联性。至于这种降维方案的学习过程，主要是依靠每个单词的上下文窗口，通过单词出现语境的统计规律，来归纳出单词之间的关联性和低维向量表示。其中的具体数学细节，可以参考[^word2vec]。当然最近的实践中，word2vec 往往被直接理解成了一个 embeded layer，其单词向量的映射规律，被置于优化特定目标的整张神经网络中一起学习。一方面原始的 word2vec 可以用于这种 embeded layer 的参数预训练，另一方面这种特定网络的 embeded layer 训练好之后也可单独拿出来做 word2vec，这种 word2vec 的效果和特性，往往和本来神经网络的任务密切相关。

Python 对应的 word2vec 模型的实现，我们可以直接利用 gensim 这一 package。这一 NLP 库提供了大量现成的模型，尤其是下文要提到的话题模型。要想利用这些模型，需要对文档预处理。文档预处理包括小写化，去标点和停词，以及词根化及最后将每篇文档拆为单词的 list。词根化 lemmatize 的意思是把单词的变形还原为原型，防止同一个单词的不同词形被记为不同单词，影响分析效果。lemmatize 实践上可以利用的工具，可以参考 [^lemma]中的介绍和对比。

有了列举每篇文档中词汇 list 的嵌套 list `doc`，就可以通过 `gensim.models.Word2Vec(doc)` 来训练 word2vec 模型。并可以进一步的根据该模型提供的方法，判断词语的相似性，甚至进行基于语境的词汇联想。具体的代码请参照文末的 jupyter notebook。

## 话题模型与相似度

为了进一步衡量文档之间的相似程度，就需要利用话题模型。比如说，相关文章推荐这种功能，就可以通过话题模型来分析文章相似度来实现。话题模型有很多算法，细节上对于联合概率分布的近似有所区别。但精神上，都是通过词袋模型，将语言文本视为单词的概率分布，通过引入隐变量－话题，将具体文档的词频分布视为话题的概率分布和每个话题中单词的概率分布的联合结果。由此，可以得出不同话题里的高频词汇，和每个文档的高频话题。话题模型是典型的无监督学习，通过对概率分布的近似，实现了一定意义上的聚类效果。这节我们将以 LDA 模型为例，演示话题模型的应用。

话题模型的文档数据，要有更多的预处理，并且需要生成 corpus。为了更好的刻画文档信息，可以对文本中同时出现的高频词组绑定成一个单词处理。这是因为词袋模型只关注词频，而不关注词的位置，词组的信息会丢失掉。因此可以通过 `gensim.models.Phrases` 来获取包含词组作为单词的模型，比如`high_temperature` `Anderson_localization` 等都会被识别为独立的单词。而对于原始的包含单词的二维 list，可以直接 `phrasesmodel[doc]` 的方式，来获取按照词组合并过的单词 list。有了 `doc` 这一单词按文档列举的 list，就可以进一步通过 `dictionary = gensim.Dictionary(doc)`，`corpus = [dictionary.doc2bow(d) for d in doc]` 来获取各种模型训练所必须的 corpus。所谓 corpus 就是包含了每个文档的各个单词词频信息的二维 list。进一步的我们还可以通过 `gensim.models.tfidfmodel.TfidfModel(corpus)` 来对词频进行调制。 tfidf 的想法其实非常简单，就是常见词词频高也不代表什么，通过 idf 压低高频词词频，防止高频却没什么意义的词干扰过强，而把本来出现不多却文档里高频 anomaly 的真正有用的词给突出出来。关于 tfidf 的直观理解和基于其最 naive 的关键词提取算法的介绍，可以参考 [^tfidf]。

完成这些预处理之后就可以通过 `gensim.models.ldamodel.LdaModel(corpus, num_topics=100, id2word = dictionary)` 来训练模型。`num_topics` 对应了预设的话题数目的超参。值得一提的是，除了初始的训练，之后 gensim 还支持模型的基于新数据的进一步训练，这一 online training 对于不断产生新数据的生产环境也非常有用。模型训练好之后，就可利用该类实现的各种方法，来展示每个话题的词频分布，每个文档在话题空间的向量等。特别的利用文档在话题空间的向量内积，可以实现文档相似度的接口`gensim.similarities.MatrixSimilarity()`。最后值得一提的是，pyLDAvis 这个库，是专门用来给 LDA 模型结果做可视化的，可以非常直观的了解各个话题的距离和词频分布。具体利用以上内容的代码，参见文末 jupyter notebook。一个完整的包括从文本预处理，到结果可视化的利用 gensim 来进行话题模型训练的例子，也可以参考[^topic]。

对于 arixv 摘要和题目数据本身的话题模型尝试结果，如果是来自不同的 subject，那么 LDA 训练效果很好，可以非常轻松，无监督的完美实现聚类。但如果只针对一个科目的文章进行训练，效果就不是很理想。这可能和一个领域科研文献的用词高度一致，非常单调有很大的关系。不过还是可以对每个话题尝试分析，比如有些可能对应 SYK model，有些对应 STM 实验，有些对应 Monte Carlo 数值计算等等，不过更多的话题分类有点像背景噪声。

## 文本自动生成

最后看一下深度神经网络的应用，这一节我们探索利用 RNN 网络深度学习，自动生成文章题目的可能性。这一工作说白了就是堆几层 RNN （主要是上 LSTM），训练的输入是若干字母，目标输出是下一个字母。这样学完之后，就可以每次按概率逐一生成字母，从而完成造句。注意到如果每一步都选择概率最大的字母，那么整个网络只能造出一句话。因此要引入温度的概念，每一步都按照一定的概率选取下一个字母，而不总是选取概率最大的字母，这就丰富了能造出句子的多样性。对应模型学习的越好，原来损失函数越小，那么就能在越高的温度，生成貌似合理的句子，从而造句能力越强，因为温度越高，句子生成的可能性越多。需要注意整个过程都是 char based，也就是按字母学习的，这样生成的每个词是对的都是一件 nontrivial 的事情，但实际上效果很好，而且字母的编码空间维度低，对于屈折多的语言也更友好（不需要关心词层面的变形）。当然 word based RNN 也不是不可以，这方面优劣对比，可以参考这个[讨论](https://datascience.stackexchange.com/questions/13138/what-is-the-difference-between-word-based-and-char-based-text-generation-rnns)。

从 keras 开始手堆这个造句器也不算难，而且这种 RNN 以好训练著称，对于网络结构和数据量的要求都不太高。不过早就有了更省事的懒人包，也是用 keras 实现的两层 LSTM，该项目叫 [textgenrnn](https://github.com/minimaxir/textgenrnn)。用这一项目，只需两行就可以完成模型的构建和训练。而且该项目 ship with 的模型带有预训练的参数，本来就可以造句了。基于这样的模型对于新数据进行优化训练，也非常的快。在话题模型时，我们已经提到，科研文章摘要的文风和选词都十分枯燥，另一方面，这也就表示了造句论文标题是很容易的，事实也是如此。在非常小的训练量（1000篇论文题目）和非常小的训练规模（只需要过10轮数据），就可以实现下列题目。

> Antiferromagnetic systems with correlated electrons
>
> The Angular Distribution of Spiral Galaxies
>
> Comment on ``Enhancement of the Tunneling Density of States in Tomonaga-Luttinger Liquids''

注意到最后一个题目，该模型还成功的学到了 latex 加引号的 syntax，并且完全和 comment 的文章类型相匹配。其题目的生成质量已经可以通过图灵测试了，反正我是不知道这些题目是不是人写的。当意识到训练是仅仅是基于字母的时候，这一结果就更显得惊人。

关于后三个还没有集成到 arxiv 项目中的 NLP 尝试，可以参考以下 jupyter notebook 的展示。这一 notebook 上边提供的按钮可以访问 google colaboratory 从而直接运行代码。该 notebook 的 gist 地址在[这里](https://gist.github.com/refraction-ray/5ce356aa40069cbbc6a80978a6d1a558)，通过此链接也可直接拿去 nbviewer，binder 或是 colaboratory 渲染使用。

<iframe src="https://nbviewer.jupyter.org/gist/refraction-ray/5ce356aa40069cbbc6a80978a6d1a558" width="100%" height="3000"></iframe>


## Reference

[^distance]: [文本相似度量方法](http://www.zmonster.me/2015/11/15/text_similarity_survey.html) 
[^rake]: [Aotomatic Keyword Extraction from Individual Documents](https://www.researchgate.net/publication/227988510_Automatic_Keyword_Extraction_from_Individual_Documents)
[^word2vec]: [word2vec 中的数学原理](https://www.cnblogs.com/peghoty/p/3857839.html)
[^lemma]: [词形还原工具对比](http://www.zmonster.me/2016/01/21/lemmatization-survey.html)
[^tfidf]: [TF-IDF与余弦相似性的应用](https://www.ruanyifeng.com/blog/2013/03/tf-idf.html)
[^topic]: [Topic Model with Gensim](https://www.machinelearningplus.com/nlp/topic-modeling-gensim-python/)