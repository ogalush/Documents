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
$ sudo service keystone stop
~~~ Port 5000がKeystoneで使われているので、一旦停止させる。
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

### Create the service entity and API endpoints
準備
```
$ export OS_TOKEN=token
$ export OS_URL=http://192.168.0.200:35357/v3
$ export OS_IDENTITY_API_VERSION=3
```

サービス作成
```
$ openstack service create  --name keystone --description "OpenStack Identity" identity
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Identity               |
| enabled     | True                             |
| id          | d4d4c4573e4546f2892ab65b220e1df2 |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+
ogalush@ryunosuke:~$ 
```

Endpoint作成
```
$ openstack endpoint create --region RegionOne identity public http://192.168.0.200:5000/v3
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 651c611d45354aa2bea32f5935b43be7 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d4d4c4573e4546f2892ab65b220e1df2 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://192.168.0.200:5000/v3     |
+--------------+----------------------------------+
ogalush@ryunosuke:~$ 

ogalush@ryunosuke:~$ openstack endpoint create --region RegionOne identity internal http://192.168.0.200:5000/v3
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 4b398962d0e64399bbf66d5ccf8b5cae |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d4d4c4573e4546f2892ab65b220e1df2 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://192.168.0.200:5000/v3     |
+--------------+----------------------------------+
ogalush@ryunosuke:~$ 

ogalush@ryunosuke:~$ openstack endpoint create --region RegionOne identity admin http://192.168.0.200:35357/v3
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | dd967a13566d4aabbc6bc311345d8ffa |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d4d4c4573e4546f2892ab65b220e1df2 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://192.168.0.200:35357/v3    |
+--------------+----------------------------------+
ogalush@ryunosuke:~$ 
```

### Create a domain, projects, users, and roles
ドメイン作成
```
ogalush@ryunosuke:~$ openstack domain create --description "Default Domain" default
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Default Domain                   |
| enabled     | True                             |
| id          | 7841afe520964a04aa50756aee42a5d3 |
| name        | default                          |
+-------------+----------------------------------+
ogalush@ryunosuke:~$
```

adminプロジェクト
```
ogalush@ryunosuke:~$ openstack project create --domain default --description "Admin Project" admin
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Admin Project                    |
| domain_id   | 7841afe520964a04aa50756aee42a5d3 |
| enabled     | True                             |
| id          | 290db0b57ea34427b01a6308d3f6e47c |
| is_domain   | False                            |
| name        | admin                            |
| parent_id   | 7841afe520964a04aa50756aee42a5d3 |
+-------------+----------------------------------+
ogalush@ryunosuke:~$ 
```

adminユーザ
```
ogalush@ryunosuke:~$ openstack user create --domain default --password-prompt admin
User Password: <パスワードを入れる>
Repeat User Password: <パスワードを入れる>
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | 7841afe520964a04aa50756aee42a5d3 |
| enabled   | True                             |
| id        | 45e936fbfe18477a9d083f902bbf0270 |
| name      | admin                            |
+-----------+----------------------------------+
ogalush@ryunosuke:~$ 
```

adminロール
```
ogalush@ryunosuke:~$ openstack role create admin
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 8c791de47d2c49b4a396df0190bf6ac5 |
| name      | admin                            |
+-----------+----------------------------------+
ogalush@ryunosuke:~$ 
```

adminユーザの紐付け
```
$ openstack role add --project admin --user admin admin
```

サービス
```
ogalush@ryunosuke:~$ openstack project create --domain default --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | 7841afe520964a04aa50756aee42a5d3 |
| enabled     | True                             |
| id          | 8c7d96939a92466f814c64458f3759a6 |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | 7841afe520964a04aa50756aee42a5d3 |
+-------------+----------------------------------+
ogalush@ryunosuke:~$ 
```

demoプロジェクト
```
ogalush@ryunosuke:~$ openstack project create --domain default --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | 7841afe520964a04aa50756aee42a5d3 |
| enabled     | True                             |
| id          | dae2d17668e242d9a639f637d21123ff |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | 7841afe520964a04aa50756aee42a5d3 |
+-------------+----------------------------------+
ogalush@ryunosuke:~$ 
```

demoユーザ
```
ogalush@ryunosuke:~$ openstack user create --domain default --password-prompt demo
User Password: <パスワードを入れる>
Repeat User Password: <パスワードを入れる>
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | 7841afe520964a04aa50756aee42a5d3 |
| enabled   | True                             |
| id        | 92b2fc5986d54c76b0a84c406a2edd1c |
| name      | demo                             |
+-----------+----------------------------------+
ogalush@ryunosuke:~$
```

userロール
```
ogalush@ryunosuke:~$ openstack role create user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | be209089532241d7b6671e6bef11f4d6 |
| name      | user                             |
+-----------+----------------------------------+
ogalush@ryunosuke:~$ 
```

demoユーザの紐付け
```
$ openstack role add --project demo --user demo user
```

### Verify operation
一時的に設定した環境変数を外して、正常動作するか試す.
```
ogalush@ryunosuke:~$  unset OS_TOKEN OS_URL
```

adminユーザの確認
```
ogalush@ryunosuke:~$ openstack --os-auth-url http://192.168.0.200:35357/v3 --os-project-domain-name default --os-user-domain-name default   --os-project-name admin --os-username admin token issue
Password: 
↓ tokenを取得できればOK.
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2016-04-23T08:49:42.458837Z                                                                                                                                                             |
| id         | gAAAAABXGykWOlteI4eehJrSd22dJxBxxDLXKYaelA1OnP0T_EYmBH18pgH3PWPFXCIXR2dN3_IKoAhHRrSo8K9HMsUuJqNhfl9ct9RKEg2yZFAfDiTONkEudbWUn91CV3eTFmmQ6obSExqTACydXN2lHEItw5uN_RqBWzPnVmSzSy6WC5KSoo0 |
| project_id | 290db0b57ea34427b01a6308d3f6e47c                                                                                                                                                        |
| user_id    | 45e936fbfe18477a9d083f902bbf0270                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
ogalush@ryunosuke:~$ 
```

demoユーザの確認
```
ogalush@ryunosuke:~$  openstack --os-auth-url http://192.168.0.200:5000/v3 --os-project-domain-name default --os-user-domain-name default  --os-project-name demo --os-username demo token issue
Password: 
↓ tokenを取得できればOK.
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2016-04-23T08:50:59.593447Z                                                                                                                                                             |
| id         | gAAAAABXGyljdemMLDNMq3xKvtgm1Jj6AjcQc6A3rMLsj9Qvt_0BztO0UQxvJeMgdyz95UdDi-D7Ox4VSaqTn5_8BgjL1KMtpWM7-CWRIX1WqMDrhht3RP4sz3jMviLLjDo40_OShdyJSk8v_if-UgVSMO_ABmKTPKtqQ0qjFAVOoH0tasFsZ3g |
| project_id | dae2d17668e242d9a639f637d21123ff                                                                                                                                                        |
| user_id    | 92b2fc5986d54c76b0a84c406a2edd1c                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
ogalush@ryunosuke:~$ 
```

### Create OpenStack client environment scripts
admin, demo両方の環境を操作できるように、環境変数を設定したファイルを準備する。

admin環境
```
$ tee ~/admin-openrc << 'EOT'
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_AUTH_URL=http://192.168.0.200:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOT

$ cat ~/admin-openrc
```
demo環境
```
tee ~/demo-openrc << 'EOT'
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://192.168.0.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOT

$ cat ~/demo-openrc 
```

確認
```
ogalush@ryunosuke:~$ source admin-openrc 
ogalush@ryunosuke:~$ openstack token issue
↓ tokenを取得できればOK.
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2016-04-23T08:55:28.409389Z                                                                                                                                                             |
| id         | gAAAAABXGypwgNQkDk9FufXPDbi8sh9Azg5m03WrLcUvQo93Zi-jdJ-qycUgf9xR7uAjOj1nHiU0rcm02e3_pstrEBGekHYoS5tLs0B-Ewegj_sWvbcljjmwqACFupEcL4okLd3NO8X2MeSK-EqPzjfKfsk9zSrdqvcQd7-PL-6nEV5rG6JbShA |
| project_id | 290db0b57ea34427b01a6308d3f6e47c                                                                                                                                                        |
| user_id    | 45e936fbfe18477a9d083f902bbf0270                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

ogalush@ryunosuke:~$ source demo-openrc 
ogalush@ryunosuke:~$ openstack token issue
↓ tokenを取得できればOK.
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2016-04-23T08:55:39.566807Z                                                                                                                                                             |
| id         | gAAAAABXGyp785XypHlt4U6HJpiuxUrBk-SsyJX5VvCZv2xSv8ki6_mI7py2NUJ2j-foSQSAYavcN6YBaGS3y4Id9NzhjI_a46XEndFZG6zY0v8D-OgdD0DOiheRxTaWK_GdMDxLHGcbJPTlVM3iAa_z57GgHqpEJ6lRaHwZjBmyPMrCvCGqn0k |
| project_id | dae2d17668e242d9a639f637d21123ff                                                                                                                                                        |
| user_id    | 92b2fc5986d54c76b0a84c406a2edd1c                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
ogalush@ryunosuke:~$ 
```

## Image service
### Install and configure
MySQL
```
$ sudo mysql -u root -p
MariaDB [(none)]> CREATE DATABASE glance;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> quit;
```

Keystone設定
```
$ source admin-openrc 
$ openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | 7841afe520964a04aa50756aee42a5d3 |
| enabled   | True                             |
| id        | 7939cfe6ce874cc9ab9fea0d71a95f2b |
| name      | glance                           |
+-----------+----------------------------------+

$ openstack role add --project service --user glance admin
$ openstack service create --name glance --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 45b639dad4ed43a7974b676c7af11582 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne image public http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 9c1298cf5fcd44b88f329f3a8b03cad3 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 45b639dad4ed43a7974b676c7af11582 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image internal http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 518d2308b7cf48748105a0683a19e6c1 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 45b639dad4ed43a7974b676c7af11582 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image admin http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | bb73a570eea349bc8bd3d387a2f373bc |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 45b639dad4ed43a7974b676c7af11582 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+
```

### Install and configure components
インストール
```
$ sudo apt-get -y install glance
```

設定
```
$ sudo vi /etc/glance/glance-api.conf
----
[database]
...
connection = mysql+pymysql://glance:password@192.168.0.200/glance

[keystone_authtoken]
...
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = password

[paste_deploy]
...
flavor = keystone

[glance_store]
...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
----

$ sudo vi /etc/glance/glance-registry.conf
----
[database]
connection = mysql+pymysql://glance:password@192.168.0.200/glance
...

[keystone_authtoken]
...
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = password

[paste_deploy]
...
flavor = keystone
----
```

反映
```
$ sudo bash -c "glance-manage db_sync" glance
$ sudo service glance-registry restart
$ sudo service glance-registry status
● glance-registry.service - OpenStack Image Service Registry
   Loaded: loaded (/lib/systemd/system/glance-registry.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2016-04-23 17:12:28 JST; 2s ago
  Process: 18302 ExecStartPre=/bin/chown glance:glance /var/lock/glance /var/log/glance /var/lib/glance (code=exited, status=0/
  Process: 18299 ExecStartPre=/bin/mkdir -p /var/lock/glance /var/log/glance /var/lib/glance (code=exited, status=0/SUCCESS)
 Main PID: 18306 (glance-registry)
    Tasks: 5 (limit: 512)
   Memory: 110.9M
      CPU: 836ms
   CGroup: /system.slice/glance-registry.service


$ sudo service glance-api restart
$ sudo service glance-api status
● glance-api.service - OpenStack Image Service API
   Loaded: loaded (/lib/systemd/system/glance-api.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2016-04-23 17:13:03 JST; 2s ago
  Process: 18353 ExecStartPre=/bin/chown glance:glance /var/lock/glance /var/log/glance /var/lib/glance (code=exited, status=0/
  Process: 18350 ExecStartPre=/bin/mkdir -p /var/lock/glance /var/log/glance /var/lib/glance (code=exited, status=0/SUCCESS)
 Main PID: 18356 (glance-api)
    Tasks: 5 (limit: 512)
   Memory: 128.9M
      CPU: 1.100s
   CGroup: /system.slice/glance-api.service
```

### Verify operation
```
$ source ~/admin-openrc
$ wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
$ openstack image create "cirros" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6                     |
| container_format | bare                                                 |
| created_at       | 2016-04-23T08:15:27Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/e7b8f2d9-7612-499f-9f09-01016ebb08e5/file |
| id               | e7b8f2d9-7612-499f-9f09-01016ebb08e5                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | 290db0b57ea34427b01a6308d3f6e47c                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13287936                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2016-04-23T08:15:28Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+

$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| e7b8f2d9-7612-499f-9f09-01016ebb08e5 | cirros | active |
+--------------------------------------+--------+--------+
```

## Compute Service
### Install and configure controller node
MySQL設定
```
$ sudo mysql -u root -p

MariaDB [(none)]> CREATE DATABASE nova_api;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> CREATE DATABASE nova;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> quit;
Bye
```

keystone設定
```
$ source ~/admin-openrc 
$ openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | 7841afe520964a04aa50756aee42a5d3 |
| enabled   | True                             |
| id        | c08307bc05064d6d971a9bd3903cf13e |
| name      | nova                             |
+-----------+----------------------------------+

$ openstack role add --project service --user nova admin

$ openstack service create --name nova --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 3e2ad499428d4352abb8c415a5e83dcb |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute public http://192.168.0.200:8774/v2.1/%\(tenant_id\)s
+--------------+----------------------------------------------+
| Field        | Value                                        |
+--------------+----------------------------------------------+
| enabled      | True                                         |
| id           | 42f6abc309444aef98f2dde136182f1e             |
| interface    | public                                       |
| region       | RegionOne                                    |
| region_id    | RegionOne                                    |
| service_id   | 3e2ad499428d4352abb8c415a5e83dcb             |
| service_name | nova                                         |
| service_type | compute                                      |
| url          | http://192.168.0.200:8774/v2.1/%(tenant_id)s |
+--------------+----------------------------------------------+

$ openstack endpoint create --region RegionOne compute internal http://192.168.0.200:8774/v2.1/%\(tenant_id\)s
+--------------+----------------------------------------------+
| Field        | Value                                        |
+--------------+----------------------------------------------+
| enabled      | True                                         |
| id           | 848560a583aa435ba4b769294c09ce0f             |
| interface    | internal                                     |
| region       | RegionOne                                    |
| region_id    | RegionOne                                    |
| service_id   | 3e2ad499428d4352abb8c415a5e83dcb             |
| service_name | nova                                         |
| service_type | compute                                      |
| url          | http://192.168.0.200:8774/v2.1/%(tenant_id)s |
+--------------+----------------------------------------------+

$ openstack endpoint create --region RegionOne compute admin http://192.168.0.200:8774/v2.1/%\(tenant_id\)s
+--------------+----------------------------------------------+
| Field        | Value                                        |
+--------------+----------------------------------------------+
| enabled      | True                                         |
| id           | 0691148ee67d414488e83e1a924b983b             |
| interface    | admin                                        |
| region       | RegionOne                                    |
| region_id    | RegionOne                                    |
| service_id   | 3e2ad499428d4352abb8c415a5e83dcb             |
| service_name | nova                                         |
| service_type | compute                                      |
| url          | http://192.168.0.200:8774/v2.1/%(tenant_id)s |
+--------------+----------------------------------------------+
```

### Install and configure components
インストール
```
$ sudo apt-get -y install nova-api nova-conductor nova-consoleauth nova-novncproxy nova-scheduler
```

設定
```
$ sudo vi /etc/nova/nova.conf
----
[DEFAULT]
...
### enabled_apis=ec2,osapi_compute,metadata
enabled_apis = osapi_compute,metadata
rpc_backend = rabbit
auth_strategy = keystone
my_ip = 192.168.0.200
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
connection = mysql+pymysql://nova:password@192.168.0.200/nova_api

[database]
connection = mysql+pymysql://nova:password@192.168.0.200/nova

[oslo_messaging_rabbit]
rabbit_host = 192.168.0.200
rabbit_userid = openstack
rabbit_password = password


[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = password

[vnc]
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip

[glance]
api_servers = http://192.168.0.200:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp
----
```

反映
```
$ sudo bash -c "nova-manage api_db sync" nova
$ sudo bash -c "nova-manage db sync" nova
```

nova再起動
```
sudo service nova-api restart
sudo service nova-consoleauth restart
sudo service nova-scheduler restart
sudo service nova-conductor restart
sudo service nova-novncproxy restart
```

### Install and configure a compute node
インストール
```
$ sudo apt-get -y install nova-compute
```

設定
```
$ sudo vi /etc/nova/nova.conf
----
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone
my_ip = 192.168.0.200
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[oslo_messaging_rabbit]
rabbit_host = 192.168.0.200
rabbit_userid = openstack
rabbit_password = password

[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = password

[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html

[glance]
api_servers = http://192.168.0.200:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp
----
```

確認
```
$ egrep -c '(vmx|svm)' /proc/cpuinfo
4 ・・・ 1 以上なら仮想化可能。

$ sudo vi /etc/nova/nova-compute.conf
----
[libvirt]
virt_type=kvm
----

$ sudo service nova-compute restart
$ sudo service nova-compute status
● nova-compute.service - OpenStack Compute
   Loaded: loaded (/lib/systemd/system/nova-compute.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2016-04-23 19:01:32 JST; 2s ago
  Process: 25044 ExecStartPre=/bin/chown nova:nova /var/lock/nova /var/log/nova /var/lib/nova (code=exited, 
  Process: 25041 ExecStartPre=/bin/mkdir -p /var/lock/nova /var/log/nova /var/lib/nova (code=exited, status=
 Main PID: 25049 (nova-compute)
    Tasks: 22 (limit: 512)
   Memory: 133.6M
      CPU: 2.236s
   CGroup: /system.slice/nova-compute.service
```

### Verify operation
確認
```
$ source ~/admin-openrc
$ openstack compute service list
+----+------------------+-----------+----------+---------+-------+----------------------------+
| Id | Binary           | Host      | Zone     | Status  | State | Updated At                 |
+----+------------------+-----------+----------+---------+-------+----------------------------+
|  4 | nova-consoleauth | ryunosuke | internal | enabled | up    | 2016-04-23T10:02:49.000000 |
|  5 | nova-scheduler   | ryunosuke | internal | enabled | up    | 2016-04-23T10:02:52.000000 |
|  6 | nova-conductor   | ryunosuke | internal | enabled | up    | 2016-04-23T10:02:52.000000 |
|  7 | nova-compute     | ryunosuke | nova     | enabled | up    | 2016-04-23T10:02:50.000000 |
+----+------------------+-----------+----------+---------+-------+----------------------------+

```

## Networking service
### Install and configure controller node
MySQL設定
```
$ sudo mysql -u root -p
MariaDB [(none)]> CREATE DATABASE neutron;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> quit;
Bye

$ source ~/admin-openrc
$ openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | 7841afe520964a04aa50756aee42a5d3 |
| enabled   | True                             |
| id        | e262e169b2d34a1fb8636a92c397f7cc |
| name      | neutron                          |
+-----------+----------------------------------+

$ openstack role add --project service --user neutron admin
```

Keystone設定
```
$ openstack service create --name neutron  --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 72e2ce57ac094b159ad8bad4c6ff1b2c |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne network public http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 18369b0a0d1e43579cdade3158583bf4 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 72e2ce57ac094b159ad8bad4c6ff1b2c |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network internal http://192.168.0.200:9696
 
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 7b368df09b8a430e949f21ff0c6330f0 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 72e2ce57ac094b159ad8bad4c6ff1b2c |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network admin http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 3f44474fe1414a71a680295a328a5c60 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 72e2ce57ac094b159ad8bad4c6ff1b2c |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+
```

### Networking Option 2: Self-service networks
インストール
```
$ sudo apt-get -y install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent  neutron-metadata-agent
```

設定
```
$ sudo vi /etc/neutron/neutron.conf
----
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
rpc_backend = rabbit
auth_strategy = keystone
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True


[database]
### connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql+pymysql://neutron:password@192.168.0.200/neutron

[oslo_messaging_rabbit]
rabbit_host = 192.168.0.200
rabbit_userid = openstack
rabbit_password = password

[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password

[nova]
auth_url = http://192.168.0.200:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = password
----

$ sudo vi /etc/neutron/plugins/ml2/ml2_conf.ini
----
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[ml2_type_vxlan]
vni_ranges = 1:1000

[securitygroup]
enable_ipset = True
----
```

### Configure the Linux bridge agent
設定
```
$ sudo vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini 
----
[linux_bridge]
physical_interface_mappings = provider:enp3s0

[vxlan]
enable_vxlan = True
local_ip = 192.168.0.200
l2_population = True

[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
----
```

### Configure the layer-3 agent
設定
```
$ sudo vi /etc/neutron/l3_agent.ini
----
[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
external_network_bridge =
----
```

### Configure the DHCP agent
設定
```
$ sudo vi /etc/neutron/dhcp_agent.ini
----
[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
----
```

### Configure the metadata agent
設定
```
$ sudo vi /etc/neutron/metadata_agent.ini
----
[DEFAULT]
...
nova_metadata_ip = 192.168.0.200
metadata_proxy_shared_secret = password
----
```

### Configure Compute to use Networking¶
```
$ sudo vi /etc/nova/nova.conf
----
[neutron]
url = http://192.168.0.200:9696
auth_url = http://192.168.0.200:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = password

service_metadata_proxy = True
metadata_proxy_shared_secret = password
----
```

反映
```
$ sudo bash -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

再起動
```
sudo service nova-api restart
sudo service neutron-server restart
sudo service neutron-linuxbridge-agent restart
sudo service neutron-dhcp-agent restart
sudo service neutron-metadata-agent restart
sudo service neutron-l3-agent restart
```

### Install and configure compute node
インストール
```
$ sudo apt-get -y install neutron-linuxbridge-agent
```

設定
```
$ sudo vi /etc/neutron/neutron.conf 
----
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone

[oslo_messaging_rabbit]
rabbit_host = 192.168.0.200
rabbit_userid = openstack
rabbit_password = password

[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password
----
```

### Networking Option 2: Self-service networks
```
$ sudo vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
----
[linux_bridge]
physical_interface_mappings = provider:enp3s0

[vxlan]
enable_vxlan = True
local_ip = 192.168.0.200
l2_population = True

[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
----
```

### Configure Compute to use Networking
設定
```
$ sudo vi /etc/nova/nova.conf
----
[neutron]
...
url = http://192.168.0.200:9696
auth_url = http://192.168.0.200:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = password
----
```

反映
```
$ sudo service nova-compute restart
$ sudo service neutron-linuxbridge-agent restart
```

### Verify operation
確認
```
$ source ~/admin-openrc
$ neutron ext-list
+---------------------------+-----------------------------------------------+
| alias                     | name                                          |
+---------------------------+-----------------------------------------------+
| default-subnetpools       | Default Subnetpools                           |
| network-ip-availability   | Network IP Availability                       |
| network_availability_zone | Network Availability Zone                     |
| auto-allocated-topology   | Auto Allocated Topology Services              |
| ext-gw-mode               | Neutron L3 Configurable external gateway mode |
| binding                   | Port Binding                                  |
| agent                     | agent                                         |
| subnet_allocation         | Subnet Allocation                             |
| l3_agent_scheduler        | L3 Agent Scheduler                            |
| tag                       | Tag support                                   |
| external-net              | Neutron external network                      |
| net-mtu                   | Network MTU                                   |
| availability_zone         | Availability Zone                             |
| quotas                    | Quota management support                      |
| l3-ha                     | HA Router extension                           |
| provider                  | Provider Network                              |
| multi-provider            | Multi Provider Network                        |
| address-scope             | Address scope                                 |
| extraroute                | Neutron Extra Route                           |
| timestamp_core            | Time Stamp Fields addition for core resources |
| router                    | Neutron L3 Router                             |
| extra_dhcp_opt            | Neutron Extra DHCP opts                       |
| dns-integration           | DNS Integration                               |
| security-group            | security-group                                |
| dhcp_agent_scheduler      | DHCP Agent Scheduler                          |
| router_availability_zone  | Router Availability Zone                      |
| rbac-policies             | RBAC Policies                                 |
| standard-attr-description | standard-attr-description                     |
| port-security             | Port Security                                 |
| allowed-address-pairs     | Allowed Address Pairs                         |
| dvr                       | Distributed Virtual Router                    |
+---------------------------+-----------------------------------------------+
~~~ 表示されればOK.
```

### Networking Option 2: Self-service networks
```
$ neutron agent-list
+--------------------------------------+--------------------+-----------+-------------------+-------+----------------+---------------------------+
| id                                   | agent_type         | host      | availability_zone | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-----------+-------------------+-------+----------------+---------------------------+
| 0bee23a5-f0b9-4ce4-9447-a01231d0844e | L3 agent           | ryunosuke | nova              | :-)   | True           | neutron-l3-agent          |
| 3f6287b9-6135-4562-a093-de902b913906 | Metadata agent     | ryunosuke |                   | :-)   | True           | neutron-metadata-agent    |
| 81819226-9d6d-4853-93e5-0a2adc592e2d | Linux bridge agent | ryunosuke |                   | :-)   | True           | neutron-linuxbridge-agent |
| b3b0a32a-7b18-4604-ab8c-e25384cabc59 | DHCP agent         | ryunosuke | nova              | :-)   | True           | neutron-dhcp-agent        |
+--------------------------------------+--------------------+-----------+-------------------+-------+----------------+---------------------------+
```
