# 康威生命游戏规则

*注：如果你已经熟悉了康威的生命游戏及其规则。请随意跳到下一节！*。

[维基百科对康威生命游戏的规则做了一个很好的描述:][wikipedia]

> 生命游戏的宇宙是一个无限的二维正交方格网格，
> 每个方格都处于两种可能状态中的一种，活或死，（或分别为有人居住和无人居住）。
> 每个单元格与其八个相邻单元格相互作用，这些单元格是水平、垂直或对角相邻的单元格。
> 在时间的每一步，都会发生以下转换：
>
> 1. 任何具有少于两个活邻居的活单元格都会死亡，就像人口不足一样。
> 2. 任何有两个或三个活邻居的活单元格都会传给下一代。
> 3. 任何拥有三个以上活邻居的活单元格都会死亡，就像人口过多一样。
> 4. 任何只有三个活邻居的死单元格都会变成活单元格，就像通过繁殖一样。
>
> 这些将自动机的行为与现实生活进行比较的规则可以浓缩为以下内容：
>
> 1. 任何有两个或三个活邻居的活单元格都能存活。
> 2. 任何具有三个活邻居的死单元格都会成为活单元格。
> 3. 所有其他活单元格在下一代中死亡。同样，所有其他死单元格保持死亡。 
>
> 初始模式构成了系统的种子。 
> 第一代是通过将上述规则同时应用于种子中的每个单元格，
> 无论是活的还是死的； 出生和死亡同时发生，
> 发生这种情况的离散时刻有时称为滴答声。 每
> 一代都是前一代的纯函数。 这些规则不断被反复应用以创造更多的世代。  

[wikipedia]: https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life

考虑以下初始宇宙:

<img src='../images/game-of-life/initial-universe.png' alt='Initial Universe' width=80 />

我们可以通过考虑每个单元格来计算下一代。左上角的单元格已死。规则 (4) 是唯一适用于死单元格的转换规则。然而，因为左上角的单元格没有正好三个活着的邻居，所以转换规则不适用，它在下一代中仍然是死的。第一行中的每个其他单元格也是如此。

当我们考虑第二行第三列的顶部活单元格时，事情变得有趣了。对于活单元格，前三个规则中的任何一个都可能适用。在这个单元格的情况下，它只有一个活着的邻居，因此规则（1）适用：这个单元格将在下一代死亡。同样的命运等待着底部的活单元格。

中间的活单元格有两个活的邻居：顶部和底部的活单元格。这意味着规则 (2) 适用，并且它在下一代中仍然存在。

最后一个有趣的例子是中间活单元格左侧和右侧的死单元格。三个活单元格都是这两个单元格的邻居，这意味着规则（4）适用，这些单元格将在下一代变得活跃。

把它们放在一起，我们在下一个滴答后得到这个宇宙：

<img src='../images/game-of-life/next-universe.png' alt='Next Universe' width=80 />

从这些简单的、确定性的规则中，出现了奇怪而令人兴奋的行为:

| Gosper's glider gun | Pulsar | Space ship |
|---|---|---|
| ![Gosper's glider gun](../images/wiki/Gospers_glider_gun.gif) | ![Pulsar](../images/wiki/Game_of_life_pulsar.gif) | ![Lighweight space ship](../images/wiki/Game_of_life_animated_LWSS.gif) |

<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/C2vgICfQawE?rel=0&amp;start=65" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>
</center>

## 练习

* 用手计算我们的例子宇宙的下一个刻度。注意到任何熟悉的东西吗？

  <details>
    <summary>答案</summary>

    它应该是例子宇宙的初始状态。

    <img src='../images/game-of-life/initial-universe.png' alt='Initial Universe' width=80 />

    这种模式是*周期性的*：它在每两个ticks之后返回到初始状态。

  </details>

* 你能找到一个稳定的初始宇宙吗？就是说，一个每一代都是一样的宇宙。

  <details>
    <summary>答案</summary>

    有无限多的稳定的宇宙! 琐碎稳定的宇宙是空宇宙。一个由活单元格组成的2乘2的正方形也是一个稳定的宇宙。

  </details>
