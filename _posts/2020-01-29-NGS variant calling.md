---
layout: post
title:  利用NGS数据进行突变检测Variant calling
date:   2020-01-29
categories: notes
---

###写在前面：
利用二代测序的数据（Next generation sequencing，NGS）进行突变检测，可以检索到很多很好的分析流程加以借鉴。但是大多数是适用于人类医学，如果应用于非模式动物植物的数据分析中还需具体问题具体分析。但是大体上，NGS数据分析基本流程是一致的，测序数据（二代测序）处理 → 比对至参考基因组（需预先对参考基因组建立index）→ 提取突变位点Variants（包括SNP和Indel，较常用的软件：GATK或者BCFtools，等等等）→ 筛选出可靠的Variants → ​​​​后续分析和应用

## 1 参考基因组的获取
参考基因组（Reference genome），一般是指某物种中的代表样本的基因组序列装配信息。随着技术水平提高，基因组组装（Assembly）版本的更新，参考基因组的精确性和准确性也随之提高。与参考基因组搭配的，还有参考转录组信息（转录本的序列）、基因注释信息（基因名、编号、起止位置等）等注释信息。

参考基因组数据可以从NCBI（ftp://ftp.ncbi.nlm.nih.gov/genomes/）、Ensembl（https://www.ensembl.org/info/data/ftp/index.html）和UCSC genome browser database这样的数据库中下载获得。

注释文件，主要是记录基因的信息，一般为GTF或者更GFF3格式（其实都是文本文件，可用文本编辑器打开），推荐使用GTF格式的注释文件。Ensembl提供GTF和GFF3两种注释文件，而NCBI只提供GFF3。不同数据库的基因组版本和注释信息版本可能不一致，因而在下载参考基因组文件时要注意记录版本号，避免将不同数据库的参考基因组和注释文件混用。如果从ncbi获取，则注释文件与参考基因组序列都从ncbi下载，并记录版本号。

NOTE：我个人喜欢从NCBI获取参考基因组，但是会将GFF3文件转化为GTF格式再使用。因为从个人经验来看，GTF文件格式更加规整，应用于其它分析时更加方便。

## 2 为参考基因组建立索引index

通常会建立三种index文件，分别是为了BWA软件、SAMtools软件和GATK软件的使用而建立。假设我们下载的参考基因组序列为reference.fa，相应的注释文件为reference.gtf。

### 2.1 为参考基因组建立BWA索引

BWA软件是常用的一款优秀的基因组比对软件，将我们的NGS数据比对至参考基因组上，获得比对结果文件（sam文件，可以转化为二进制的bam文件）。
BWA软件建立index有不同的方式，以`-a`参数用来选择建库方式，10M以下基因组用默认的`is`。10M以上的基因组`-a`设置成`bwtsw`。
```
bwa index -a bwtsw reference.fa
```
执行完毕后，生成以reference为前缀的一系列文件，后缀分别为amb,ann,bwt,pac和sa。

### 2.2为参考基因组建立SAMtools索引

使用的SAMtools的`faidx`命令。
```
samtools faidx reference.fa
```
执行完毕后，生成`reference.fa.fai`文件。

### 2.3为参考基因组建立GATK索引
使用`CreateSequenceDictionary` 命令生成`reference.dict`文件

GATK4 以前的版本
```
java -jar picard.jar CreateSequenceDictionary \

    REFERENCE=reference.fa \

	OUTPUT=reference.dict
```

GATK4
```
gatk CreateSequenceDictionary -R reference.fa -O reference.dict
```

以上是参考基因组数据的准备。接下来对原始数据进行预处理。

NGS原始文件一般是以`fastq`或`fq`为后缀的文本文件。未经过质控的原始文件中可能包含测序中接头序列信息，低质量碱基和序列信息，不利于后续分析进行。因而需要对原始文件进行质控处理，获得clean data。

## 3 原始数据的质控
有很多工具可以达到去除接头序列，去除低质量碱基或序列的目的，这里使用的Trimmomatic软件。假设原始数据是经过Illumina Hiseq平台获得的双端测序数据，双端序列文件分别为raw_R1.fq和raw_R2.fq。简单的质控可以通过
```
java -jar trimmomatic.jar PE -threads 4 -phred33 -trimlog trim.log raw_R1.fq raw_R2.fq clean_R1.fq Fsingle.fq clean_R2.fq Rsingle.fq ILLUMINACLIP:adapters.fa:2:30:10 LEADING:30 TRAILING:30 SLIDINGWINDOW:4:15 MINLEN:50
```
得到质控后的clean_R1.fq和clean_R2.fq进行后续比对分析。这两个文件中包含质量合格的双端测序的reads。

## 4 将reads比对至参考基因组
这里使用BWA软件的MEM算法完成。
```
bwa mem reference.fa clean_R1.fq clean_R2.fq -o clean2reference.sam
```
这里其实也使用了2.1中提到的bwa建立的索引文件，执行完成后生成`clean2reference.sam`文件，包含了全部reads的比对结果。

sam文件其实是符合特定格式的文本文件，可以用less等命令查看。使用SAMtools工具将sam文件转化为bam二进制文件，减少文件体积，方便进行计算。
```
samtools view -bS -o clean2reference.bam clean2reference.sam
```
执行结束之后，生成clean2reference.bam文件，clean2reference.sam可以被删除以节约磁盘空间。

## 5 对bam文件进行预处理，以方便后续分析

### 5.1 排序并增加样本信息

```
#gatk4
gatk AddOrReplaceReadGroups -I clean2reference.bam -O clean2reference.sort.bam -SO coordinate -ID Sample1 -LB L1 -PU P1 -PL illumina -SM Sample1 --CREATE_INDEX true
```
其中`-SO coordinate`是指对文件进行排序，`-ID -LB -PU -PL -SM` 等是指对文件补充相应的样本信息，`--CREATE_INDEX true`是指生成结果文件clean2reference.sort.bam的同时，对其生成index文件。

### 5.2 去重复
```
gatk MarkDuplicates -I clean2reference.sort.bam -O clean2reference.sort.dedup.bam -M Sample1.metrics --REMOVE_DUPLICATES TRUE --CREATE_INDEX true --TMP_DIR ./temp 
```
执行结束之后生成clean2reference.sort.dedup.bam文件，进行后续的分析。

