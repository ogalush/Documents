# HAProxy入門
HAProxyが数年前から流行っているので実際に触ってみる。  
環境: ubuntu16.04  
構成:
```
[client]--(http://192.168.0.101/)-->[haproxy]--+-->[slave-1(10.0.0.5)]
                                               +-->[slave-2(10.0.0.10)]
```
参考:
* [HAProxyを使い始めてみる](http://qiita.com/saka1_p/items/3634ba70f9ecd74b0860)
* [HAProxy Configuration Manual](http://cbonte.github.io/haproxy-dconv/configuration-1.6.html#2.4)

## インストール
apt-cache searchでhaproxyに関連するパッケージを検索。以下の3つを入れてみる。
```
$ apt-cache search haproxy
$ sudo apt-get -y install haproxy haproxyctl vim-haproxy
---
haproxy - fast and reliable load balancing reverse proxy
haproxyctl - Utility to manage HAProxy
vim-haproxy - syntax highlighting for HAProxy configuration files
---
```

## 設定
haproxy.cfgを設定する.  
セクションがglobal, frontend, backendの3つある.  
globalは全体設定で、frontendは入口側の設定、backendはメンバ側の設定となる.
```
$ sudo cp -pv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.def
$ sudo vi /etc/haproxy/haproxy.cfg
----
追記
## TEST Port80 Members
frontend http-in
  bind *:80
  default_backend servers

backend servers
  server slave-1 10.0.0.5:80 maxconn 32
  server slave-2 10.0.0.10:80 maxconn 32
----

$ sudo haproxy -f /etc/haproxy/haproxy.cfg -c
Configuration file is valid
~~~ valid=有効なので設定内容はOK.
```

## 反映
### rsyslog
HAProxyのログ出力としてrsyslogを使用しているので反映する.  
(/etc/rsyslog.d/49-haproxy.conf)
```
$ sudo service rsyslog restart
```

### HAProxy
```
$ sudo service haproxy restart
$ sudo service haproxy status
sudo: unable to resolve host haproxy-test
● haproxy.service - HAProxy Load Balancer
   Loaded: loaded (/lib/systemd/system/haproxy.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2017-01-09 01:49:50 JST; 2s ago
     Docs: man:haproxy(1)
           file:/usr/share/doc/haproxy/configuration.txt.gz
  Process: 2355 ExecStartPre=/usr/sbin/haproxy -f ${CONFIG} -c -q (code=exited, status=0/SUCCESS)
 Main PID: 2358 (haproxy-systemd)
    Tasks: 3
   Memory: 1.2M
      CPU: 5ms
   CGroup: /system.slice/haproxy.service
           ├─2358 /usr/sbin/haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
           ├─2361 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds
           └─2362 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds

Jan 09 01:49:50 haproxy-test systemd[1]: Starting HAProxy Load Balancer...
Jan 09 01:49:50 haproxy-test systemd[1]: Started HAProxy Load Balancer.
Jan 09 01:49:50 haproxy-test haproxy-systemd-wrapper[2358]: haproxy-systemd-wrapper: executing /usr/sbin/haproxy -f /etc/ha
Jan 09 01:49:50 haproxy-test haproxy[2361]: Proxy http-in started.
Jan 09 01:49:50 haproxy-test haproxy[2361]: Proxy http-in started.
Jan 09 01:49:50 haproxy-test haproxy[2361]: Proxy servers started.
Jan 09 01:49:50 haproxy-test haproxy[2361]: Proxy servers started.
-----
```

## slave設定
slaveサーバ全台にapache2を入れる.
```
$ sudo apt-get -y install apache2
$ netstat -ln  |grep :80
tcp6       0      0 :::80                   :::*                    LISTEN
```

## 確認
ログにservers/slave1, servers/slave2と出力されているので振り分けられていることが分かる.
```
$ tail -F /var/log/haproxy.log
----
Jan  9 02:06:46 haproxy-test haproxy[2362]: 192.168.0.12:51386 [09/Jan/2017:02:06:46.226] http-in servers/slave-2 143/0/0/1/144 200 3469 - - ---- 1/1/0/1/0 0/0 "GET / HTTP/1.1"
Jan  9 02:06:46 haproxy-test haproxy[2362]: 192.168.0.12:51386 [09/Jan/2017:02:06:46.371] http-in servers/slave-1 19/0/0/1/20 304 125 - - ---- 1/1/0/1/0 0/0 "GET /icons/ubuntu-logo.png HTTP/1.1"
----
```

## 残課題
* slaveサーバの死活監視/サービスイン/サービスアウト  
→ synapseで利用可能なように見える.  
[Synapse と Serf でサービスディスカバリ](http://blog.ryotarai.info/blog/2014/04/01/service-discovery-by-syanpse-with-serf/)
* OpenStack(Neutron LBAAS)で簡単に出来そうにも見える.  
[Load Balancer as a Service (LBaaS)](http://docs.openstack.org/liberty/ja/networking-guide/adv-config-lbaas.html)
