---
layout:     post
title:      "计算机图形学总结"
subtitle:   "Summary of Computer Graphics"
date:       2021-05-09
header-img: "img/livestreamprotocol.jpg"
author:     "CMB"
tags:
    - 图形图像
---

## 变换矩阵

#### 摘要

变换矩阵 (Transformation Marices) 在图形学中的重要性不用多说，一切物体的缩放，旋转，位移，都可以通过变换矩阵作用得到。同时在投影 (projection) 变换的时候也有很多应用，本文将会介绍一些简要的变换矩阵。

##### 2D线性变换

我们将如下图所示的简单矩阵乘法定义为对向量 $(x,y)^T$ 的线性变换。

$$
\begin{bmatrix}{a_{11}}&{a_{12}}\\{a_{21}}&{a_{22}}\end{bmatrix} \begin{bmatrix}{x}\\{y}\end{bmatrix} = \begin{bmatrix}{a_{11}x + a_{12}y}\\{a_{21}x + a_{22}y}\end{bmatrix}
$$

#### 缩放(scaling)

缩放变换是一种沿着**坐标轴作用**的变换，定义如下:

$$
scale(s_{x}, s_{y}) = \begin{bmatrix}{s_{x}}&{0}\\{0}&{s_{y}}\end{bmatrix}
$$

即除了 $(0,0)^T$ 保持不变之外，所有的点变为 $(s_{x}x.s_{y}y)^T$ 。举两个简单例子：

![](https://pic2.zhimg.com/80/v2-ae48c1455b68fc5593b8788287235309_720w.jpg)

![](https://pic4.zhimg.com/80/v2-b1a9d665cece63d7e984936641b7ebab_720w.jpg)

#### 剪切(shearing)

shear 变换直观理解就是把物体一边固定，然后拉另外一边，定义如下：

$$
shear-x(s) = \begin{bmatrix}{1}&{s}\\{0}&{1}\end{bmatrix},shear-y(s) = \begin{bmatrix}{1}&{0}\\{s}&{1}\end{bmatrix}
$$

分别对应了向"拉伸" x 轴，和"拉伸" y 轴 直观理解见如下两图：

![](https://pic2.zhimg.com/80/v2-f4aa66cbabade26576291e780feca4f9_720w.jpg)

![](https://pic1.zhimg.com/80/v2-aa233efa70a9bae5152b33da75934f3c_720w.jpg)

#### 旋转(rotation)

旋转可以说是又一个十分重要的变换矩阵了，如下图，我们希望用一个变换矩阵表示将向量 **a** 旋转到向量 **b** 的位置

![](https://pic4.zhimg.com/80/v2-2fa721a848bea3d598ccf5c228228adb_720w.jpg)

记为：

$$
rotate(\phi) = \begin{bmatrix}{cos\phi}&{- sin\phi}\\{sin\phi}&{cos\phi}\end{bmatrix}
$$

我们可做如下推导得到该矩阵，记向量长度为 $r$，则不难得到

$$
x_{a} = r\ cosa,\\
y_{a} = r\ sina.
$$

进一步我们可以将旋转之后的向量 **b** 的坐标 $x,y$ 用如下表示：

$$
x_{b} = r\ cos(a + \phi) - y_{a}sin\phi,\\
y_{b} = y_{a}cos\phi + x_{a}sin\phi.
$$

此时不难得出该结果即为 $rotate(\phi)\times(x_{a},y_{a})^T$ 的结果了，证明结束。（注意该式是**逆时针(countercklockwise)旋转**，且**原点为旋转中心**！）

举例逆时针旋转 45° 效果如下:

![](https://pic2.zhimg.com/80/v2-a46dafc052b4e363775e98dadf53a049_720w.jpg)

### 3D线性变换

其实知道 2 维推 3 维还是非常直观的，只有推 3 维旋转的时候有一点要注意一下。

#### 3D缩放(scaling)，剪切(shearing)，旋转(rotation)

缩放不用多说：

$$
scale(s_{x},s_{y},s_{z}) = \begin{bmatrix}{s_{x}}&{0}&{0}\\{0}&{s_{y}}&{0}\\{0}&{0}&{s_{z}}\end{bmatrix}
$$

剪切也十分类似：

$$
shear-x(d_{y},d_{z}) = \begin{bmatrix}{1}&{d_{y}}&{d_{z}}\\{0}&{1}&{0}\\{0}&{0}&{1}\end{bmatrix}
$$

对于 3 维旋转来说一共有 3 类矩阵，分别对应绕 $x$ 轴，$y$ 轴，$z$ 轴旋转，同时有很关键的一点要注意！我们所采用的是右手系，所以在二维之中其逆时针旋转矩阵是x轴向y轴旋转，对应到 3 维便是绕z轴旋转($x$ 轴转向 $y$ 轴)，不难推出绕x轴旋转($y$ 转向 $z$)，绕y轴旋转($z$ 转向 $x$), 如果想不明白，右手螺旋定则试一试就知道了！ $x$->$y$->$z$->$x$......

因此理解了上面这点，对应二维旋转矩阵能够类推得到三维旋转矩阵如下：

$$
rotate-z(\phi) = \begin{bmatrix}{cos\phi}&{-sin\phi}&{0}\\{sin\phi}&{cos\phi}&{0}\\{0}&{0}&{1}\end{bmatrix}
$$

> 绕z轴，故 z 不变，且 x 转向 y，左上角与二维逆时针旋转矩阵相同

$$
rotate-x(\phi) = \begin{bmatrix}{1}&{0}&{0}\\{0}&{cos\phi}&{-sin\phi}\\{0}&{sin\phi}&{cos\phi}\end{bmatrix}
$$

> 绕x轴，故 x 不变，且 y 转向 z，右下角与二维逆时针旋转矩阵相同

绕y轴会有一点不同，但只要记住需要z转向x，很快便能反应过来

$$
rotate-y(\phi) = \begin{bmatrix}{cos\phi}&{0}&{sin\phi}\\{0}&{1}&{0}\\{-sin\phi}&{0}&{cos\phi}\end{bmatrix}
$$

其实到这里可以下一个结论，可以看到**任意旋转都是正交矩阵**，因此他们的逆便是他们的转置，而一个旋转矩阵的逆所对应的几何解释便是，我反着转这么多，比如我逆时针转 30°，转置便是顺时针 30°

> 注：以上所有的旋转都是针对原点来说，那么如何对围绕任何一个轴(3D)旋转呢?

#### **3D绕任意轴旋转**

我们只有绕 $x,y,z$ 旋转的方法，怎么随便给一个轴让你绕着他旋转呢！很直观的，先把该轴给先旋转到 $x$ 或者 $y$ 或者 $z$ 轴上，接着就可以应用上面已经得到的标准旋转矩阵，最后要记得把该轴逆旋转回它原先的地方就完成了，用矩阵表示如下：

$$
R_{1}R_{x}R_{1}^T \times (x,y,z)^T
$$

这里的 $R_{x}$ 是知道的，也就是 3 维下绕 $x$ 轴旋转的矩阵，那么根据之前的分析问题只剩怎么求 $R_{1}$ 了，设我们想围绕旋转的轴为 $u$，$R_{1}$便是将 $u$ 旋转到 $x$ 的矩阵。 具体来说这里需要以 $u$ 为一轴，构造一个 3 维正交坐标系，然后将 $u-x$ 对齐，那么其它两轴就肯定和 $y$ 和 $z$ 对齐了！

$$
w = t \times u\\
v = u \times w
$$

此时 $u, w, v$ 便是我们构造出的新坐标系(这里运用了一些叉乘的几何性质)。

现在得到了$u,w,v$ 对应 $x,y,z$ 如何将我们的新坐标系与原始坐标系重合呢，这其实很简单，直接取 $R_{1} = (u,w,v)$, 该旋转矩阵的含义便是将 $x,y,z$ 旋转到 $u,w,v$ 的旋转矩阵(直接计算 $R_{1} \times x，R_{1} \times y，R_{1} \times z$试试便一目了然)。

旋转矩阵是正交矩阵，旋转矩阵的转置便是它的逆，也是几何意义上的反作用，因此 $R^T$ 便是将 $u,w,v$ 旋转到 $x,y,z$ 的矩阵了。现在我们知道了 $R_{1}$ 知道了$R_{x}$，那么围绕任意轴的旋转也就得到了！

$$
\begin{bmatrix}{x_{u}}&{x_{v}}&{x_{w}}\\{y_{u}}&{y_{v}}&{y_{w}}\\{z_{u}}&{z_{v}}&{z_{w}}\end{bmatrix} \begin{bmatrix}{cos\phi}&{-sin\phi}&{0}\\{sin\phi}&{cos\phi}&{0}\\{0}&{0}&{1}\end{bmatrix} \begin{bmatrix}{x_{u}}&{y_{u}}&{z_{u}}\\{x_{v}}&{y_{v}}&{z_{v}}\\{x_{w}}&{y_{w}}&{z_{w}}\end{bmatrix}
$$

中间那个矩阵换成 $rotate - x(\phi)$ 即可。

> 参考：[3blue1brown的线代本质](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1Ys411k7yQ%3Ffrom%3Dsearch%26seid%3D15562968547395149083)

### 仿射变换

其实读到这大部分矩阵变换都已经说完了，只剩最后一个位移，同时也会引入齐次坐标为了更好将位移与 rotation，scaling 结合再一起，这样能够有旋转 scale，又有 translation 的变换，称之为仿射变换。

#### 位移(translation)

位移就是：

$$
x^, = x + x_{t}\\
y^, = y + y_{t}
$$

线性变换如下：

$$
x^, = m_{11}x + m_{12}y\\
y^, = m_{21}x + m_{22}y
$$

如果 2 维变换只用 2 维矩阵，3 维变换只用 3 维矩阵，是不可能将二者合在一起用一个矩阵表示的，所以很自然的，引入一维新的坐标，称之为齐次坐标：$(x,y)$->$(x,y,1)$


$$
\begin{bmatrix}{x^,}\\{y^,}\\{1}\end{bmatrix} = \begin{bmatrix}{m_{11}}&{m_{12}}&{x_{t}}\\{m_{21}}&{m_{22}}&{y_{t}}\\{0}&{0}&{1}\end{bmatrix} \begin{bmatrix}{x}\\{y}\\{1}\end{bmatrix} = \begin{bmatrix}{m_{11}x + m_{12}y + x_{t}}\\{m_{21} + m_{22} + y_{t}}\\{1}\end{bmatrix}
$$

可以用一个矩阵即表示线性变换，又表示位移了！三维其实同理，完全一样，不过有点要注意的是，对于带有齐次坐标的变换，会先进行线性变换(即 scale，rotation，shear 这些)，之后才会进行位移！

> 注：最后一维为 1，表示点(point), 为 0 表示方向(direction).
>
> 方向的位移没有意义，方向始终不会变。 当然，第四维不是只能是 1 和 0，在投影变换中，齐次坐标会有更多的作用。