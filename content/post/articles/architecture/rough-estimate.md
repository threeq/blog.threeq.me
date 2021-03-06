---
title: 架构师：《百度的春晚战事》给我们的启示 — 粗略估算
date: 2019-03-13
lastmod: 2019-03-13
draft: false
keywords: [架构,微服务,注册中心,分布式,架构设计,架构分析,容量估算,容量分析,粗略估算,Little定律]
description: ""
categories:
 - 架构
tags:
 - 架构
 - docker
 - 微服务
toc: true
comment: true

autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false

contentCopyright: false
reward: false
mathjax: true
mathjaxEnableSingleDollar: false

flowchartDiagrams:
  enable: true
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""
---

春节期间大家一边看着春晚，一边拿着百度 APP 顺利的抢着红包，作为一枚标准程序员的我，同时不停刷着各个技术网站和论坛，依据之前“春晚红包”的经验，出问题后很快就会有问题分析文章出来（特别是第一波红包），借此机会又可以学习一波（毕竟别人的经验，就是我们最好的学习教材）。但是今年并没有，也没有相关技术文章出来，可能大家没有太大感觉，作为给客户做过春节红包系统的我（当然我们的量是很小的，当时我们还盯着给客户做的春节红包系统呢^_^）发自内心对百度技术的佩服：抗住了，百度牛逼！。作为标准程序员的我们，怎么能停止在佩服的层面就行了呢？对吧！百度是怎么抗住的？用了多少服务器？怎么估算的的服务器？这些问题一直困惑我，也促使我不断思考。直到春节后百度出了一篇文章 [《百度的春晚战事》](https://mp.weixin.qq.com/s/W9Nbq64v9doYPxcCLBsqNQ) ，简要介绍了这次“百度春晚项目“的大致流程，项目过程也是波澜壮阔、成叠起伏（建议大家可以看一下），不过其中包含的少量技术数据才是我关注的重点，不过通过这些数据也能粗粒度的解惑我之前问题，这里就从`系统容量`估算角度阐述我得到的启示——**粗略估算**。

<!--more-->

在文章里面提到预估流量每秒峰值 *5000w*，每分钟峰值 *10亿* （这里的流量没有具体说明是什么，我们就理解成接口调用次数），经过计算需要使用 *10w* 台服务器支持，根据文章描述也确实是使用了 10w 台服务器。10w 台服务器支撑14亿人的访问量，是否真的可行？我们有没有一个比较简单的模型预估其可行性，这里可以使用**Little定律** 进行简单的估算：

* 10w 台服务器 14亿人 => 14亿/10w = 1.4w 人/台，意味着平均**一台服务器要支撑14000万人**请求

* 假定在抢红包期间平均每个发起 **1** 次请求（这个具体的平均请求数可以根据业务实现分析出来）

文章里提到预估流量峰值每秒 5000w，每分钟峰值 10亿，初看这里是有矛盾的: 5000w/s，到1分钟应该是 `5000w/s*60s =30亿/分钟` 。其实对于抢红包峰值持续时间是很短的，针对现在用户对移动端的响应要求 **2s~5s：最好是2s内，5s已经极限了** 。这样10亿的估值就不难理解了，设定峰值持续时间 T，一般平均访问量 v，则有
$$
0.5亿 * T + (60-T) * v = 10亿
$$
（由于红包系统的特殊性这里选择简单的方形波计算，大家有兴趣也可以选择泊松分布进行计算）

> * T=2，\\(v\approx2000w/s\\)  ，约为峰值的40%
> * T=5，\\(v\approx1400w/s\\) ，约为峰值的30%
> * T=10，\\(v\approx1000w/s​\\) ，约为峰值的20%
> * T=15，\\(v\approx555w/s\\) ，约为峰值的11%
> * T=20，\\(v\approx0w/s\\) ，约为峰值的0%

抢红包系统在访问分布上峰值将会远远高于其他时间的平均访问。

* 预估流量峰值每秒 5000w，每台服务器请流量 \\( 5000w/10w = 500请求/秒 \\)

* 在手机端抢红包，在 **2s** 内应答（包含用户思考时间和响应时间）是很好的体验

**Little定律** $$N=X \cdot R​$$

> **N**: 平均负载
>
> **X**: 吞吐量
>
> **R**: 平均响应时间

其中 \\(X=500次/s\\) , \\( R=2s \\) ，则有
$$
N=500次/s * 2s = 1000次
$$
这样计算整个系统的在峰值期间参与抢红包的人数大概在 1000*10w = 1亿 左右，占全国总人数的 7%~10%左右。

上面的分析过于简化并不准确，并且只是按照一般的 web 系统分析，也欢迎大家拍砖。下面才是重点

# 估算实例 - 抢红包/秒杀活动系统

一个仅支持企业员工抢红包的系统，企业员工抢红包系统本身的请求量不算大，就算是全员参与企业的人数也是有限的（一般在万到十万这个级别），但是会有一些其他辅助功能(比如会有特定的活动页、红包排行等)，这就会增加活动过程中当个用户发起的请求量，从而增加系统的整体请求量。

```flow
st=>start: Start
e=>end: End
op2=>operation: 请求活动信息(红包或秒杀)
op3=>operation: 执行活动请求(抢红包或秒杀)
op4=>operation: 查看活动结果(红包结果和排行或秒杀结果)

st->op2->op3->op4->e
```

从活动开始到结束，大体分成3个阶段，每个阶段至少会有一个请求，同时也有部分用户并不会走完所有流程，这里假定平均每个用户发起**4个请求**。一个用户参加活动到结束，整个操作时间加响应等待时间不超过**2s**，这就是每个用户在系统的平均停留时间。

企业总人数：10000

同时在线：5000

总请求：20000

完成服务时间：10s

每秒需要处理：2000

如果一台服务器TPS/QPS在 300~500 左右

所需服务器：4~7



**架构设计，知易行难，实践是最好的捷径。**

[编程珠玑](https://u.jd.com/6GnhwE)

[编程珠玑(续)](https://u.jd.com/yGVuCB)