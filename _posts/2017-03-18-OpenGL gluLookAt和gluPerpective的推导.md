---
---
大家都知道`gluLookAt`和`gluPerspective`函数分别对应对模型的`model、view、perspective`三种变换的`view`和`perspective`变换。而其内部实现都是一个4x4矩阵，而函数参数影响矩阵内的数值。下面我们来简单讨论一下这两个矩阵的推导，和它们做了哪些工作。

---

首先，说一点预备知识。大家都知道OpenGL的坐标系是右手坐标系。如果按照平常习惯放置x轴和y轴的话(x轴正方向从左至右，y轴正方向从下至上)，z轴正方向则为从前向后，即人眼睛看向x轴负方向，如图：
![](../assets/images/opengl/axis.PNG)
其中红、绿、蓝(rgb)色实线分别代表x、y、z轴正方向，而虚线代表负方向。我们往OpenGL里面输入的点坐标就是在这个坐标系下了。

让我们稍微回顾一下渲染流程。首先顶点着色器对输入的顶点属性进行计算，往往是计算以得到矩阵变换后的顶点的新坐标。如果是实时渲染的话，很可能光照也是在这一阶段计算的(黑暗之魂3作为16年的3A游戏，光照结果也是插值得到的)。然后是组装图元、剪裁、光栅化、片元着色、测试、混合。

注意剪裁这一步。图元组装完成之后，并不是所有的图元都会被光栅化并参与到片元着色，而是在一定范围内的图元才可以。而如果一个图元部分在范围内，部分在范围外，还会被沿着范围边界分割，生成完全在范围内的新的图元，来参加后续计算。

那么怎么界定这个范围呢？

这个范围其实就是一个以原点为中心，边长为2的正方体。正方体内的点(就是该点x、y、z坐标分别都在-1~1以内)，就会被光栅化，参与到片元计算。如图所示。
![](../assets/images/opengl/device.PNG)
图中两个圆形代表z轴与正方体平面的交点，为了方便读者理解，没有其他意思。

大家还记得人眼看向的方向是z轴负方向吗？请注意图中有阴影的平面。正方体内的模型，经过光栅化后形成片元，最后在这个平面上的投影，也就是要输出到平面上的图像了(当然得到这个投影还要很多步骤，但大体可以这么理解)。

---

下面我们来看一下啊`gluLookAt`这个函数。

首先看它的函数原型
```c++
void gluLookAt(GLdouble eyeX, GLdouble eyeY, GLdouble eyeZ, GLdouble centerX, GLdouble centerY, GLdouble centerX, GLdouble upX, GLdouble upY, GLdouble upZ);
```
这个函数的参数很好理解：虽然有9个参数，但明显每3个参数组成一个向量，代表了一个三维空间中的点，或者一个向量(`eye`和`center`是点，`up`是一个向量)。那么我们要做的也很好理解：在`eye`这个位置以`up`为上，看向`center`(也就是z轴负方向为`eye`->`center`)构建新的坐标系。

既然要构造一个新坐标系，那么肯定要构造一个矩阵。而新坐标系以`eye`为原点，那么往往还要做平移。

大家知道，在三维空间中，旋转和重建坐标系没什么本质不同，那我们就现在构造这个旋转矩阵。

首先确定z轴。因为OpenGL中镜头总是看向z轴负方向，所以z轴为：
```c++
vec axisZ = center - eye;
axisZ.normalize();
```
新坐标系的y轴当然和给定的参数`up`有关，但不一定会和`up`共线或者平行。首先，给定的`up`向量不能和我们刚才求出的z轴平行([The UP vector must not be parallel to the line of sight from the eye point to the reference point.](https://www.khronos.org/registry/OpenGL-Refpages/gl2.1/xhtml/gluLookAt.xml))。那么，`up`向量就和z轴能唯一确定一个平面。那么y轴就是这平面上和z轴垂直的一个单位向量(当然，y轴和`up`夹角要小于90°)。y轴的求法：
```c++
vec axisY = up - dot(up, axisZ); // 就是up向量减去其z轴分量，然后归一化
axisY.normalize();
```
知道了y轴和z轴，那么x轴自然也能求了：
```c++
vec axisX = cross(axisY, axisZ); // 注意参数不能写反，叉乘的乘数不可交换。
axisX.normalize(); // 其实y轴z轴都是单位向量，不归一化也可以。
```
这样我们得到了新坐标系的3个基。

当然`gluLookAt`的实际实现是一个更巧妙一点的方法。

当我们知道z轴和`up`之后，我们可以先求x轴：
```c++
vec axisX = cross(up, axisZ);
axisX.normalize();
```
然后再求y轴：
```c++
vec axisY = cross(axisZ, axisX);
axisY.normalize() // 其实z轴y轴都是单位向量，不归一化也可以
```
知道了新坐标系的三个基之后，我们就可以构建这个旋转矩阵了。

$$
 \begin{matrix}
 axisX.x & axisX.y & axisX.z \\
 axisY.x & axisY.y & axisY.z \\
 axisZ.x & axisZ.y & axisZ.z \\
 \end{matrix}
$$

矩阵如上。但是光有矩阵还不够。想要做到在`eye`点观察目标的效果，还要将坐标原点移动到`eye`的位置。当然，相同的操作我们也可以解释为"坐标系中物体朝相反方向移动"。

线性变换是没法做移动的。因此我们要做一个4x4的仿射变换矩阵。

那么很容易理解，我们这个做移动的仿射变换矩阵如下：

$$
\begin{matrix}
1 & 0 & 0 & -eyeX \\
0 & 1 & 0 & -eyeY \\
0 & 0 & 1 & -eyeZ \\
0 & 0 & 0 & 1 \\
\end{matrix}
&&

我们当然希望做一次矩阵运算就解决了旋转和位移的问题。因此要合并矩阵。那么我们需要先将旋转矩阵也处理成1个4x4的仿射变换矩阵。如下：

$$
\begin{matrix}
axisX.x & axisX.y & axisX.z & 0 \\
axisY.x & axisY.y & axisY.z & 0 \\
axisZ.x & axisZ.y & axisZ.z & 0 \\
0 & 0 & 0 & 1 \\
\end{matrix}

我们现在要做的事就是合并这两个矩阵。那么问题来了：
我们是先做平移还是先做旋转呢？
---
答案是先做平移。

道理很简单：我们现在的平移矩阵做的平移运动，是用现有坐标系下的基来衡量的。

如果先做旋转的话，我们再做平移运动，就是用新坐标系的基来做运动了。

打个比方来说：

我们俩游泳池的长度是50，单位是米。如果我们改变了测量的基，变成尺的话，再引用50这个数据就不对了。要想获得正确的游泳池长度，就需要用50这个数据，乘以从米到尺的转换矩阵(假设这个转换是用矩阵来做的的话。。)。

回到我们`gluLookAt`函数推导的内容。如果你想先做旋转，再做正确的平移的话，需要将平移矩阵也乘以旋转矩阵以得到新坐标系下的平移矩阵。但仔细想想的话，这种方式与先平移，再旋转没有本质区别。

因此我们最后得到的`gluLookAt`的旋转矩阵就是

$$
\begin{matrix}
axisX.x & axisX.y & axisX.z & axisX*eye \\
axisY.x & axisY.y & axisY.z & axisY*eye \\
axisZ.x & axisZ.y & axisZ.z & axisZ*eye \\
0 & 0 & 0 & 1 \\
\end{matrix}

---

(未完待续)
