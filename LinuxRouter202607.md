## 概要
NEC IX2215 Routerを経由するとホームページを開くタイミングの微妙の待ちが気になるため入れ替えを考えた.  
今回は実験がてら、2026.6に退役させた ECS Livaを利用してLinuxRouterを試す.

## 構成
### 接続構成
```
[マンションインターネット]
↑
内蔵NIC 1000Base-T
[ECS Liva]
Baffalo LUA-U3-A2G/C (USB3.x) 2.5GBase-T
↑
[Planex FX2G-08EM2]
↑
SOLO10-TB3 2.5GBase-T
[MacBook Pro]
```
LAN: 192.168.3.0/24  
WAN: 100.64.0.0/22 + IPv6 (ndProxy + radvd)  

### 環境
ECS Liva  
OS: Debian 13  
→ メモリ 2GBのマシンの都合で入るOSがこれだけだった。

## 大まかな流れ
* USB NIC認識
* 2.5Gbpsリンクアップ
* WAN DHCP取得
* IPv4 NAT
* IPv6 RA受信・LAN配布
* DNSキャッシュ
* IX2215と入れ替えて体感比較


## 構築
### USB NIC認識
NICをUSBに差す
```
----
$ sudo lsusb -t
/:  Bus 001.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/6p, 480M
/:  Bus 002.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/1p, 5000M
    |__ Port 001: Dev 002, If 0, Class=Vendor Specific Class, Driver=r8152, 5000M
----
→ 速い方のHubに刺さってるためOK.

----
$ sudo lsusb
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 002 Device 002: ID 0bda:8156 Realtek Semiconductor Corp. USB 10/100/1G/2.5G LAN
----
→ NICデバイスを認識している.

----
$ sudo ethtool -i enx50c4dd1d1f9b
driver: r8152
version: v1.12.13
firmware-version: rtl8156a-2 v2 04/27/23
expansion-rom-version: 
bus-info: usb-0000:00:14.0-1
supports-statistics: yes
supports-test: no
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: no
----
→ ドライバも新しそう。
```

### 2.5Gbpsリンクアップ
```
$ cp -pv /etc/network/interfaces ~
$ sudo vim /etc/network/interfaces
----
ogalush@liva:~$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

## WAN
auto enp3s0
iface enp3s0 inet dhcp
iface enp3s0 inet6 auto

## LAN
auto enx50c4dd1d1f9b
iface enx50c4dd1d1f9b inet static
   address 192.168.3.254
   netmask 255.255.255.0
   gateway 192.168.3.254
   dns-nameservers 192.168.3.254 8.8.8.8
iface enp3s0 inet6 auto

ogalush@liva:~$
----

$ sudo service networking status
$ sudo service networking restart
→ IPアドレスが変わるため、接続が切れる.
```

再接続する。
```
$ ssh -A ogalush@192.168.3.254
$ sudo reboot
→ netstat -rnする古いNICのルーティングテーブルも残っているため。
```

### WAN DHCP取得
WAN側のNICへWANのケーブルを差す.
IPアドレスを拾えることが出来ればOK.
```
$ ip addr show enp3s0
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether c0:3f:d5:4b:b5:5f brd ff:ff:ff:ff:ff:ff
    altname enxc03fd54bb55f
    inet 100.64.1.41/22 brd 100.64.3.255 scope global dynamic noprefixroute enp3s0
       valid_lft 86248sec preferred_lft 75448sec
    inet6 240b:10:a460:6100:ac90:1615:6d27:8319/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 2591844sec preferred_lft 604644sec
    inet6 fe80::3c7:bbfc:6464:31b2/64 scope link 
       valid_lft forever preferred_lft forever

$ sudo ethtool enp3s0
[sudo] password for ogalush: 
Settings for enp3s0:
        Supported ports: [ TP    MII ]
        Supported link modes:   10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
        Supported pause frame use: Symmetric Receive-only
        Supports auto-negotiation: Yes
        Supported FEC modes: Not reported
        Advertised link modes:  10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
        Advertised pause frame use: Symmetric Receive-only
        Advertised auto-negotiation: Yes
        Advertised FEC modes: Not reported
        Link partner advertised link modes:  10baseT/Half 10baseT/Full
                                             100baseT/Half 100baseT/Full
                                             1000baseT/Full
        Link partner advertised pause frame use: Symmetric
        Link partner advertised auto-negotiation: Yes
        Link partner advertised FEC modes: Not reported
        Speed: 1000Mb/s
        Duplex: Full
        Auto-negotiation: on
        master-slave cfg: preferred slave
        master-slave status: slave
        Port: Twisted Pair
        PHYAD: 0
        Transceiver: external
        MDI-X: Unknown
        Supports Wake-on: pumbg
        Wake-on: d
        Link detected: yes
$
→ IPアドレスを拾えているためOK.

ogalush@liva:~$ netstat -rn
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
0.0.0.0         100.64.1.1      0.0.0.0         UG        0 0          0 enp3s0
100.64.0.0      0.0.0.0         255.255.252.0   U         0 0          0 enp3s0
172.17.0.0      0.0.0.0         255.255.0.0     U         0 0          0 docker0
192.168.3.0     0.0.0.0         255.255.255.0   U         0 0          0 enx50c4dd1d1f9b
ogalush@liva:~$ 
→ ルーティングテーブルも 100.64.0.0/22 の方がExternalとなっているためOK.
```


### IPv4 NAT
パケット転送設定
```
$ echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-router.conf
net.ipv4.ip_forward=1
$ cat /etc/sysctl.d/99-router.conf 
net.ipv4.ip_forward=1
$ sudo systemctl restart systemd-sysctl
```

NAT設定
iptables後継のnftableを利用する。
```
元々入れていたdockerを一度止める。
検証用途なので一時的に。
----
ogalush@liva:~$ sudo systemctl stop docker
Stopping 'docker.service', but its triggering units are still active:
docker.socket
ogalush@liva:~$ sudo systemctl stop docker.socket
ogalush@liva:~$ sudo systemctl disable docker docker.socket
Synchronizing state of docker.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install disable docker
Removed '/etc/systemd/system/multi-user.target.wants/docker.service'.
Removed '/etc/systemd/system/sockets.target.wants/docker.socket'.
ogalush@liva:~$ 
$ sudo reboot
----
```

NAT設定
・Before
```
$ sudo nft list ruleset
→ ルールなし。

$ ping -I enx50c4dd1d1f9b 8.8.8.8
PING 8.8.8.8 (8.8.8.8) from 192.168.3.254 enx50c4dd1d1f9b: 56(84) bytes of data.
From 192.168.3.254 icmp_seq=1 Destination Host Unreachable
ping: sendmsg: No route to host
From 192.168.3.254 icmp_seq=2 Destination Host Unreachable
From 192.168.3.254 icmp_seq=3 Destination Host Unreachable
From 192.168.3.254 icmp_seq=5 Destination Host Unreachable
From 192.168.3.254 icmp_seq=6 Destination Host Unreachable
From 192.168.3.254 icmp_seq=7 Destination Host Unreachable
^C
--- 8.8.8.8 ping statistics ---
8 packets transmitted, 0 received, +6 errors, 100% packet loss, time 7137ms
pipe 4
$
→ まだ届かない.
```

・rule設定
nftablesファイルに入れて一括設定するやり方でいく。
```
$ sudo cp -pv /etc/nftables.conf ~
'/etc/nftables.conf' -> '/home/ogalush/nftables.conf'


ogalush@liva:~$ cat /etc/nftables.conf
#!/usr/sbin/nft -f

flush ruleset

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        oifname "enp3s0" masquerade
    }
}

table inet filter {
    chain input {
        type filter hook input priority filter;
        policy accept;
    }

    chain forward {
        type filter hook forward priority filter;
        policy drop;

        ct state established,related accept
        iifname "enx50c4dd1d1f9b" oifname "enp3s0" accept
    }

    chain output {
        type filter hook output priority filter;
        policy accept;
    }
}
ogalush@liva:~$
```

設定反映
```
$ sudo nft -f /etc/nftables.conf
$ sudo nft list ruleset
table ip nat {
        chain postrouting {
                type nat hook postrouting priority srcnat; policy accept;
                oifname "enp3s0" masquerade
        }
}
table inet filter {
        chain input {
                type filter hook input priority filter; policy accept;
        }

        chain forward {
                type filter hook forward priority filter; policy drop;
                ct state established,related accept
                iifname "enx50c4dd1d1f9b" oifname "enp3s0" accept
        }

        chain output {
                type filter hook output priority filter; policy accept;
        }
}
$
```

永続化
```
$ sudo systemctl enable nftables
Created symlink '/etc/systemd/system/sysinit.target.wants/nftables.service' → '/usr/lib/systemd/system/nftables.service'.
$ sudo systemctl restart nftables
$ sudo systemctl status nftables
```

### IPv6 RA受信・LAN配布
状態確認
```
$ ip -6 addr show enp3s0
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    altname enxc03fd54bb55f
    inet6 240b:10:xx::xx/64 scope global dynamic mngtmpaddr proto kernel_ra 
       valid_lft 2591847sec preferred_lft 604647sec
    inet6 240b:10:xx::xx/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 2591015sec preferred_lft 603815sec
    inet6 fe80::3c7:xx:xx/64 scope link 
       valid_lft forever preferred_lft forever

$ ip -6 route
240b:10:xx::/64 dev enp3s0 proto kernel metric 256 expires 2591743sec pref medium
240b:10:xx::/64 dev enp3s0 proto ra metric 1002 pref medium
fe80::/64 dev enp3s0 proto kernel metric 256 pref medium
fe80::/64 dev enx50c4dd1d1f9b proto kernel metric 256 pref medium
default via fe80::77:696a:2000:0 dev enp3s0 proto ra metric 1002 pref medium
default via fe80::77:696a:2000:0 dev enp3s0 proto ra metric 1024 expires 1543sec hoplimit 64 pref medium
```
→ WAN側で、/64で拾っているので、ndproxyでRAを仲介させる。

パケット転送設定
```
$ cat /etc/sysctl.d/99-ipv6-router.conf
net.ipv6.conf.all.forwarding = 1
net.ipv6.conf.enp3s0.accept_ra = 2

$ sudo systemctl restart systemd-sysctl

$ /usr/sbin/sysctl net.ipv6.conf.all.forwarding
net.ipv6.conf.all.forwarding = 1
$ /usr/sbin/sysctl net.ipv6.conf.enp3s0.accept_ra
net.ipv6.conf.enp3s0.accept_ra = 2

$ ip -6 addr show enp3s0
```
→ 「accept_ra=2」は、Routerへ昇格してもRAを受ける設定。
これがないとRouterAdvertizedを受けない様になるらしい。

パケット転送設定確認
```
$ ip -6 addr show enp3s0
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    altname enxc03fd54bb55f
    inet6 240b:10::xx/64 scope global dynamic mngtmpaddr proto kernel_ra 
       valid_lft 2591543sec preferred_lft 604343sec
    inet6 240b:10::xx/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 2590302sec preferred_lft 603102sec
    inet6 fe80::3c7:bbfc:6464:31b2/64 scope link 
       valid_lft forever preferred_lft forever

$ ip -6 route
240b:10::/64 dev enp3s0 proto kernel metric 256 expires 2591537sec pref medium
240b:10::/64 dev enp3s0 proto ra metric 1002 pref medium
fe80::/64 dev enp3s0 proto kernel metric 256 pref medium
fe80::/64 dev enx50c4dd1d1f9b proto kernel metric 256 pref medium
default via fe80::77:696a:2000:0 dev enp3s0 proto ra metric 1002 pref medium
default via fe80::77:696a:2000:0 dev enp3s0 proto ra metric 1024 expires 1337sec hoplimit 64 pref medium
ogalush@liva:~$ 
```

ndProxyインストール
```
$ sudo apt install -y ndppd
$ cat /etc/ndppd.conf
proxy enp3s0 {
    router yes
    timeout 500

    rule 240b:10:a460:6100::/64 {
        iface enx50c4dd1d1f9b
    }
}
```

radvdインストール
```
$ sudo apt install -y radvd
$ cat /etc/radvd.conf
cat: /etc/radvd.conf: No such file or directory
```

radvd.conf設定
★設定に気をつけること。間違えると上流へ迷惑をかけるため。
```
$ cat /etc/radvd.conf
interface enx50c4dd1d1f9b {
    AdvSendAdvert on;

    prefix 240b:10:a460:6100::/64 {
        AdvOnLink on;
        AdvAutonomous on;
    };
};
```
→ 一旦Prefix固定で設定する。
Prefix自動読み込みはあとでも出来るらしい。

ndProxy, radvd起動
```
$ sudo systemctl enable --now ndppd
$ sudo systemctl enable --now radvd
```

このままだとルーティングテーブルのWANが優先されるため、metricを入れたstatic routeを入れてローカルへ向ける。
```
$ sudo ip -6 route add 240b:10:a460:6100::/64 dev enx50c4dd1d1f9b metric 10
$ ip -6 route show |head -n 3
240b:10:a460:6100::/64 dev enx50c4dd1d1f9b metric 10 pref medium
240b:10:a460:6100::/64 dev enp3s0 proto kernel metric 256 expires 2591834sec pref medium
240b:10:a460:6100::/64 dev enp3s0 proto ra metric 1002 pref medium

永続化。
----
$ cat /etc/network/interfaces
...
## WAN
auto enp3s0
iface enp3s0 inet dhcp
iface enp3s0 inet6 auto
+  post-up ip -6 route replace 240b:10:a460:6100::/64 dev enx50c4dd1d1f9b metric 10
...
```

### DNSキャッシュ
軽量高速DNS Resolver のunboundを入れる。
```
$ sudo apt install -y unbound
$ sudo vim /etc/unbound/unbound.conf.d/router.conf
$ cat /etc/unbound/unbound.conf.d/router.conf
server:
    interface: 127.0.0.1
    interface: 192.168.3.254

    access-control: 127.0.0.0/8 allow
    access-control: 192.168.3.0/24 allow

    verbosity: 1

    prefetch: yes
    cache-min-ttl: 300
    cache-max-ttl: 600

    rrset-cache-size: 64m
    msg-cache-size: 32m

    hide-identity: yes
    hide-version: yes

forward-zone:
    name: "."
    forward-addr: 1.1.1.1
    forward-addr: 1.0.0.1

remote-control:
    control-enable: yes
$

$ sudo unbound-checkconf
unbound-checkconf: no errors in /etc/unbound/unbound.conf

$ sudo systemctl enable unbound
Synchronizing state of unbound.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable unbound

$ sudo systemctl restart unbound
$ sudo systemctl status unbound
● unbound.service - Unbound DNS server
     Loaded: loaded (/usr/lib/systemd/system/unbound.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-07-04 04:51:07 JST; 5s ago

$ sudo unbound-control status
version: 1.22.0
verbosity: 1
threads: 1
modules: 3 [ subnetcache validator iterator ]
uptime: 13 seconds
options: reuseport control(namedpipe)
unbound (pid 1573) is running...

DNSキャッシュ利用状況の確認は以下で可能。
$ sudo unbound-control stats_noreset | grep cache
thread0.num.cachehits=0
thread0.num.cachemiss=2
total.num.cachehits=0
total.num.cachemiss=2
```

確認
```
$ dig +short @192.168.3.254 google.com
192.178.230.101
192.178.230.100
192.178.230.113
192.178.230.139
192.178.230.138
```
→ 引けるためOK.


### DHCPv4 サーバ
インストール
```
$ sudo apt install -y isc-dhcp-server
```

設定
```
$ sudo cp -vp /etc/default/isc-dhcp-server ~
'/etc/default/isc-dhcp-server' -> '/home/ogalush/isc-dhcp-server'

$ sudo vim /etc/default/isc-dhcp-server
----
- INTERFACESv4=""
+ INTERFACESv4="enx50c4dd1d1f9b"
INTERFACESv6=""
----

$ sudo cp -vp /etc/dhcp/dhcpd.conf ~
'/etc/dhcp/dhcpd.conf' -> '/home/ogalush/dhcpd.conf'

$ sudo vim /etc/dhcp/dhcpd.conf
----
authoritative;

default-lease-time 86400;
max-lease-time 604800;

option domain-name "localdomain";

subnet 192.168.3.0 netmask 255.255.255.0 {
    range 192.168.3.10 192.168.3.99;
    option routers 192.168.3.254;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.168.3.255;
    option domain-name-servers 192.168.3.254;
}
----

Syntaxチェック
----
~$ sudo dhcpd -t
Internet Systems Consortium DHCP Server 4.4.3-P1
Copyright 2004-2022 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/
Config file: /etc/dhcp/dhcpd.conf
Database file: /var/lib/dhcp/dhcpd.leases
PID file: /var/run/dhcpd.pid
----
```

起動
```
$ sudo systemctl enable isc-dhcp-server
$ sudo systemctl restart isc-dhcp-server
```

クライアント側の確認
```
% ipconfig getpacket en7
op = BOOTREPLY
htype = 1
flags = 0x0
hlen = 6
hops = 0
xid = 0xbd21277d
secs = 9
ciaddr = 0.0.0.0
yiaddr = 192.168.3.10
siaddr = 0.0.0.0
giaddr = 0.0.0.0
chaddr = 0:30:93:10:31:67
sname = 
file = 
options:
Options count is 8
dhcp_message_type (uint8): ACK 0x5
server_identifier (ip): 192.168.3.254
lease_time (uint32): 0x15180
subnet_mask (ip): 255.255.255.0
router (ip_mult): {192.168.3.254}
domain_name_server (ip_mult): {192.168.3.254}
domain_name (string): localdomain
end (none): 
```

(8) ポート開放
WireGuard用にポート開放しているので開ける。
```
$ cat /etc/nftables.conf
...
    chain input {
        type filter hook input priority filter;
        policy accept;
+        ct state established,related accept
+        iifname "lo" accept
+        ip6 daddr 240b:10:a460:6100:32c5:99ff:fe29:d3f8 udp dport 51820 accept
    }
...
    chain forward {
        type filter hook forward priority filter;
        policy drop;
        ct state established,related accept
+        iifname "enp3s0" oifname "enx50c4dd1d1f9b" udp dport 51820 accept
        iifname "enx50c4dd1d1f9b" oifname "enp3s0" accept
    }
```

反映
```
$ sudo nft -f /etc/nftables.conf
```

### IX2215と入れ替えて体感比較
Webページを開いたりしてレスポンスや挙動を確認する。
家庭内LANを使うだけであればレスポンスは良くなった。  
ただし、企業利用などの規模だとLinuxRouterの処理力が不足気味にも見えるの相応の使い勝手という印象。  

以上
