---
layout: post
title: "gaon chart 年榜的一点分析"
date: 2018-02-02
excerpt: "从数据看K-Pop的发展变化"
tags: [mathematica,data science]
comments: true
---
* toc
{:toc}

## 引言

可以说 K-Pop 的一大乐趣，就是重视数据，很多东西可量化，可比较。G 榜的数据免费且网站内容非常简洁有序（人话：好爬），还是挺良心的。正好17年年榜出来不久，先从难度最小的各年年榜数据来分析下，看看有什么好玩的事情。 G 榜年榜从2010年开始，不过10年只有排名没有具体数据，因此我们选取11到17这七年的年榜数据作为研究对象。如果需要的话，之后也可以综合周榜进行更细致的分析。

这种爬网站附带数据分析的活，一听就是 python 的老本行，不过文艺一下，用 Mathematica 来一发。这种抓取网页能力几乎为0的语言都可以完成爬虫，也确实说明 G 榜的页面真是业界良心。不过 jekyll 这个模板用的插件并不支持 mathematica 代码栏高亮，我本地的 typora 编辑器是支持的，暂时就这样吧，有时间再折腾插件，效果就是下文的代码看起来非常的吃藕（其实写的也挺丑，毕竟数据量太小，完全不需要关心性能优化）。

另外，用了 mathematica，当然是配合导出的 pdf 格式~~高清无码大图~~，那就只能嵌到 html 里了，毕竟水土不太服。这样的后果是，可能部分浏览器不兼容，看不到图片，至少桌面版的chrome是可以渲染的。此外我也会将pdf图片的链接附于每张图片后，可以直接点击打开观看。

## 音源部分

### 数据

首先看音源部分，可以定量的数据包括年榜排名前100歌曲的下载量和流媒体播放量，爬虫简单到令人发指，只需

~~~mathematica
Address = 
 Table["http://gaonchart.co.kr/main/section/chart/online.gaon?\
nationGbn=T&serviceGbn=S1040&targetTime=" <> ToString[year] <> 
   "&hitYear=" <> ToString[year] <> "&termGbn=year", {year, 2011, 
   2017}]; 
streaming = Import[#, "Data"] & /@ Address;
Address = 
 Table["http://gaonchart.co.kr/main/section/chart/online.gaon?\
nationGbn=T&serviceGbn=S1020&targetTime=" <> ToString[year] <> 
   "&hitYear=" <> ToString[year] <> "&termGbn=year", {year, 2011, 
   2017}]; 
download = Import[#, "Data"] & /@ Address;
~~~

七年的音源数据就已经都存储在`download`和`streaming`两个变量 list 中了。最好写一些常用的转化函数和常量方便后边数据处理

~~~mathematica
 ToNum[num_] := ToExpression[StringTrim[StringReplace[num, "," -> ""]]];
 (* Transform num in string format to a expression format *)
 dTotal = 
 Table[Sum[ToNum[download[[i, 3, 4, j + 1, 4]]], {j, 1, 100}], {i, 1, 7}];
 sTotal = 
 Table[Sum[ToNum[streaming[[i, 3, 4, j + 1, 4]]], {j, 1, 100}], {i, 1, 7}]; 
 (* the total number of download or streaming for top 100 songs in each year *)
 cc[data1_, data2_] := 
 Mean[(data1 - Mean[data1])*(data2 - Mean[data2])]/(StandardDeviation[
     data1]*StandardDeviation[data2]);
 (* function to compute the cross correlation of two time series *)
~~~

### 流媒下载比例变化

这几乎是拿到数据，最明显和最先看到的特征。我们将每年年榜100首歌的流媒体或下载数据求和，这可以代表当年音源流媒体和下载的特征数据，就得到下图。（为了下载数量和明显更大的流媒体数量可比，我们进行跨下载和流媒的类型比较时，都将下载数量乘以 25）。

~~~mathematica
ListLinePlot[{Table[
   Sum[ToNum[download[[i, 3, 4, j + 1, 4]]]*25, {j, 1, 100}], {i, 1, 7}], 
   Table[
   Sum[ToNum[streaming[[i, 3, 4, j + 1, 4]]], {j, 1, 100}], {i, 1, 7}]}]
~~~

| <embed src="/images/gaon_ds_transition.pdf"  type='application/pdf'> |
| :--------------------------------------: |
| *2011到2017， 年榜前100流媒体与下载数量总和随时间的演变 [查看大图](/images/gaon_ds_transition.pdf)* |

明显流媒体作为主流势不可挡，下载量相比2011年几乎减半。这一趋势应该在全世界都是通用的。网速带宽的提升使得除了收藏癖（没错，就是我）之外，没有什么理由需要下载歌曲。我们还可以把下载和流媒按比例加起来，观察音源销售总量随时间的变化。

~~~mathematica
ListLinePlot[Transpose[{yTotal, sTotal + dTotal*25}]]
~~~

| <embed src="/images/gaon_digit_total.pdf"  type='application/pdf'> |
| :--------------------------------------: |
| *2011到2017， 年榜前100音源销售总量随时间的演变 [查看大图](/images/gaon_digit_total.pdf)* |

音源销售量随着经济发展和物价上涨理应一直上升，不过2013年出现了一个明显的低谷，甚至音源销售总量只有两年前的 70%。那一年中，音源价格上调，造成音源数据严重波动，也直接导致了下载和流媒播放的价格差扩大，近一步促进了流媒体数据的起飞。

### 所谓三大

G 榜数据包含了歌曲的发行公司，于是可以很容易的比较各个公司间在音源成绩上的实力。这里选取 SM，YG 和 JYP 三间较有影响力的经纪公司为例，来展示他们所发行的进入TOP100的歌曲占总共TOP100歌曲的各种比例。首先我们计算下，每年各公司可以进入年榜的歌曲数目。

~~~mathematica
BarChart[Transpose[
  Table[Sum[
      If[StringCount[download[[year, 3, 4, i + 1, 5]], #] > 0, 1, 
       0], {i, 1, 100}], {year, 1, 7}] & /@ {"YG", "SM", "JYP"}], 
 ChartLabels -> {Placed[yTotal, Above], 
   Placed[{"YG", "SM", "JYP"}, Top]}]
~~~

| <embed src="/images/gaon_three_nb.pdf"  type='application/pdf'> |
| :--------------------------------------: |
| *2011到2017， 三间经纪公司发行歌曲进入下载量TOP100的数目 [查看大图](/images/gaon_three_nb.pdf)* |

我们也可以直接计算出七年间进入下载年榜的歌曲总数（多年入榜记为多首），分别为 YG 73首，SM 39首，JYP 30首。嗯 $$73>39+30=69$$，不尴尬。接下来除了数量还要看歌曲的质量（音源具体数据）。

~~~mathematica
ListLinePlot[
 Table[Sum[
      If[StringCount[streaming[[year, 3, 4, i + 1, 5]], #] > 0, 
       ToNum[streaming[[year, 3, 4, i + 1, 4]]], 0], {i, 1, 100}], {year, 1, 7}]/sTotal 
       & /@ {"YG", "SM", "JYP"}, 
 PlotLegends -> Placed[{"YG", "SM", "JYP"}, {Center, Above}], 
 DataRange -> {2011, 2017}]
~~~

| <embed src="/images/gaon_three_s.pdf"  type='application/pdf'> |
| :--------------------------------------: |
| *2011到2017， 三间经纪公司发行音乐流媒体占TOP100的比例 [查看大图](/images/gaon_three_s.pdf)* |

毫无悬念，YG 在音源流媒体占比方面碾压其他两家，我们还可以把SM和JYP成绩加起来，更残酷的看。

| <embed src="/images/gaon_two_down.pdf"  type='application/pdf'> |
| :--------------------------------------: |
| *2011到2017， YG 与 SM JYP 俩家公司之和的音源流媒体TOP100占比 [查看大图](/images/gaon_two_down.pdf)* |

由此可见，YG 一家音源销售稳稳吊打另外两家之和，唯一失手是16年，鬼知道16年，菊花都在想啥。哦，对了，16年 Bigbang 并没有发歌（年末的没来得及计入），可能这一条就够了。可以计算 YG 占据的年榜流媒销售量平均值在 11.5%。最后我们看一下，考虑下载和流媒的音源销售总量，三家公司又是怎样的格局。


| <embed src="/images/gaon_four_t.pdf"  type='application/pdf'> |
| :--------------------------------------: |
| *2011到2017， 三家公司音源综合成绩 [查看大图](/images/gaon_four_t.pdf)* |

没什么悬念啦，依旧稳稳的吊打。还有个有趣的事，有没有发现 SM 和 JYP 两家公司的音源成绩是此起此伏，而 YG 和他们是此起彼伏。口说无凭，用上[交叉关联函数](https://en.wikipedia.org/wiki/Cross-correlation)，算算就知道了。SM 和 JYP 的关联约为正的 0.7，而 YG 和他们的相关性是 －0.4，显现出了明显的正负相关。不知有没有什么背后相爱相杀的故事。

### 具体艺人

这个直接上图吧，近七年来音源最强的十组艺人。这一计算对于每一组团体艺人，还尽量包括了其子团体，solo等音源成绩。比如 BlockB 包括了 Zico 的solo成绩，少女时代包括了 TTS 和泰妍 solo 等（但这种统计可能不全或者漏掉，非常粗略）。此外 OST 和节目曲也包含其中。总之就是演唱者名字里有艺人名字就算。最后效果如下（如果你发现了音源总量可以进入 TOP 10 而我漏算的艺人，记得提醒我）：

~~~mathematica
SortBy[{Sum[
      Sum[If[StringCount[streaming[[year, 3, 4, i + 1, 3]], #] > 0, 
        ToNum[streaming[[year, 3, 4, i + 1, 4]]], 0], {i, 1, 
        100}], {year, 1, 7}], 
     Sum[Sum[If[StringCount[download[[year, 3, 4, i + 1, 3]], #] > 0, 
        ToNum[download[[year, 3, 4, i + 1, 4]]]*25, 0], {i, 1, 
        100}], {year, 1, 7}], #} & /@ Flatten[artist] // N, -(#[[1]] + #[[2]]) &]
  (* artist is a list of artists with some or | relation to include subunit and solo *)
~~~




| <embed src="/images/gaon_digital_top10.pdf"  type='application/pdf'> |
| :--------------------------------------: |
| *2011到2017， 年榜音源总量 TOP10艺人 [查看大图](/images/gaon_digital_top10.pdf)* |

只能说IU一骑绝尘。不过这只是这七年，之前的谎言一天一天之类的神曲还没算入，否则大棒可能更强些。毕竟11年好日子之后，这七年几乎包括了她的最好成绩。不过最传奇的大概还是 Busker Busker，如果知道他们的发专数量，再看这一成绩就只能膜拜。这个音源名单紧随其后的艺人还有 TWICE，Zion.T， Apink，PSY 和 GFRIEND。

为了更直观的说明，我们将以上十组艺人的音源综合总量放入七年音源年榜总量中（标记4到10，对应上面条形图的十组艺人）。IU 的进入年榜的音源发售总量已经占到了这七年年榜的 4.8%，一组艺人占据 5% 的音乐市场，在韩国这种激烈的竞争下，非常难得。

| <embed src="/images/gaon_10_pie.pdf"  type='application/pdf'> |
| :--------------------------------------: |
| *年榜音源总量 TOP10艺人占七年音源总量的比例 [查看大图](/images/gaon_10_pie.pdf)* |

单独拿出艺人看看每年的音源总量也挺有趣的，比如大棒。我们来对比下大棒每年进入年榜部分的音源综合成绩和整个 YG 公司的音源综合成绩。

| <embed src="/images/gaon_yg_bb.pdf"  type='application/pdf'> |
| :--------------------------------------: |
| *2011到2017， 年榜音源成绩，YG vs BIGBANG  [查看大图](/images/gaon_yg_bb.pdf)* |

对此，没有评论。。。。所以要是去掉了一组艺人，三家公司就一个起跑线了。

再来看一下，top5 艺人的每年流媒占比变化情况。前七年综合，还是脸红去年十首且音源综合数据占比也在约10%是最强的。

| <embed src="/images/gaon_ratio_top5.pdf"  type='application/pdf'> |
| :--------------------------------------: |
| *2011到2017，音源 TOP5 艺人流媒占比变化情况  [查看大图](/images/gaon_ratio_top5.pdf)* |
