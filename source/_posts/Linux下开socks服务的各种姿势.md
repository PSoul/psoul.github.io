---
title: Linux下开socks服务的各种姿势
date: 2017-09-11 17:34:10
tags: 渗透
---
在Linux内网渗透中，开个socks服务出来漫游内网当然是最惬意的事情了，以下姿势均为网上搜集，配合端口反弹效果更佳

首先是Python版本，当然是猪猪侠大牛的s5.py了

S5.PY：

``` python
#!/usr/bin/python 
# Filename s5.py 
# Python Dynamic Socks5 Proxy 
# Usage: python s5.py 1080 
# Background Run: nohup python s5.py 1080 & 
# Email: ringzero@557.im 

import socket, sys, select, SocketServer, struct, time 

class ThreadingTCPServer(SocketServer.ThreadingMixIn, SocketServer.TCPServer): pass
class Socks5Server(SocketServer.StreamRequestHandler): 
    def handle_tcp(self, sock, remote): 
        fdset = [sock, remote] 
        while True: 
            r, w, e = select.select(fdset, [], []) 
            if sock in r: 
                if remote.send(sock.recv(4096)) <= 0: break 
            if remote in r: 
                if sock.send(remote.recv(4096)) <= 0: break 
    def handle(self): 
        try: 
            pass # print 'from ', self.client_address nothing to do. 
            sock = self.connection 
            # 1. Version 
            sock.recv(262) 
            sock.send("\x05\x00"); 
            # 2. Request 
            data = self.rfile.read(4) 
            mode = ord(data[1]) 
            addrtype = ord(data[3]) 
            if addrtype == 1:       # IPv4 
                addr = socket.inet_ntoa(self.rfile.read(4)) 
            elif addrtype == 3:     # Domain name 
                addr = self.rfile.read(ord(sock.recv(1)[0])) 
            port = struct.unpack('>H', self.rfile.read(2)) 
            reply = "\x05\x00\x00\x01" 
            try: 
                if mode == 1:  # 1. Tcp connect 
                    remote = socket.socket(socket.AF_INET, socket.SOCK_STREAM) 
                    remote.connect((addr, port[0])) 
                    pass # print 'To', addr, port[0]  nothing do to. 
                else: 
                    reply = "\x05\x07\x00\x01" # Command not supported 
                local = remote.getsockname() 
                reply += socket.inet_aton(local[0]) + struct.pack(">H", local[1])
            except socket.error: 
                # Connection refused 
                reply = '\x05\x05\x00\x01\x00\x00\x00\x00\x00\x00' 
            sock.send(reply) 
            # 3. Transfering 
            if reply[1] == '\x00':  # Success 
                if mode == 1:    # 1. Tcp connect 
                    self.handle_tcp(sock, remote) 
        except socket.error: 
            pass #print 'error' nothing to do . 
        except IndexError: 
            pass 
def main(): 
    filename = sys.argv[0]; 
    if len(sys.argv)<2: 
        print 'usage: ' + filename + ' port' 
        sys.exit() 
    socks_port = int(sys.argv[1]);     
    server = ThreadingTCPServer(('', socks_port), Socks5Server) 
    print 'bind port: %d' % socks_port + ' ok!' 
    server.serve_forever() 
if __name__ == '__main__': 
    main() 
```

[原文链接](http://zone.wooyun.org/content/3486)

接下来是Perl版本，适用于某些情况下机器无Python的情况

默认账户名密码都是hidden，端口44269

将文件保存为socks.pl

``` perl
#!/usr/bin/perl 


$auth_enabled = 0; 
$auth_login = "hidden"; 
$auth_pass = "hidden"; 
$port = 44269; 

use IO::Socket::INET; 

$SIG{'CHLD'} = 'IGNORE'; 
$bind = IO::Socket::INET->new(Listen=>10, Reuse=>1, LocalPort=>$port) or  die "Нельзя забиндить порт $port\n"; 

while($client = $bind->accept()) { 
$client->autoflush(); 

if(fork()){ $client->close(); } 
else { $bind->close(); new_client($client); exit(); } 
} 

sub new_client { 
local $t, $i, $buff, $ord, $success; 
local $client = $_[0]; 
sysread($client, $buff, 1); 

if(ord($buff) == 5) { 
  sysread($client, $buff, 1); 
  $t = ord($buff); 

  unless(sysread($client, $buff, $t) == $t) { return; } 

  $success = 0; 
  for($i = 0; $i < $t; $i++) { 
    $ord = ord(substr($buff, $i, 1)); 
    if($ord == 0 && !$auth_enabled) { 
      syswrite($client, "\x05\x00", 2); 
      $success++; 
      break; 
    } 
    elsif($ord == 2 && $auth_enabled) { 
      unless(do_auth($client)){ return; } 
      $success++; 
      break; 
    } 
  } 

  if($success) { 
    $t = sysread($client, $buff, 3); 

    if(substr($buff, 0, 1) == '\x05') { 
      if(ord(substr($buff, 2, 1)) == 0) { 
        ($host, $raw_host) = socks_get_host($client); 
        if(!$host) {  return; } 
        ($port, $raw_port) = socks_get_port($client); 
        if(!$port) { return; } 
        $ord = ord(substr($buff, 1, 1)); 
        $buff = "\x05\x00\x00".$raw_host.$raw_port; 
        syswrite($client, $buff, length($buff)); 
        socks_do($ord, $client, $host, $port); 
      } 
    } 
  } else { syswrite($client, "\x05\xFF", 2); }; 
} 
$client->close(); 
} 

sub do_auth { 
local $buff, $login, $pass; 
local $client = $_[0]; 

syswrite($client, "\x05\x02", 2); 
sysread($client, $buff, 1); 

if(ord($buff) == 1) { 
  sysread($client, $buff, 1); 
  sysread($client, $login, ord($buff)); 
  sysread($client, $buff, 1); 
  sysread($client, $pass, ord($buff)); 

  if($login eq $auth_login && $pass eq $auth_pass) { 
    syswrite($client, "\x05\x00", 2); 
    return 1; 
  } else { syswrite($client, "\x05\x01", 2); } 
} 

$client->close(); 
return 0; 
} 

sub socks_get_host { 
local $client = $_[0]; 
local $t, $ord, $raw_host; 
local $host = ""; 

sysread($client, $t, 1); 
$ord = ord($t); 
if($ord == 1) { 
  sysread($client, $raw_host, 4); 
  @host = $raw_host =~ /(.)/g; 
  $host = ord($host[0]).".".ord($host[1]).".".ord($host[2]).".".ord($host[3]); 
} elsif($ord == 3) { 
  sysread($client, $raw_host, 1); 
  sysread($client, $host, ord($raw_host)); 
  $raw_host .= $host; 
} elsif($ord == 4) { 
  #ipv6 - not supported 
} 

return ($host, $t.$raw_host); 
} 

sub socks_get_port { 
local $client = $_[0]; 
local $raw_port, $port; 
sysread($client, $raw_port, 2); 
$port = ord(substr($raw_port, 0, 1)) << 8 | ord(substr($raw_port, 1, 1)); 
return ($port, $raw_port); 
} 

sub socks_do { 
local($t, $client, $host, $port) = @_; 

if($t == 1) { socks_connect($client, $host, $port); } 
elsif($t == 2) { socks_bind($client, $host, $port); } 
elsif($t == 3) { socks_udp_associate($client, $host, $port); } 
else { return 0; } 

return 1; 
} 

sub socks_connect { 
my($client, $host, $port) = @_; 
my $target = IO::Socket::INET->new(PeerAddr => $host, PeerPort => $port, Proto => 'tcp', Type => SOCK_STREAM); 

unless($target) { return; } 

$target->autoflush(); 
while($client || $target) { 
  my $rin = ""; 
  vec($rin, fileno($client), 1) = 1 if $client; 
  vec($rin, fileno($target), 1) = 1 if $target; 
  my($rout, $eout); 
  select($rout = $rin, undef, $eout = $rin, 120); 
  if (!$rout  &&  !$eout) { return; } 
  my $cbuffer = ""; 
  my $tbuffer = ""; 

  if ($client && (vec($eout, fileno($client), 1) || vec($rout, fileno($client), 1))) { 
    my $result = sysread($client, $tbuffer, 1024); 
    if (!defined($result) || !$result) { return; } 
  } 

  if ($target  &&  (vec($eout, fileno($target), 1)  || vec($rout, fileno($target), 1))) { 
    my $result = sysread($target, $cbuffer, 1024); 
    if (!defined($result) || !$result) { return; } 
    } 

  if ($fh  &&  $tbuffer) { print $fh $tbuffer; } 

  while (my $len = length($tbuffer)) { 
    my $res = syswrite($target, $tbuffer, $len); 
    if ($res > 0) { $tbuffer = substr($tbuffer, $res); } else { return; } 
  } 

  while (my $len = length($cbuffer)) { 
    my $res = syswrite($client, $cbuffer, $len); 
    if ($res > 0) { $cbuffer = substr($cbuffer, $res); } else { return; } 
  } 
} 
} 

sub socks_bind { 
my($client, $host, $port) = @_; 
} 

sub socks_udp_associate { 
my($client, $host, $port) = @_; 
}
```
