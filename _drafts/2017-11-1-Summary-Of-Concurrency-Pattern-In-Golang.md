## Background

This post illustrates several types of concurrency patterns which are most used in my work environment. It's all about how to use and control routines, those may exist as goroutines in golang or threads in c/c++.   

You may don't want a operation blocking the whole system. Or to limit routines within a certain amount, or even control them more precisely.   

Here are rough classifications of patterns:

* unlimited concurrency
* limited concurrency
* procise concurrency
* elastic concurrency

And few other related stuff would also be involved.

## Use Case Illustration

I wrote serveral **echo-server** as the use case to illustrate the patterns, it simply return **server-ip:server-port** and **what client send**.  

Note that those snippets should not be used in production environment, cause I really didn't do enough tests.


## Unlimited Concurrency


Since goroutines are lighter and the amount is larger, if goroutines can be manipulated well, so will c threads.  

Let's step into the first pattern: 

![](/Users/zhaizenghui/singchia.github.io/img/concurrency/unlimited-golang.png)

It's a really easy sample, just start a new goroutine to handle a new incoming connection. You can copy codes [here](https://gist.github.com/singchia/baac98c0f1a76851ac6b7ddd3a01c21f) for your own tests.

## Limited Concurrency