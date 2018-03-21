---
layout: post
title:  群体结构与群体混杂
date:   2016-03-05
categories: notes
---

Structured population
1）从若干个遗传群体内抽取部分个体，合并到一起即可构建一个混杂群体，这是抽样造成的混杂。

2）群体遗传学中的群体混杂/群体结构，一般是指由于recent admixture，群体之间存在基因交流而造成的。

主成分方法（PCA）和STRUCTURE/Admixture是常用的检测群体结构的方法。
与PCA相比，STRUCTURE/Admixture基于群体遗传的模型，更贴切群体遗传结构的分析要求。假定K个祖先群体，计算现有群体中各个祖先群体来源遗传成分的比例。STRUCTURE/Admixture计算结果类似。

fastStructure是STRUCTURE的快速版本，下载地址和参考文档：https://rajanil.github.io/fastStructure/

Admixture，计算速度同样比STRUCTURE快很多，下载地址和参考文档：https://www.genetics.ucla.edu/software/admixture/download.html

以Admixture为例，假定K=2~10，分别计算

`for ((k=2;K<11);K++);do ./admixture --cv example.bed $k|tee example_${k}.log;done`

得到的结果文件为`example.${k}.Q`

提取cross validation error结果

`grep -h CV example*.log>CV.out`

使用[structurePlot.R]绘制K=3时的结果（`example.3.Q`）

{% highlight bash %}
source('structurePlot.R')
tbl=read.table("example.3.Q")
popGroups=read.table("pop_info.txt",col.names = c('Group','pop',"id"))
# pop_info.txt
# 第一列为群体分组，必须命名为Group
# 第二列为群体名称，可以与第一列相同
# 第三列为个体名称

mergedAdmixtureTable=cbind(tbl,popGroups)
orderedAdmixtureTable=mergedAdmixtureTable[order(mergedAdmixtureTable$Group),]
head(orderedAdmixtureTable)
pdf("Admixture_K3.pdf",height=4,width=7)
par(mar=c(10,5,5,5))
structurePlot(orderedAdmixtureTable,k=3,main="K=3")
dev.off()
{% endhighlight %}

[structurePlot.R]:https://github.com/geneticswithme/hello_genetics/blob/master/structurePlot.R




