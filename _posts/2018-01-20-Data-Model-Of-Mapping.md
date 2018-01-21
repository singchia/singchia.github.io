---
layout: post
title: 「映射」的数据模型设计
header-img: "img/mapping/mapping.png"
tags: 
    - mapping
    - redis
    - design
---

## 1. 背景
在做Faas项目的时候，遇到这样的架构设计需求。

> 请求到来时需要根据请求的类型发往不同的目标，但是这个「类型到目标」的映射关系除了程序化外还希望提供管理员接口可以手工操作。  

初期的设计中我们就将该映射关系单独提出来作为状态单独维护，不过后续快速变化的需求使我多次重新考虑用更解耦和的模型来优化这个过程。本文就是记录前前后后对这种映射关系的模型设计。  

## 2. 一对一和一对多
初期的设计时为了满足下面的几个目标：

* 请求到达时能够快速根据类型找到目标
* 映射关系作为状态存储起来，操作与状态解耦和
* 为了快速上线减少自开发代码

我们将这个状态维护在持久化```redis```中，基本可满足上述条件，所以```redis```中维护的状态大约如下：

	type a request -> Destination 1
	type b request -> Destination 2
	type c request -> Destination 3
	
这样就可以让```类型a```和```类型c```的请求发往```目标1```，而```类型b```发往```目标2```了。    

后来我们发现```类型a```的请求数目远大于```类型b```和```类型c```，仅```目标A```已经来不及处理请求了，需要**扩容**，添加与```目标1```同类型的目标来处理```类型a```的请求。于是状态变成**一对多**：
	
	type a request -> Destination 1'0, 1'1
	type b request -> Destination 2
	type c request -> Destination 3  

问题是我们发现```目标1```来不及处理请求，在重新操作映射时只能遍历整个映射表以免漏掉需要改变的映射，只因为根据目标找不到请求类型。如果加上目标到请求类型的映射，就会变成：

	type a request -> Destination 1'0, 1'1
	type b request -> Destination 2'0
	type c request -> Destination 3'0  
	
	Destination 1'0 -> type a request
	Destination 1'1 -> type a request
	Destination 2   -> type b request
	Destination 3   -> type c request
	
这种映射解决了遍历的问题，但是如果```类型a```的请求因为升级或者其他需要要求**更换目标**，那么这个映射表需要多改变2条映射，究其原因是**没有将目标与请求类型解耦和**。  

## 2. 组对组

为了解决上一节```类型a```请求变化的问题，可以让类型和目标解耦和来解决，因此引入**目标组**，让目标和目标组内聚，请求类型与目标解耦合：

	type a request -> group 1
	type b request -> group 2
	type c request -> group 3 
	
	group 1 contains { destination 1'0, 1'1 }
	group 2 contains { destination 2'0 }
	group 3 contains { destination 3'0 }
	
	destination 1'0 -> group 1
	destination 1'1 -> group 1
	destination 2'0 -> group 2
	destination 3'0 -> group 3
	
虽然又多了一些映射，但对于**更换目标**这种需求可以更优雅的做到，例如```类型a```的请求需要发往**目标组4**：

	                  group 1
	type a request -> group 4
	type b request -> group 2
	type c request -> group 3 
	
	group 1 contains { destination 1'0, 1'1 }
	group 2 contains { destination 2'0 }
	group 3 contains { destination 3'0 }
	group 4 contains { destination 4'0 }
	
	destination 1'0 -> group 1
	destination 1'1 -> group 1
	destination 2'0 -> group 2
	destination 3'0 -> group 3
	destination 4'0 -> group 4
	
如果需求仅止于次，似乎多出来的目标组解耦和并没有更多的说服力，但如果**目标组**不仅可以处理一种类型的请求呢，这样刚刚不再被映射的**目标组1**还可以被其他类型的请求使用，如果将条件再放松点，**目标组根本不跟类型相关，可以处理任何类型的请求，设定映射的唯一原因是根据目标组内目标数目的不同来给不同类型的请求分配不同数目的计算资源**。  

这时我想让**目标组1**处理```类型b```和```类型c```的资源，仅仅需要重新指向：

	type b request -> group 1
	type c request -> group 1
	type a request -> group 4
	                  group 2
	                  group 3 
	
	group 1 contains { destination 1'0, 1'1 }
	group 2 contains { destination 2'0 }
	group 3 contains { destination 3'0 }
	group 4 contains { destination 4'0 }
	
	destination 1'0 -> group 1
	destination 1'1 -> group 1
	destination 2'0 -> group 2
	destination 3'0 -> group 3
	destination 4'0 -> group 4

实际上至此已满足原项目大部分需求，但为了更方便管理请求类型、更近一步解耦和和设计对称，再引入**请求类型组**，例如业务上就可以看成**请求类型组A**的消息全部发往**目标组1**，具体的请求类型应该属于哪个类型组以及具体的目标应该属于哪个目标组根本只与自己相关：

	group A contains { type b,c request }
	group B contains { type a }
	
	type b request -> group A
	type c request -> group A
	type a request -> group B
	
	group A -> group 1
	group B -> group 4
               group 2
	           group 3 
	
	group 1 contains { destination 1'0, 1'1 }
	group 2 contains { destination 2'0 }
	group 3 contains { destination 3'0 }
	group 4 contains { destination 4'0 }
	
	destination 1'0 -> group 1
	destination 1'1 -> group 1
	destination 2'0 -> group 2
	destination 3'0 -> group 3
	destination 4'0 -> group 4
	
这些都是在目标组不与状态相关的条件下多出来的设计，实际上即使是组对组的所实现的映射关系呈现出来仍然有点繁杂。尤其是借助```redis```最基本的```key-value```实现出来，为了保证所有操作的原子性，这些甚至需要完全使用```redis EVAL```定义的lua脚本来做。  

[go-mapper](https://github.com/singchia/go-mapper)就是这样的实现，提供rest接口对映射进行操作。

## 3. tag标记实现
如果目标组跟请求类型相关，组对组的模型就不一定是最好的抽象方式，如果按照最不宽松的条件来，一个目标组只能处理一个类型的请求，那么可能会引起目标组暴涨，这样再多一层抽象也没有太多必要了。  

对于这种情况，可以把每一种请求类型看成一个标签（tag a, tag b, tag c...），而目标则可以持有多份标签，表示可以处理多种类型的请求：

	destination 1 (tag a, tag b)
	destination 2 (tag c)
	destination 3 (tag b)

为了让某个标签的请求到来时能够更快找到持有该标签的目标，可以为该标签建立索引：

	tag a contains { destination 1 }
	tag b contains { destination 1, 3}
	tag c contains { destination 2}
	
这样如果想要去除一种标签或者目标都比较方便。

## 4. topic与队列实现
将每个目标看成一个只能发布固定```topic```的队列，而请求的类型则为需要发布的```topic```。这种实现仅仅在```redis```中维护一个映射已经无法满足高可用的需求，例如在目标宕掉的情况下，如何维持这个队列依旧可用。  

这种情况可以维护一个```proxy```，来维持某个```topic```下的所有队列的连接。而所有的消息失败与重试都先由该```proxy```来内部消化并返回回执：

	+-------------------+---------------+---------------+
	|                   |               | topic a queue |
	|                   | topic a proxy | topic a queue |
	|request with topic +---------------+---------------+
	|                   |               | topic b queue |
	|                   | topic b proxy | topic b queue |
	+-------------------+---------------+---------------+

	
