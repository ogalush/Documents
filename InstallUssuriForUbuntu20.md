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

## OpenStack packages for Ubuntu
cloud-archive:UssuriはUbuntu18.04用となるのでスキップ.
```
$ sudo add-apt-repository cloud-archive:ussuri
 Ubuntu Cloud Archive for OpenStack Ussuri
 More info: https://wiki.ubuntu.com/OpenStack/CloudArchive
Press [ENTER] to continue or Ctrl-c to cancel adding it.

cloud-archive for Ussuri only supported on bionic
```
[OpenStackのUssuriリリースがUbuntu 18.04 LTSと20.04 LTSで利用可能に](https://jp.ubuntu.com/blog/openstack%E3%81%AEussuri%E3%83%AA%E3%83%AA%E3%83%BC%E3%82%B9%E3%81%8Cubuntu-18-04-lts%E3%81%A820-04-lts%E3%81%A7%E5%88%A9%E7%94%A8%E5%8F%AF%E8%83%BD%E3%81%AB)を見ると利用できるはずだが.

デフォルトでussuriパッケージをインストールできる模様.
```
$ apt show python3-openstackclient | head -n 5

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

Package: python3-openstackclient
Version: 5.2.0-0ubuntu1
Priority: optional
Section: python
Source: python-openstackclient
$ 
```
[python-openstackclient 5.2.0-0ubuntu1 source package in Ubuntu](https://launchpad.net/ubuntu/+source/python-openstackclient/5.2.0-0ubuntu1)
```
python-openstackclient 5.2.0-0ubuntu1 source package in Ubuntu
Changelog
python-openstackclient (5.2.0-0ubuntu1) focal; urgency=medium
  * New upstream release for OpenStack Ussuri. ・・・★Ussuri用のリリースと書いてある.
  * d/control: Align (Build-)Depends with upstream.
 -- James Page <email address hidden>  Mon, 30 Mar 2020 15:59:21 +0100
```

インストール
```
$ sudo apt -y install python3-openstackclient
$ sudo apt -y update
$ sudo apt -y upgrade
$ sudo apt -y dist-upgrade
$ sudo apt -y autoremove
```

## SQL database for Ubuntu
https://docs.openstack.org/install-guide/environment-sql-database-ubuntu.html

### Install and configure components¶
```
$ sudo apt -y install mariadb-server python3-pymysql
```
