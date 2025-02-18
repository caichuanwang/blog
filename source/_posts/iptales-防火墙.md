---
title: iptables 防火墙
date: {{ date }}
tags: 
    - linux
categories: 
    - linux
description: IPtables是什么？防火墙又是什么？什么是四表五链？怎么理解呢？
---


### IPtables

#### 背景

在日常linux的使用中，数据流量是经过`硬件(网卡)`  ----> `linux内核空间` ------> `用户空间(应用程序)`，而如果想对流量进行控制，linux提供了命令`iptables`

在linux中，我们的常用的防火墙工具`iptables`并不是真正的防火墙，它只是一个代理，而真正有拦截和网络地址转换能力的是linux内核中的`netfilter`，这个软件位于内核中，如图

![image-20241204162130092](http://gicgo-images.oss-cn-shanghai.aliyuncs.com/img/image-20241204162130092.png)

#### 链

在流量的运行中，我们将其经过的路径称之为`链`

当客户端访问服务器的web服务时，客户端发送报文到网卡，网卡将数据发送到内核中解析进行TCP/IP 拆包解析并转发至相应的端口，如果我们想要防火墙能够达到”防火”的目的，则需要在内核中设置关卡，所有进出的报文都要通过这些关卡，经过检查后，符合放行条件的才能放行，符合阻拦条件的则需要被阻止，于是，就出现了`INPUT`关卡和`OUTPUT`关卡，而这些关卡在iptables中不被称为”关卡”,而被称为”`链`”。

而这两个链都是针对进入用户空间的，但是我们的linux服务器是具有转发功能的，也可以看成一个路由器，这样部分流量从网卡进来后可以经过内核转发出网卡，是不经过用户空间的，这就诞生了`PREROUTING 预路由`、`FORWARD 转发` 、`POSTROUTING 路由后` 这三个关卡，如图所示

![image-20241204164132569](http://gicgo-images.oss-cn-shanghai.aliyuncs.com/img/image-20241204164132569.png)

在实际的流量运行中，并不是每个流量都会经过五个链，而是根据流量的目的和功能走不同的链，比如

到本机某进程的报文：PREROUTING –> INPUT

由本机转发的报文：PREROUTING –> FORWARD –> POSTROUTING

由本机的某进程发出报文（通常为响应报文）：OUTPUT –> POSTROUTING



我们之前改成关卡为链，那为什么在`iptables`里面称之为`链`呢？我们知道，防火墙的作用在于报文的匹配`规则`,然后执行相应的`动作`，所以报文在经过关卡时，就会进行规则匹配，而且某个关卡可能有`多个规则`，那个报文会将每个规则都匹配一遍，这样多个规则就形成了一个`链`，如图

![image-20241204170722369](http://gicgo-images.oss-cn-shanghai.aliyuncs.com/img/image-20241204170722369.png)



#### 表

我们对每个链都放置了一串规则，那么也会有一些规则是相似的，比如某一类都可以修改报文，某一类都可以对IP和端口过滤，这样，我们就把具有类似功能的规则凡在一起，形成了`表`

`iptables`命令带了个`s`，说明这个命令控制了很多和IP有关的表格。那有哪些表格呢？

- `filter`表    这是最常用的表格，负责`过滤`功能，防火墙；内核模块：`iptables_filter`，在使用命令的时候，如果没有指定表格，那就表示是filter表格
- `nat`表  常用的表，   `network address translation`，`网络地址转换`功能；内核模块：`iptable_nat`
- `mangle`表  拆解报文，做出修改，并重新封装 的功能；iptable_mangle ，不常用
- `raw`表  关闭nat表上启用的连接追踪机制；iptable_raw  不常用



#### 表链关系

因为数据流量 的走向关系，某些链里注定没有某一类规则，也就是没有某个表，同时也注定某些表里没有某个链路，他们像是在不同的维度对规则的集合

![image-20241204172214028](http://gicgo-images.oss-cn-shanghai.aliyuncs.com/img/image-20241204172214028.png)

> 注意：上图中的顺序不代表现实的顺序

##### 每个链路有哪些表

PREROUTING    的规则可以存在于：raw表，mangle表，nat表。

INPUT      的规则可以存在于：mangle表，filter表，（centos7中还有nat表，centos6中没有）。

FORWARD     的规则可以存在于：mangle表，filter表。

OUTPUT     的规则可以存在于：raw表mangle表，nat表，filter表。

POSTROUTING    的规则可以存在于：mangle表，nat表。



##### 每个表存在于哪些链中

表（功能）<–>  链（钩子）：

raw   表中的规则可以被哪些链使用：PREROUTING，OUTPUT

mangle  表中的规则可以被哪些链使用：PREROUTING，INPUT，FORWARD，OUTPUT，POSTROUTING

nat   表中的规则可以被哪些链使用：PREROUTING，OUTPUT，POSTROUTING（centos7中还有INPUT，centos6中没有）

filter  表中的规则可以被哪些链使用：INPUT，FORWARD，OUTPUT



> iptables ----> 表 -------> 链 -----> 规则   
>
> iptables有不用的表，表中有不同的链，链中有多个表，表中有不同的规则



##### 总结

数据报通过防火墙的流程

![image-20241204172908959](http://gicgo-images.oss-cn-shanghai.aliyuncs.com/img/image-20241204172908959.png)



#### 规则

每个规则由匹配条件和处理动作组成

匹配条件分成基本匹配条件和拓展匹配条件

##### 匹配条件

###### 基本匹配条件

源地址Source IP，目标地址 Destination IP



除了上述的条件可以用于匹配，还有很多其他的条件可以用于匹配，这些条件泛称为扩展条件，这些扩展条件其实也是netfilter中的一部分，只是以模块的形式存在，如果想要使用这些条件，则需要依赖对应的扩展模块。

###### 拓展匹配条件

源端口Source Port, 目标端口Destination Port





##### 处理动作

处理动作在iptables中被称为`target`（这样说并不准确，我们暂且这样称呼），动作也可以分为基本动作和扩展动作。

此处列出一些常用的动作，之后的文章会对它们进行详细的示例与总结：

**ACCEPT**：允许数据包通过。

**DROP**：直接丢弃数据包，不给任何回应信息，这时候客户端会感觉自己的请求泥牛入海了，过了超时时间才会有反应。

**REJECT**：拒绝数据包通过，必要时会给数据发送端一个响应的信息，客户端刚请求就会收到拒绝的信息。

**SNAT**：源地址转换，解决内网用户用同一个公网地址上网的问题。

**MASQUERADE**：是SNAT的一种特殊形式，适用于动态的、临时会变的ip上。

**DNAT**：目标地址转换。

**REDIRECT**：在本机做端口映射。

**LOG**：在/var/log/messages文件中记录日志信息，然后将数据包传递给下一条规则，也就是说除了记录以外不对数据包做任何其他操作，仍然让下一条规则去匹配。



#### 命令

```bash
iptables (optional) (parmater)
```



```bash
-t, --table table 对指定的表 table 进行操作， table 必须是 raw， nat，filter，mangle 中的一个。如果不指定此选项，默认的是 filter 表。

# 通用匹配：源地址目标地址的匹配
-p：指定要匹配的数据包协议类型；选项：tcp udp imcp等， `all表示所有`
-s, --source [!] address[/mask] ：把指定的一个／一组地址作为源地址，按此规则进行过滤。当后面没有 mask 时，address 是一个地址，比如：192.168.1.1；当 mask 指定时，可以表示一组范围内的地址，比如：192.168.1.0/255.255.255.0。
-d, --destination [!] address[/mask] ：地址格式同上，但这里是指定地址为目的地址，按此进行过滤。
-i, --in-interface [!] <网络接口name> ：指定数据包的来自来自网络接口，比如最常见的 `eth0` 和 `lo`表示localhost 。注意：它只对 INPUT，FORWARD，PREROUTING 这三个链起作用。如果没有指定此选项， 说明可以来自任何一个网络接口。同前面类似，"!" 表示取反。
-o, --out-interface [!] <网络接口name> ：指定数据包出去的网络接口。只对 OUTPUT，FORWARD，POSTROUTING 三个链起作用。
--dport 表示端口号

# 查看管理命令
-L, --list [chain] 列出链 chain 上面的所有规则，如果没有指定链，列出表上所有链的所有规则。

# 规则管理命令
-A, --append chain rule-specification 在指定链 chain 的末尾插入指定的规则，也就是说，这条规则会被放到最后，最后才会被执行。规则是由后面的匹配来指定。
-I, --insert chain [rulenum] rule-specification 在链 chain 中的指定位置插入一条或多条规则。如果指定的规则号是1，则在链的头部插入。这也是默认的情况，如果没有指定规则号。
-D, --delete chain rule-specification -D, --delete chain rulenum 在指定的链 chain 中删除一个或多个指定规则。
-R num：Replays替换/修改第几条规则

# 链管理命令（这都是立即生效的）
-P, --policy chain target ：为指定的链 chain 设置策略 target。注意，只有内置的链才允许有策略，用户自定义的是不允许的。
-F, --flush [chain] 清空指定链 chain 上面的所有规则。如果没有指定链，清空该表上所有链的所有规则。
-N, --new-chain chain 用指定的名字创建一个新的链。
-X, --delete-chain [chain] ：删除指定的链，这个链必须没有被其它任何规则引用，而且这条上必须没有任何规则。如果没有指定链名，则会删除该表中所有非内置的链。
-E, --rename-chain old-chain new-chain ：用指定的新名字去重命名指定的链。这并不会对链内部造成任何影响。
-Z, --zero [chain] ：把指定链，或者表中的所有链上的所有计数器清零。

-j, --jump target <指定目标> ：即满足某条件时该执行什么样的动作。target 可以是内置的目标，比如 ACCEPT，也可以是用户自定义的链。
-h：显示帮助信息；

```



> 一个表中多个规则是按照number编号从1开始依次执行的，所以规则的顺序很重要
>
>  
>
> 表中的编号永远都是从1开始，没有间断的，也就说如果删除3编号，那么原来的4编号的规则编号就会变成3



##### 命令选项顺序

```bash
iptables -t 表名 < -A/I/D/R > 规则链名 [规则号] < -i/o 网卡名 > -p 协议名 < -s 源IP/源子网 > --sport 源端口 < -d 目标IP/目标子网 > --dport 目标端口 -j 动作
```

