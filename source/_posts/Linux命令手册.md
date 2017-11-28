---
title: linux下常用手册
date: 2017-09-12 10:05:36
tags:
	- Linux
	- 运维
categories: 运维
---

各种linux命令杂项备忘

<!-- more -->


### 搜索文件内容
grep -i 'xx' *

-i为不区分大小写

---

### Linux下如何查看版本信息

Linux下如何查看版本信息， 包括位数、版本信息以及CPU内核信息、CPU具体型号等等，整个CPU信息一目了然。

 
　　1、# uname －a   （Linux查看版本当前操作系统内核信息）
 
　　Linux localhost.localdomain 2.4.20-8 #1 Thu Mar 13 17:54:28 EST 2003 i686 athlon i386 GNU/Linux
 
　　2、# cat /proc/version （Linux查看当前操作系统版本信息）
 
      Linux version 2.4.20-8 (bhcompile@porky.devel.redhat.com)
      (gcc version 3.2.2 20030222 (Red Hat Linux 3.2.2-5)) #1 Thu Mar 13 17:54:28 EST 2003
 
　　3、# cat /etc/issue  或cat /etc/redhat-release（Linux查看版本当前操作系统发行版信息）
 
　　Red Hat Linux release 9 (Shrike)

　　4、# cat /proc/cpuinfo （Linux查看cpu相关信息，包括型号、主频、内核信息等）

	processor        : 0
	vendor_id         : AuthenticAMD
	cpu family        : 15
	model             : 1
	model name      : AMD A4-3300M APU with Radeon(tm) HD Graphics
	stepping         : 0
	cpu MHz          : 1896.236
	cache size       : 1024 KB
	fdiv_bug         : no
	hlt_bug          : no
	f00f_bug        : no
	coma_bug      : no
	fpu                : yes
	fpu_exception   : yes
	cpuid level      : 6
	wp                : yes
	flags             : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 syscall mmxext lm 3dnowext 3dnow
	bogomips      : 3774.87

　　5、# getconf LONG_BIT  （Linux查看版本说明当前CPU运行在32bit模式下， 但不代表CPU不支持64bit）
 
　　32
 
　　6、# lsb_release -a

### Ubuntu一键PPTP VPN

    wget http://www.ilucong.net/file/pptp-debian.sh && sh pptp-debian.sh  

### Ubuntu CentOS 一键L2TP

``` bash
#!/bin/bash

if [ $(id -u) != "0" ]; then
    printf "Error: You must be root to run this tool!\n"
    exit 1
fi
clear
printf "
####################################################
#                                                  #
# This is a Shell-Based tool of l2tp installation  #
# Version: 1.3                                     #
# Author: Hong Chen                                  #
# For Ubuntu 32bit and 64bit                       #
#                                                  #
####################################################
"
vpsip=`ifconfig  | grep 'inet addr:'| grep -v '127.0.0.1' | cut -d: -f2 | awk 'NR==1 { print $1}'`

iprange="10.0.99"
echo "Please input IP-Range:"
read -p "(Default Range: 10.0.99):" iprange
if [ "$iprange" = "" ]; then
    iprange="10.0.99"
fi

mypsk="vpsyou.com"
echo "Please input PSK:"
read -p "(Default PSK: vpsyou.com):" mypsk
if [ "$mypsk" = "" ]; then
    mypsk="vpsyou.com"
fi

clear
get_char()
{
SAVEDSTTY=`stty -g`
stty -echo
stty cbreak
dd if=/dev/tty bs=1 count=1 2> /dev/null
stty -raw
stty echo
stty $SAVEDSTTY
}
echo ""
echo "ServerIP:"
echo "$vpsip"
echo ""
echo "Server Local IP:"
echo "$iprange.1"
echo ""
echo "Client Remote IP Range:"
echo "$iprange.2-$iprange.254"
echo ""
echo "PSK:"
echo "$mypsk"
echo ""
echo "Press any key to start..."
char=`get_char`
clear

apt-get -y update
apt-get -y upgrade
apt-get -y install libgmp3-dev bison flex libpcap-dev ppp iptables make gcc lsof vim
mkdir /ztmp
mkdir /ztmp/l2tp
cd /ztmp/l2tp
apt-get install openswan
rm -rf /etc/ipsec.conf
touch /etc/ipsec.conf
cat >>/etc/ipsec.conf<<EOF
config setup
    nat_traversal=yes
    virtual_private=%v4:10.0.0.0/8,%v4:192.168.0.0/16,%v4:172.16.0.0/12
    oe=off
    protostack=netkey

conn L2TP-PSK-NAT
    rightsubnet=vhost:%priv
    also=L2TP-PSK-noNAT

conn L2TP-PSK-noNAT
    authby=secret
    pfs=no
    auto=add
    keyingtries=3
    rekey=no
    ikelifetime=8h
    keylife=1h
    type=transport
    left=$vpsip
    leftprotoport=17/1701
    right=%any
    rightprotoport=17/%any
EOF
cat >>/etc/ipsec.secrets<<EOF
$vpsip %any: PSK "$mypsk"
EOF
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf
sed -i 's/#net.ipv6.conf.all.forwarding=1/net.ipv6.conf.all.forwarding=1/g' /etc/sysctl.conf
sysctl -p
iptables --table nat --append POSTROUTING --jump MASQUERADE
for each in /proc/sys/net/ipv4/conf/*
do
echo 0 > $each/accept_redirects
echo 0 > $each/send_redirects
done
/etc/init.d/ipsec restart
ipsec verify
cd /ztmp/l2tp
wget http://downloads.sourceforge.net/project/rp-l2tp/rp-l2tp/0.4/rp-l2tp-0.4.tar.gz
tar zxvf rp-l2tp-0.4.tar.gz
cd rp-l2tp-0.4
./configure
make
cp handlers/l2tp-control /usr/local/sbin/
mkdir /var/run/xl2tpd/
ln -s /usr/local/sbin/l2tp-control /var/run/xl2tpd/l2tp-control
cd /ztmp/l2tp
apt-get install xl2tpd
mkdir /etc/xl2tpd
rm -rf /etc/xl2tpd/xl2tpd.conf
touch /etc/xl2tpd/xl2tpd.conf
cat >>/etc/xl2tpd/xl2tpd.conf<<EOF
[global]
ipsec saref = yes
[lns default]
ip range = $iprange.2-$iprange.254
local ip = $iprange.1
refuse chap = yes
refuse pap = yes
require authentication = yes
ppp debug = yes
pppoptfile = /etc/ppp/options.xl2tpd
length bit = yes
EOF
rm -rf /etc/ppp/options.xl2tpd
touch /etc/ppp/options.xl2tpd
cat >>/etc/ppp/options.xl2tpd<<EOF
require-mschap-v2
ms-dns 8.8.8.8
ms-dns 8.8.4.4
asyncmap 0
auth
crtscts
lock
hide-password
modem
debug
name l2tpd
proxyarp
lcp-echo-interval 30
lcp-echo-failure 4
EOF
cat >>/etc/ppp/chap-secrets<<EOF
test l2tpd test123 *
EOF
touch /usr/bin/zl2tpset
echo "#/bin/bash" >>/usr/bin/zl2tpset
echo "for each in /proc/sys/net/ipv4/conf/*" >>/usr/bin/zl2tpset
echo "do" >>/usr/bin/zl2tpset
echo "echo 0 > \$each/accept_redirects" >>/usr/bin/zl2tpset
echo "echo 0 > \$each/send_redirects" >>/usr/bin/zl2tpset
echo "done" >>/usr/bin/zl2tpset
chmod +x /usr/bin/zl2tpset
iptables --table nat --append POSTROUTING --jump MASQUERADE
zl2tpset
xl2tpd
cat >>/etc/rc.local<<EOF
iptables --table nat --append POSTROUTING --jump MASQUERADE
/etc/init.d/ipsec restart
/usr/bin/zl2tpset
/usr/local/sbin/xl2tpd
EOF
clear
ipsec verify
printf "
####################################################
#                                                  #
# This is a Shell-Based tool of l2tp installation  #
# Version: 1.3                                     #
# Author: Zed Lau                                  #
# For Ubuntu 32bit and 64bit                       #
#                                                  #
####################################################
if there are no [FAILED] above, then you can
connect to your L2TP VPN Server with the default
user/pass below:

ServerIP:$vpsip
username:test
password:test123
PSK:$mypsk

"
```

