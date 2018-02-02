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

另外，用了 mathematica，当然是配合导出的 pdf 格式~~高清无码大图~~，那就只能嵌到 html 里了，毕竟水土不太服。这样的后果是，可能部分浏览器不兼容，看不到图片，因此我也会将pdf图片的链接附于每张图片后，可以直接点击打开观看。

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

七年的音源数据就已经都存储在`download`和`streaming`两个变量 list 中了。我们还需要定一些常用的转化函数和常量方便后边数据处理

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
   Sum[ToNum[streaming[[i, 3, 4, j + 1, 4]]], {j, 1, 100}], {i, 1, 7}]},
   PlotLegends -> {"download", "streaming"}]
~~~



| <embed src="/images/gaon_ds_transition.pdf"  type='application/pdf'> |
| :--------------------------------------: |
| *2011到2017， 流媒体与下载数量的演变 [查看大图](/images/gaon_ds_transition.pdf)* |






| <embed src="/images/gaon_digital_top10.pdf"  type='application/pdf'> |
| :--------------------------------------: |
| *2011到2017， 年榜音源 TOP10 [查看大图](/images/gaon_digital_top10.pdf)* |

