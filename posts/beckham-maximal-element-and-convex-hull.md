# 贝克汉姆，极大元与凸包

- pubdate: 2012-07-11

---

## 序

这是一篇一个月前就应该写好的文章。

这是一篇伪学术伪科普文。

## 起

这故事还得从上一次年级大会说起。会上辅导员表扬了两位女生，评价如下：“一个是学习最好的运动员兼运动能力最好的学生，一个是学习最好的干部和办事能力最强的好学生”。当场我就联想到了人们对足球运动员大卫·贝克汉姆的经典评价：“踢得好的没他长得帅，长得帅的没他踢得好”。

类似的评价直观上给人一种“要满足这种条件好像很少甚至唯一”的感觉。那么究竟是不是呢？对于给那两位女生的评价，的确；但对于贝克汉姆们，就不是了。这里要注意前后两种评价标准的不同，前一种是两类人+两种属性（[运动员]中[学习]最好兼[学生]中[运动能力]最强），后一种则是一类人+两种属性（[足球运动员]中[踢得好]兼[足球运动员]中[长得帅]）。

今天要说的，和贝克汉姆有关。

## 承

贝克汉姆那个的问题其实是一个[极大元（Maximal Element）][maximal_element]问题。极大元是一个偏序集（Partially Ordered Set）中满足特定属性的元素。什么是偏序集呢？就是一个定义了偏序的集合。那什么是偏序呢？偏序是一种二元关系（Binary Relation），对集合`P`中任意元素`a, b, c`满足如下条件（不妨把这个关系记作`<`）：

* `a < a`
* 如果`a < b`且`b < a`，那么`a = b`
* 如果`a < b`且`b < c`，那么`a < c`

这3个属性，从上到下依次是自反性（Reflexivity）、反对称性（Anti-symmetry）和传递性（Transitivity）。不过这么说还是有点抽象，让我们看看偏序关系的典型例子：首先是自然数集上的*整除关系*（这里用`a | b`记作`a`能整除`b`）：

* 任意自然数`a`，`a | a`
* 如果`a | b`且`b | a`，那么`a = b`
* 如果`a | b`且`b | c`，那么`a | c`

其次是一个集合的[幂集][power_set]的*子集包含*关系（这里用`A C B`表示`B`包含`A`）：

* 集合`P`的任意子集`A`，`A C A`
* 如果`A C B`且`B C A`，那么`A = B`
* 如果`A C B`且`B C D`，那么`A C D`

第二个偏序关系可以用Hasse图（Hasse Diagram）来表示，比如建立在集合`{x, y, z}`的幂集上的该关系用Hasse图表示如下：

<img alt="Hasse Diagram" src="http://upload.wikimedia.org/wikipedia/commons/thumb/e/ea/Hasse_diagram_of_powerset_of_3.svg/400px-Hasse_diagram_of_powerset_of_3.svg.png" />

(From: [Wikipedia](http://upload.wikimedia.org/wikipedia/commons/thumb/e/ea/Hasse_diagram_of_powerset_of_3.svg/400px-Hasse_diagram_of_powerset_of_3.svg.png))

理解了偏序，我们就可以定义极大元了。一个偏序集上的极大元满足如下条件：

* 如果`m`是定义了偏序`<`的偏序集`P`上的极大元，不存在这样的`x`，使得`m < x`

之前定义偏序的时候，有没有觉得直观上偏序有一种“发散”的感觉（比如整除关系），当然这不是一定的。但一定的是，偏序关系中，任意两元素**不一定可比**，比如上面子集包含的例子中，`{x, y}`和`{y, z}`这两个子集互不包含，所以无法比较。极大元就是每个可比较的“串”中最大的那个，换句话说，极大元可能有唯一（比如`{x, y, z}`是上面例子中的唯一极大元，但极大元也不一定存在（想想整除关系）。如果任意两个元素都可比呢？这种情况下，偏序就变成了全序（Totally Ordered），如果极大元若存在则一定唯一，也就是最大元（maximum）。

再回到贝克汉姆的例子，如果集合`P`是所有足球运动员的集合，定义在`P`上的偏序`<`是：

* `B < A`：B不比A更帅也不比A技术更好

所以说……贝克汉姆不一定是唯一满足条件的人，因为存在技术比贝克汉姆差但长得比他帅或者长得没他帅但技术比他好的运动员，对于这些人（即`P`中的所有极大元）人，我们都用“踢得好的没他长得帅，长得帅的没他踢得好”来评价他们。

## 转

那么，如何快速找到最大元呢？最简单的方法当然是两两对比，当可比较时剔除较小的元素，复杂度为`O(N^2)`。为了寻找更快的算法，我Google了一下，在Stackoverflow上发现了[这个][sto_exam]，同时我惊奇地发现，这是上学期算法课期中考试的一道题！当时我给出了一个`O(NlogN)`的算法。然后，老外的回答让我一身冷汗：

> That's a really really **evil** exam question (unless your instructor told you about one of the O(n log h) algorithms for convex hull, in which case it's merely evil).

evil？convex hull？不管怎样，原来最优算法是O(NlogH)的！老外继续说道：

> The problem is called 2D MAXIMA and is the typically the domain of computational geometers because of its close relationship to computing convex hulls. I can't find a good description of a O(n log h) algorithm for the maxima problem, but Timothy Chan's O(n log h) algorithm for 2D CONVEX HULL should give you the flavor.

Wow，原来这个问题是一个经典的计算几何问题。在二维/三维平面上可以用[Chan's Algorithm][chans_algo]达到`O(NlogH)`的时间复杂度。

[Convex hull][convex_hull]，中文叫做凸包。在二维平面上，形象点说就是n个点的“边界点”，所有n个点都落在这些点构成的边界中。想象一条橡皮筋套住这n个点的情形，最后直接由橡皮筋连起来的那些点就是凸包了，如图所示：

<img alt="Convex Hull" src="http://upload.wikimedia.org/wikipedia/commons/thumb/d/de/ConvexHull.svg/400px-ConvexHull.svg.png" />

(From: [Wikipedia](http://upload.wikimedia.org/wikipedia/commons/thumb/d/de/ConvexHull.svg/400px-ConvexHull.svg.png))

可以看出，求所有极大元其实是求凸包的子问题：如果我们能求出凸包，再进行一次`O(H)`的扫描，就能找到所有极大元。让我们先来看一个算法：[Jarvis March][jarvis_march]。这个算法的思想很简单，首先，最左的那个点一定属于凸包，然后每一轮都从上一个属于凸包的点向上方作垂直的射线，将这条射线顺时针旋转，遇到的第一个点就是下一个属于凸包的点。如图所示：

<img alt="Jarvis March" src="http://upload.wikimedia.org/wikipedia/commons/thumb/d/de/Jarvis_march_convex_hull_algorithm_diagram.svg/400px-Jarvis_march_convex_hull_algorithm_diagram.svg.png" />

(From: [Wikipedia](http://upload.wikimedia.org/wikipedia/commons/thumb/d/de/Jarvis_march_convex_hull_algorithm_diagram.svg/400px-Jarvis_march_convex_hull_algorithm_diagram.svg.png))

由于每一轮都找出一个属于凸包的点，而每一轮都需要扫描所有点，所以最终的时间复杂度是`O(NH)`。另外，这个算法又叫做Gift Wrapping，想想用丝带包装礼物的情景，是不是很形象呢？

我们接着看[Graham Scan][graham_scan]。这个算法的思想是这样的：先找出`y`值最小的点（这个点必然属于凸包），然后计算所有其他点和这个点与X轴正方向形成的夹角大小，并排序。然后按序扫描这些点，扫描到一个点时，都假定这个点在凸包中，万一*很巧*，这`N`个点正好都在凸包中，那么如果你从第一个点开始按照这些点被加入的顺序行走，每次拐弯应该都是“左转”。实际情况不可能每次都这么巧，所以如果你还是按照这些点被扫描的顺序走，但“右转”了，这说明上一个被假定在凸包中的点其实不在凸包中。如图所示：

<img alt="Graham Scan" src="http://upload.wikimedia.org/wikipedia/commons/thumb/e/ed/Graham_Scan.svg/300px-Graham_Scan.svg.png" />

(From: [Wikipedia](http://upload.wikimedia.org/wikipedia/commons/thumb/e/ed/Graham_Scan.svg/300px-Graham_Scan.svg.png))

这个算法中排序需要`O(NlogN)`，扫描需要`O(N)`，所以最终是一个`O(NlogN)`的算法。

为什么要说一个`O(NH)`、一个`O(NlogN)`的算法呢？因为[Chan's Algorithm][chans_algo]是一个分治的算法，每一个子问题中都先用Graham Scan（或其他`O(NlogN)`的算法）解决，然后再对所有子问题的中间解进行一次Jarvis March。除了分治，Chan's Algorithm还用到了迭代尝试的思想，具体可参见上面的链接。

## 合

最后总结下这篇文章。故事从枯燥的年级大会八卦到了贝克汉姆，然后从一个颇具智慧的评价联系到了极大元问题，又从如何求极大元的问题搜索到了一个相关的问题——凸包。在求凸包这个问题上，算法的时间复杂度从`O(N^2)`到`O(NH)`，再到`O(NlogN)`，最后到`O(NlogH)`，算法的优化之美跃然于眼前。

诚然，我们生活的世界，是一个充满了艺术、需要用感性去体验的世界，但点点滴滴之中也不乏科学的道理。之所以写这篇文章，不仅想记录使用网络发现知识的惊喜与快乐，也是我对理论与生活结合的坚信。

[sto_exam]: http://stackoverflow.com/questions/8500840/algorithm-to-compute-maximal-points-in-pointset
[chans_algo]: http://en.wikipedia.org/wiki/Chan%27s_algorithm
[maximal_element]: http://en.wikipedia.org/wiki/Maximal_element
[power_set]: http://en.wikipedia.org/wiki/Power_set
[convex_hull]: http://en.wikipedia.org/wiki/Convex_hull
[jarvis_march]: http://en.wikipedia.org/wiki/Jarvis_march
[graham_scan]: http://en.wikipedia.org/wiki/Graham_scan
