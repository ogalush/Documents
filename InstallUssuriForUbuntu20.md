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
$ sudo cp -rafv /etc/mysql ~
$ sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf 
-----
-bind-address            = 127.0.0.1
+bind-address            = 192.168.3.200
+default-storage-engine = innodb
+innodb_file_per_table = on
+max_connections = 4096

-character-set-server  = utf8mb4
-collation-server      = utf8mb4_general_ci
+character-set-server = utf8
+collation-server = utf8_general_ci
-----
```

### Finalize installation
```
$ sudo service mysql restart
$ sudo mysql_secure_installation
Enter current password for root (enter for none): [enter]
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
Adding user "openstack" ...
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
Setting permissions for user "openstack" in vhost "/" ...
```

## Memcached for Ubuntu
### Install and configure components
ubuntu20.04はpython2系を入れようとすると`E: Package 'python-memcache' has no installation candidate`となるため、  
hoge-python3fooでインストールしていく.
```
$ sudo apt -y install memcached python3-memcache
$ sudo cp -pv /etc/memcached.conf ~
'/etc/memcached.conf' -> '/home/ogalush/memcached.conf'
$ sudo sed -i 's/-l 127.0.0.1/-l 192.168.3.200/g' /etc/memcached.conf 
$ diff --unified=0 ~/memcached.conf /etc/memcached.conf 
--- /home/ogalush/memcached.conf        2020-07-05 20:28:00.418229945 +0900
+++ /etc/memcached.conf 2020-07-05 20:30:45.475942429 +0900
@@ -35 +35 @@
--l 127.0.0.1
+-l 192.168.3.200
$
```

### Finalize installation
```
$ sudo service memcached restart
$ sudo service memcached status
● memcached.service - memcached daemon
 Loaded: loaded (/lib/systemd/system/memcached.service; enabled; vendor preset: enabled)
 Active: active (running) since Sun 2020-07-05 20:31:50 JST; 3s ago
...
```
