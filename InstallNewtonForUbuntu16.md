## Install OpenStack Newton on Ubuntu 16.04

ドキュメント: [OpenStack Docs](http://docs.openstack.org/newton/install-guide-ubuntu/)  
インストール先: ryunosuke(192.168.0.200)
```
ogalush@livaserver:~$ ssh -c aes128-ctr -A ogalush@192.168.0.200
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-47-generic x86_64)
ogalush@ryunosuke:~$ uname -a
Linux ryunosuke 4.4.0-47-generic #68-Ubuntu SMP Wed Oct 26 19:39:52 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
ogalush@ryunosuke:~$ 
```

## Environment
### Network Time Protocol (NTP)
ntpdがdefaultでinstallされているので対応なし
```
$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 0.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 1.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 2.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 3.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp.ubuntu.com  .POOL.          16 p    -   64    0    0.000    0.000   0.000
+v157-7-235-92.z 103.1.106.69     2 u   47  128  377    4.260   -1.575   0.776
+li400-120.membe 157.7.208.12     3 u   37  128  377    4.392   -1.853   0.942
*extendwings.com 133.243.238.163  2 u   41  128  377    4.276   -2.192   1.347
-time.platformni 150.101.114.190  2 u  102  256  377    5.153   -5.328   1.619
-alphyn.canonica 132.246.11.231   2 u   44  128  377  196.992   -1.019   0.333
ogalush@ryunosuke:~$ 
```

### OpenStack packages
#### Enable the OpenStack repository
```
$ sudo apt install software-properties-common
Reading package lists... Done
Building dependency tree       
Reading state information... Done
software-properties-common is already the newest version (0.96.20.4).
~~~ 既に入っている。
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.

$ sudo add-apt-repository cloud-archive:newton
 Ubuntu Cloud Archive for OpenStack Newton
 More info: https://wiki.ubuntu.com/ServerTeam/CloudArchive
Press [ENTER] to continue or ctrl-c to cancel adding it

Reading package lists...
Building dependency tree...
Reading state information...
The following NEW packages will be installed:
  ubuntu-cloud-keyring
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 5,086 B of archives.
After this operation, 34.8 kB of additional disk space will be used.
Get:1 http://jp.archive.ubuntu.com/ubuntu xenial/universe amd64 ubuntu-cloud-keyring all 2012.08.14 [5,086 B]
Fetched 5,086 B in 0s (111 kB/s)
Selecting previously unselected package ubuntu-cloud-keyring.
(Reading database ... 138021 files and directories currently installed.)
Preparing to unpack .../ubuntu-cloud-keyring_2012.08.14_all.deb ...
Unpacking ubuntu-cloud-keyring (2012.08.14) ...
Setting up ubuntu-cloud-keyring (2012.08.14) ...
Importing ubuntu-cloud.archive.canonical.com keyring
OK
Processing ubuntu-cloud.archive.canonical.com removal keyring
gpg: /etc/apt/trustdb.gpg: trustdb created
OK
ogalush@ryunosuke:~$
```

#### Finalize the installation
```
$ sudo apt -y update && sudo apt -y dist-upgrade
→ 通常のPackageUpdateが行われる。

$ sudo apt -y install python-openstackclient
→ PythonクライアントがインストールされればOK.
```

### SQL database
```
$ sudo apt -y install mariadb-server python-pymysql

MySQLのbind-address設定が50-server.cnfに入っているので、編集する。
$ sudo grep -iRH 'bind-address' /etc/mysql/
/etc/mysql/mariadb.conf.d/50-server.cnf:bind-address            = 127.0.0.1

バックアップ
$ sudo cp -pv /etc/mysql/mariadb.conf.d/50-server.cnf ~
'/etc/mysql/mariadb.conf.d/50-server.cnf' -> '/home/ogalush/50-server.cnf'

設定変更
$ sudo vi /etc/mysql/mariadb.conf.d/50-server.cnf
----
[mysqld]
bind-address            = 192.168.0.200
...
## for OpenStack Settings.
default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
----

※mysqldの文字コードが、デフォルト設定でutf8mb4になっている。utf8の4Byte文字も許容するCodeの模様。
OpenStackドキュメントのutf8よりも良いと思われるのでそのままで。
http://dev.classmethod.jp/cloud/aws/utf8mb4-on-rds-mysql/

反映
$ sudo service mysql restart
$ sudo service mysql status
● mysql.service - LSB: Start and stop the mysql database server daemon
   Loaded: loaded (/etc/init.d/mysql; bad; vendor preset: enabled)
   Active: active (running) since Sat 2016-11-26 17:52:38 JST; 2s ago
     Docs: man:systemd-sysv-generator(8)

Security Settings.
$ sudo mysql_secure_installation
Set root password? [Y/n] n
 ... skipping.
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
Thanks for using MariaDB!
```

### Message queue
```
インストール
$ sudo apt -y install rabbitmq-server

設定
$ sudo rabbitmqctl add_user openstack password
Creating user "openstack" ...

$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
Setting permissions for user "openstack" in vhost "/" ...
```

### Memcached
#### Install and configure components
```
インストール
$ sudo apt -y install memcached python-memcache

設定
$ sudo sed -i 's/-l 127.0.0.1/-l 192.168.0.200/g' /etc/memcached.conf 
$ sudo service memcached restart
$ sudo service memcached status
● memcached.service - memcached daemon
   Loaded: loaded (/lib/systemd/system/memcached.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2016-11-26 18:01:36 JST; 5s ago
 Main PID: 11438 (memcached)
    Tasks: 6
   Memory: 548.0K
      CPU: 3ms
   CGroup: /system.slice/memcached.service
           └─11438 /usr/bin/memcached -m 64 -p 11211 -u memcache -l 192.168.0.200

確認
$ sudo netstat -lpn |grep 11211
tcp        0      0 192.168.0.200:11211     0.0.0.0:*               LISTEN      11438/memcached 
udp        0      0 192.168.0.200:11211     0.0.0.0:*                           11438/memcached 
~~~ Interfaceに紐づいているIPアドレスでLISTENしているのでOK. 
```
