---
layout: post
title: "Concurrency Patterns: Summary And Implementation"
header-img: "img/concurrency/concurrency.png"
tags: 
    - concurrency-patterns
    - golang
    - goroutines
    - threads
---


## Background

This post illustrates several types of concurrency patterns which are most used in my work environment. It's all about how to use and control routines which may exist as goroutines in golang or threads in c/c++.   

You may want to keep an operation from blocking the whole system. To limit routines within a certain amount, or even control them more precisely.   

Here are rough classifications of patterns:

* unlimited concurrency
* limited concurrency
* elastic concurrency
* precise concurrency

And few other related stuff would also be involved.

## Use Case Illustration

I wrote serveral **echo-server**s which simply return **"server-ip:server-port what-client-send"** as the use case to illustrate the patterns .  

You can copy those codes into your workspace, run the server and using **telnet** or **netcat** to connect it and test it.

**Note that those snippets should not be used in production environment, since I really didn't do enough tests.**


## Unlimited Concurrency


Since goroutines are lighter and the amount is larger, if goroutines can be manipulated well, so will c threads.  

Let's step into the first pattern: 

~~~
package main

import (
    "flag"
    "net"
    "os"
)

func main() {
    var addr string
    flag.StringVar(&addr, "addr", ":1202", "listen addr")
    flag.Parse()

    l, err := net.Listen("tcp", addr)
    if err != nil {
        os.Exit(1)
    }
    defer l.Close()

    for {
        conn, err := l.Accept()
        if err != nil {
            os.Exit(1)
        }
        go func(conn net.Conn) {
            buf := make([]byte, 1024)
            for {
                len, err := conn.Read(buf)
                if err != nil {
                    conn.Close()
                    return
                }
                conn.Write(append([]byte(addr+" "), buf[:len]...))
            }
        }(conn)
    }
}
~~~

It's a really easy snippet, just starts a new goroutine to handle a new incoming connection. You can also get codes [here](https://gist.github.com/singchia/baac98c0f1a76851ac6b7ddd3a01c21f).

## Limited Concurrency

The original thought about limited concurrency is using certain routines to handle some common events with **signals** or **messages** or **shared memory**.  

Before reading the snippet below, you may would want to get the dependency:
	
	go get -u github.com/singchia/go-hammer
	
It's just a **circular linker** for saving connections in this example.  

~~~
package main

import (
	"flag"
	"net"
	"os"
	"time"

	"github.com/singchia/go-hammer/circulinker"
)

func main() {
	var addr string
	flag.StringVar(&addr, "addr", ":1202", "listen addr")
	flag.Parse()

	l, err := net.Listen("tcp", addr)
	if err != nil {
		os.Exit(1)
	}
	defer l.Close()

	connMap := make(map[net.Conn]circulinker.CircuID)
	cl := circulinker.NewCirculinker()

	for i := 0; i < 100; i++ {
		go func() {
			buf := make([]byte, 1024)
			for {
				time.Sleep(time.Microsecond * 50)
				if len(connMap) == 0 {
					continue
				}

				cl.Rightshift()
				conn := cl.RetrieveCur().(net.Conn)
				conn.SetReadDeadline(time.Now().Add(time.Microsecond * 50))
				len, err := conn.Read(buf)
				E, ok := err.(net.Error)
				if ok && E.Timeout() {
					continue
				}
				if err != nil {  
				
					conn.Close()
					cl.Delete(connMap[conn])
					delete(connMap, conn)
					continue
				}
				conn.Write(append([]byte(addr+" "), buf[:len]...))
			}

		}()
	}

	for {
		conn, err := l.Accept()
		if err != nil {
			os.Exit(1)
		}
		id := cl.Add(conn)
		connMap[conn] = id
	}
}
~~~ 

The snippet keeps every incoming connection into a circular linker, and 100 goroutines check if any connection got data with ```net.Conn.Read()```iteratively.  

Since **golang**'s native implementation about network data reading and writing is blocking, to use ```net.Conn.SetReadDeadline``` is the only way to simulate **non-blocking** operation. You can also get code [here](https://gist.github.com/singchia/d725754dbc7f83c9fa75be6da0b28326)
 
Note that it's not a good design, because I'm using shared memory insteading of messages or signals, It may cause mess and confusion about the complicated status.  

## Elastic Concurrency 

Sometimes I want to adjust the amount of routines at runtime rather than keeping using initial number at start time which may be too small or too big as the system goes.  

The snippet below takes standard input to allow user expand or shrink routines, in your case, you can use some strategies like **request-handle-rate-based-adjustment** to replace **standard-input-adjustment**, but in our example, we gotta stay that way.  

~~~
package main

import (
	"flag"
	"fmt"
	"net"
	"os"
	"sync"
	"time"
)

var addr string
var closeChan = make(chan struct{}, 1024)
var messageChan = make(chan net.Conn, 1024)
var feedbackChan = make(chan net.Conn, 1024)

func main() {
	flag.StringVar(&addr, "addr", ":1202", "listen addr")
	flag.Parse()

	l, err := net.Listen("tcp", addr)
	if err != nil {
		os.Exit(1)
	}
	defer l.Close()

	connMap := make(map[net.Conn]bool)
	mutex := new(sync.RWMutex)

	go func() {
		for {
			select {
			case conn := <-feedbackChan:
				mutex.Lock()
				delete(connMap, conn)
				mutex.Unlock()
			default:
				mutex.RLock()
				for conn := range connMap {
					messageChan <- conn
				}
				mutex.RUnlock()
			}
		}

	}()

	go func() {
		for {
			conn, err := l.Accept()
			if err != nil {  
			
				os.Exit(1)
			}
			mutex.Lock()
			connMap[conn] = true
			mutex.Unlock()
		}
	}()

	for {
		var i int
		_, err := fmt.Scanf("%d", &i)
		if err != nil {  
		
			fmt.Println("please input an integer", err.Error())
			continue
		}
		if i <= 0 {
			shrinkGoRoutines(-i)
		} else {
			expandGoRoutines(i)
		}
	}
}

func expandGoRoutines(num int) {
	var i int
	for i = 0; i < num; i++ {
		go func() {
			buf := make([]byte, 1024)
			for {
				select {
				case conn := <-messageChan:
					conn.SetReadDeadline(time.Now().Add(time.Microsecond * 50))
					len, err := conn.Read(buf)
					E, ok := err.(net.Error)
					if ok && E.Timeout() {
						continue
					}
					if err != nil {  
					
						conn.Close()
						feedbackChan <- conn
						continue
					}
					conn.Write(append([]byte(addr+" "), buf[:len]...))

				case <-closeChan:
					return
				}
			}
		}()
	}
}

func shrinkGoRoutines(num int) {
	var i int
	for i = 0; i < num; i++ {
		closeChan <- struct{}{}
	}
}
~~~

The first goroutine in ```main``` function is responsible for deleting closed connection and dispatch connections to worker goroutines, the second one accepts connections and add into the map.  

Then the ```main``` function continuously takes standard input to expand or shrink worker goroutines.  

You can also get code [here](https://gist.github.com/singchia/9b45bd1a2cb4241c4b9aa8b850f17205), and I personally implemented [one](https://github.com/singchia/go-scheduler).

## Precise Concurrency

Since goroutines and threads are stateless, we can combine them with some certain resources for precise control.  

My thought in golang is to generate a new channel for a new goroutine, then use this channel represents and control this goroutine.   

Other thing directly related to goroutine is **goroutine id**, so as thread, but it's deprecated that using **goroutine id** in production environment.

This can be used in some scenarios like binding **mysql instance**, **file descriptor** to some certain goroutines or threads.
