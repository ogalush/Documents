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

## Hosts
```
$ sudo cp -pv /etc/hosts ~
$ sudo vim /etc/hosts
---
192.168.0.200 ryunosuke.localdomain ryunosuke
→ 追記
---

$ diff -u ~/hosts /etc/hosts
--- /home/ogalush/hosts 2013-06-07 23:31:32.000000000 +0900
+++ /etc/hosts  2019-02-24 16:56:09.282865680 +0900
@@ -1,2 +1,3 @@
 127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
 ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
+192.168.0.200 ryunosuke.localdomain ryunosuke
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

## SQL database for RHEL and CentOS
[Doc](https://docs.openstack.org/install-guide/environment-sql-database-rdo.html)
```
$ sudo yum -y install mariadb mariadb-server python2-PyMySQL
$ sudo cp -rafv /etc/my.cnf* ~
$ cat << _EOS_ > ~/openstack.cnf
[mysqld]
bind-address = 192.168.0.200
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
_EOS_
$ sudo cp -v ~/openstack.cnf /etc/my.cnf.d
$ sudo systemctl enable mariadb.service
$ sudo systemctl start mariadb.service

$ sudo mysql_secure_installation
Enter current password for root (enter for none): 
OK, successfully used password, moving on...
Set root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

## Message queue for RHEL and CentOS
[Doc](https://docs.openstack.org/install-guide/environment-messaging-rdo.html)
```
$ sudo yum -y install rabbitmq-server
$ sudo systemctl enable rabbitmq-server.service
$ sudo systemctl start rabbitmq-server.service

$ sudo rabbitmqctl add_user openstack password
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

## Memcached for RHEL and CentOS
[Doc](https://docs.openstack.org/install-guide/environment-memcached-rdo.html)
```
$ sudo yum -y install memcached python-memcached
$ sudo cp -pv /etc/sysconfig/memcached ~
$ sudo sed -i 's/127.0.0.1,::1/192.168.0.200/g' /etc/sysconfig/memcached
$ diff -u ~/memcached /etc/sysconfig/memcached
--- /home/ogalush/memcached     2018-03-01 18:49:35.000000000 +0900
+++ /etc/sysconfig/memcached    2019-02-24 16:54:13.502753139 +0900
@@ -2,4 +2,4 @@
 USER="memcached"
 MAXCONN="1024"
 CACHESIZE="64"
-OPTIONS="-l 127.0.0.1,::1"
+OPTIONS="-l 192.168.0.200"
$

$ sudo systemctl enable memcached.service
$ sudo systemctl start memcached.service
```

## Etcd for RHEL and CentOS
[Doc](https://docs.openstack.org/install-guide/environment-etcd-rdo.html)
```
$ sudo yum -y install etcd
$ sudo cp -pv /etc/etcd/etcd.conf ~
$ sudo vim /etc/etcd/etcd.conf
$ diff -u ~/etcd.conf /etc/etcd/etcd.conf |egrep '^(\+|\-)'
--- /home/ogalush/etcd.conf     2019-02-14 01:56:03.000000000 +0900
+++ /etc/etcd/etcd.conf 2019-02-24 17:01:06.974445818 +0900
-ETCD_LISTEN_CLIENT_URLS="http://localhost:2379"
+ETCD_LISTEN_PEER_URLS="http://192.168.0.200:2380"
+ETCD_LISTEN_CLIENT_URLS="http://192.168.0.200:2379"
-ETCD_NAME="default"
+ETCD_NAME="ryunosuke"
-#ETCD_INITIAL_ADVERTISE_PEER_URLS="http://localhost:2380"
-ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"
+ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.200:2380"
+ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.200:2379"
-#ETCD_INITIAL_CLUSTER="default=http://localhost:2380"
-#ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
-#ETCD_INITIAL_CLUSTER_STATE="new"
+ETCD_INITIAL_CLUSTER="ryunosuke=http://192.168.0.200:2380"
+ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
+ETCD_INITIAL_CLUSTER_STATE="new"
$

$ sudo systemctl enable etcd
$ sudo systemctl start etcd
```
