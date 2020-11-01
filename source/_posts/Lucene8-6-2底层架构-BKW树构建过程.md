---
title: Lucene8.6.2底层架构-BKW树构建过程
date: 2020-11-01 16:22:41
tags: Lucene、BKW树、Point
toc: true
categories: Lucene
---
针对数值型的倒排索引，Lecene从6.X引入了BKD树结构，BKD全称：Block K-Dimension Balanced Tree。在此之前，数值型查找和String结构一样，使用<a href="https://kkewwei.github.io/elasticsearch_learning/2020/02/25/Lucene8-2-0%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-%E8%AF%8D%E5%85%B8fst%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/">FST结构</a>）建立索引，FST结构针对精确匹配存在较大的优势，但是数值型很大部分使用场景为范围查找, BKD树就是解决这类使用场景的。

# 使用示例

