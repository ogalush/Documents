# Install OpenStack Rocky on Ubuntu 20.04
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.3.200(192.168.3.200)  
設定ファイル: [URL](URL)
```
ogalush@ryunosuke:~$ uname -a
Linux ryunosuke 5.4.0-40-generic #44-Ubuntu SMP Tue Jun 23 00:01:04 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
ogalush@ryunosuke:~$
```

## Networking
### Configure network interfaces
127.0.1.1 がホスト名に紐づいているので、実際のIPアドレスへ置換する.
```
$ sudo cp -pv /etc/hosts /tmp/hosts
'/etc/hosts' -> '/tmp/hosts'
$ sudo vim /etc/hosts
$ diff --unified=0 /tmp/hosts /etc/hosts
--- /tmp/hosts  2020-05-05 00:38:36.503499586 +0900
+++ /etc/hosts  2020-07-05 19:52:04.440981711 +0900
@@ -2 +2 @@
-127.0.1.1 ryunosuke
+192.168.3.200 ryunosuke
$
```

## Network Time Protocol (NTP)
時刻同期できているのでOK.
```
$ dpkg -l ntp
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name           Version                  Architecture Description
+++-==============-========================-============-=================================================
ii  ntp            1:4.2.8p12+dfsg-3ubuntu4 amd64        Network Time Protocol daemon and utility programs
$ ntpq -p |grep '*'
*ntp-a2.nict.go. .NICT.           1 u   52   64  177    4.558    1.035   2.728
$
```
