# Install OpenStack Rocky on Ubuntu 18.04 ComputeNode
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.0.210  
設定ファイル: [Rocky-InstallConfigs](https://github.com/ogalush/Rocky-InstallConfigs)
```
$ uname -a
Linux hayao 4.15.0-45-generic #48-Ubuntu SMP Tue Jan 29 16:28:13 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
```

# Prepare
## resolv.conf
nameserver 127.0.0.53になった事象があったので、参照先が怪しい場合は対応する.
```
$ grep nameserver /etc/resolv.conf 
nameserver 127.0.0.53
$ sudo rm -f /etc/resolv.conf
$ sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
$ ls -l /etc/resolv.conf 
lrwxrwxrwx 1 root root 32 Feb 17 15:56 /etc/resolv.conf -> /run/systemd/resolve/resolv.conf
$ grep nameserver /etc/resolv.conf 
nameserver 192.168.0.254
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
--- /tmp/hosts  2019-02-16 01:44:10.191132478 +0900
+++ /etc/hosts  2019-02-17 15:57:30.640683535 +0900
-127.0.1.1      hayao
+192.168.0.210  hayao
$
```

## Network Time Protocol (NTP)
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
   Active: active (running) since Sun 2019-02-17 15:59:29 JST; 4s ago
     Docs: man:chronyd(8)
...
---

$ chronyc sources
210 Number of sources = 9
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^- golem.canonical.com           2   6    33    17   +832us[ +832us] +/-  151ms
^- pugot.canonical.com           2   6    17    21   +811us[ +811us] +/-  156ms
^- chilipepper.canonical.com     2   6    17    21  +5329us[+5329us] +/-  145ms
^- alphyn.canonical.com          2   6    17    22  +2935us[+2956us] +/-  119ms
^- ntp1.jst.mfeed.ad.jp          2   6    17    22   -412us[ -391us] +/-   73ms
^- sv1.localdomain1.com          2   6    17    23   -484us[ -973us] +/-   44ms
^- y.ns.gin.ntt.net              2   6    17    23  -1762us[-1742us] +/-  101ms
^- ntp3.jst.mfeed.ad.jp          2   6    17    23   -235us[ -214us] +/-  109ms
^* ntp-a3.nict.go.jp             1   6    17    22    +35us[  +55us] +/- 2589us
```

## CPU Performance (任意)
CPUガバナーをPerformanceへ変更してレスポンスを上げておく
```
https://askubuntu.com/questions/1021748/set-cpu-governor-to-performance-in-18-04
$ sudo apt-get -y install cpufrequtils
$ echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
$ sudo systemctl disable ondemand
$ sudo reboot
$ cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
performance
performance
performance
performance
$
→ Performance表示となっていればOK.
```

# OpenStack packages for Ubuntu
## Enable the OpenStack repository
```
$ sudo apt -y install software-properties-common
$ sudo add-apt-repository cloud-archive:rocky
```

## Finalize the installation
```
$ sudo apt -y install python-openstackclient
$ sudo apt-get -y update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade && sudo apt-get -y autoremove
$ sudo shutdown -r now
```
