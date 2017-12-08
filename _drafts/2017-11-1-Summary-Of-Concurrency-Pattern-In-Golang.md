This document illustrates several types of concurrency patterns which are most used in my work environment. It's all about how to use and control coroutines which exist as goroutines in golang. you may don't want a operation block the processing, or limit conroutines within a certain amount, or control them more precisely.   
Here are rough classifications of patterns:

* unlimited concurrency
* limited concurrency
* elastic concurrency
* procise concurrency

And few other patterns about messaging and dependency would also be involved.

## Unlimited Concurrency


Since goroutines are lighter and larger, if goroutines can be manipulated well, 