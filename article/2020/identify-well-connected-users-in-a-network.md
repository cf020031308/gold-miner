> * 原文地址：[Identify well-connected Users in a Network](https://towardsdatascience.com/identify-well-connected-users-in-a-social-network-19ea8dd50b16)
> * 原文作者：[Guenter Roehrich](https://medium.com/@GuenterRoehrich)
> * 译文出自：[掘金翻译计划](https://github.com/xitu/gold-miner)
> * 本文永久链接：[https://github.com/xitu/gold-miner/blob/master/article/2020/identify-well-connected-users-in-a-network.md](https://github.com/xitu/gold-miner/blob/master/article/2020/identify-well-connected-users-in-a-network.md)
> * 译者：[cf020031308](https://github.com/cf020031308)
> * 校对者：

# 识别网络中连接紧密的用户

本文主要介绍如何使用无向图和 Scipy 的稀疏矩阵实现（COO）来存储数据和分析用户之间的连接。

![“连接” —— 照片来自 [Unsplash](https://unsplash.com?utm_sourceu003dmedium&utm_mediumu003dreferral) 上的 [NASA](https://unsplash.com/@nasa?utm_sourceu003dmedium&utm_mediumu003dreferral)]( https://cdn-images-1.medium.com/max/13292/0*Ry3dSWy5ckRtkTHY)

最近我发现我的一位前雇主建立了一个类似 Facebook 的社交网络，有大量数据亟待分析。社交数据已相当成熟，应用也很广，因此我决定对其进行更深入的研究。

通过网上调研，我发现一种可行的“计算连接质量”的有趣方法，其核心是数三角形：通过两个用户之间共同连接的第三者的数量判断他俩连接的紧密程度。为了便于理解，我们看一个简单的例子。

> **A 认识 B，B 认识 C**。很简单的关系，不存在三角形，这不是我们要关注的。
>
> **A 认识 B**，**B 认识 C**，**C 认识 A**。对，这就是我们想要的，因为在三个人的群体一个人最多有 2 个连接。因此，这里 2 就是最大的紧密连接数。

![一个无向图的例子](https://cdn-images-1.medium.com/max/2000/1*d4TCH2_ayplfLN2EUAtxLg.png)

该图的矩阵表示如下（[triangle.py](https://gist.github.com/guenter-r/87196fcca6a6cda0d211a2e14fc200f4/raw/0fa50b13c874938646d4a730b6c6b1e1073cf2f9/triangle.py)）：

```python
graph = np.array([
    [0,1,1,0,0,0,0,0],
    [1,0,1,1,1,0,0,1],
    [1,1,0,0,0,0,0,0],
    [0,1,0,0,1,0,1,1],
    [0,1,0,1,0,1,0,0],
    [0,0,0,0,1,0,0,0],
    [0,0,0,1,0,0,0,0],
    [0,1,0,1,0,0,0,0]
])
```

[卡内基·梅隆大学](http://www.math.cmu.edu/~ctsourak/tsourICDM08.pdf) 发表过一篇在矩阵中计算三角形的有趣论文[1]，提供了容易理解的伪代码片段——不过我们后面研究的将会是另一种方法。Charalampos E. Tsourakakis （卡内基·梅隆大学）在该论文中提出了一种已被广泛验证的高精度估算方法（精确度超过 95％），即通过“特征值的立方和”计算网络中含有的三角形个数.

![C. E. Tsourakikis 提出的“特征三角形”](https://cdn-images-1.medium.com/max/2000/1*IA7JteyZb7Na8iA2sc269w.png)

翻译成人话就是，用 $\lambda$ 表示无向图的邻接矩阵的特征值，计算所有特征值的立方和就可以算出图中的三角形数量。注意前面除以的 6：三角形有三个点，每个点数紧密连接数了两次，所以要除以这个分母——稍后我们会再次见识到这个神奇的 6。

如果你也是一个“特征人”（来自 MIT 教授 Gilbert Strang 的玩笑话[2]），肯定会喜欢上面的“特征三角形”公式。顺便提一下 Tsourakakis 的另一个快速估算法，只需要输入邻接矩阵和可接受误差（如果你使用的是我们前面的这个小例子，1e-5 就够了）：

![Charalampos E. Tsourakakis：大型真实网络中的三角形快速计数（原文）](https://cdn-images-1.medium.com/max/2000/1*6-camzK0rSQJy9T963QmRQ.png)

对计算机程序来说“特征三角形”一个非常简单的任务，如下是一个简单的 Python 实现（[eigen_triangle.py](https://gist.githubusercontent.com/guenter-r/f784a8cb3c0aebb80407597f37b30878/raw/5ef3ffaef03be56a79f4c1da72506b03f59737e4/eigen_triangle.py)）：

```python
from numpy import linalg

def return_eig_triang(graph):
  w,v = linalg.eig(graph)
  w,v
  no_of_triangles = round(np.sum(w**3) / 6,1)
  return no_of_triangles # 矩阵中的三角形总数
```

由此可计算得我们的例子中总共有 3 个三角形，这一点我们从图中也可以看到。三个三角形分别是 **(0,1,2)**，**(1,3,4)** 和 **(1,3,7)**，因此对比用户 **5** 和 **6**，用户 **0,1,2,3,4,7** 在网络中的连接要更为紧密。

---

前面为了观察输出，我们使用的是一个特殊的数据例子。现在我们将继续讨论更加随机的情况，为此先创建一个人工的稀疏数据集。我随机生成了一个有 20 个用户的简单的数据集——实际网络的规模当然要大得多，因此我们将使用坐标表示的稀疏矩阵，虽然对我们这个 20 个用户的人工数据集来说还没必要用稀疏矩阵。

创建一个随机数据集：[create_coo.py](https://gist.githubusercontent.com/guenter-r/1a516c9cc197624b0247a87550a32b90/raw/02d7c1bf7663a4f871249ac579d897b4f1724cf0/create_coo.py)

```python
# 创建稀疏数据集
connections_list = []

for i in range(31):
    connections = np.random.choice(21,2) # 可能有重复数据，先试试看能不能得到三角形……
    connections_list.append(list(connections))

print(connections_list)
```

现在可以看看我们生成的随机数据——请注意我们还将继续使用本文第一个小例子，来验证此过程有效。通过前面的代码我们随机生成了用户之间的一些连接。如果我们看到有连接 (8, 7) 和 (16, 8)，自然希望再有连接 (16, 7)，这样就形成紧密的三角形了。但注意，用户不能与自己连接成为朋友！

**随机**函数生成的数据如下——乍一看，我觉得可以用（重复的连接没关系，但要避免自己与自己连接）：

```
[[2, 3], [18, 3], [14, 10], [2, 10], [3, 12], [10, 11], [19, 16], [7, 2], [6, 0], [14, 9], [11, 5], [12, 3], [19, 8], [9, 0], [4, 18], [12, 19], [10, 4], [3, 2], [16, 6], [7, 3], [2, 18], [10, 11], [8, 18], [2, 10], [19, 1], [17, 3], [13, 4], [2, 0], [17, 2], [4, 0], [1, 17]]
```

初步检查 X 坐标与 Y 坐标，可以发现 **{2, 3, 7, 17}** 出现在了三角形中。检查的方式很简单，只要看连接的两端是否有 2 就可以了。因为我们打算将稀疏数据存储在专门的数据结构中，这里我们谈谈稀疏矩阵。为了便于理解，我写了如下一个简单的代码片段，对称地添加坐标（[coo_creation.py](https://gist.githubusercontent.com/guenter-r/d96cd2a91f9d5e5d400142d2f859a1c6/raw/d20c19f40440c3295d5b5063cbd81fd28457a198/coo_creation.py)）：

```python
x,y = [],[]

for element in connections_list:
    xi = element[0]
    yi = element[1]
    
    x.append(xi)
    x.append(yi) # 对称添加
    y.append(yi)
    y.append(xi) # 对称添加

# 稀疏矩阵中的值为 1
vals = np.ones(len(x))

# numpy(scipy.sparse)
from scipy.sparse import coo_matrix
m = coo_matrix((vals,(x,y)))
```

前面生成的连接集合中，很可能就含有三角形。为了计算大规模网络中某个结点（它的连接由邻接矩阵中的一行 **X[该点索引值]** 表示）所在的三角形个数——这不同于前面我们计算整个网络中的三角形个数的方法——我们需要另一种计算方法。

在计算特定结点所在的三角形数量这个问题上我被困了不少时间，做了些练习后，我从 Richard Vuduc 教授[3]（UC Berkeley, Georgia Tech）的一本书中学到了一个非常好的线性代数方法。

首先，我们计算邻接矩阵 M 与自身的点积（M 和其转置 M.T 相等）。这会得到从一个用户到另一个用户的所有两步连接数，但不相连的用户之间（甚至自己到自己）也可能有两步连接，因而对于数三角形的任务来说会存在“计数过高”。不过当点积再与初始矩阵 M 按元素相乘后，这些多余的计数就会被“归零”。

**真是一个优雅的方案！**

```python
M = (m.dot(m)) * m # 点积与按元素相乘
```

这样我们就终于能得到连接紧密的用户了。最后，我们将计算每个用户所在的三角形数量。注意任何给出的数字（译者注：指后文的“三角形值”）都不能解释为“用户 m 所在的三角形有 n 个”，但其结果可以用来排序，以便找到网络中连接最紧密或最疏松的用户。

注意：正如前面提到的，如果我们把所有连接数加起来并除以神奇的 6（每个三角形除以 3，每条连除以 2），就又能得到整个网络中的三角形数量了。

从前面的初步检查中已经知道，在用户 2 的连接中最有可能出现三角形，这也可以在我们的示例数据中看出来。高三角形值下面用星号突出显示：

```
user:  0 	Triangle value:  [[66.]]
user:  1 	Triangle value:  [[26.]]
user:  2 	Triangle value:  [[187.]] *
user:  3 	Triangle value:  [[167.]] *
user:  4 	Triangle value:  [[69.]]
user:  5 	Triangle value:  [[13.]]
user:  6 	Triangle value:  [[22.]]
user:  7 	Triangle value:  [[70.]]
user:  8 	Triangle value:  [[30.]]
user:  9 	Triangle value:  [[24.]]
user:  10 	Triangle value:  [[127.]] *
user:  11 	Triangle value:  [[59.]]
user:  12 	Triangle value:  [[71.]]
user:  13 	Triangle value:  [[15.]]
user:  14 	Triangle value:  [[34.]]
user:  15 	Triangle value:  [[0.]]
user:  16 	Triangle value:  [[15.]]
user:  17 	Triangle value:  [[77.]]
user:  18 	Triangle value:  [[93.]]
user:  19 	Triangle value:  [[39.]]
```

从这个例子中我们可以看出：与三角形相关的结点确实具有相对较高的值。但是，我们还需要额外考虑 **“简单连接也会增加整体值”**。这在没有三角形的 **10** 和有三角形的 **17** 情况下尤其明显。（译者注：这部分问题太多，翻译就不润色了，后面统一补充说明。）

---

如前所述，我们再看看刚开始的例子，确保我们的结果合理。因为用户 1 涉及了全部 3 个三角形，我们自然期望它有最高的值。通过完全一样的步骤，我们得到了如下结果，证实了我们的期望：

```
user:  0 	Triangle value:  2
user:  1 	Triangle value:  6
user:  2 	Triangle value:  2
user:  3 	Triangle value:  4
user:  4 	Triangle value:  2
user:  5 	Triangle value:  0
user:  6 	Triangle value:  0
user:  7 	Triangle value:  2
```

---

关于如何找出无向图中紧密连接的结点，并用用稀疏矩阵储存和处理网络数据，希望我的这个小例子对您有用，也希望得知您将这些逻辑用到其它问题上。下次见 🔲

---

[1] Charalampos E. Tsourakakis, Fast Counting of Triangles in Large Real Networks: Algorithms and Laws

[2] Prof. Gilbert Strang, MIT, https://www.youtube.com/watch?v=cdZnhQjJu4I

[3] Prof. Richard Vuduc, CSE 6040, https://cse6040.gatech.edu/fa17/details/

---

**译者注：** 原文的理论没问题，但代码实现中的问题较严重，导致结果完全不对。

1. 作者生成的连接集合有重复的，比如 [2, 10] 出现了两次，这导致按他代码生成的矩阵中，部分元素会大于 1。因此不论是点积还是按元素相乘，结果都会很大。正确的做法是添加一行 `m = m.sign()` 使矩阵中元素只有 0 和 1；
2. numpy 中的 '*' 不一定是按元素相乘。具体来说，如果操作的类型是 `numpy.ndarray` 才是按元素相乘，如果是 `numpy.matrix` 或 `scipy.sparse.coo.coo_matrix` 则是点积，与 `dot()` 一样。所以作者代码中 M 的结果实际上是两次点积（三次方），应改为 `M = m.dot(m).multiply(m)`；
3. 其实这样的 M 中每个元素值就刚好是行列对应的两个结点间的三角形数量，但作者似乎因为代码问题导致结果不对，因而强行说这个值不等于三角形数量只能用来排序。

修正的代码汇总如下：

```python
# 这里会看到其中部分元素大于 1
print(m.todense())

# 使大于 1 的元素为 1
m = m.sign()

# 正确的点积后按元素相乘
M = m.dot(m).multiply(m)

# 输出为 [[0. 0. 6. 6. 0. 0. 0. 2. 0. 0. 0. 0. 0. 0. 0. 0. 0. 2. 2. 0.]]
# 即 2、3 有 6 个三角形，7 有 2 个三角形，……
print(M.sum(axis=1).T)
```

> 如果发现译文存在错误或其他需要改进的地方，欢迎到 [掘金翻译计划](https://github.com/xitu/gold-miner) 对译文进行修改并 PR，也可获得相应奖励积分。文章开头的 **本文永久链接** 即为本文在 GitHub 上的 MarkDown 链接。

---

> [掘金翻译计划](https://github.com/xitu/gold-miner) 是一个翻译优质互联网技术文章的社区，文章来源为 [掘金](https://juejin.im) 上的英文分享文章。内容覆盖 [Android](https://github.com/xitu/gold-miner#android)、[iOS](https://github.com/xitu/gold-miner#ios)、[前端](https://github.com/xitu/gold-miner#前端)、[后端](https://github.com/xitu/gold-miner#后端)、[区块链](https://github.com/xitu/gold-miner#区块链)、[产品](https://github.com/xitu/gold-miner#产品)、[设计](https://github.com/xitu/gold-miner#设计)、[人工智能](https://github.com/xitu/gold-miner#人工智能)等领域，想要查看更多优质译文请持续关注 [掘金翻译计划](https://github.com/xitu/gold-miner)、[官方微博](http://weibo.com/juejinfanyi)、[知乎专栏](https://zhuanlan.zhihu.com/juejinfanyi)。
