---
title: Elasticsearch中排序和相关性
date: 2015-07-18 10:00:00
tags: sorting,relevance,tf/idf
author: maclaon
comments: false
---
# 相关性
目前计算相关性主要是词频统计，最经典的莫过于`TF-IDF`，下面来着重讲一下对应的概念。

## TF-IDF
> Term frequency: 当前文档中，单词honeymoon在该document中的tweet字段中出现多少次？
> Inverse document frequency: 单词honeymoon在所有document中的tweet字段中出现多少次？

最终效果：tf-idf倾向于过滤掉常见的词语，保留重要的词语。如果某个词或短语在一篇文章中出现的频率TF高，并且在其他文章中很少出现，则认为此词或者短语具有很好的类别区分能力，适合用来分类。


