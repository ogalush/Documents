## Install OpenStack Pike on Ubuntu 16.04
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/openstack-services.html)  
インストール先: ryunosuke(192.168.0.200)  
```
ogalush@livaserver:~$ ssh -c aes128-ctr -A ogalush@192.168.0.200
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.4.0-109-generic x86_64)
ogalush@ryunosuke:~$ uname -a
Linux ryunosuke 4.4.0-109-generic #132-Ubuntu SMP Tue Jan 9 19:52:39 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
ogalush@ryunosuke:~$
```
「Minimal deployment」の最小構成を構築する. 

## Environment
### OS Update
```
$ sudo apt-get -y update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade
```

### ntp
```
$ ntpq -p
→ 時刻同期していればOK.
ogalush@ryunosuke:~$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 0.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 1.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 2.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 3.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp.ubuntu.com  .POOL.          16 p    -   64    0    0.000    0.000   0.000
+golem.canonical 192.150.70.56    2 u    6   64  177  235.278    1.376  58.327
+juniperberry.ca 140.203.204.77   2 u    8   64  177  250.272    9.587  58.079
+64.71.128.26 (1 216.218.254.202  2 u    4   64   77  110.667   -0.890  51.408
+y.ns.gin.ntt.ne 249.224.99.213   2 u    2   64   77    7.290   -3.315  49.621
+extendwings.com 10.84.87.146     2 u   72   64   76    8.171   -8.905  51.148
+x.ns.gin.ntt.ne 249.224.99.213   2 u   67   64   37    5.258  -12.825  50.265
*sv01.azsx.net   103.1.106.69     2 u    3   64   77    9.597   -2.385  49.716
+balthasar.gimas 181.170.187.161  3 u    4   64   77   68.825   30.177  77.336
+sv1.localdomain 133.243.238.163  2 u   67   64   37    5.650   -8.568  49.828
ogalush@ryunosuke:~$ 
```

## Keystone Installation Tutorial
[Document](https://docs.openstack.org/keystone/pike/install/)
### Install and configure
```
# Setup MariaDB
$ sudo apt-get -y install mariadb-server
$ sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf 
----
#bind-address           = 127.0.0.1
bind-address            = 0.0.0.0
----
$ sudo service mysql restart
$ sudo service mysql status
● mysql.service - LSB: Start and stop the mysql database server daemon
   Loaded: loaded (/etc/init.d/mysql; bad; vendor preset: enabled)
   Active: active (running) since Sat 2018-01-20 22:58:06 JST; 6s ago
....
$ netstat -ln |grep 3306
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN  
→ 0.0.0.0でLISTENできていればOK.

ogalush@ryunosuke:/etc/mysql$ sudo /usr/bin/mysql_secure_installation
----
Enter current password for root (enter for none): 
Set root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
----
$ sudo mysql
→ MySQLのプロンプトが表示されればOK.
```

### Prerequisites
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;
```

### Install and configure components
install
```
$ sudo apt install -y keystone  apache2 libapache2-mod-wsgi
```
Config
```
$ sudo vim /etc/keystone/keystone.conf
----
[database]
##connection = sqlite:////var/lib/keystone/keystone.db
connection = mysql+pymysql://keystone:password@192.168.0.200/keystone
...
[token]
provider = fernet
...


----
```
