---
title: 在线广告系统预算控制和流量预估的研究
date: 2017-03-14 14:45:38
categories:
- 技术杂谈
tags:
- 计算广告
mathjax: true
---
<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default">
</script>

> 本文主要是针对国外的计算广告论文 Predicting Traffic of Online Advertising in Real-time Bidding Systems from Perspective of Demand-Side Platforms 进行技术学习的一次分享，同时拓展性的研究了竞价预算和流量预测的方法，具体的参考论文以及实现代码见最后的参考文献地址

<!-- more -->

随着互联网的发展和用户的增长，广告行业从传统的线下广告模式，逐步转变为线上广告模式，同时，由于大数据分析技术的运用，线上广告模式相比于传统广告也体现了巨大的优越性。广告主之间相互竞争，通过竞价的方式，将自己的广告投放在运营媒体的广告位上。

对需求方平台算法的优化是广大广告主的诉求，是广告网络产品透明化和开放化的催生产物，使得线上广告变得越来越依赖大数据和计算导向的方向发展。预算控制和流量预估的研究是当前的一个热点，流量预测直接影响到广告主获得优质广告流量的能力，进而决定广告预算的性价比并影响广告营销的效果，成为DSP系统(Demand Side Platfrom, 需求方平台)中十分重要的一个环节。

![](http://wx2.sinaimg.cn/mw1024/78d85414ly1fo48r9x850j20x40hagtp.jpg "图1 实时竞价系统的工作流")

## 预算控制(Budget Control)
### 预算控制的意义
通过实时竞价方式投放在线广告，能够更为准确地将广告的预算花费到那些更有可能产生回报的广告展示机会上，从而使广告收益得到优化，通常情况下，广告商需要为每个广告营销活动设置投放周期内的预算，需求方平台按照一系列优化算法买下尽可能多的符合广告目标群体的广告展示机会，因此，在有限预算的情况下对每次广告展示机会进行出价并给出合理的竞标价格是实时竞价的关键，而针对此的优化算法的主要目标分为以下几点：

1. 满足广告预算的花销计划，广告主希望在广告活动周期内相对平稳的消耗预算，保证在更长时间内有曝光效果，拥有持续性的广告推广效果。
2. 降低数据资讯花费，DSP 通常需要结合自己的第一方数据和来自 DMP 的第三方数据，更好的进行性能评估而做出更理想的决策，减少不必要的数据咨询次数可以降低这方面的经济花销。
3. 同时得到广告的投放和性能目标，对效果类广告来说首要任务是获得性能目标，尽可能的找到更多潜在的客户，对于品牌广告而言则需要达到更多的人群接触，获得更多的投放机会。

为了实现后两个目标，需要依赖广告预算步进算法(Budget Pacing)，步进算法是指对于各种历史行为数据的分析，得出一个符合近期趋势的广告预算的分配计划，这个分配计划是预算的花销额与实践的变化趋势，且去要控制时机的花销进度去接近这个分配计划来最终保证预算的按计划分配。

对于那些没有预算分配计划的广告活动，如图上图所示，可能出现集中恶性的结果：

1. 广告主的广告活动在一天中较早的时间就完全耗尽。而错失之后可能出现的高价值广告流量购买机会。
2. 广告主的出价整体偏低的，难以赢得adx的竞拍，导致预算花费进度不如预期而出现预算剩余，难以达到广告投放目标。


### 理想的预算控制
![](http://wx2.sinaimg.cn/mw1024/78d85414ly1fo4g8zaomvj211y0i3mx7.jpg "图2 竞标和曝光次数趋势图")

为了解决上面的问题，Joaquin等人利用动态规划和变分法证明了理想的预算花费不是线性也不是均匀的，而是正比于广告交易量，也就是广告流量，因此，就产生了一种根据广告流量变化趋势的分配计划，如上图所示。将分配计划正比于广告流量的变化趋势能够使的广告投放到一天内任何一个用户的几率均等，最大程度地保证广告均匀的分配到受众群体上，而不是均匀分布在每个时间段内。

## 流量预测(Traffic Prediction)
论文中提到的流量预测的方法都是基于高维度的特征提取然后建模预估流量，由于可能存在上万种特征，如果不进行降维处理，很难进行运算，因此，从用户特征，上下文，广告属性上万个特征中抽取出具有代表性的基准特征，当产生一次预测时，通过这几种基准特征为标准建立模型来预测流量，这种方法需要极大的训练样本和训练时间，很难在工程上实现。

## 论文具体算法实现

### 流量问题分析
在实际的广告系统中，广告请求数高度依赖于用户的行为，与此同时，用户的行为模式具有一定的规律性，工作日和休息日，白天和晚上所带来的流量请求均有即可循，因此，通过分析用户的行为来预测在线广告的流量走势具有一定的可操作性。

由于不同的时间段流量差异较大，该文将一天按时间分为T个时间段，如果取$T = 24$， 则每天的每个时间段为一个小时，因此，对流量问题的分析模型可以量化为以下公式的求极值的问题。

{% raw %}
$${\arg _\Theta }\min \sum\limits_{\forall d,t} {loss(h({X_{d,t}};\Theta ),{Y_{d,t}}),1 \le d \le D,1 \le t \le T}\tag{1} $$
{% endraw %}
其中，$D$ 为训练数据的天数， ${Y_{d,t}}$ 为在 d 天的 t 时刻的实际流量，${h({X_{d,t}};\Theta )}$ 为预测模型函数，计算值和实际值的均方差为损失函数，通过训练，寻找最优的模型来使得损失函数最小。

### 系统框架

![](http://wx3.sinaimg.cn/mw1024/78d85414ly1fo5unrqrxvj20ot09pt92.jpg "图2 系统架构图")

上图是整个系统的架构图，利用前D天的历史数据来生成当日的模型，首先提取出前D天的历史数据的特征，然后训练出当天需要用到的数据模型，当需要进行预测的时间点来到的时候，代入当前的时间点的特征，通过流量预测模型来得到此刻时间点的流量。


![](http://wx1.sinaimg.cn/mw1024/78d85414ly1fo5unqs8yzj20oj09odg5.jpg "图3 流量预测模块")

在整个系统框架中，最重要的为流量预测模型，流量预测模型中首先会更新上一个时间点的真实流量，检测出异常流量并对异常流量进行平滑处理，然后根据历史数据获取到当前的特征
，根据获取的特征进行线性回归即可得到真正的预测模型。

### 特征选取
从dsp的角度出发，而不是单纯的利用在线广告的一些特征，提取出一下三个特征

1. LastNDayReqs: 最近N天的当前Slot点的请求数
2. LastSlotReqs: 上一次Slot点的请求数(主要针对流量异常的解决方案)
3. SlotNumber: 当前的时间段

其中前两个参数分别代表了长期和短期因素对流量的影响，如果仅仅使用这两个元素的，进行流量预测产生的结果如下所示，

![](http://wx3.sinaimg.cn/mw1024/78d85414ly1fnylmc94zaj215w0i60x0.jpg "图4 忽略SlotNumber预测结果对比")

通过图中的对比可以发现，在任意时段对于流量的预测都过高或者过低，当流量发生变化的时候，对于流量的预测总是滞后于实际的流量值，因此，选择 SlotNumber 这一特征来弥补这种差距。

SlotNum 是长度为 T 的一个二进制的一维数组，对于其定义如下
{% raw %}
$$SlotNu{m^k}(i) = 1,k = i$$
$$SlotNu{m^k}(i) = 0,k \ne i$$
{% endraw %}
其中 ${i \in (1,T)}$，举个例子，当T=24时，则
{% raw %}
$$SlotNu{m^1} = [1,0.......0,0]$$
{% endraw %}
### 流量异常处理
由于在实际的线上广告系统中，其每天的流量规律往往不是一成不变的，对于流量的预测需要将流量异常的情况考虑在内，产生异常流量的原因主要包括两个部分，一是由于突发性事件造成的流量异常，例如某次事件的爆发，某次网络的迁移等，还有一种就是一种流量趋势的变化，在对流量建模的过程中需要将这种异常情况对于因变量进行一种平滑处理，为了解决流量异常的问题，通过综合考虑前k天的数据来对异常点的流量进行平滑处理。
### SmoothedLastSlotReqs
通过均衡前k天的t时间点的流量来判断t时间点的流量是否正常，其具体的步骤如下:

1. 获取到上一个时间点t的流量值, 计为
{% raw %}
  $$originalLastReqs = Reqs(0,t)\tag{2} $$
{% endraw %}  
其中 $t$ 为当天的时间点，$Reqs$ 为前k天的流量数据，其为二维数组 

2. 计算前k天的算术平均值和几何平均值
{% raw %}
$$avg = {1 \over k}\sum\limits_{i = 1}^k {Reqs(i,t)}\tag{3}  $$
$$st{d^2} = {1 \over k}\sum\limits_i^k {{{(reqs(i,t) - avg)}^2}}\tag{4}  $$
{% endraw %} 
3. 判断实际流量是否是异常流量，如果是正常流量，则不进行平滑，否则进行平滑操作得到最终的 
{% raw %}
$${{\left\| {originalLastReqs - avg} \right\|} \over {std}} > tol\tag{5}$$ 
{% endraw %} 
其中tol为设定的阈值，根据经验值取，如果该计算结果大于预知，则
{% raw %}
$$lastSlotReq{s^{k + 1}} = {\rm{ }}\prod\limits_{i = 0}^k {Reqs(k,t)}\tag{6} $$
{% endraw %}
反之，则
{% raw %}
$$lastSlotReqs = originalLastReqs\tag{7}$$
{% endraw %}

### SmoothedNDayReqs
对于 $SmoothedNDayReqs$ 参数的平滑操作类似，由于该特征代表着对于流量的一种长期的效应，如果流量呈现一种上升或者下降的趋势，该特征需要能反映出这种趋势，对于该特征的值的计算如下：
{% raw %}
$$lastNDaysReq{s^k} = \prod\limits_{i = 1}^k {Reqs(t + 1,i)} \tag{8}$$
{% endraw %}
### PredictTraffic
由于对于损失函数的最优化计算是一个不适定的过程，通过引入正则项来求出其最优解，对于公式1增加正则项目修改如下：
{% raw %}
$${\arg _\Theta }\min \sum\limits_{\forall d,t} {loss(({\omega ^T}{X_{d,t}} + b) - {Y_{d,t}}),1 \le d \le D,1 \le t \le T} \tag{9}$$
{% endraw %}
其中， {% raw %}${X_{d,t}} = [lastNDayreqs,lastSLotreqs,slotNum]$ , ${{\omega ^T}}$ {% endraw %}和$b$为正则项，通过找到使得损失函数最小的$({\omega ^T},b)$来求的最优化模型，得到最优化模型后对于流量预测到步骤如下所示：

1. 获取到最新的最近的流量计为$Reqs(i,j)$,其中 $i$ 为天数，$j$ 为时间片的值。
2. 获取到需要进行预测的时间片的值，计为 $t$。
3. 计算参数 $lastNDayReq{s_t} = SmoothNDayReqs(Reqs,t,T,k)$
4. 计算参数 $lastSlotReq{s_t} = SmoothLastSlotReqs(Reqs,t,T,k)$
5. ${X_{t}} = [lastNDayreqs_{t},lastSLotreqs_{t},slotNum_{t}]$
6. 将参数带入模型计算预测值$h({X_t};\omega ,b)$

## 线性回归

线性回归的一般问题就是，针对给出的数据，拟合出一个能够较为准确预测出输出结果的线性模型，针对上文定义好的模型进行线性回归:
{% raw %}
$$f(d,t) = {\omega ^T}X(d,t) + b \tag{10}$$

$$J(\omega ,b) = {1 \over 2}\sum\limits_{d = 1}^D {\sum\limits_{t = 1}^T {{{(f(d,t) - y(d,t))}^2}} } \tag{11}$$
{% endraw %}

上式中:

* $f(d,t)$ 是预测值
* $y(d,t)$ 是真实值
* $J(\omega ,b)$ 是代价函数
* $X(d,t)$是输入值
* $\omega ,b$是回归方程需要求解的参数

$J(\omega ,b)$ 表示了预测结果和真实结果的误差，其值越小表明预测结果越接近真实结果，因此，需要找到一组 $J(\omega ,b)$ 使得 $J(\omega ,b)$ 能够最小，根据代价函数，分别求取关于$J(\omega ,b)$的偏导数如下：
{% raw %}
$${{\partial J(\omega ,b)} \over {\partial \omega }} = \sum\limits_{d = 1}^D {\sum\limits_{t = 1}^T {X(d,t)(f(d,t) - y(d,t))} } \tag{12}$$

$${{\partial J(\omega ,b)} \over {\partial b}} =  - \sum\limits_{d = 1}^D {\sum\limits_{t = 1}^T {(f(d,t) - y(d,t))} } \tag{13}$$
{% endraw %}
对于 $J(\omega ,b)$ 的最小化的问题，可以将其看作是自变量为$J(\omega ,b)$的函数，这样最小化的问题即可变成函数的极值点为偏导数为0的点，因此:
{% raw %}
$${{\partial J(\omega ,b)} \over {\partial \omega }} =  - \sum\limits_{d = 1}^D {\sum\limits_{t = 1}^T {X(d,t)(f(d,t) - y(d,t))} }  = 0 \tag{14}$$

$${{\partial J(\omega ,b)} \over {\partial b}} =  - \sum\limits_{d = 1}^D {\sum\limits_{t = 1}^T {(f(d,t) - y(d,t)) = 0} } \tag{15}$$
{% endraw %}
求解方程组得到：
{% raw %}
$$\omega  = {{\sum\limits_{d = 1}^D {\sum\limits_{t = 1}^T {(X(d,t) - \bar X(d,t))(y(d,t) - \bar y(d,t))} } } \over {\sum\limits_{d = 1}^D {\sum\limits_{t = 1}^T {{{(X(d,t) - \bar X(d,t))}^2}} } }} \tag{16}$$
$$b = \bar y(d,t) - \omega \bar X(d,t) \tag{17}$$
{% endraw %}

上式中

* $\bar X(d,t)$ 表示输入参数的均值
* $\bar y(d,t)$ 表示实际值的均值

最小二乘法的优势在于计算简单，快速，并且找到的估计参数是全局极小值；缺点是对于异常值极其敏感。

## 算法结果
我们模拟生成近半个月的流量并根据论文的算法进行建模和参数预估，模拟生成半个月的流量数据，并选取其中的最近的7天作为训练样本，即

$$D = 7,T = 24$$

对生成的数据进行线性回归得到曲线如下，目前只取了前一天的曲线图：

![](http://wx1.sinaimg.cn/mw1024/78d85414ly1fo5n915e9jj211y0i3mxe.jpg "图5 回归曲线与真实曲线对比")

其最近七天的roc曲线和整体的roc曲线如下图所示

![](http://wx3.sinaimg.cn/mw1024/78d85414ly1fo5n91oi6xj211y0i3mxa.jpg "图6 最近七天ROC曲线")

![](http://wx4.sinaimg.cn/mw1024/78d85414ly1fo5n921039j211y0i3wet.jpg "图7 最近七天ROC曲线")

可以看出，其整体的拟合结果效果只有极少数的点呈现离散。


## 参考文献

1. [ Predicting Traffic of Online Advertising in Real-time Bidding Systems from Perspective of Demand-Side Platforms](https://github.com/wzhe06/Ad-papers/blob/master/Budget%20Control/Predicting%20Traffic%20of%20Online%20Advertising%20in%20Real-time%20Bidding%20Systems%20from%20Perspective%20of%20Demand-Side%20Platforms.pdf)
2. [Facebook 广告系统背后的Pacing算法](https://developers.facebook.com/docs/marketing-api/pacing)
3. [Github代码地址](https://github.com/Harren/traffic_predicting.git)






  













