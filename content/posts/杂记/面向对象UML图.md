---
author: "李昌"
title: "面向对象UML图"
date: "2021-12-26"
tags: ["UML", "OOB"]
categories: ["杂记"]
ShowToc: true
TocOpen: true
---

## 1. 用例图

用例图主要用于定义系统的功能需求，它描述了系统的参与者与系统提供的用例之间的关系，用例图仅从参与者使用系统的角度描述系统中的信息。

**图例**  
![1](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226151623.png)
![2](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226151655.png)


**示例**  
![3](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226151743.png)

## 2. 时序图（顺序图）
顺序图描述了对象以及对象之间传递的消息，强调对象之间的交互是按照时间的先后顺序发生的，这些特定顺序发生的交互序列从开始到结束需要一定的时间。在顺序图中主要包括了以下 4 种元素。
● 对象
● 生命线
● 激活
● 消息

**图例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226151855.png)

**示例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226151931.png)

## 3. 协作图
协作图与顺序图一样，也是用于描述系统中各对象的交互关系并展现对象间的消息传递，但两者侧重点不同，顺序图着重于描述交互的时间顺序，而协作图着重于描述协作对象间的交互和连接。还可以从另一个角度来看两种图的定义，顺序图是按照时间的顺序布图，而协作图是按照空间来布图。

**图例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226152426.png)

**示例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226152444.png)

**顺序图与协作图的关系**  
顺序图和协作图在语义上是等价的，它们之间可以进行互相转换。
例如上面的协作图可以等价转化为顺序图：  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226152551.png)

## 4. 类图
类图描述了类和类间关系，它从静态角度来表示一个系统，因此类图属于一种静态图。类图是 UML 建模中最基本和最重要的一类图。

**图例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226152658.png)
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226152711.png)

**示例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226152803.png)

## 5. 对象图
对象图是类图的实例，几乎使用与类图完全相同的标识。它们的不同点在于对象图显示类的多个对象实例，而不是实际的类。一个对象图是类图的一个实例。由于对象存在生命周期，因此对象图也是有生命周期的，它只能在系统某一时间段存在。

**示例** 
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226152932.png)

## 6. 包图
创建包图的主要作用是：
- 描述需求的高阶概述。
- 描述设计的高阶概述。
- 在逻辑上把一个复杂的图模块化。
- 组织源代码。
- 对框架进行建模。

**图例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226153104.png)

**示例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226153142.png)

## 7. 状态图
状态图主要用来描述一个特定对象的所有可能状态以及由于各种事件的发生而引起状态之间的转移。通过状态图可以知道一个对象、子系统、系统的各种状态及其收到的消息对其状态的影响。通常创建一个 UML 状态图是为了以下的研究目的：研究类、角色、子系统或构件的复杂行为。

状态图主要由起点、终点和状态组成，各状态由转移连接在一起。状态是对象执行某项活动或等待某个事件时的条件。转换是两个状态之间的关系，它由某个事件触发，然后执行特定的操作或评估并导致特定的结束状态。

状态图适合于描述跨越多个用例的单个对象的行为，而不适合描述多个对象之间的行为协作。为此，常常将状态图与其他技术组合使用。

**图例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226153329.png)
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226153341.png)

**示例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226153419.png)


## 8. 活动图
活动图是用来描述达到一个目标所实施一系列活动的过程，描述了系统的动态特征。活动图类似结构化程序课程中的流程图，不同之处在于它支持并行活动。活动图和状态图的主要区别在于状态图侧重从行为的结果来描述，以状态为中心；活动图侧重从行为的动作来描述，以活动为中心。活动图用来为一个过程中的活动序列建模，而状态图用来为对象生命期中的各离散状态建模。

**图例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226153524.png)
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226153535.png)

**示例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226153605.png)

## 9. 构件图
构件是系统的模块化部分，它封装了自己的内容，且它的声明在其环境中是可以替换的；构件利用提供接口和请求接口定义自身的行为，它起类型的作用。

**图例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226153810.png)
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226153829.png)

**示例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226153854.png)

![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226153922.png)

## 10. 部署图
部署图描述了整个系统的软硬件的实际配置，它表示了系统在运行期间的体系结构、硬件元素（节点）的构造和软件元素是如何被映射在那些节点之上的。

部署图可以帮助系统的有关人员了解系统中各个构件部署在什么硬件上，以及这些硬件之间的交互关系。

**图例**    
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226155030.png)
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226155050.png)

**示例**  
![](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20211226155109.png)

## References
[《面向对象技术及UML教程》 李磊，王养廷主编](http://lnbook.wenqujingdian.com/Public/editor/attached/file/20180317/20180317234200_64961.pdf)














