---
layout: post
title: 浅谈代理
header-img: "img/proxy/proxy.png"
tags: 
    - ssh
    - socks5
    - iptables
    - lvs
    - nat
    - proxy
---

## 1. 背景
本文所讨论的```代理```是计算机网络范畴的概念，在[维基百科](https://en.wikipedia.org/wiki/Proxy)有```代理服务```的名词定义：

> Proxy server, a computer network service that allows clients to make indirect network connections to other network services.  
> 译：代理服务器，一种允许客户端间接地连接到其他网络服务的服务。  

```代理```所做的正是将```A```不能或者不好到达```C```的流量：

	A !=> C

通过```代理B```来到达：
	
	A => B => C 
	

```代理```之所以重要是因为从普通用户到专业运维都会不可避免地跟它打交道。 普通用户可能需要访问正常不能访问的网站，或者希望自己玩游戏不会卡顿。专业运维可能需要用更方便地方式访问部署服务的外网服务器，或者希望外网服务器进入的流量能够更均衡或者更高可用地落到真正提供服务的内网服务器上。  

本文主要从工具和原理等方面来谈一谈```代理```。  


## 2. ```ssh```端口转发
本节所讨论的```ssh```为```OpenSSH SSH client```，也是常用的远程登录程序。其客户端基本用法如下：
	
	ssh [-1246AaCfGgKkMNnqsTtVvXxYy] \
	[-b bind_address] [-c cipher_spec] [-D [bind_address:]port] \
	[-E log_file] [-e escape_char] [-F configfile] [-I pkcs11] \
	[-i identity_file] [-J [user@]host[:port]] [-L address] \
	[-l login_name] [-m mac_spec] [-O ctl_cmd] [-o option] \
	[-p port] [-Q query_option] [-R address] [-S ctl_path] \
	[-W host:port] [-w local_tun[:remote_tun]] \
	[user@]hostname [command]

### 2.1. ```ssh -L```本地端口转发

	场景描述：
	A机器内网地址为：192.168.0.10
	B机器公网地址为：220.220.0.10 内网地址为：172.16.0.11
	C机器内网地址为：172.16.0.10
	A能够访问B
	B能够访问C
	A不能访问C
	B、C机器上都运行了sshd；A机器上有ssh客户端
	现在需要A能够访问C机器上的1202上端口
	
本地转发需要使用```-L```选项。在```ssh```的```man```中有这样一段：

> Specifies that connections to the given TCP port or Unix socket on the local (client) host are to be forwarded to the given host and port, or Unix socket, on the remote side.  This works by allocating a socket to listen to either a TCP port on the local side, optionally bound to the specified bind_address, or to a Unix socket.  Whenever a connection is made to the local port or socket, the connection is forwarded over the secure channel, and a connection is made to either host port hostport, or the Unix socket remote_socket, from the remote machine.  

在主机```C```上使用```nc```来监听```1202```端口

	> nc -l 172.16.0.10 1202

我们在主机```A```上：
	
	> ssh -L 7777:172.16.0.10:1202 220.220.0.10
	
紧接着在主机```A```上连接本地```7777```端口并发送```hello```：

	> nc 127.0.0.1 7777
	hello
	
在主机```C```上将会看到消息正常到达，即本机```777```端口的流量经过主机```220.220.0.10```转发给```172.16.0.10```的端口```1202```。 以下为各个主机中相关```socket```情况：
	
**主机```A```：**

	Active Internet connections (servers and established)
	Proto Recv-Q Send-Q Local Address           Foreign Address         	State       PID/Program name
	tcp        0      0 127.0.0.1:7777          0.0.0.0:*               	LISTEN      26071/ssh
	tcp        0      0 127.0.0.1:40900         127.0.0.1:7777          	ESTABLISHED 26677/nc
	tcp        0      0 127.0.0.1:7777          127.0.0.1:40900         	ESTABLISHED 26071/ssh
	tcp        0      0 192.168.0.10:48674      220.220.0.10:22         	ESTABLISHED 26071/ssh
	tcp6       0      0 ::1:7777                :::*                    	LISTEN      26071/ssh
	
可以看出本机上```ssh```监听了```7777```端口并使用端口```48674```与```220.220.0.10:22```建立了连接；```nc```使用端口```40900```与```777```建立了连接。
	
**主机```B```：**

	Proto Recv-Q Send-Q Local Address           Foreign Address         	State       PID/Program name
	tcp        0      0 0.0.0.0:22              0.0.0.0:*               	LISTEN      28336/sshd
	tcp        0      0 220.220.0.10:22         192.168.0.10:48674      	ESTABLISHED 26072/sshd: root@pt
	tcp        0      0 172.16.0.11:57922       172.16.0.10:1202        	ESTABLISHED 26072/sshd: root@pt
	tcp6       0      0 :::22                   :::*                    	LISTEN      28336/sshd

可以看到一个来自主机```A```的连接以及一个内网连接```172.16.0.11:57922```到```172.16.0.10:1202```。

**说明：**有公网设备相关工作经验的读者肯定会疑惑为什么```B公网地址```与```A内网地址```直接相通，请忽略这点，因为这个场景环境是使用```netns(linux network namespace)```搭建，实际上所有的地址都是内网地址。


**主机```C```：**

	Active Internet connections (servers and established)
	Proto Recv-Q Send-Q Local Address           Foreign Address         	State       PID/Program name
	tcp        0      0 0.0.0.0:22              0.0.0.0:*               	LISTEN      28342/sshd
	tcp        0      0 172.16.0.10:1202        172.16.0.11:57922       	ESTABLISHED 26571/nc
	tcp6       0      0 :::22                   :::*                    	LISTEN      28342/sshd
 
 抓包可以看出流量过程：```A:40900 -> A:7777 => A:48674 -> B:22 => B:57922 -> C:1202```
 
### 2.2. ```ssh -R```远程端口转发

	场景描述：
	A机器公网地址为：220.220.0.10
	B机器内网地址为：192.168.0.10
	C机器内网地址为：172.16.0.10
	B能够访问A
	B能够访问C
	A不能访问C
	A、C机器上都运行了sshd；B机器上有ssh客户端
	现在需要A能够访问C机器上的1202上端口
	
远程转发需要使用```-R```选项。在```ssh```的```man```中有这样一段：

> Specifies that connections to the given TCP port or Unix socket on the remote (server) host are to be forwarded to the given host and port, or Unix socket, on the local side.  This works by allocating a socket to listen to either a TCP port or to a Unix socket on the remote side.  Whenever a connection is made to this port or Unix socket, the connection is forwarded over the secure channel, and a connection is made to either host port hostport, or local_socket, from the local machine.  

在主机```C```上使用```nc```来监听```1202```端口

	> nc -l 172.16.0.10 1202
	
我们在主机```B```上：

	> ssh -R 7777:172.16.0.10:1202 220.220.0.10

紧接着在主机```A```上连接本地```7777```端口并发送```hello```：

	> nc 127.0.0.1 7777
	hello
	
在主机```C```上将会看到消息正常到达，即本机```777```端口的流量经过主机```192.168.0.10```转发给```172.16.0.10```的端口```1202```。 以下为各个主机中相关```socket```情况：

**主机```A```：**

	Active Internet connections (servers and established)
	Proto Recv-Q Send-Q Local Address           Foreign Address         	State       PID/Program name
	tcp        0      0 127.0.0.1:7777          0.0.0.0:*               	LISTEN      18636/sshd: root@pt
	tcp        0      0 0.0.0.0:22              0.0.0.0:*               	LISTEN      28870/sshd
	tcp        0      0 127.0.0.1:7777          127.0.0.1:40918         	ESTABLISHED 18636/sshd: root@pt
	tcp        0      0 220.220.0.10:22         192.168.0.10:50718      	ESTABLISHED 18636/sshd: root@pt
	tcp        0      0 127.0.0.1:40918         127.0.0.1:7777          	ESTABLISHED 18731/nc
	tcp6       0      0 ::1:7777                :::*                    	LISTEN      18636/sshd: root@pt
	tcp6       0      0 :::22                   :::*                    	LISTEN      28870/sshd
	
可以看出```sshd```监听了```7777```端口并接受来自主机```B```的连接```220.220.0.10:22```和```192.168.0.10:50718```。```nc```使用```40918```与```sshd```监听的```7777```建立了连接。 

**主机```B```：**

	Active Internet connections (servers and established)
	Proto Recv-Q Send-Q Local Address           Foreign Address         	State       PID/Program name
	tcp        0      0 0.0.0.0:22              0.0.0.0:*               	LISTEN      28336/sshd
	tcp        0      0 192.168.0.10:50718      220.220.0.10:22         	ESTABLISHED 18635/ssh
	tcp        0      0 192.168.0.10:57940      172.16.0.10:1202        	ESTABLISHED 18635/ssh
	tcp6       0      0 :::22                   :::*                    	LISTEN      28336/sshd
	
一个是上述```主机B```到```主机A```的连接，一个是到```主机C```的连接。

**主机```C```：**

	Active Internet connections (servers and established)
	Proto Recv-Q Send-Q Local Address           Foreign Address         	State       PID/Program name
	tcp        0      0 0.0.0.0:22              0.0.0.0:*               	LISTEN      28342/sshd
	tcp        0      0 172.16.0.10:1202        192.168.0.10:57940       	ESTABLISHED 18605/nc
	tcp6       0      0 :::22                   :::*                    	LISTEN      28342/sshd
	
抓包可以看出流量过程：```A:40918 -> A:7777 => A:22 -> B:50718 => B:57940 -> C:1202```。

```tcp```是通的，那么自然```http```也通了，最常使用这种方法场景的是从公网访问内网的```web服务器```。
 
## 3. SOCKS5协议

虽然端口转发可以解决某一个端口的代理问题，但我们还有很多动态端口转发的需求，即我们希望将目的包封装起来，发给```代理服务```，```代理服务```收到后解包再按照该目的地址转发，只所以是动态，是因为```代理服务```会根据解包后目的地址不同建立不同的长连接。

有一个名为```SOCKS```的```代理协议```比较流行，它定义了一种动态端口转发的协议。在我[SOCKS5协议「RFC1928翻译」](http://www.singchia.com/2018/03/21/RFC1928-Socks-Protocol-Version-5/)中可以了解细节。```RFC1928```描述的```SOCKS5```协议发布于```1996```，甚至早于```1999```的```HTTP/1.1(RFC2616)```协议。

### 3.1. ShadowSocks
```SOCKS5```协议的实现很多，其中```ShadowSocks```因为```The Great Firewall ```的原因被很多同学所使用。由于```SOCKS5```协议只描述了客户端到服务端之间的协议，因此在服务端如何把包发往目标机器是可以自己决定的。

```ShadowSocks```的做法是分为```ss客户端```和```ss服务端```，**注意**```ss客户端```和```ss服务端```之间的通信并非```SOCKS5```协议实现，而```客户端```到```ss客户端```的过程是符合```SOCKS5```协议的。即```ss客户端```为```SOCKS5```服务端，```客户端```封包是由操作系统支持的，这里的```客户端```可以是浏览器。

如果读者本地也安装了```ShadowSocks```的```ss客户端```，会发现通常该```ss客户端```默认是监听本地```127.0.0.1:1086```端口的（虽然```RFC```提到通常使用```1080```端口）。如果你在```lo```网卡上抓包，访问```google.com```就可以抓到```SOCKS5```的包：

![](/img/proxy/socks5-stream.png)

中间列为负载的```16进制```显示，右列为```acsii```显示，对比```SOCKS5```协议，可以发现：
	
	>> 05 01 00                                           ...
	#客户端发起SOCKS5(0x05)流程，认证方法占一个字节(0x01)，无需认证(0x00)
	
	<< 05 00                                              ..
	#SOCKS5(0x05)服务端表示可以不认证(0x00)
	
	>> 05 01 00 03 0e 77 77 77  2e 67 6f 6f 67 6c 65 2e   .....www .google.
	#客户端发起SOCKS5(ox05)连接(0x01)，保留一字节(0x00)，地址类型为域名(0x03)，域名13字节(0x0e)
	>> 63 6f 6d 01 bb                                     com..
	#域名和端口为www.google.com:443
	
	<< 05 00 00 01 00 00 00 00  00 00                     ........ ..
	#返回成功，未返回绑定的地址和端口，这在协议中本来就是可选的
	
而协商成功后包已经全都属于数据面了，即业务层的内容，给```SOCKS5```服务端会直接转发。

### 3.2. ```ssh -D```动态端口转发
动态端口转发需要```-D```选项，在```ssh```的```man```中有这样一段：

> Specifies a local “dynamic” application-level port forwarding.  This works by allocating a socket to listen to port on the local side, optionally bound to the specified bind_address. Whenever a connection is made to this port, the connection is forwarded over the secure channel, and the application protocol is then used to determine where to connect to from the remote machine.  Currently the SOCKS4 and SOCKS5 protocols are supported, and ssh will act as a SOCKS server.  Only root can forward privileged ports.  Dynamic port forwardings can also be specified in the configuration file.

可以看出```ssh```也是同样作为```SOCKS5```服务端工作的，读者也可以使用这种方式来实现动态端口转发。

## 4. NAT
上述的几种方法里都是从工具的角度来达到日常代理需求，总的来说不够透明。再看```代理```的行为，```A```不能或者不好到达```C```的流量：

	A !=> C

通过```代理B```来到达：
	
	A => B => C 
	
说明这里```B```的网络与```C```是通的，我们还是用```2.1节```的场景来说明：
	
	场景描述：
	A机器内网地址为：192.168.0.10
	B机器公网地址为：220.220.0.10 内网地址为：172.16.0.11
	C机器内网地址为：172.16.0.10
	A能够访问B
	B能够访问C
	A不能访问C
	
这次我们附上场景图：

![](/img/proxy/before-nat.png)

此时再考虑前面章节所描述的工具，所有让```A```到```B```的流量为了能够到```C```，在```B```上都会改变流量的```目的ip地址```为```C```的```ip```地址。

那么自然能想到最简单的就是```NAT```了，所谓```NAT(RFC2663)```即```地址转换协议```，属于网络层，在```RFC2663```文档中描述了典型应用场景，局域网中的主机需要连通```互联网```则需要做```地址转换```，把```私有ip```转换成```公网ip```，反之依然。

在```RFC2663 4.1.1节```中描述了基本```NAT```：

> With Basic NAT, a block of external addresses are set aside for translating addresses of hosts in a private domain as they originate sessions to the external domain. For packets outbound from the private network, the source IP address and related fields such as IP, TCP, UDP and ICMP header checksums are translated. For inbound packets, the destination IP address and the checksums as listed above are translated.


所以要实现一个基本```NAT```，除了最基本的地址转换外，还需要重新计算包的```校验和```。

### 4.1. iptables

虽然```NAT(RFC2663)```的描述是在末端路由器上，但我们更关注服务器，```netfilter```是```linux2.4.x+内核```的包过滤框架，在这个框架中开发的软件允许包过滤、地址转换以及包修改。而```iptables```是其用户空间的命令行工具，可以用来配置和管理上述功能。

在[netfilter-hacking-HOWTO](https://www.netfilter.org/documentation/HOWTO//netfilter-hacking-HOWTO-3.html#ss3.1)文档中描述了```netfilter```的架构，我把其中一张图拿出来如下：

	   --->PRE------>[ROUTE]--->FWD---------->POST------>
       Conntrack    |       Mangle   ^    Mangle
       Mangle       |       Filter   |    NAT (Src)
       NAT (Dst)    |                |    Conntrack
       (QDisc)      |             [ROUTE]
                    v                |
                    IN Filter       OUT Conntrack
                    |  Conntrack     ^  Mangle
                    |  Mangle        |  NAT (Dst)
                    v                |  Filter
   
由于我们把```B```充当```代理服务器```，所以当包从```A```到```B```，在图中```PRE ROUTE```即服务器路由数据包之前把包的目的地址修改成```C```的，这个过程称为```DNAT```。而在```C```回包到```B```时，再把源地址修改成```B```的，这个过程称为```SNAT```，通常在```POST ROUTE```实现。
 
**注意：**对网络不熟悉的读者可能有点不明白其中细节，在我后面的博文中会有关于网络的介绍，读者可以更关注在实现上。

在主机```C```上使用```nc```来监听```1202```端口：

	> nc -l 172.16.0.10 1202
	
在主机```B```上使用```iptables```来实现```NAT```，```220.220.0.10```所在网卡为```eth0```，```172.16.0.11```所在网卡为```eth1```：

	> sysctl -w net.ipv4.ip_forward=1 #允许内核站转发数据包，临时生效
	> sysctl -w net.ipv4.conf.eth1.proxy_arp=1 #允许内网网卡转发arp
	> iptables -t nat -A PREROUTING -i veth1 -p tcp --dport 1202 -j DNAT --to-destination 172.16.0.10:1202
	> iptables -t nat -A POSTROUTING -o veth1 -j SNAT -s 172.16.0.10 --to-source 220.220.0.10
	
紧接着在主机```A```上连接```B:1202```端口并发送```hello```：

	> nc 220.220.0.10 1202
	hello
 
以下为各个主机中相关```socket```情况：

**主机```A```：**

	Active Internet connections (servers and established)
	Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name     Timer
	tcp        0      0 192.168.0.10:60114      220.220.0.10:1202       ESTABLISHED 6945/nc              off (0.00/0/0)

在主机```A```看来就是与```B```建立的长连接。
 
**主机```B```：**

上没有这条```tcp```长连接，因为是```NAT```。

**主机```C```：**

	Active Internet connections (servers and established)
	Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name     Timer
	tcp        0      0 172.16.0.10:1202        192.168.0.10:60114      ESTABLISHED 6943/nc              off (0.00/0/0)
	
在主机```C```上能才能看到真实情况，因为我们没有对从```B```发过来的包做```SNAT```，所以在```C```看来所有的包都是与```A```直接相通的。

### 4.2. LVS

```LVS(linux virtual server)```是[章文嵩](https://baike.baidu.com/item/%E7%AB%A0%E6%96%87%E5%B5%A9)早年开发的负载均衡器，依赖于```netfilter```，所以其在内核态也需要跑一个模块，叫```ip_vs```，同样的，如果你的linux内核版本高于```2.4.x```，那么：

	> lsmod ip_vs
	
是能够找到这个内核模块的，如果没有安装，请使用```insmod```来安装刚刚找到的模块。

与```netfilter```类似，```LVS```提供了用户态工具来```ipvsadm```来控制```LVS```的行为，从负载均衡角度来说，```ipvsadm```还是比较容易使用的，我大概调研了下也是使用```netlink```来实现，但官网没有开放具体协议，所以目前使用工具。

把刚刚场景创建的```iptables ```规则全部删除，此外在上述场景再添加一台目标机器```D```：

	D机器内网地址为：172.16.1.10

我们希望```A```发给```B```的流量能够无感均衡地发送到```C```和```D```。需要注意的是由于后端的服务器已经超过1台，很多L4(传输层)和L7(应用层)的协议是有状态的，这就需要共享相同状态的流量需要发送到同一台后端服务器上，不过不用担心，```LVS```的内核模块都已经做好了

	