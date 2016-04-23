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
$ sudo vi /etc/mysql/mariadb.conf.d/50-server.cnf
----
[mysqld]
...
bind-address            = 0.0.0.0
...
###character-set-server  = utf8mb4
###collation-server      = utf8mb4_general_ci

default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
character-set-server = utf8
----

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

## Identity service
### DB作成
```
$ sudo mysql -u root -p

MariaDB [(none)]> CREATE DATABASE keystone;
Query OK, 1 row affected (0.00 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.00 sec)
MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)
MariaDB [(none)]> quit;
Bye
```

### Keystoneインストール
```

インストール後に自動起動しないようにする設定とのこと。
$ echo "manual" | sudo tee /etc/init/keystone.override
$ cat /etc/init/keystone.override 
manual

$ sudo apt-get -y install keystone apache2 libapache2-mod-wsgi
```

### Keystone 設定
```
$ sudo vi /etc/keystone/keystone.conf
----
[DEFAULT]
admin_token = token
...

[database]
### connection = sqlite:////var/lib/keystone/keystone.db
connection = mysql+pymysql://keystone:password@192.168.0.200/keystone
...

[token]
...
provider = fernet
----

反映
$ sudo bash -c "keystone-manage db_sync" keystone

fernet(?)の初期化
$ sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
```

### Configure the Apache HTTP server
```
$ sudo vi /etc/apache2/apache2.conf
----
## for OpenStack
ServerName ryunosuke.localdomain
----

$ sudo vi /etc/apache2/sites-available/wsgi-keystone.conf
----
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/bin>
        Require all granted
    </Directory>
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/bin>
        Require all granted
    </Directory>
</VirtualHost>
----

$ sudo ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled
$ ls -l /etc/apache2/sites-enabled/wsgi-keystone.conf 
lrwxrwxrwx 1 root root 47 Apr 23 16:24 /etc/apache2/sites-enabled/wsgi-keystone.conf -> /etc/apache2/sites-available/wsgi-keystone.conf

$ sudo rm -v /var/lib/keystone/keystone.db
$ sudo service apache2 restart
$ sudo service apache2 status
● apache2.service - LSB: Apache2 web server
   Loaded: loaded (/etc/init.d/apache2; bad; vendor preset: enabled)
  Drop-In: /lib/systemd/system/apache2.service.d
           └─apache2-systemd.conf
   Active: inactive (dead) since Sat 2016-04-23 16:25:45 JST; 19s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 15279 ExecStop=/etc/init.d/apache2 stop (code=exited, status=0/SUCCESS)
  Process: 15261 ExecStart=/etc/init.d/apache2 start (code=exited, status=0/SUCCESS)
```
