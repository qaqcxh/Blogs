---
title: 支配树求解算法——理论篇
categories: 
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

![]({{ site.baseurl }}/Img/dfstree.svg)

G中属于dfs树上的边称为树边，非树边分为前向、返祖、横跨边三种。前向边指向子孙，返祖边指向祖先，横跨边指向不同子树。可以证明**横跨边总是由大指向小**：

1. 观察到：$$\forall v > u \Rightarrow v是u的子孙，或者v位于NearestCommonAncestor(u,v)的更右子树 $$
2. 假设G中存在$$ u\rightsquigarrow v的横跨边，且u < v $$
3. 那么在执行$$ dfs(u)并返回前，一定会检查其子节点v，而u\rightsquigarrow v不被选为树边，说明v已经被访问过 $$
4. 而$$ v > u 且此时u的祖先中不存在更右的子树 \Rightarrow v是u的子孙 \Rightarrow u\rightsquigarrow v属于正向边。与假设矛盾 $$

### 2.2 记号

我们使用如下记号来表示G与T中的边以及路径：

|      | 边                        | 任意路径                               | 非0路径                                |
| ---- | ------------------------- | -------------------------------------- | -------------------------------------- |
| 图G  | $$ x\rightsquigarrow y $$ | $$ x\overset{*p} \rightsquigarrow y $$ | $$ x \overset{+q}\rightsquigarrow y $$ |
| 树T  | $$ x\rightarrow y $$      | $$ x\overset{*p} \rightarrow y $$      | $$ x \overset{+q}\rightarrow y $$      |

路径上方有一个\*或者\+标记，后跟一个可选的路径名称。\*表示路径长度(边数)任意，可以是0，此时两个端点重合；\+表示路径长度至少为1。$$ x\overset{*}\rightsquigarrow y $$表示G中x到y的一条路径。$$ x\overset{*}\rightarrow y $$表示dfs树上x到y的一条路径。

由于每个节点含有先序遍历的值，因此$$ x\overset{*}\rightarrow y \Rightarrow x \leq y $$，$$ x \overset{+}\rightarrow y \Rightarrow x < y $$

我们用$$ v \in x \overset{*p}\rightsquigarrow y $$表示v在p上的一点。不加说明的话v可以是x或者y，如果限制为p上的内部节点的话，则不包含两端点。

> 引理1(路径引理)：$$ \forall v \overset{*p}\rightsquigarrow w, 如果v \leq w \Rightarrow存在p上的一点u，是v和w的公共祖先 $$

> 证明：如果v=w，结论显然成立。如果v$$ < $$w，我们删除T中v w的所有公共祖先，使得不存在树边连通v w所在子树。那么v要到达w必须通过非树边连通两个子树，前向边与返祖边无法跨越子树，而横跨边只能从大到小，因此都不成立。那么v只能通过删除的祖先到达w。

## 3. 支配与支配树

如果从r出发到w的所有路径都经过v，那么就说v支配w或者说w的一个支配是v，我们用dom(w)表示w的支配。根据定义，每个点是自己的支配，定义除了自己外的支配为严格支配(proper dominator)。所有严格支配中最小的那个被称为最近支配(immediate dominator)，记作idom(w)。

显然idom(w)存在且唯一，因为r保证了dom(w)存在，而每个点的值不同保证了唯一。通过最近支配关系，我们可以构建一个有向图，边(x,y)表示x是y的最近支配点，去掉该图的方向后可以得到一棵支配树。简要证明如下：

> 性质：支配关系是一种偏序，具有反自反与传递性。
>
> * 传递性：a支配b, b支配c $$ \Rightarrow $$ a支配c。(根据定义显然)
> * 反自反性：a支配b $$ \Rightarrow $$ b=a或者b支配a。b=a时显然，$$ a \ne b $$时，假设a支配b且b支配a，那么不失一般性设a < b，则dfs到a时，b还未被遍历，此时有一条$$ r \overset{*} \rightarrow a $$且不经过b的路，与b支配a矛盾。
>
> 证明：
>
> * 首先可以证明有向图中不可能有环。否则环中任意两点u v($$ u \ne v $$)，根据传递性有u支配v，v支配u，与反自反性矛盾。
> * 其次可以证明有向图中，r可以到达所有点。设任意点u，沿着最近支配点反向遍历，易知该路径上的点不重复，否则有环。加上点的数目是有限的，因此最后一定会到达r。那么反过来就是r到该点的路径。
>
> * 由上一步可知，去掉方向后的无向图是连通的，加上除了r之外每个点只有一个最近支配点，因此图的\|V\| = \|E\| + 1，必然无环，是一棵树(无环连通图)。



在支配树中，$$ v \rightarrow  w $$表示$$ v = idom(w) $$。$$ v \overset{*}\rightarrow w $$表示v支配w。

## 4. 迭代数据流算法

不少编译参考书都会介绍支配集的数据流求解算法。该算法基于以下发现：

1. 对于任意点w，如果点x属于w所有前驱的支配集的交集，那么x就是w的一个支配点。用公式表示如下：

    $$
    x \in w \cup \{\cap_{v \in pred(w)} dom(v)\} \Rightarrow x \in dom(w)
    $$

2. 对于任意点w，如果x是w的一个支配点，那么x要么是w，要么支配w的所有前驱。否则存在不经过x的路径$$ r \overset{*} \rightsquigarrow pred(w) \rightsquigarrow w $$，与x是w的支配矛盾。用公式表示如下：

   
   $$
   x \in dom(w) \Rightarrow x \in  w \cup \{\cap_{v \in pred(w)} dom(v)\}
   $$


结合这两个公式，我们可以得到如下数据流方程：

$$
dom(w) \equiv w \cup \{\cap_{v \in pred(w)} dom(v)\}
$$

根据该方程，可以直接得到该算法的伪代码：

```cpp
 // dominator of the start node is the start itself
 Dom(r) = {r}
 // for all other nodes, set all nodes as the dominators
 for each n in V - {r}
     Dom(n) = V;
 // iteratively eliminate nodes that are not dominators
 while changes in any Dom(n)
     for each n in V - {r}:
         Dom(n) = {n} union with intersection over Dom(p) for all p in pred(n)
```

求得所有点的支配集后，取其中先序遍历值最大的点作为最近支配点，进而可构建支配树。因为算法需要维护一个支配关系的二维表，因此更新次数为$$ O(n^2 )$$，而每次更新需要检测所有受影响的前驱，所以总体复杂度为$$ O(mn^2) $$

## 5. Lengaur-Tarjan算法

Lengaur与Tarjan在1979年提出了一种接近线性时间的算法，该算法首先计算每个点的半支配点(见下文)，然后用半支配点来计算最近支配点。我们先介绍算法使用到的各种结论，然后给出该算法的伪代码。

### 5.1 半支配点

为了引出半支配点，我们考虑路径$$ v \overset{+q} \rightsquigarrow w $$，其中q的内部节点都比w大。可以断言w的任何支配点u，都是v的祖先。为了得到该结论，考虑路径$$ r \overset{*p} \rightarrow v \overset{+q} \rightsquigarrow w $$，因为q中的内部点都比w大，而u是w的支配，有u < w，所以$$ u \notin q$$。又因为u存在任何到w的路径上，所以必然有$$ u \in p $$，即u是v的祖先。

如果我们取所有上述路径的最小节点v。那么就可以排除所有比v大的点为w的支配点了。我们定义该最小节点v为w的半支配点，记为sdom(w)。其定义如下：


$$
sdom(w) = min\{v | \exists v\overset{+q}\rightsquigarrow w,满足q的所有内部节点都比w大\}
$$


对于$$ v \overset{+q} \rightsquigarrow w$$，且内部节点大于w的路径，称其为半支配路径，称v为候选半支配点。注意到$$ parent(w) \rightarrow w$$没有内部节点，也满足条件，所以$$parent(w)$$是w的一个候选半支配点。

最小的候选半支配就是我们定义的半支配点，显然它存在且不同于w，因为有$$ sdom(w) \leq parent(w) < w $$。

> 引理2: 对于任意不为r的点w，存在dfs树上的路径$$ r \overset{*} \rightarrow id(w) \overset{*} \rightarrow sdom(w) \overset{+} \rightarrow w $$

> 证明：
>
> 因为sdom(w)到w存在半支配路，且$$sdom(w) \leq parent(w) < w $$，由路径引理知半支配路上存在sdom(w)与w的公共祖先，因为半支配路的内部节点都比w大，不可能是w的祖先，因此sdom(w)是w的祖先。即$$ sdom(w) \overset{+} \rightarrow w $$。
>
> 因为idom(w)存在于所有r到w的路径上，且idom(w)不为w，有$$ idom(w) \overset{+} \rightarrow w $$。
>
> 因为$$ r \overset{*} \rightarrow sdom(w) \overset{+q} \rightsquigarrow w $$(q是半支配路)是一条r到w的路，idom(w)必然包含在其中，而q中的内部节点都比w大，内部节点不可能是w的祖先，因此idom(w)只能在$$ r \overset{*} \rightarrow sdom(w) $$上，所以有$$ r \overset{*} \rightarrow idom(w) \overset{*} \rightarrow sdom(w) \overset{+} \rightarrow w $$。 

### 5.2 计算半支配点

对于所有w, $$ w \neq r$$，计算其sdom(w)可以用于后续计算支配点。记$$sdom_1(w)$$为满足半支配路中无内部节点的最小候选半支配点；记$$sdom_2(w)$$为满足半支配路含内部节点的最小候选半支配点。那么$$ sdom(w) = min(sdom_1(w), sdom_2(w))$$。

因为$$parent(w)$$是半支配路没有内部节点的候选半支配点，所以$$sdom_1(w)$$总是存在的。但$$sdom_2(w)$$不一定存在，那么设其为$$ \infty $$

> 命题1：$$sdom_1(w)$$是w的最小前驱。

> 证明：根据定义显然。

> 命题2: $$sdom_2(w)$$是最小的sdom(u)，其中u > w，且存在w的一个非树边前驱v，u是v的祖先。用公式表示如下：
>
> $$ sdom_2(w) = min \{sdom(u) | u > w，存在v使得有路径u \overset{*} \rightarrow v \rightsquigarrow w\} $$

> 证明：
>
> 1. 我们先证明任何满足公式右边条件的sdom(u)是一个候选的$$sdom_2(w)$$，这样可以保证$$min(sdom(u)) \ge sdom_2(w)$$；
> 2.  同时，我们证明$$sdom_2(w)$$是一个sdom(u)，保证等号能取得。这样就可推出$$sdom_2(w) = min(sdom(u))$$。
>
> 先证sdom(u)是一个候选的$$sdom_2(w)$$：
>
> 根据条件，有$$ sdom(u) \overset{+p} \rightsquigarrow u \overset{*q} \rightarrow v \rightsquigarrow w $$(p是半支配路，q是dfs树路)，显然p中的内部节点都大于u，进而大于w。q中的内部节点都是u的子孙，因而大于u，进而大于w。而v是w的非树边，则只能是横跨边或者返祖边，否则如果是前向边则v及其祖先都比w小，不存在大于w的点u，矛盾。而横跨边与返祖边都是从大指向小，因此v大于w。所以sdom(u)是w的一个候选半支配点，且该半支配路存在内部节点u，因此sdom(u)是一个候选的$$sdom_2(w)$$。
>
> 再证$$sdom_2(w)$$是某个sdom(u)。其中，u > w且存在路径$$ u \overset{*} \rightarrow v \rightsquigarrow w $$
>
> 设$$sdom_2(w)$$对应的半支配路上的内部最小节点为u，则有$$sdom_2(w) \overset{+p} \rightsquigarrow u \overset{+q} \rightsquigarrow w$$。因为u在w的半支配路上，$$ u > w $$，又$$sdom(u) \overset{+} \rightsquigarrow u$$中的内部节点都大于u，因而$$sdom(u) \overset{+} \rightsquigarrow u \overset{+q} \rightsquigarrow w$$是w的一条半支配路，$$ sdom_2(w) \le sdom(u)$$。又因为u是p,q的最小值，所以p中的内部节点都比u大，$$sdom_2(w)$$是u的一个候选半支配，有$$sdom(u) \le sdom_2(w)$$。因此$$sdom(u) = sdom_2(w)$$。
>
> 考虑q中倒数第二个点v，因为$$ v > w $$，所以v不是w的祖先，那么v到w是非树边，即$$ u \overset{*s} \rightsquigarrow v \rightsquigarrow w (s位于q上)$$。现在考虑u,v之间的关系，根据路径引理，可知s中存在u,v的公共祖先，而s中u最小，所以u也一定是一个公共祖先。否则存在u,v的某个非u的公共祖先，与u是s中的最小点矛盾。那么我们得到$$ u \overset{*} \rightarrow v \rightsquigarrow w$$
>
> 证毕

### 5.3 半支配点求解伪代码

由5.2节的公式知计算点w的半支配点，我们只需遍历其前驱v，取最小sdom(v)值作为$$sdom_1(w)$$；然后对大于w的前驱，遍历它及其大于w的祖先u，取最小sdom(u)作为$$sdom_2(w)$$即可。

因此，按照逆先序遍历的方式求解sd(w)，保证计算w时，大于它的点都已计算完。伪代码如下：

```cpp
compute_sdom() {
    for w in N...1 { // 逆先序遍历点w
        sd1[w] = sd2[w] = INF; // 初始化w的sd1,sd2为无穷
        for v in pred(w) {
            if (v < w) {
                sd1[w] = min(sd1[w], v);
            } else {
                sd2[w] = min(sd2[w], sdom(u)); //u是所有已遍历的节点中，v的祖先，且sdom值最小的那个
            }
        }
        sd[w] = min(sd1[w], sd2[w]);
    }
}
```

计算sdom(u)需要从v出发，不断沿着dfs树中的祖先遍历来计算最小sdom值，可以通过使用路径压缩的并查集来加速查找。这种方式下，计算所有点的半支配点的复杂度是$$mlog(n)$$，不过原始论文中提到可以设计更复杂的数据结构来优化查找，使复杂度降低为线性。

### 5.4 计算最近支配点

由半支配点计算最近支配点需要用到两个定理。而得到这两个定理还需要引入两个引理：

> 引理3(括号引理)：$$ idom(w) \overset{+} \rightarrow v \overset{+} \rightarrow w \equiv idom(w) \overset{*} \rightarrow idom(v) \overset{+} \rightarrow w $$. 

> 证明：先证一个结论，$$ idom(v) \overset{*} \rightarrow idom(w) \overset{+} \rightarrow v \overset{+} \rightarrow w$$不存在(1)。
>
> 假设存在(1)中的路。因为idom(w)不是v的支配点，所以存在路径$$ r \overset{*p} \rightsquigarrow v$$，且p不经过idom(w)。又v是w的祖先，所以有一条$$ r \overset{*p} \rightsquigarrow v \overset{+} \rightarrow w$$ 的路，该路不经过idom(w)，矛盾。
>
> 现在来证恒等式左边如推出右边。因idom(v)是v的祖先，有$$idom(v) \overset{+} \rightarrow v \overset{+} \rightarrow w$$。因此确定idom(v)，idom(w)的相对大小即可，而根据结论(1)可知idom(w)不可能是idom(v)的子孙，那么只能是$$ idom(w) \overset{*} \rightarrow idom(v) \overset{+} \rightarrow v \overset{+} \rightarrow w$$。
>
> 证右边推出左边类似，只需确定w,v的相对大小。把(1)中的w,v名字交换可知v不可能是w的子孙。

引理3很好记，把idom(v),v分别看作左右括号，由括号匹配规则就能知道不存在([)]的形式，也就对应$$ dom(v) \overset{*} \rightarrow idom(w) \overset{+} \rightarrow v \overset{+} \rightarrow w $$不存在。

> 引理4：如果v是w的祖先，且从v到w的树路上经过的所有点y，$$v \overset{+} \rightarrow y \overset{*} \rightarrow w $$都有$$sd(y) \ge v$$，那么v是w的一个支配点。

> 证明：我们反证如果v不是w的支配点，会产生矛盾。
>
> 假设v不是w的支配，则存在$$ r \overset{*p} \rightsquigarrow w$$，p不经过v。设y是p中离v最近的子孙。因为w是满足条件的一个候选，所以y存在。设x是p中在y之前且小于y的最近的点。该点存在，因为根节点r是x的一个候选。
>
> 因为p中x到y上的所有内部点都大于y，所以x是y的一个候选半支配点。有$$sdom(y) \le x$$。又因为x < y，根据路径引理，x是y的祖先(p中其他点都大于y，不可能是祖先)。由于y是v最近的位于p上的子孙，所以x不可能位于y和p之间，那么x只能是v的祖先了。加上x在p上，p中不含v，所以$$ x \ne v$$，于是我们得到 $$ x < v $$。
>
> 结合$$ sdom(y) \le x$$以及 $$ x < v$$，有$$ sdom(y) < v $$，与条件矛盾。证毕。

引理4将半支配与支配建立起了连续，可用于证明后续的定理1:

> 定理1：设$$\bar{w}$$是sdom(w)的到w的树路上，sdom值最小的点。即
> $$
> sdom(\bar{w}) = min\{sdom(u)  |  \exists u, sdom(w) \overset{+} \rightarrow u \overset{*} \rightarrow w\}
> $$
> 。则$$ idom(w) = idom(\bar{w}) $$。

> 证明：
>
> 对$$ \forall y \in idom(\bar{w}) \overset{*} \rightarrow sdom(\bar{w}) \overset{*} \rightarrow sdom(w) \overset{+} \rightarrow \bar{w} \overset{*} \rightarrow w $$，我们将y所在的路径分成两部分，则$$ y \in idom(\bar{w}) \overset{+} \rightarrow \bar{w} $$ 或者$$ y \in sdom(w) \overset{+} \rightarrow w$$。
>
> 先看第一部分路径，根据括号引理有$$ idom(\bar{w}) \overset{*} \rightarrow idom(y) \overset{+} \rightarrow \bar{w} $$，进而有$$ sdom(y) \ge idom(y) \ge idom(\bar{w}) $$，所以有$$ sdom(y) \ge idom(\bar{w})$$。
>
> 再看第二部分路径，根据$$\bar{w}$$的定义，其位于该路径上，且sdom值最小，所以有$$ sdom(y) \ge sdom(\bar{w}) \ge idom(\bar{w}) $$
>
> 因此对于$$ y \in idom(\bar{w}) \overset{+} \rightarrow w$$，$$ sdom(y) \ge idom(\bar{w}) $$。根据引理4，有$$ idom(\bar{w}) \overset{*} \rightarrow idom(w) $$
>
> 根据$$idom(w) \overset{*} \rightarrow sdom(w) \overset{+} \rightarrow \bar{w} \overset{+} \rightarrow w $$，有$$ idom(w) \overset{+} \rightarrow \bar{w} \overset{+} \rightarrow w$$，根据括号引理，有$$ idom(w) \overset{*} \rightarrow idom(\bar{w}) $$
>
> 综上，$$ idom(\bar{w}) = idom(w) $$

根据定理1，当$$\bar{w} \ne w$$时，可通过其祖先$$ \bar{w} $$的最近支配点来求得。并且如果我们按照先序遍历的方式进行求解，那么在计算idom(w)时$$ idom(\bar{w}) $$已经算完。但当$$ \bar{w} = w$$ 时，就需要用到定理2了：

> 定理2：如果$$ w = \bar{w}$$，则$$idom(w) = sdom(w)$$

> 证明：
>
> 对$$ sdom(w) \overset{+} \rightarrow y \overset{*} \rightarrow w$$中的任意点y，有$$ sdom(y) \ge sdom(\bar{w}) = sdom(w)$$，直接根据引理4，有$$ sdom(w) \overset{*} \rightarrow idom(w) $$。又由引理2，有$$ idom(w) \overset{*} \rightarrow sdom(w) $$，所以$$ idom(w) = sdom(w) $$。

结合定理1，2。我们得到了idom(w)的计算公式：

​	
$$
id(w) = \left\{
\begin{aligned}
id(\bar{w}) & \quad if\ w \ne \bar{w} \\
sd(w) & \quad otherwise\\
\end{aligned}
\right.
$$


### 5.5 最近支配点求解伪代码

这里给一个高度抽象的伪代码/doge。

```cpp
void LengualTarjan(G with N+1 nodes) {
   	preorderDFS(r); // do dfs and assign pre-order number to node, root is 0
    // x is in bucket[k] means sdom[x] == bucket[k]
    for k in 1...N {
        bucket[k] = {};
    }
    // compute sdom
    compute_sdom();
    // fill in bucket[k]
    fill_bucket();
    // compute wbar for all w
    for k in 1...N {
        for w in bucket[k] {
            wbar[k] = node with minimul sdom value in sdom[w]-> w
        }
    }
    // compute idom
    for w in 1...N {
        if w = wbar[w] {
            id[w] = sd[w]
        } else {
            id[w] = id[wbar[w]
        }
    }
}
```





## 6. Semi-NCA算法

17年之后，LLVM官方开始使用SemiNCA算法了，该算法理论复杂度是$$O(n^2)$$,但实际运行时间比Lengaur-Tarjan更好。该算法步骤如下：

1. 与Lengaur-Tarjan一样计算每个点的半支配点。
2. 按照增量的方式构建支配树D。构建时，按照节点的先序遍历值，逐一将其加入D中。当处理点w时，沿着D中的树路径$$r\overset{*}\rightarrow parent(w)$$，找到最深的，且值小于等于sdom(w)的点x，x就是w的最近支配点。在D中新增$$x\rightarrow w$$。需要注意parent(w)是w在dfs树中的父节点，所有节点的值都是dfs树的先序遍历值。

该算法比较简单，其正确性基于以下两个引理：

> 引理5：对于任意非根节点w，其最近支配点idom(w)是支配树D中，sdom(w)与parent(w)的最近公共祖先。

> 证明：我们先证idom(w)是支配数D中sdom(w)与parent(w)的公共祖先。然后证明idom(w)是最近的公共祖先。
>
> 1. 根据引理2以及sdom(w)的定义，易知$$idom(w) \overset{*} \rightarrow sdom(w) \overset{*} \rightarrow parent(w) \overset{+} \rightarrow w$$。根据括号引理，可知$$idom(w) \overset{*} \rightarrow idom(sdom(w))$$以及$$idom(w) \overset{*} \rightarrow idom(parent(w))$$，所以idom(w)是D中sdom(w)与parent(w)的公共祖先。
> 2. 假设idom(w)不是D中sdom(w)与parent(w)的最近公共祖先，而是点v。那么就有$$idom(w)\overset{+}\rightarrow v \overset{*} \rightarrow sdom(w) \overset{*} \rightarrow parent(w) \rightarrow w$$。因为v不是w的支配，所以存在路径$$idom(w)\overset{+p} \rightsquigarrow w$$,且p中不含v。设x是p中离w最近的，且小于w的点。因为idom(w)是一个候选，所以该点存在。那么在$$x\overset{+q}\rightsquigarrow w$$中，q是w的一条半支配路。又由路径引理，x是w的一个祖先。因此有$$idom(w) \overset{*} \rightsquigarrow x \overset{*} \rightsquigarrow parent(w) \rightarrow w$$，我们找到了一条不经过v，但是经过parent(w)的路。与v是D中parent(w)的祖先(v支配parent(w))矛盾。因此idom(w)就是D中sdom(w)与parent(w)的最近公共祖先。

> 引理6：支配树D中任何$$idom(w) \overset{+} \rightarrow _D parent(w)$$的内部节点都大于sdom(w)

> 证明：考虑dfs的路径$$r \overset{*p} \rightarrow idom(w) \overset{*q} \rightarrow sdom(w) \overset{*z} \rightarrow parent(w) \rightarrow w$$,D中从r到parent(w)上的所有点一定位于该路径上。那么$$idom(w) \overset{+} \rightarrow _D parent(w)$$上的内部点一定位于q或者z中。如果位于q中，根据该点支配parent(w)以及括号引理，有该点支配sdom(w)，与idom(w)是sdom(w)，parent(w)的最近公共支配矛盾。因此该点只能在z中，必然都大于sdom(w)。

根据引理6直接得到了前面的SemiNCA算法的步骤2。

## 参考文献

1. [支配树-维基百科](https://en.wikipedia.org/wiki/Dominator_(graph_theory))
2. [An Explanation of Lengauer-Tarjan Dominators Algorithm](https://www.cs.utexas.edu/users/misra/Lengauer+Tarjan.pdf)
3. [Finding Dominators in Practice](https://renatowerneck.files.wordpress.com/2016/06/gtw06-dominators.pdf)
