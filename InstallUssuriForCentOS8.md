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

## SQL database for RHEL and CentOS
https://docs.openstack.org/install-guide/environment-sql-database-rdo.html
### Install and configure components
```
$ sudo yum -y install mariadb mariadb-server python3-PyMySQL
$ sudo cp -rafv /etc/my.cnf.d /tmp
$ sudo vim /etc/my.cnf.d/mariadb-server.cnf
----
[mysqld]
...
+ bind-address = 192.168.3.200
+ default-storage-engine = innodb
+ innodb_file_per_table = on
+ max_connections = 4096
+ collation-server = utf8_general_ci
+ character-set-server = utf8
----
```
### Finalize installation
```
$ sudo systemctl enable mariadb.service
$ sudo systemctl restart mariadb.service
$ sudo systemctl status mariadb.service
$ sudo mysql_secure_installation
Enter current password for root (enter for none):なし
Set root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

## Message queue for RHEL and CentOS
https://docs.openstack.org/install-guide/environment-messaging-rdo.html
### Install and configure components
```
$ sudo yum -y install rabbitmq-server
$ sudo systemctl enable rabbitmq-server.service
$ sudo systemctl restart rabbitmq-server.service
$ sudo rabbitmqctl add_user openstack password
Adding user "openstack" ...
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
Setting permissions for user "openstack" in vhost "/" ...
```

## Memcached for RHEL and CentOS
https://docs.openstack.org/install-guide/environment-memcached-rdo.html
### Install and configure components
```
$ sudo yum -y install memcached python3-memcached
$ sudo cp -pv /etc/sysconfig/memcached /tmp
$ sudo vim /etc/sysconfig/memcached
----
- OPTIONS="-l 127.0.0.1,::1"
+ OPTIONS="-l 127.0.0.1,::1,192.168.3.200"
----
```
### Finalize installation
```
$ sudo systemctl enable memcached.service
$ sudo systemctl restart memcached.service
$ sudo netstat -lnp |grep 11211
tcp        0      0 192.168.3.200:11211     0.0.0.0:*               LISTEN      9001/memcached      
tcp        0      0 127.0.0.1:11211         0.0.0.0:*               LISTEN      9001/memcached      
tcp6       0      0 ::1:11211               :::*                    LISTEN      9001/memcached    
```

## Etcd for RHEL and CentOS
https://docs.openstack.org/install-guide/environment-etcd-rdo.html
### Install and configure components
```
$ sudo yum -y install etcd
$ sudo cp -rafv /etc/etcd /tmp
$ sudo vim /etc/etcd/etcd.conf
-----
$ diff --unified=0 /tmp/etcd/etcd.conf /etc/etcd/etcd.conf |grep -v '^@@'
--- /tmp/etcd/etcd.conf 2020-01-29 01:51:46.000000000 +0900
+++ /etc/etcd/etcd.conf 2020-07-26 16:05:16.713981639 +0900
-#ETCD_LISTEN_PEER_URLS="http://localhost:2380"
-ETCD_LISTEN_CLIENT_URLS="http://localhost:2379"
+ETCD_LISTEN_PEER_URLS="http://192.168.3.200:2380"
+ETCD_LISTEN_CLIENT_URLS="http://192.168.3.200:2379"
-ETCD_NAME="default"
+ETCD_NAME="ryunosuke"
-#ETCD_INITIAL_ADVERTISE_PEER_URLS="http://localhost:2380"
-ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"
+ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.3.200:2380"
+ETCD_ADVERTISE_CLIENT_URLS="http://192.168.3.200:2379"
-#ETCD_INITIAL_CLUSTER="default=http://localhost:2380"
-#ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
-#ETCD_INITIAL_CLUSTER_STATE="new"
+ETCD_INITIAL_CLUSTER="ryunosuke=http://192.168.3.200:2380"
+ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
+ETCD_INITIAL_CLUSTER_STATE="new"
-----
```
### Finalize installation
```
$ sudo systemctl enable etcd
$ sudo systemctl restart etcd
$ sudo systemctl status etcd
```
