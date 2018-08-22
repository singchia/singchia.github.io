---
layout: post
title: An Introduction Of Hierarchical Timing Wheels
header-img: "img/timer/timer.png"
catalog: true
tags: 
    - timingwheel
    - algorithms-datastructures
    - high-performance
---

## Background
Timer is a so important component be needed in many projects to limit a action or to trigger an event, we used to start a thread or coroutine to implement one for rapid development. But some of you may encountered the scenario that requires lot of timers, like developing a restful server or a message server, then a high-performance timer is strongly needed. we got serveral options to choose:

*	Ordered Linked Timer List
* 	Heap Based Timer
*	Hashed Timing Wheels
*	Hierarchical Timing Wheels

And this documentation illustrates hierarchical timing wheels algorithm and an implemention of it.

## Algorithm

I drew some easy-understand diagrams which are time ordered to describe the algorithm, and some corresponding comments had been written next to the graphic content.  
Let's name a fixed internal of time going-by as **ticking**, naming a pre-set-able event as **tick** which be called _**timer**_ in common, and  **timer** presents the algorithm. So let's start with the case with simple 2 wheels.  

#### ticking 0
Assuming we got a clean enviroment and doing nothing at ticking 0.
![](/img/timingwheels/ticking0.jpg)

#### ticking 1
Calculate 2 5-ticking-after ticks's positions in wheels respectively and insert into right place.
![](/img/timingwheels/ticking1.jpg)

#### ticking 6
Since nothing special happened from ticking 1 to ticking 5, they've all omitted. At ticking 6, the **ticks** set at ticking 1 fired.
![](/img/timingwheels/ticking6.jpg)

#### ticking 7
Add another 3 **ticks** into timer.
![](/img/timingwheels/ticking7.jpg)

#### ticking 8 
This ticking is important because it illustrates what happened when a lower wheel's ticking rotated back at position 0, and this, will raise the ticking of higher wheel.
![](/img/timingwheels/ticking8.jpg)

#### ticking 24
Two ticks set at ticking 8 fired, one of them ended and another one be set into lower wheel.
![](/img/timingwheels/ticking24.jpg)

#### ticking 27
The tick in wheel 0 fired.
![](/img/timingwheels/ticking27.jpg)

#### ticking 56
The last tick set at ticking 7 fired and be set into lower wheel.
![](/img/timingwheels/ticking56.jpg)

#### ticking 63
The last tick fired and ended.
![](/img/timingwheels/ticking63.jpg)

#### ticking 64
Everything cleaned up as ticking 0.
![](/img/timingwheels/ticking64.jpg)

## Implementation
see implementation in golang [here](https://github.com/singchia/go-timer)
