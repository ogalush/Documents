# Install OpenStack Ussuri on CentOS8 (ComputeNode)
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.3.210  
設定ファイル: [Ussuri-InstallConfigsForCentOS8](hoge)
```
[ogalush@hayao ~]$ uname -n
hayao.localdomain
[ogalush@hayao ~]$ cat /etc/redhat-release 
CentOS Stream release 8
[ogalush@hayao ~]$ uname -a
Linux hayao.localdomain 4.18.0-277.el8.x86_64 #1 SMP Wed Feb 3 20:35:19 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
[ogalush@hayao ~]$
```
# Environment
## OS
```
[ogalush@hayao ~]$ uname -n
hayao.localdomain
[ogalush@hayao ~]$ cat /etc/hostname 
hayao.localdomain
[ogalush@hayao ~]$ cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.3.210 hayao hayao.localdomain ・・・追記
[ogalush@hayao ~]$ 
```
## ntp
https://docs.openstack.org/install-guide/environment-ntp-controller.html
```
[ogalush@hayao ~]$ chronyc sources
210 Number of sources = 4
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^+ ernie.gerger-net.de           3  10   377   268  -1198us[-1198us] +/-  131ms
^* 79.133.44.139                 1   9   377   729   -831us[-1889us] +/-  126ms
^+ ntp1.m-online.net             2  10   377   282  +1787us[+1787us] +/-  140ms
^+ ns1.blazing.de                3  10   377   282  +5364us[+5364us] +/-  136ms
[ogalush@hayao ~]$ 
→ 同期しているためOK.
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
