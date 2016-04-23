## Install OpenStack Mitaka on Ubuntu 16.04

ドキュメント: [OpenStack Docs](http://docs.openstack.org/mitaka/install-guide-ubuntu/environment.html)  
インストール先: ryunosuke(192.168.0.200)  
```
ogalush@ryunosuke:~$ uname -a
Linux ryunosuke 4.4.0-21-generic #37-Ubuntu SMP Mon Apr 18 18:33:37 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
```

## Environment
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

### OpenStack packages
```
$ sudo apt-get -y install software-properties-common
$ sudo add-apt-repository cloud-archive:mitaka
 Ubuntu Cloud Archive for OpenStack Mitaka
 More info: https://wiki.ubuntu.com/ServerTeam/CloudArchive
Press [ENTER] to continue or ctrl-c to cancel adding it

cloud-archive for Mitaka only supported on trusty
~~~ このコマンドはubuntu16.04では使用できないらしい。mitaka使えるん？

$ sudo apt-get -y update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade
```

### OpenStack Client
```
$ sudo apt-get -y install python-openstackclient
```

### SQL database
```
$ sudo apt-get -y install mariadb-server python-pymysql

$ sudo tee /etc/mysql/conf.d/openstack.cnf << 'EOT'
[mysqld]
bind-address = 192.168.0.200
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
character-set-server = utf8
EOT

$ sudo service mysql restart
$ sudo service mysql status
● mysql.service - LSB: Start and stop the mysql database server daemon
   Loaded: loaded (/etc/init.d/mysql; bad; vendor preset: enabled)
   Active: active (running) since Sat 2016-04-23 15:32:12 JST; 13s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 6933 ExecStop=/etc/init.d/mysql stop (code=exited, status=0/SUCCESS)
  Process: 6969 ExecStart=/etc/init.d/mysql start (code=exited, status=0/SUCCESS)
    Tasks: 27 (limit: 512)
   Memory: 78.7M
      CPU: 264ms
   CGroup: /system.slice/mysql.service
           ├─6998 /bin/bash /usr/bin/mysqld_safe
           ├─6999 logger -p daemon err -t /etc/init.d/mysql -i
           ├─7158 /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/lib/mysql/plugin --user=mysql --sk
           └─7159 logger -t mysqld -p daemon error


$ sudo mysql_secure_installation
...
Enter current password for root (enter for none): <パスワードを入れる>
Change the root password? [Y/n] y
New password: <パスワードを入れる>
Re-enter new password: <パスワードを入れる>
Password updated successfully!

Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

### Message queue
RabbitMQを入れる。
```
$ sudo apt-get -y install rabbitmq-server
$ sudo rabbitmqctl add_user openstack password
Creating user "openstack" ...
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
Setting permissions for user "openstack" in vhost "/" ...
```

### Memcached
```
$ sudo apt-get -y install memcached python-memcache
$ sudo cp -pv /etc/memcached.conf /etc/memcached.conf.def
'/etc/memcached.conf' -> '/etc/memcached.conf.def'

$ sudo sed -i 's/127.0.0.1/192.168.0.200/g' /etc/memcached.conf
$ diff -u /etc/memcached.conf.def /etc/memcached.conf
----
--l 127.0.0.1
+-l 192.168.0.200
----

$ sudo service memcached restart
$ sudo service memcached status
● memcached.service - memcached daemon
   Loaded: loaded (/lib/systemd/system/memcached.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2016-04-23 15:45:57 JST; 2s ago
 Main PID: 9279 (memcached)
    Tasks: 6 (limit: 512)
   Memory: 932.0K
      CPU: 3ms
   CGroup: /system.slice/memcached.service
           └─9279 /usr/bin/memcached -m 64 -p 11211 -u memcache -l 192.168.0.200
```

