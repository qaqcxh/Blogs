---
title: 支配树求解算法——理论篇
categories: 
 - LLVM
 - Graph Theory
---

* toc
{:toc}

# 支配树求解算法—理论篇

## 1. 引言

支配关系是图论中的一个重要课题，被广泛用于编译分析算法，如构建SSA形式的IR、检测循环、支持其他分析优化pass等。单纯求解支配树并不复杂，可以构建简单数据流方程迭代求解。但这种方式效率不高，于是LLVM先后使用了Lengaur-Tarjan算法与SemiNCA算法作为内置的支配树求解算法。本文将介绍这些算法的理论基础，并对算法做整体描述，后续的实践篇将讲解LLVM中的代码实现。

## 2. 前置知识与约定

### 2.1 dfs树

给定一个图G，G的任意一点都可以从节点r到达，那么从r进行dfs，按照先序遍历的方式给节点赋值，就可以得到一棵dfs树(记为T)。下图展示了G与T的关系。

![](/home/qwqcxh/Blogs/Img/dfstree.svg)

G中属于dfs树上的边称为树边，非树边分为前向、返祖、横跨边三种。前向边指向子孙，返祖边指向祖先，横跨边指向不同子树。可以证明**横跨边总是由大指向小**：

1. 假设存在$$ \x $$

### 2.2 记号

$$
A\xrightarrow{yy} A\xhookrightarrow{xx}b
$$





## 3. 支配与支配树



## 4. 迭代数据流算法



## 5. Lengaur-Tarjan算法



## 6. Semi-NCA算法



