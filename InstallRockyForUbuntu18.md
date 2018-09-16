# Install OpenStack Rocky on Ubuntu 18.04
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.0.200(192.168.0.200)  
設定ファイル: [URL](URL)
```
$ uname -a
Linux ryunosuke 4.15.0-34-generic #37-Ubuntu SMP Mon Aug 27 15:21:48 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

# Host networking
## Configure network interfaces
127.0.1.1 がホスト名に紐づいているので、実際のIPアドレスへ置換する.
```
$ sudo cp -pv /etc/hosts /tmp/hosts
[sudo] password for ogalush: 
'/etc/hosts' -> '/tmp/hosts'
$ sudo vim /etc/hosts
$ diff -u /tmp/hosts /etc/hosts |egrep '^(\+|\-)'
--- /tmp/hosts  2018-09-16 14:40:00.059247346 +0900
+++ /etc/hosts  2018-09-16 15:01:06.148425262 +0900
-127.0.1.1      ryunosuke
+192.168.0.200  ryunosuke
```

# Network Time Protocol (NTP)
```
$ sudo dpkg -r ntp
$ sudo apt -y install chrony
$ dpkg -l |grep chrony
ii  chrony 3.2-4ubuntu4.2 amd64 Versatile implementation of the Network Time Protocol
$ sudo cp -ar /etc/chrony ~
$ sudo vim /etc/chrony/chrony.conf
$ diff -urBb ~/chrony /etc/chrony 2> /dev/null |egrep '^(\+|\-)'
--- /home/ogalush/chrony/chrony.conf    2018-08-20 15:00:29.000000000 +0900
+++ /etc/chrony/chrony.conf     2018-09-16 15:19:47.082452675 +0900
+
+server ntp.nict.jp iburst
+allow 10.0.0.0/24
+allow 192.168.0.0/24
----

$ sudo service chrony restart
$ sudo service chrony status
● chrony.service - chrony, an NTP client/server
   Loaded: loaded (/lib/systemd/system/chrony.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-09-16 15:20:50 JST; 10s ago
---

$ chronyc sources
210 Number of sources = 9
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^- chilipepper.canonical.com     2   6    77    64  -1219us[-1276us] +/-  148ms
^- golem.canonical.com           2   6   177     1  +3375us[+3375us] +/-  147ms
^- pugot.canonical.com           2   6   177     1  +3079us[+3079us] +/-  146ms
^- chilipepper.canonical.com     2   6    77    64  +1327us[+1270us] +/-  141ms
^- ns1.alza.is                   2   6   177     0    -53us[  -53us] +/-  175ms
^- 78.140.251.2                  2   6    77    64    +12ms[  +12ms] +/-  150ms
^- 2001:bc8:30a3:100::1          2   6    77    63  +6799us[+6742us] +/-  154ms
^- ntp1.ams1.nl.leaseweb.net     2   6    77    64  +1787us[+1730us] +/-  243ms
^* ntp-a3.nict.go.jp             1   6   177     2  -2094ns[  -56us] +/- 2202us
```
