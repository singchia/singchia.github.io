---
layout: post
title: Summary Of Concurrency Pattern
header-img: "img/concurrency/concurrency.png"
tags: 
    - concurrency-pattern
    - golang
    - goroutine
---


## Background

This post illustrates several types of concurrency patterns which are most used in my work environment. It's all about how to use and control routines, those may exist as goroutines in golang or threads in c/c++.   

You may don't want a operation blocking the whole system. Or to limit routines within a certain amount, or even control them more precisely.   

Here are rough classifications of patterns:

* unlimited concurrency
* limited concurrency
* elastic concurrency
* precise concurrency

And few other related stuff would also be involved.

## Use Case Illustration

I wrote serveral **echo-server** as the use case to illustrate the patterns, it simply return **server-ip:server-port** and **what client send**.  

You can copy those codes into your workspace, run the server, then using **telnet** or **netcat** to connect it and test it.

**Note that those snippets should not be used in production environment, since I really didn't do enough tests.**


## Unlimited Concurrency


Since goroutines are lighter and the amount is larger, if goroutines can be manipulated well, so will c threads.  

Let's step into the first pattern: 

![](/img/concurrency/unlimited-golang.png)

It's a really easy snippet, just start a new goroutine to handle a new incoming connection. You can get codes [here](https://gist.github.com/singchia/baac98c0f1a76851ac6b7ddd3a01c21f).

## Limited Concurrency

The original thought about limited concurrency is using certain routines to handle some common events with **signals** or **messages** or **shared memory**.  

Before reading the snippet below, you may would want to get the dependency:
	
	go get -u github.com/singchia/go-hammer
	
It's just a **circular linker** for saving connections in this example.  

![](/img/concurrency/limited-golang1.png)
![](/img/concurrency/limited-golang2.png)  

The snippet keeps every incoming connection into a circular linker, and 100 goroutines check if any connection got data with ```net.Conn.Read()```iteratively.  

Since **golang**'s native implementation about network data reading and writing is blocking, to use ```net.Conn.SetReadDeadline``` is the only way to simulate **non-blocking**. You can get code [here](https://gist.github.com/singchia/d725754dbc7f83c9fa75be6da0b28326)
 
Note that it's not a good design, because I'm using shared memory insteading of messages, It may cause mess and confusion about the complicated status.  

## Elastic Concurrency 

Sometimes I want adjust the amount of routines at runtime rather than keeping using initial number at start time which may be too small or too big as the system goes.  

The snippet below takes standard input to allow user expand or shrink routines, in your case, you can use some strategy like **request-handle-rate-based-adjustment** to replace **standard-input-adjustment**, but in our example, we gotta stay that way.  

![](/img/concurrency/elastic-golang1.png)
![](/img/concurrency/elastic-golang2.png)
![](/img/concurrency/elastic-golang3.png)  

The first goroutine in ```main``` function is responsible for deleting closed connection and dispatch connections to worker goroutines, the second one accepts connections and add into map.  

Then the ```main``` function continuously takes standard input to expand or shrink worker goroutines.  

You can get code [here](https://gist.github.com/singchia/9b45bd1a2cb4241c4b9aa8b850f17205), and I personally implemented [one](https://github.com/singchia/go-scheduler).

## Precise Concurrency

Since goroutines or threads are stateless, we can combine them with some certain resources for precise control.  

My thought in golang is to generate a new channel for a new goroutine, then use this channel represents and control this goroutine.   

Other thing directly related to goroutine is **goroutine id**, so as thread, but it's really deprecated using **goroutine id** in production environment.

This can be used in some scenarios like binding **mysql instance**, **file descriptor** to some certain goroutines or threads.
