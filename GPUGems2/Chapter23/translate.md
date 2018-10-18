# 前言

在为GeForce 6800开发Nalu demo的时候，面对的最大的一个问题就是如何实时的渲染真实的头发。以前的NVIDIA demo中的角色也有头发，但是他们都很短、暗淡且静态。对于Nalu，我们打算实现在水下的、长的、流动的金发。本章节我们将介绍我们在实现这个目标时使用到的技术。其中包含，模拟头发的运动、计算头发自阴影的算法，以及一个反射模型用来模拟光线穿过单缕发丝时发生散射的现象。将这些元素组合在一起后，就可以实时的制造出极致真实的头发图像。
在Nalu头发着色的背后，有一个每帧生成头发几何体和控制动力学和碰撞的系统。基本上可以将它分为两个部分：几何体生成和动力学/碰撞计算。
整个头发是有4095个独立的（individual）使用线段图元绘制的发丝制作的。使用123000个顶点来渲染头发。将这些顶点拿来计算动力学和碰撞检测简直难以想象的慢，所以我们使用控制发丝：Nalu的发型可以用几百个发丝的较小的集合来描述和控制，虽然渲染需要几千个。所有昂贵的动力学计算都只运算在这些控制发丝上。
我们没有时间和工具来手动对这些控制发丝结构做动画，就算有工具，对那么多控制发丝手动做动画也很困难。许多的微妙的辅助运动是必须的，从而使得头发看起来逼真。基于物理的动画帮了大忙。
当然，如果程序化动画被引入的时候，一些对于头发动作的控制会丢失（对于我们的情况，丢失了90%的控制）。碰撞检测和响应可以引入不需要的头发表现，妨碍了创作的动画。我们的动画师为了明白动力学的内部运作花了大量工作，并且创造了一些解决方法。一些额外的工具被添加到工程方面，用来获得剩下10%的对人类在动力学上的控制。

# 头发几何

## 布局和种植

控制发丝结构用以大致的描述发型。我们从一个用maya创建的专用的几何体种植控制发丝，它用来表示头皮（渲染时不可见）。头皮的每个顶点沿着法线方向种植一根控制发丝。种植是100%程序化的。

## 控制头发

当我们有了一个控制发丝集合时，我们把它用于物理、动力学和碰撞计算，以便为头发添加程序化动画。在这个demo里，动作几乎全部依赖于动力学。虽然这个系统可能看起来有吸引力，当时我们更希望有一个人工可控的系统。当我们想去编写动画，我们需要能够“伪造”或者“修理”头发的表现。另外，我们的头发动力学在某些情况下还不够真实的像真正的头发那样移动。有一个更好的人为控制可以允许我们将动作制作的更加令人信服。

## 数据流

因为头发是完全动态的且每帧都变化，我们需要每帧重建最终的渲染发丝的集合。图23-4展示了这些数据是如何在过程中移动的。动画的控制发丝被转化为贝塞尔曲线并且曲面细分为平滑的曲线。平滑的曲线被插值来增加头发的密度。被插值的发丝的集合被传给引擎用以最终的帧渲染。我们使用一个动态的顶点缓冲来保存这些顶点数据。

## 曲面细分

为每条控制发丝添加顶点用来平滑发丝，这些操作组成了头发的曲面细分。我们增加了多于5倍的顶点，从7个到36个。如图所示。
为了计算这些新顶点的位置，我们将这些控制发丝转换为贝塞尔曲线，通过（每帧）计算它们的切线，并用这些切线来计算贝塞尔控制点。从这些贝塞尔曲线，我们计算出额外顶点的位置，用以平滑控制发丝。
这些控制发丝将会通过插值被复制，来创建头发的密度集合，为最终渲染做准备。

## 插值

使用头皮网格的拓扑结构来创建插值发丝。如图。每个三角形的末端都包含三个平滑过的控制发丝。我们想要在三角面的内部来填充发丝，所以我们三重插值控制发丝的坐标，来创建新的平滑的发丝。这些平滑的控制发丝和平滑的插值发丝拥有相同数量的顶点。
为了将发丝填充到三角面内，我们使用了重心坐标系来创建新的、插值的发丝。例如，插值发丝Y是基于三个重心系数来计算的。
（公式）
生成两个[0,1]间随机数，如果它们的和大于1，用1减掉较大的一个值，然后再用1减掉那两个的和来设置第三个值，这样，它们的和就为1，我们就可以决定在什么位置种植发丝来增加头发到期望的密度。

# 动力学和碰撞

Nalu的头发动力学是基于例子系统的，如图。每个非插值的控制发丝顶点以粒子的形式做动画。这些粒子在发丝上并不是均匀的间隔的。控制发丝的分段会随着与头颅的距离的增大而增大。这样就允许在不添加太多顶点的情况下拥有更长的头发。
对于这个项目，我们选取Verlet积分来计算粒子的动作，因为这个方法比Euler积分更趋向于稳定，比Runge-Kutta积分更加简单。相关说明不在这个章节的范围之内，不过如果你想学习更多的在游戏中使用Verlet积分的方法，看Jakobsen 2003。

## 约束

当粒子四处游走的时候，控制发丝的长度必须保持不便来避免拉伸。为了实现这个，我们对控制发丝里的粒子之间使用约束。如果粒子之间过近，约束会让它们退后，如果它们之间相距过远，约束会使它们收缩线段。当然，如果我们拉一个粒子，相邻线段的长度会变得无效，所以这种修改需要迭代的执行。经过几次迭代之后，这个系统收敛到一个期望的结果。如图。

## 碰撞

我们尝试了很多碰撞检测的技术。我们需要保持这个过程简单而快速。对于这个demo，多个球体的控制表现的很好用，而且它是实现起来最简单的。如图。
一开始，这个解决方案没有计划中那么好用，因为某些发丝的线段比碰撞球体还大。因为我们在检测点和球之间的碰撞，所以无法防止一根发丝的两个端点与球体相交。
为了防止这种事情发生，我们引入了“珍珠构造”到控制发丝的碰撞数据中，如图所示。每个球体将会碰撞在粒子上的另一个球体，而碰撞一个点。
我们也尝试了检测线段和球体之间的碰撞。不是很难，但是它没有使用“珍珠”的方法快。我们的边碰撞检测工作的良好，不过它也有稳定性问题，可能由于我们的碰撞响应的代码。无奈如何，我们有了对于我们目标的一个工作足够良好的方法。

## 鱼鳍

美人鱼的原始概念里包含了长且软的鱼鳍，我们把这个特性作为这个角色的一个重要的部分。固定的鱼鳍被创建和蒙皮到骨骼。但是，玛雅里，在动画过程中，鱼鳍只是跟随身体，看起来挺僵硬的。如图。
幸运的是，我们的头发动力学代码也可以被用于执行布料模拟。所以，在实时引擎中，我们计算了一个布料模拟并把布料结果混合到蒙皮几何体中。绘制了一个权重映射来定义有多少物理会被混合进去。我们运用越多的物理，鱼鳍看起来就越柔软。相反的，混合越多的蒙皮，就会得到越刚体化的运动结果。我们想要鱼鳍的根比末梢更加的刚体化，并且权重映射允许我们准确的做到，因为我们可以将我们想要的物理对蒙皮的比例准确的刷到每一个布料顶点上。如图。最后，这个合并允许我们制造软且真实的鱼鳍，如图。

# 头发着色

头发的着色问题可以分为两个部分：一个对于头发的局部的反射模型和一个计算发丝间的自阴影的方法。

## 头发的实时反射模型

对于我们的反射模型，我们选择了在Marschner et al. 2003介绍的模型。我们选择它是因为它是一个全面的、基于物理的头发反射的表示方法。
Marschner反射模型可以公式化为四维度双向散射函数
（公式）
这里thetai属于[-pi/2,pi/2]且phii属于[0,2pi]	,是以极坐标形式表示的输入方向，thetao和phio是极坐标形式的光线方向。S函数是一个完整的描述，头发纤维如何散射和反射光线。如果我们可以计算这个函数，我们就可以对任意光照位置计算表面的着色。
因为估计S是非常昂贵的，我们想要避免在每个像素上计算它。一个可能的解决方案是保存S到一个lookup表里，并在运行时读取它。这个lookup表可以编码成一个纹理，允许我们在像素着色器中访问它。
不幸的是，这个S函数包含4个参数，GPU没有对四维纹理的本地支持。这就意味着我们需要某种方案／格式来对四维函数编码成二维纹理。幸运的是，如果我们小心的处理表的lookup，我们可以只使用一个数量不大的二维映射来编码完全的四维函数。

### Marschner反射模型

Marschner发射模型把每个单独的发丝纤维当作一个半透明的圆柱体，且考量光照穿过头发的可能的路径。三种路径类型会被考量，用不同的路径符号来为它们贴标签。路径符号将光照的每一个路径表示为一个字符串，每个表示了光射线与表面的一种类型的交互。R路径表示从发丝纤维的表面反弹回观察者的光线。TT路径表示折射入发丝并再次折射出来到观察者的光线。TRT路径表示折射入发丝纤维，在内部表面发射回来，再次折射回观察者的光线。对于每种情况，R表示光线反射，T表示光线被表面投射或者折射。
Marshner et al. (2003) 展示了，每个路径对发丝反射的独特并视觉重要的方面的贡献，允许了以前的头发反射模型不可能存在的真实度。图示展示了三个对头发的外观贡献最多的三个反射路径。
我们可以把反射模型写为以下公式：
（公式）
每个Sp项都可以进一步的因式分解为两个函数的积。Mp方程描述了反射上的theta角的效果。另一个方程Np，捕获在phi方向上的反射。如果我们假设一个完美的圆形的头发纤维，那么我们可以将M和N写成一个更小的角度的集合的项。定义辅助角度thetad=thetai-thetao和phid=phii-phio，前面方程的每一个项都可以写成：
（公式）
以这种形式，我们可以看到，M和N都是只有两个参数的函数。这就意味着，我们可以对每个方程计算一个lookup表，并把他们编码成为一个二维的纹理。二维纹理是理想的，因为GPU对二维纹理做了优化。我们也可以使用GPU的插值和mipmap功能来估算着色走样。
虽然我们保存了六个函数，但是它们中间很多只有一个通道，且可以保存到同一个纹理。MR，MTT和MTRT每个都只有一个通道，所以把它们一起打包到第一个lookup纹理中。NR是单通道的，但是NTT和NTRT都是三通道的。我们把NTT和NR一起存到第二个lookup纹理中。为了对demo改善性能并且降低纹理的使用，我们做了一个简化假设MTT=MTRT。这就允许我们将NTT和NTRT保存到相同的纹理中，并将纹理的数量从三降到了二。如图所示。
虽然模型是以角度的形式来表述的，但是从向量计算这些角度需要发三角函数。这很昂贵，我们想尽可能的避免它。我们计算正弦值而不是传递thetai和thetao给第一个lookup。
sinthetai = light 点乘 tangent
sinthetao = eye 点乘 tangent
我们可以把M表示为sinthetai和sinthetao，省去了shader里的一些数学计算。
添加一小点儿更多的工作，我们还可以计算cosphid。我们首先将观察向量和光线向量垂直投影到发丝上
lightPerp=light-(light dot tangent) cross tangent
eyeperp = eye - (eye dot tangent) cross tangent
然后从公式
lightPerp dot eyePerp = ||lightPerp|| multiply ||eyePerp|| multiply cosphid
我们可以计算cosphid
cosphid = (eyePerp dot lightPerp) multiply power((eyePerp dot eyePerp) multiply (lightPerp dot lightPerp), -0.5)
剩下还没有计算的就是thetad。为了计算它，我们将thetad表示为一个thetai和thetao的方程。因为我们准备好使用一个lookup表，用thetai和thetad索引的，我们可以添加额外的通道到lookup表来保存thetad。
23-1表里，概要的表示的我们的shader的伪代码。
在我们的实现里，lookup表在CPU里计算。我们使用8bit每分量的格式的128x128的纹理。8bit的格式要求我们把数值缩放到[0,1]范围里。作为结果，我们不得不添加一个额外的缩放系数到shader里来平衡这些项的相对强度。如果我们希望更多的精度，我们可以跳过这个步骤并使用16bit浮点纹理来保存lookup表。我们发现，对于我们的目标，这个是不必要的。
为了计算这些lookup表中的一个，我们不得不写一个程序来计算函数M和N。这些函数太复杂了，就不再这里描述了。Marschner et al. 2003提供了完整的描述和推导。
注意，在lookup表里编码了很多对于反射模型的其他参数。包含高光的宽度和强度，头发的折射的颜色和指数，等等。在我们的实现中，我们允许这些参数在运行时可被改变，并且我们实时的重计算lookup表。

### 固定几何体

虽然我们使用这个头发反射模型来渲染发丝，发丝表示为线段条，将这个反射模型延伸到用固定几何体表示的头发也是可能的。我们可以使用表面的一个主要的切线，而不是使用线段条的切线。最终，我们必须考虑到表面的自遮挡。乘以一个额外的项(wrap + dot(N,L))/(1+wrap)来实现这个东西，其中dot(N,L)是法线和光线的点积，且wrap是介于0和1之间的值，用来控制多远的光照可以被允许环绕模型。这是一个简单的近似，用来模拟光照流过头发。

## 头发的实时体积阴影

实时应用中的阴影一般使用两种方法中的一种来计算：模板阴影体积或者阴影映射。不幸的事，这两个技术中的哪个都无法很好的对头发计算阴影。头发中使用的几何体的绝对数量会使模板阴影体积变得棘手，并且在高度详细的几何体（例如头发）上，阴影映射表现出严重的走样。
替代的，我们使用一个为了渲染头发阴影特别设计的方法。不透明度阴影映射扩展了阴影映射来处理体积对象和反走样（Kim and Neumann 2001）。不透明度阴影映射最初是在GeForce2级硬件上实现的，相对于当下的可编程的GPU，它的灵活现是非常受限的。最初的实现没有在大数据集上实现实时的性能，并且它需求一大部分的算法要在CPU上运行。GeForce 6系列硬件拥有了足够的可编程性，我们就可以将大部分算法在GPU上执行。这样做，我们可以在大量发丝数据集上实现实时性能。

### 不透明度阴影映射

相比使用类似传统阴影映射的离散的测试，我们使用不透明度阴影映射，它可以允许阴影值是分数。这样意味着，对比对于遮挡的简单的二分测试，我们需要一个测量在给定的像素上，（在灯光空间中）灯光穿过深度z的百分比。使用以下公式给出：
（公式）
T(x,y,z)是像素位置（x,y）上，光线穿过深度z的分数。sigma被称为不透明厚度，r(x,y,z)是笑容系数，它描述了在点(x,y,z)上，每个单位距离上，光照被吸收的百分比。K值是一个常量，当sigma=1时，T接近0（在数字精度中）。这就允许我们忽略在[0,1]之外的sigma值。
不透明度阴影映射的想法是在一个z的离散集合上（z0...zn-1）计算sigma。然后我们对最近的两个值进行插值来计算在两者之间的sigma值。
（公式）
这是一个合理的近似，因为sigma是一个z的单调递增函数。我们选择n=16，z0是头发在灯光空间中的近平面，z15时头发在灯光空间的远平面。其他平面是均匀距离的分布，所以zi=z0+i dz，dz=(z15-z0)/16。注意，因为r是0，它在头发体积之外，对于所有的x和y，sigma(x,y,z0)=0，所以我们只需要保存z1到z15的sigam值。
Kim & Neumann(2001)指出，积分的sigma可以使用图形硬件的添加混合来计算。我们的实现也用到了硬件混合，但是它也使用了shader来减少CPU的工作量。

### 更新的实现

幼稚的方案会把sigma(x,y,zi)存到16张纹理里。这将要求我们渲染16次来生成全部的不透明度阴影映射。我们第一个意见是保存sigma只需要1个通道，所以我们可以将4个sigma值打包到一个4通道的纹理中，并同时渲染它们。使用这个方案，我们可以将渲染通道的数量从16降低到4.
我们可以比4渲染通道做的更好，如果我们使用多渲染对象（MRT）。MRT是一个允许我们同时渲染最多4张不同纹理的特性。如果我们使用MRT，我们可以同时渲染4个分离的4通道纹理，允许我们仅在一个渲染通道里渲染出来所有的16个通道。
现在，如果我们简单的叠加混合每个通道，然后我们就可以得到对每一层有贡献的所有头发。我们真正想要的是，对于每一层i，它只被zhair < z的头发部分所影响。用一个shader，我们可以简单的输出0，如果zhair > z。

### 执行lookup

给出一个不透明度阴影映射，我们必须计算在点(x,y,z)的T，给出不透明度阴影映射的切片。我们对两个最近的切片的sigma使用线性插值来实现这个。sigma(x,y,z)的值是sigma(x,y,z0)到sigma(x,y,zn)的线性组合。特别的。
（公式）
我们可以在顶点着色器中对所有的16个值计算wi=|z - zi|/dz。
v2f.OSM1weight = max(0.0.xxxx, 1 - abs(dist - depth1) * inverseDeltaD);
v2f.OSM2weight = max(0.0.xxxx, 1 - abs(dist - depth2) * inverseDeltaD);
v2f.OSM3weight = max(0.0.xxxx, 1 - abs(dist - depth3) * inverseDeltaD);
v2f.OSM4weight = max(0.0.xxxx, 1 - abs(dist - depth4) * inverseDeltaD);
为了改善效率，我们在顶点着色器中计算这些权重，并把它们直接的传递给片元着色器。虽然这些结果与在片元着色器中计算这些权重并不是数学等同的，但是对于我们的目标，这足够接近了。
当这些权重都被计算了，这就是一个简单的计算和的问题了。
（公式）
因为数据是对齐的，我们可以
（公式）
在一个点积中计算，并且我们可以同样的使用点积计算i=4-7,i=8-11和i=12-15的和。
```
half density = 0;
density = dot(h4tex2D(OSM1, v2f.shadowCoord.xy), v2f.OSM1weight); 
density += dot(h4tex2D(OSM2, v2f.shadowCoord.xy), v2f.OSM2weight);
density += dot(h4tex2D(OSM3, v2f.shadowCoord.xy), v2f.OSM3weight); 
density += dot(h4tex2D(OSM4, v2f.shadowCoord.xy), v2f.OSM4weight);
```
最终，我们从光学密度上计算透射率，并获得一个0到1的值，用来描述光从光源到达点(x,y,z)处的光线分数。我们将这个值乘以着色值来获得最终的头发颜色。
```
half shadow = exp(-5.5 * density);
```	
如图。

# 总结与展望

我们展示了，现在已经有可能实时的模拟头发渲染的各方面：从动画和动力学到渲染和着色。我们希望我们的系统可以为在交互应用（例如游戏）中实时渲染头发提供依据。
虽然这里的想法已经被应用到头发的动画和渲染，但是这不只是它们的全部应用场景。Marschner反射模型拥有一个天然的因式分解，我们可以用来分解它到纹理lookup中。这个方法可以被扩展到任何使用类似因式分解的反射模型（McCool et al. 2001）。这些数字的因式分解包含分析因式分解的所有好处，还有一些例外，少量的错误。
此外，不透明度阴影映射是在渲染头发中非常有用的，也可以被用于深度映射失效的情况，例如，它们可以被用于体积表现，例如云和烟，或者高度详细的对象，例如稠密的叶子。
GPU变得越来越灵活，寻找将更多的工作传递给它们的方法是非常有价值的。这里不光包含明显的并行任务例如曲面细分和插值，同样包含哪些传统上交给GPU完成的领域，例如碰撞检测和物理。
效率不是我们的关注点，我们也注重让头发更加的对开发者可控，那么做样式和动画会变得简单。更多的挑战在前头，但是我们希望在次世代应用中看到更加真实的头发。

# 引用文献




