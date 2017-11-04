This documentation illastrates several types of concurrency patterns which are most used in my work environment. It's all about how to use and control coroutines which exist as goroutines in golang. you may need that a operation do not block the processing, or limit certain amount of conroutines, or control them more precisely.   
Here are rough classifications of patterns:


Since goroutines are lighter and larger, if goroutines can be manipulated well, 