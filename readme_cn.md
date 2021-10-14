# ![SVGNest](http://svgnest.com/github/logo2.png)

**SVGNest**: 一个基于浏览器的向量排样工具。

**Demo:** [svgnest](http://svgnest.com)（需要 [SVG](https://developer.mozilla.org/en-US/docs/Web/SVG) 和 [web workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers) 的支持）。移动端提醒：运行这个demo对CPU消耗较大

参考文献(PDF):

- [López-Camacho *et al.* 2013](http://www.cs.stir.ac.uk/~goc/papers/EffectiveHueristic2DAOR2013.pdf)[^López-Camacho,2013]
- [Kendall 2000](http://www.graham-kendall.com/papers/k2001.pdf)[^Kendall,2000]
- [E.K. Burke *et al.* 2006](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.440.379&rep=rep1&type=pdf)[^Burke,2006]

## 什么是排样？

给定一个正方形的材料和一些字母用于激光切割：

![letter nesting](http://svgnest.com/github/letters.png)

我们需要将所有的字母排放到正文形中，尽可能的少使用材料。如果一个正方形不能满足，我们可以找到正方形需求的最小数量。

在CNC的世界里面这个称为[排样](http://sigmanest.com/)，有[软件](http://www.mynesting.com/)在为[工业界客户](http://www.hypertherm.com/en/Products/Automated_cutting/Nesting_software/) 做[这件事](http://www.autodesk.com/products/trunest/overview)，并且[非常昂贵](http://www.nestfab.com/pricing/)。

SVGnest是一个免费的、开源的替代品，它使用[^Burke,2006]中的解决这个问题，使用基因算法用于全局优化。它可以工作在任意形状的容器和凹面部件下，并且效果与存在的商业软件可以看齐。

![non-rectangular shapes](http://svgnest.com/github/shapes.png)

它也支持部件内嵌部件，可以在其他部件的孔洞中排放部件。

![non-rectangular shapes](http://svgnest.com/github/recursion.png)

## 使用

确保所有的部件都已经转化为轮廓，并且没有轮廓覆盖。上传SVG文件，并且选择其中一个轮廓作为箱子。

所有其他的轮廓会自动作为排样部件来运行。

## 算法概述

在矩形箱式排样问题中存在[好的启发式](http://cgi.csc.liv.ac.uk/~epa/surveyhtml.html)，但是在真实的世界我们关注的是不规则的形状。

排样策略有两部分组成：

- 布局策略（如：怎样在箱子中插入每个块）
- 优化策略（如：最好的排列顺序）

### 排放形状

这里的关键概念是“临界多边形”

给定多边形 A 和 B，我们希望“轨道” B 环绕 A，并且它们保持接触而不相交。

![No Fit Polygon example](http://svgnest.com/github/nfp.png)

轨道输出的是NFP。NFP包括了B与先前放置部分保持接触的所有可能的位置。然后，我们可以选择NFP上的一个点使用某种启发式算法找出排放的位置。

同样，我们可以构造一个“内拟合多边形”为箱子和部件。与NFP的过程相似，除了轨道多边形在固定多边形的内部。

当两个或者更多部件被放置，我们可以将先前放置的部件的NFP进行合并。

![No Fit Polygon example](http://svgnest.com/github/nfp2.png)

这表示我们需要计算 $O(n\log n)$ 个NFP去完成首次排样。当存在方法去减轻这个任务时，我们采用优化算法中拥有很好性质的暴力算法。

### 优化

当可以排放部件时，我们需要优化插入的顺序。下面是个插入顺序的坏例子：

![Bad insertion order](http://svgnest.com/github/badnest.png)

如果大的“C”最后排放，凸的空间就没有办法使用，因为所有的部件都已经被排放完成。

为了解决这个问题，我们使用“首次使用降充”启发式算法。大的部件先排放，小的部分后排放。这个是符合直觉的，因为小的部件倾向于扮演“沙子“的角色填充大的部件留下的空隙。

![Good insertion order](http://svgnest.com/github/goodnest.png)

当这个策略给了好的开始，我们就希望更多的解空间。我们可以简单的随机化插入顺序，但是我们可以使用基因算法做得更好。（了解基因算法可以参考[这篇文章](http://www.ai-junkie.com/ga/intro/gat1.html)）。

## 评估拟合

在我们的基因算法中，插入的顺序和部件的旋转来自于基因。拟合函数遵行下面的规则：

1. 最小化不可以排放部件的数目（部件由于旋转导致无法拟合任何箱子）
2. 最小化箱子的使用数目
3. 最小化所有排放部件的宽度

第三个条件是相当随意的，因为我们也可以为矩形边界或者最小的凹壳进行优化。在真实的世界，使用的材料趋向于切割成矩形，并且这些选择趋向于将未使用的材料切成长条形。

因为基因中小的突变在整个拟合过程中会导致潜在的大的变化，种群的个体会变得非常小。通过缓存NFP，新的个体的评估速度会非常快。

## 性能

![SVGnest comparison](http://svgnest.com/github/comparison1.png)

与商业软件相比，共同运行5分钟后效果相似。

## 配置参数

- **Space between parts:** 部件之间的最小距离（如：激光切口、CNC 的偏移量等）
- **Curve tolerance:** Bezier路径或者弧的线性逼近容易的最大误差，设置为SVG的单元或者像素。如果曲面部件有轻度重叠可以减少这个值。
- **Part rotations:** 为每个部件估计的*可能的*旋转次数。如：4只代表基本方向。更大的值可以改进结果，但是会收敛得更慢。
- **GA population:** 基因算法的种群大小。
- **GA mutation rate:** 每个基因或者部件的位置发生突变的概率。值从1~50
- **Part in part:** 启用后，将部件置于其他部件的孔洞中。默认情况下是关闭的，因为它可能是资源密集型计算。
- **Explore concave areas:** 启用后，付出一些性能和放置的稳健性为代价解决凹面的问题

![Concave flag example](http://svgnest.com/github/concave.png)

## To-do

- ~~Recursive placement (putting parts in holes of other parts)~~
- Customize fitness function (gravity direction, etc)
- kill worker threads when stop button is clicked
- fix certain edge cases in NFP generation

# 参考文献

[^Burke,2006]:Burke E K, Hellier R S R, Kendall G, et al. Complete and robust no-fit polygon generation for the irregular stock cutting problem[J]. European Journal of Operational Research, 2007, 179(1): 27-49.
[^Kendall,2000]:Kendall G. Applying meta-heuristic algorithms to the nesting problem utilising the no fit polygon[D]. University of Nottingham, 2000.
[^López-Camacho,2013]:López-Camacho E, Ochoa G, Terashima-Marín H, et al. An effective heuristic for the two-dimensional irregular bin packing problem[J]. Annals of Operations Research, 2013, 206(1): 241-264.
