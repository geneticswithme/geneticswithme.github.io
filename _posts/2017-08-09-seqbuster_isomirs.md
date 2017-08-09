---
layout: post
title:  seqbuster分析miRNA/isomirs基本流程
date:   2017-08-09
categories: notes
---
[seqbuster](https://github.com/lpantano/seqbuster)

[tutorial](http://seqcluster.readthedocs.io/mirna_annotation.html#processing-of-reads)



## 1 miRNA quality control

* 先用fastQC查看数据情况

* cutadpt 去除adapter序列，miRNA数据一般只需要去除3' 的adapter。



## 2 准备本地alignment所需的database，放入文件夹“DB”内

hairpin.fa and miRNA.str from miRBase site



## 3 比对reads到mirRNA的database （miraligner.jar, 需使用java1.7）

```

java -Xmx 16g -jar miraligner.jar -sub 1 -trim 3 -add 3 -s hsa -i input_trimmed.fq -db DB -o output

#output: output.mirna

```

以上步骤全部参考seqbuster的tutorial。得到的output.mirna文件。本来想使用miraligner.jar 的 `-freq` 参数直接输出miRNA的频率，但不知道为什么输出文件不能被isomiRs包直接读取，试了几次后干脆直接在output.mirna基础上重新统计frequency。



## 4 post analysis with R package isomiRs

bioconductor上的R package [isomiRs](https://bioconductor.org/packages/release/bioc/html/isomiRs.html)


```

library ( "plyr" )

df <- read.table ( "output.mirna", header=T, comment.chr="!" )

colnames_order = c( "seq","name","freq","mir","start","end","mism","add","t5","t3","s5","s3","DB","ambiguity" )

aa = ddply ( df, ~mir+start+end+mism+add+t5+t3+s5+s3+seq+DB+ambiguity, nrow )

colnames(aa)[13] = "freq"

aa[ , "name"] = paste0 ( aa[ ,'mir'], "_x", aa[ ,"freq"] )

newdf = aa[ , colnames_order ]

write.table ( newdf, file = "isomiRs_input.mirna", sep = "\t", quote=F, row.names=F )

```

然后用IsomirDataSeqFromFiles读取文件，参考isomiRs包的说明文档进行后续分析。













