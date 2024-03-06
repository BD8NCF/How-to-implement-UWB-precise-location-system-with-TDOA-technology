这是一个系列文章《如何从零开始实现TDOA技术的 UWB 精确定位系统》第5部分。

**重要提示（劝退说明）：**

> **Q：做这个定位系统需要基础么？**

> A：文章不是写给小白看的，需要有电子技术和软件编程的基础

> **Q：你的这些硬件/软件是开源的吗？**

> A：不是开源的。这一系列文章是授人以“渔”，而不是授人以“鱼”。文章中我会介绍怎么实现UWB定位系统，告诉你如何克服难点，但不会直接把PCB的Gerber文件给你去做板子，不会把软件的源代码给你，不会把编译好的固件给你。我不会给你任何直接的结果，我只是告诉你方法。

> **Q：我个人对UWB定位很兴趣，可不可以做出一个定位系统？**

> A：如果是有很强的硬件/软件背景，并且有大量的时间，当然可以做得出来。文章就是写给你看的！

> **Q：我是商业公司，我想把UWB定位系统搞成一个商业产品。**

> A：当然可以。这文章也是写给你看的。如果你想自己从头构建整个系统，看了我的文章后，只需要画电路打板；构思软件结构再编码。就这样，所有的难点我都会在文中提到，并介绍了解决方法。**你不需要招人来做算法研究**。如果你想省事省时间，可以直接购买我们的电路图(AD工程文件)，购买我们的软件源代码，然后快速进入生产环节。（网站: https://uwbhome.top）


回顾一下，前几篇文章我们介绍了基站和标签的硬件设计，以及基站和标签的固件设计，包括时钟同步等等的要点，现在我们介绍定位引擎的设计，当然重点是TDOA算法。

# 定位引擎设计-TDOA算法

使用DW1000的定位系统，通常使用的定位方案是TOF或者TDOA。

TOF方案在DecaWave公司提供的例程，还有Trek1000的代码中都有介绍，通过测距得到Tag与几个Anchor之间的距离，然后使用 Trilateration 算法计算Tag的坐标，具体算法参见wikipedia: https://en.wikipedia.org/wiki/Trilateration。

我们使用TDOA方案。Tag发出定位UWB包后，被定位区域内的几个Anchor收到，各个Anchor记录下收到这个UWB包的时间戳，改善到定位引擎RTLE，由RTLE根据各个Anchor收到该UWB包的时间差计算Tag的坐标。通常，这个计算坐标的算法叫 Multilateration，具体介绍参考 https://en.wikipedia.org/wiki/Pseudo-range_multilateration。
Wikipedia上介绍的是GPS，其原理与UWB一样的。

另外，TDOA定位有下行和上行两方案。GPS使用的是下行方案，也就是说卫星发出定位信号，GPS接收机收到各个卫星发出的定位信号后，根据收到各个卫星的信号的时间差计算自己的坐标；上行则是由被定位的Tag发出定位信号，由各个负责接收，坐标计算定位引擎集中进行计算。

上行下行两种方案各有优缺点。对于UWB定位，绝大多数情况是系统想知道Tag在哪里，而不是Tag想知道自己在哪里；另外，上行方式对Tag的要求低，Tag只要能发出一个UWB信号就可以了，不需要有太多的计算能力。对电力的要求也很低。例如Tag可能会做成工牌，或者手环，如果使用上行方案，我们就可以把Tag做得很小。

反之，如果使用下行方案，则要求Tag有坐标计算能力，对MCU的要求会比较高；因为需要经常计算坐标，休眠时间也会很短。这些因素，都会有比较大的电力消耗。想想以前的GPS终端，都是一个很大的手持机，电池的待机时间都很短。随着时间的推移，技术不断进步，最重要的是市场越来越大，开发商有利润有动力不断研发，近年来GPS才能做得很小，集成进手机里面。

有很多论文涉及Multilateration算法，几乎每篇论文都会说自己的算法很厉害，列举出一堆数据来证明自己很厉害，然后再列出一堆让人看不明白的数学公式。

对于TDOA系统，我们一定要有一个观念：时间差就是距离差！

Tag发出的无线电波到达不同的基站的时间差，其本质是因为Tag与各个基站之间的距离不同，无线电波飞行需要时间。因为距离远近不同，所以无线电波到达各个基站的时间才会不同。我们可以把时间差乘以光速(其实是无线电波在空气中的传播速度)，就可以得到距离差了。

## Andersen 的算法
我们使用的第一个Multilateration算法是André Chr. Andersen 设计的。这篇文章发表在 André Chr. Andersen 的博客上，http://blog.andersen.im/2012/07/signal-emitter-positioning-using-multilateration。
很遗憾，他的博客现在打不开了，可能他已经没有维护了。André Chr. Andersen在文章后面还提供了 MATLAB 代码，Paul Hayes把这段 MATLAB 代码翻译为 Python 代码，可以在 github 上找到 https://github.com/paulhayes/MultilaterationExample 。

这个算法是针对超声波定位的，但是本质上跟 UWB 定位是一样的，重要的是这是一个简单易实现的 Multilateration 算法，改一改就可以用在 uwb 的坐标计算上。
非常感谢 André Chr. Andersen 和 Paul Hayes 两位，如果没有他们的算法和代码，也许我们这个项目就提前中止了。

Andersen 的算法可以用，但是也有一些问题。主要体现在两个方面：
  1. 计算出来的坐标不太准。标签在区域中心的时候比较准，但是当标准在区域边缘时，误差会变得很大。
  2. 算法中迭代的过程中可能会出错，导致有时算不出坐标。因为各种因素影响，我们得到的TDOA值肯定是不准确的、有误差的，实际上我们期望得到的是一个近似值。但是误差太大，有时会得出很离谱的坐标，这个就比较尴尬了。

我们后来自己研发了两个算法，最终采用的是最小二乘法，在速度和精度方面得到较好的平衡。

以下是 Paul Hayes 重写的 Python 代码。。

```Python
###########################
#
# Python rewrite of multilateration technique by André Andersen in his [blog post](http://blog.andersen.im/2012/07/signal-emitter-positioning-using-multilateration).
#
############################

from numpy import *
from numpy.linalg import *
import json
#speed of sound in medium
v = 3450
numOfDimensions = 3
nSensors = 5
region = 3
sensorRegion = 2

#choose a random sensor location
emitterLocation = region * ( random.random_sample(numOfDimensions) - 0.5 )
sensorLocations = [ sensorRegion * ( random.random_sample(numOfDimensions)-0.5 ) for n in range(nSensors) ]
p = matrix( sensorLocations ).T

#Time from emitter to each sensor
sensorTimes = [ sqrt( dot(location-emitterLocation,location-emitterLocation) ) / v for location in sensorLocations ]

c = argmin(sensorTimes)
cTime = sensorTimes[c]

#sensors delta time relative to sensor c
t = sensorDeltaTimes = [ sensorTime - cTime for sensorTime in sensorTimes ]

ijs = range(nSensors)
del ijs[c]

A = zeros([nSensors-1,numOfDimensions])
b = zeros([nSensors-1,1])
iRow = 0
rankA = 0

for i in ijs:
	for j in ijs:
		A[iRow,:] = 2*( v*(t[j])*(p[:,i]-p[:,c]).T - v*(t[i])*(p[:,j]-p[:,c]).T )
		b[iRow] = v*(t[i])*(v*v*(t[j])**2-p[:,j].T*p[:,j]) + \
		(v*(t[i])-v*(t[j]))*p[:,c].T*p[:,c] + \
		v*(t[j])*(p[:,i].T*p[:,i]-v*v*(t[i])**2)
		rankA = matrix_rank(A)
		if rankA >= numOfDimensions :
			break
		iRow += 1
	if rankA >= numOfDimensions:
		break

calculatedLocation = asarray( lstsq(A,b)[0] )[:,0]

print "Emitter location: %s " % emitterLocation
print "Calculated position of emitter: %s " % calculatedLocation
```

这段代码可以很容易就翻译成Java，我们的第一版定位引擎是使用Java编写的。实际上我把它翻译为Java没花多少时间，你应该也可以。

Andersen的算法我们用了一段时间之后，发现最大的问题就是当标签靠近基站围成的多边形边缘时，误差比较大。然后，我们还发现，可以把基站的坐标从多边形往外扩，可以得到更准确的坐标。这很不科学。

## 坐标质量评估
这时候，我研究出了一个坐标质量评估算法，对计算出的坐标的质量进行评估。Tag的坐标计算出来之后，我们不知道它对不对，如果有误差，误差有多大。怎么办呢？

其实也很简单，我们有了一个坐标，我们可以计算这个坐标到各个基站的距离，我们可以得到这个坐标到各个基站的距离差，这个距离差其实就是时间差，因为时间差本质上就是距离差。我们拿这个距离差(时间差)与基站实际信号的时间差(距离差)进行比较，可以得到一个分值，这个分值就代表计算出的坐标的质量。

坐标质量评估很重要，几乎与TDOA算法一样重要。它会在整个系统中扮演很重要的角色。

例如，有两个相邻的定位区域，Tag在边界上的时候，两个区域都会计算出Tag的坐标。通常，这两个坐标的值不会相同，而是有差异。那么，哪个坐标才是正确的呢？通过坐标质量评估，我们可以知道哪个坐标更接近真实值。
当然，这个说法是简化了的，实际情况会复杂得多。例如，Tag从区域A移动到区域B，在边界上，如果只是如前所述简单判断哪个坐标的质量更高，就输出它。那么，如果把输出的Tag坐标在地图上连成线，我们会看到在边界上有一个跳跃，这个跳跃就是从区域A切换到区域B的瞬间。由于误差的存在，这个质量评估并不一定准确。如果Tag在边界上停留了一会儿，我们可能会看到输出的坐标不断在区域A和B之间切换，跳过去又跳过来。我以后的文章再来讨论这种情况怎么办。



## 第二个 Multilateration 算法
现在可以请出我们的第二个算法，这个算法很简单，也很直观。我们把定位区域分为4块，类似笛卡尔坐标的4个象限，我们现在有了4个小的矩形。我们假设这4个矩形的中心点可能是Tag的坐标，我们分别计算这4个点的质量，取质量最高的那个点，Tag应该在它对应的区域里面；然后我们再把这个矩形分为4块，再计算更小的4个矩形的中心点的质量，取质量最高的那个点，Tag应该在这个小矩形内……如此下去，直到某个时候，例如矩形的边长小于30cm，这时我们就可以认为这个小于30cm的矩形的中心就坐标就是Tag的坐标。
是不是很简单啊。
但是，这里有一个假设，Tag在质量最高的那个坐标所属的矩形内! 这个假设成立么？数学我不知道怎么证明，但是实事上它是成立的，因为这个算法可以正常工作。
但是，这个算法有一个问题，就是计算量比较大，坐标计算速度比较慢。

这个算法的要点在于，我们把这个坐标计算过程看成一个数学函数，它是收敛的，它最后收敛在一个点上，那就是我们想要的坐标。

## 第三个 Multilateration 算法
然后我们再研究出了第三个算法。其实这个算法跟第二个算法类似，但是收敛的方法不一样，我们使用最小二乘法。最小二乘法有更快的收敛速度，也不需划分4个矩形。我们根据初始参数，得到一个向量，它指向坐标所在的方向，不断迭代，这个向量不断接近真实坐标。当迭代结束的时候，这个向量就是我们需要的坐标。

从本质上来讲，Multilateration 的算法不是按照标准的解方程的方式来解方程。我们在中学数学中都学过解方程，对于多元方程式，可以通过一些技巧消元，求到某个未知数的值，然后代入其他的方程式，再求其他未知数的值。但是这个套路无法用于实践，有两个因素：
一是误差很大，严格数学的意义上看，这个方程式是无解的，实际上我们要求的是一个近似值；
二是我们有冗余参数。如果计算2维坐标，3个基站就够了。但是可能会有4个或者更多的基站能提供数据，多出来的基站提供的数据能够让我们得到更近似(精确)的坐标。但如果按标准的方程解法，这些冗余的参数之间的矛盾的。

Multilateration的实用算法基本上都是想办法求近似值，多数都是使用逼近法进行迭代。通过一些初始参数估计解的范围，缩小范围之后再次估计解的范围，得到一个更小的范围，不断迭代，满足某个条件之后结束迭代。这个条件要么是得到一个适合的精度的解了；要么是求解失败，因为误差太大，函数无法收敛。

这几篇文章写到这里，使用TDOA技术实现UWB精确定位的最有价值的技术都介绍完了。如果你之前对UWB不了解，看起来会比较费力，因为我基本上都只是介绍技术要点，而不是做科普。如果你正在研发类似的系统，你应该可以开始写代码了。

*接下来，我会再写几篇文章，介绍一些技术细节。*

*这几篇文章的内容看起来有点乱，确实也有点乱。有点理解那些写网络连载小说的作者了，想到哪里写到哪里，还要有连贯性，太难了。不像平时写技术方案，你可以反复修改、推敲。*


