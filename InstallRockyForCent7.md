# Install OpenStack Rocky for CentOS7
[OpenStack Installation Guide](https://docs.openstack.org/install-guide/)  
Ubuntu18.04版がSegmentation Failtエラーが頻繁に出て不安定なため、CentOS7で構築をしてみる.
## 環境
```
[ogalush@ryunosuke ~]$ uname -n
ryunosuke.localdomain
[ogalush@ryunosuke ~]$ cat /etc/redhat-release 
CentOS Linux release 7.6.1810 (Core) 
[ogalush@ryunosuke ~]$ uname -a
Linux ryunosuke.localdomain 3.10.0-957.el7.x86_64 #1 SMP Thu Nov 8 23:39:32 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
[ogalush@ryunosuke ~]$ ip addr show |grep 192.168.0.200
    inet 192.168.0.200/24 brd 192.168.0.255 scope global noprefixroute enp3s0
[ogalush@ryunosuke ~]$ 
```

# Environment
## Configure network interfaces
[Doc](https://docs.openstack.org/install-guide/environment-networking-controller.html)
```
[ogalush@ryunosuke ~]$ grep -e 'DEVICE' -e 'TYPE' -e 'ONBOOT' -e 'BOOTPROTO' /etc/sysconfig/network-scripts/ifcfg-enp3s0 
TYPE=Ethernet
BOOTPROTO=none
DEVICE=enp3s0
ONBOOT=yes
[ogalush@ryunosuke ~]$
→ マニュアル通り.
```

## NTP
```
[ogalush@ryunosuke ~]$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
-sv1.localdomain 133.243.238.244  2 u   21   64  377    3.536  -21.530   4.919
+s97.GchibaFL4.v 133.243.238.244  2 u   29   64  377    6.664  -26.797  16.482
+chobi.paina.net 203.178.138.38   2 u   27   64  377    9.825  -24.327   7.463
*ntp-5.jonlight. 133.243.238.243  2 u   33   64  377    3.349  -19.459   4.138
[ogalush@ryunosuke ~]$
→ すでにインストールしてあるのでOK.
```

## OpenStack packages for RHEL and CentOS
```
$ sudo yum -y install subscription-manager
$ sudo subscription-manager repos --enable=rhel-7-server-optional-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms
System certificates corrupted. Please reregister.
→ なぜかうまく言ってない.
$ sudo yum -y install centos-release-openstack-rocky
→ なぜかインストールはうまく言った.
----
Installed:
  centos-release-openstack-rocky.noarch 0:1-1.el7.centos                                                             

Dependency Installed:
  centos-release-ceph-luminous.noarch 0:1.1-2.el7.centos      centos-release-qemu-ev.noarch 0:1.0-4.el7.centos       
  centos-release-storage-common.noarch 0:2-2.el7.centos       centos-release-virt-common.noarch 0:1-1.el7.centos     

Complete!
[ogalush@ryunosuke ~]$
----

##$ sudo yum install https://rdoproject.org/repos/rdo-release.rpm
→ RDOを入れようとするとqueenが入るのでやめる.

$ sudo yum -y update
$ sudo yum -y install python-openstackclient
$ sudo yum -y install openstack-selinux
→ SELinuxがデフォルトで有効になってるので、よしなにやってくれるらしい.
```
