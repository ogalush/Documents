# Install OpenStack Ussuri on CentOS8
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.3.200(192.168.3.200)  
設定ファイル: [URL](URL)
```
$ uname -n
ryunosuke.localdomain
$ cat /etc/redhat-release 
CentOS Linux release 8.2.2004 (Core) 
[ogalush@ryunosuke ~]$ uname -a
Linux ryunosuke.localdomain 4.18.0-193.6.3.el8_2.x86_64 #1 SMP Wed Jun 10 11:09:32 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
```
# Environment
## OS
CentOS8でインストールを行う.  
CentOS7は、ussuri向けのRPMが無いため.
## Hosts
```
$ uname -n
ryunosuke.localdomain
$ cat /etc/hostname 
ryunosuke.localdomain
$ cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.3.200 ryunosuke ryunosuke.localdomain
$
```
## ntp
https://docs.openstack.org/install-guide/environment-ntp-controller.html
```
$ chronyc sources
210 Number of sources = 4
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* gpg.n1zyy.com                 2   6   377    25  -4076us[-7488us] +/-   89ms
^+ login-vlan194.budapest2.>     2   6   377    24  -7054us[-7054us] +/-  169ms
^+ 138.68.183.179                3   6   377    24  +9476us[+9476us] +/-  171ms
^+ www.bochum.solar              2   6   377    25  +1840us[+1840us] +/-  133ms
→ 同期してる.

$ chronyc tracking
Reference ID    : C0630208 (gpg.n1zyy.com)
Stratum         : 3
Ref time (UTC)  : Sun Jul 26 06:38:51 2020 ・・・最終確認時刻(UTC, JST=UTC+0900)
System time     : 0.002377091 seconds slow of NTP time・・・NTPサーバーと自端末時刻の誤差
Last offset     : -0.003412287 seconds
RMS offset      : 0.002153547 seconds
Frequency       : 25.055 ppm fast
Residual freq   : -3.757 ppm
Skew            : 12.161 ppm
Root delay      : 0.171737716 seconds
Root dispersion : 0.006856591 seconds
Update interval : 64.4 seconds
Leap status     : Normal

読み方.
https://hackers-high.com/linux/easy-chrony-settings/
```

## OpenStack packages for RHEL and CentOS
https://docs.openstack.org/install-guide/environment-packages-rdo.html
```
$ sudo yum -y install centos-release-openstack-ussuri
$ sudo yum config-manager --set-enabled PowerTools
$ sudo yum -y upgrade
$ sudo yum -y install python3-openstackclient
$ sudo yum -y install openstack-selinux
```
