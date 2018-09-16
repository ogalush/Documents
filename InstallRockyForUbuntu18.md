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

# OpenStack packages
## OpenStack packages for Ubuntu
### Enable the OpenStack repository
```
$ sudo apt -y install software-properties-common
$ sudo add-apt-repository cloud-archive:rocky
...
Get:2 http://jp.archive.ubuntu.com/ubuntu bionic-updates InRelease [88.7 kB]
Get:3 http://jp.archive.ubuntu.com/ubuntu bionic-backports InRelease [74.6 kB]
Hit:4 http://security.ubuntu.com/ubuntu bionic-security InRelease
Ign:5 http://ubuntu-cloud.archive.canonical.com/ubuntu bionic-updates/rocky InRelease
Get:6 http://ubuntu-cloud.archive.canonical.com/ubuntu bionic-updates/rocky Release [7,879 B]
Get:7 http://ubuntu-cloud.archive.canonical.com/ubuntu bionic-updates/rocky Release.gpg [543 B]
Get:8 http://ubuntu-cloud.archive.canonical.com/ubuntu bionic-updates/rocky/main amd64 Packages [112 kB]
Get:9 http://ubuntu-cloud.archive.canonical.com/ubuntu bionic-updates/rocky/main i386 Packages [112 kB]
Fetched 395 kB in 4s (105 kB/s)
Reading package lists... Done
$
```

### Finalize the installation
```
$ sudo apt -y install python-openstackclient
$ sudo apt-get -y update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade && sudo apt-get -y autoremove
$ sudo shutdown -r now
```

## SQL database for Ubuntu
### Install and configure components
```
$ sudo apt -y install mariadb-server python-pymysql
$ sudo cp -ra /etc/mysql ~
$ sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf
$ diff -urBb ~/mysql /etc/mysql 2> /dev/null |egrep '^(\+|\-)'
--- /home/ogalush/mysql/mariadb.conf.d/50-server.cnf    2018-07-08 17:14:42.000000000 +0900
+++ /etc/mysql/mariadb.conf.d/50-server.cnf     2018-09-16 15:46:26.879647819 +0900
-bind-address           = 127.0.0.1
+bind-address           = 192.168.0.200
-character-set-server  = utf8mb4
-collation-server      = utf8mb4_general_ci
+character-set-server = utf8
+collation-server = utf8_general_ci
+
+default-storage-engine = innodb
+innodb_file_per_table = on
+max_connections = 4096
----

$ sudo service mysql restart
$ sudo service mysql status
● mariadb.service - MariaDB 10.1.34 database server
   Loaded: loaded (/lib/systemd/system/mariadb.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-09-16 15:47:42 JST; 14s ago
$ sudo mysql_secure_installation
Set root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

## Message queue for Ubuntu
### Install and configure components
```
$ sudo apt -y install rabbitmq-server
$ sudo rabbitmqctl add_user openstack password
Creating user "openstack"
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
Setting permissions for user "openstack" in vhost "/"
```

## Memcached for Ubuntu
### Install and configure components
```
$ sudo apt -y install memcached python-memcache
$ sudo sed -i 's/-l 127.0.0.1/-l 192.168.0.200/' /etc/memcached.conf 
$ grep 192.168.0.200 /etc/memcached.conf 
-l 192.168.0.200
```

### Finalize installation
```
$ sudo service memcached restart
$ sudo service memcached status
● memcached.service - memcached daemon
   Loaded: loaded (/lib/systemd/system/memcached.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-09-16 15:52:39 JST; 4s ago

$ netstat -ln |grep 11211
tcp        0      0 192.168.0.200:11211     0.0.0.0:*               LISTEN   
→ 変更したIPアドレスでLISTENできているのでOK.
```

## Etcd for Ubuntu
### Install and configure components
```
$ sudo apt -y install etcd
$ sudo vim /etc/default/etcd
----
name: 'ryunosuke'
data-dir: /var/lib/etcd
initial-cluster-state: 'new'
initial-cluster-token: 'etcd-cluster-01'
initial-cluster: ryunosuke=http://192.168.0.200:2380
initial-advertise-peer-urls: http://192.168.0.200:2380
advertise-client-urls: http://192.168.0.200:2379
listen-peer-urls: http://0.0.0.0:2380
listen-client-urls: http://192.168.0.200:2379
----
```

### Finalize installation
```
$ sudo systemctl enable etcd
$ sudo systemctl restart etcd
$ sudo systemctl status etcd
● etcd.service - etcd - highly-available key value store
   Loaded: loaded (/lib/systemd/system/etcd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-09-16 16:05:15 JST; 11s ago
     Docs: https://github.com/coreos/etcd
```
