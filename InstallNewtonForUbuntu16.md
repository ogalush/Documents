## Install OpenStack Newton on Ubuntu 16.04

ドキュメント: [OpenStack Docs](http://docs.openstack.org/newton/install-guide-ubuntu/)  
インストール先: ryunosuke(192.168.0.200)  
Repo: https://github.com/ogalush/Newton-InstallConfigs
```
ogalush@livaserver:~$ ssh -c aes128-ctr -A ogalush@192.168.0.200
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-47-generic x86_64)
ogalush@ryunosuke:~$ uname -a
Linux ryunosuke 4.4.0-47-generic #68-Ubuntu SMP Wed Oct 26 19:39:52 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
ogalush@ryunosuke:~$ 
```

## Environment
### Disable ipv6
ipv6が入っているとインスタンスの起動が遅かったりするので、無効化する.
```
$ sudo vim /etc/sysctl.conf
----
## For OpenStack
net.ipv6.conf.all.disable_ipv6 = 1
----
$ sudo sysctl -p
```

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
...

$ sudo apt -y install python-openstackclient
→ PythonクライアントがインストールされればOK.
```

#### Finalize the installation
```
$ sudo apt -y update && sudo apt -y upgrade && sudo apt -y dist-upgrade && sudo apt -y autoremove
→ 通常のPackageUpdateが行われる。
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
###character-set-server  = utf8mb4
###collation-server      = utf8mb4_general_ci
collation-server = utf8_general_ci
character-set-server = utf8
...
## for OpenStack Settings.
default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
----

※mysqldの文字コードが、デフォルト設定でutf8mb4になっている。utf8の4Byte文字も許容するCodeの模様。
　http://dev.classmethod.jp/cloud/aws/utf8mb4-on-rds-mysql/

[memo] MySQLの文字コードをデフォルトのutf8mb4で実施すると、あとでErrorになる。
Document通りutf8を使用する。
----
・出力されたエラーメッセージ
ogalush@ryunosuke:~$ sudo bash -c "keystone-manage db_sync" keystone
2016-11-26 18:16:39.046 14288 ERROR oslo_db.sqlalchemy.exc_filters [-] DBAPIError exception wrapped from (pymysql.err.InternalError) (1071, u'Specified key was too long; max key length is 767 bytes') [SQL: u'\nCREATE TABLE migrate_version (\n\trepository_id VARCHAR(250) NOT NULL, \n\trepository_path TEXT, \n\tversion INTEGER, \n\tPRIMARY KEY (repository_id)\n)\n\n']

・Newton on 16.04 Xenial and mysql's new default utf8mb4 charset 
https://bugs.launchpad.net/openstack-manuals/+bug/1575688
----

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


## Identity service
認証サービスであるKeyStoneをインストールする。
### Install and configure
#### Prerequisites
```
MySQLの設定
$ sudo mysql -u root
MariaDB [(none)]> CREATE DATABASE keystone;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> quit;
```

#### Install and configure components
```
インストール
$ sudo apt -y install keystone

設定
$ sudo vi /etc/keystone/keystone.conf
-----
[database]
## connection = sqlite:////var/lib/keystone/keystone.db
connection = mysql+pymysql://keystone:password@192.168.0.200/keystone
...
[token]
provider = fernet
-----

反映
$ sudo bash -c "keystone-manage db_sync" keystone

keystone設定
$ sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage bootstrap --bootstrap-password password --bootstrap-admin-url http://192.168.0.200:35357/v3/ --bootstrap-internal-url http://192.168.0.200:35357/v3/ --bootstrap-public-url http://192.168.0.200:5000/v3/ --bootstrap-region-id RegionOne
```

#### Configure the Apache HTTP server
```
httpd設定
$ echo 'ServerName 192.168.0.200' | sudo tee -a /etc/apache2/apache2.conf 

不要ファイル削除
$ sudo rm -vf /var/lib/keystone/keystone.db
removed '/var/lib/keystone/keystone.db'

反映
$ sudo service apache2 restart
$ sudo service apache2 status
● apache2.service - LSB: Apache2 web server
   Loaded: loaded (/etc/init.d/apache2; bad; vendor preset: enabled)
  Drop-In: /lib/systemd/system/apache2.service.d
           └─apache2-systemd.conf
   Active: active (running) since Sat 2016-11-26 18:33:31 JST; 2s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 15071 ExecStop=/etc/init.d/apache2 stop (code=exited, status=0/SUCCESS)
  Process: 15105 ExecStart=/etc/init.d/apache2 start (code=exited, status=0/SUCCESS)
    Tasks: 95
   Memory: 37.3M
      CPU: 130ms
   CGroup: /system.slice/apache2.service
           ├─15123 /usr/sbin/apache2 -k start
           ├─15126 (wsgi:keystone-pu -k start
           ├─15127 (wsgi:keystone-pu -k start
           ├─15128 (wsgi:keystone-pu -k start
           ├─15129 (wsgi:keystone-pu -k start
```

#### Finalize the installation
環境変数設定
```
$ export OS_USERNAME=admin
$ export OS_PASSWORD=password
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=default
$ export OS_PROJECT_DOMAIN_NAME=default
$ export OS_AUTH_URL=http://192.168.0.200:35357/v3
$ export OS_IDENTITY_API_VERSION=3
```

### Create a domain, projects, users, and roles
```
Project作成
$ openstack project create --domain default --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 140a73faaa904dc49b1b21b2cecd1db2 |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
+-------------+----------------------------------+

$ openstack project create --domain default --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | f79866ddad734d739c7fd0511d284b55 |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | default                          |
+-------------+----------------------------------+

User作成
$ openstack user create --domain default --password-prompt demo
User Password: password
Repeat User Password: password
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 28e270123c6b467fb35d5c74550e6a26 |
| name                | demo                             |
| password_expires_at | None                             |
+---------------------+----------------------------------+

Role作成
$ openstack role create user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 146d43aa973a49c793495fe3bb71f524 |
| name      | user                             |
+-----------+----------------------------------+

Role追加
$ openstack role add --project demo --user demo user

```

### Verify operation
```
・admin user, request an authentication token:
$ unset OS_AUTH_URL OS_PASSWORD
$ openstack --os-auth-url http://192.168.0.200:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
Password: password
+------------+----------------------------------------------------------------------------------------------+
| Field      | Value                                                                                        |
+------------+----------------------------------------------------------------------------------------------+
| expires    | 2016-11-26 10:44:26+00:00                                                                    |
| id         | gAAAAABYOVl6o6lPIcKXVyEQebeA8pfJ2hKLHKlxIfQMZsaMTW53x2_JARdXMP2oqInZmLzP1BHjWklKomrY5a-wgFRB |
|            | FP4VYbJGlNWEZbJOMuQVviHonftnWUCrgVBQxPANoJ2vHYFuEP7qy40l4kOlGjnFqrntxyqSZ104C6oh5mthOldMvaQ  |
| project_id | 7d691f1c7898412d9d9fe831ca484032                                                             |
| user_id    | 49f7f89501284e82ad403b392d0effbc                                                             |
+------------+----------------------------------------------------------------------------------------------+

・As the demo user, request an authentication token:
$ openstack --os-auth-url http://192.168.0.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name demo --os-username demo token issue
Password: password
+------------+----------------------------------------------------------------------------------------------+
| Field      | Value                                                                                        |
+------------+----------------------------------------------------------------------------------------------+
| expires    | 2016-11-26 10:45:59+00:00                                                                    |
| id         | gAAAAABYOVnXVreD4xHixg8iAA9PwlBOBw9QWH10wzKVnmjmkKa83fnIWY8Dy3WQ-G46AiJ9GPqJFWRbbkbyiXduw6nm |
|            | xFDdWxxV1MiPO6y5W8dqrKpfBvYzvJJvj2Z0S28tK6FJg9NvEKHxTqDt_VBKntls6jeAzlrsOo0SKhXY50ld-55l0-Q  |
| project_id | f79866ddad734d739c7fd0511d284b55                                                             |
| user_id    | 28e270123c6b467fb35d5c74550e6a26                                                             |
+------------+----------------------------------------------------------------------------------------------+
```

### Create OpenStack client environment scripts
OpenStackを操作するための環境設定ファイルを作成する。
```
・adminユーザ用
$ echo 'export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.0.200:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2' > ~/admin-openrc

$ source ~/admin-openrc
$ openstack token issue
+------------+----------------------------------------------------------------------------------------------+
| Field      | Value                                                                                        |
+------------+----------------------------------------------------------------------------------------------+
| expires    | 2016-11-26 10:49:59+00:00                                                                    |
| id         | gAAAAABYOVrH27u9KToGo1zz_K0iJdEpCZ1yn66ucsB_GVVh_w-bfXql_wO2oYRkOSAT0G4a26GH9V-              |
|            | X8kMW6WZtjXoYYQHD6mHLDfWzZnX5uphaLraD1GuynL3PJDfQswHqXpFvnU-                                 |
|            | 1n53g6EkSaCFdqjO2B08NB3ZwX9lcjPJ_G9hmnBAsDN8                                                 |
| project_id | 7d691f1c7898412d9d9fe831ca484032                                                             |
| user_id    | 49f7f89501284e82ad403b392d0effbc                                                             |
+------------+----------------------------------------------------------------------------------------------+


・demoユーザ用
$ echo 'export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.0.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2' > ~/demo-openrc

$ source ~/demo-openrc
$ openstack token issue
+------------+----------------------------------------------------------------------------------------------+
| Field      | Value                                                                                        |
+------------+----------------------------------------------------------------------------------------------+
| expires    | 2016-11-26 10:50:43+00:00                                                                    |
| id         | gAAAAABYOVrzYn9eeQ-VU_5WdZGXZcjwhTxrhyIU3SeYtjYbFyoSbN0dHVUKdkAxcUqujjXNBYq4H-               |
|            | V_UUvQAoeqGCR8raJEC9phcF6Y--                                                                 |
|            | JLSvLlIV_QnvApWzScvYDmsB1E5nQYRCgx9FFB4GXCWAmOnXlM5ZvOjARUJ_UvKGb67CXWVbeCMro                |
| project_id | f79866ddad734d739c7fd0511d284b55                                                             |
| user_id    | 28e270123c6b467fb35d5c74550e6a26                                                             |
+------------+----------------------------------------------------------------------------------------------+
```

## Image service
### Install and configure
#### Prerequisites
```
MySQL設定
$ sudo mysql -u root
MariaDB [(none)]> CREATE DATABASE glance;
Query OK, 1 row affected (0.00 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.00 sec)
MariaDB [(none)]> quit;
Bye

KeyStone設定
$ source ~/admin-openrc
$ openstack user create --domain default --password-prompt glance
User Password: password
Repeat User Password: password
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 63e3ca0770b24002b7cb103c5661b611 |
| name                | glance                           |
| password_expires_at | None                             |
+---------------------+----------------------------------+

$ openstack role add --project service --user glance admin
$ openstack service create --name glance --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 9f038111b95a41c8984653c184bc9faf |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne image public http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 24687c46a54d4e29b1a057878937f7ac |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 9f038111b95a41c8984653c184bc9faf |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image internal http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 7145be636ab544ae9c2a8456c6ef4d1e |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 9f038111b95a41c8984653c184bc9faf |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image admin http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 4f8bb292030e45e1995dd33158da810f |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 9f038111b95a41c8984653c184bc9faf |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+
```

#### Install and configure components
```
インストール
$ sudo apt -y install glance

設定
$ sudo vi /etc/glance/glance-api.conf
----
[database]
...
connection = mysql+pymysql://glance:password@192.168.0.200/glance
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = password
...
[paste_deploy]
flavor = keystone
...

[glance_store]
stores = file,http
default_store = file 
filesystem_store_datadir = /var/lib/glance/images/
----

$ sudo vi /etc/glance/glance-registry.conf
----
[database]
...
connection = mysql+pymysql://glance:password@192.168.0.200/glance
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = password
...

[paste_deploy]
flavor = keystone
----

反映
$ sudo bash -c "glance-manage db_sync" glance
```

#### Finalize installation
```
$ sudo service glance-registry restart
$ sudo service glance-registry status
● glance-registry.service - OpenStack Image Service Registry
   Loaded: loaded (/lib/systemd/system/glance-registry.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2016-11-26 19:11:23 JST; 4s ago
  Process: 17021 ExecStartPre=/bin/chown glance:adm /var/log/glance (code=exited, status=0/SUCCESS)
  Process: 17018 ExecStartPre=/bin/chown glance:glance /var/lock/glance /var/lib/glance (code=exited, status=
  Process: 17014 ExecStartPre=/bin/mkdir -p /var/lock/glance /var/log/glance /var/lib/glance (code=exited, st
 Main PID: 17025 (glance-registry)
    Tasks: 5

$ sudo service glance-api restart
$ sudo service glance-api status
● glance-api.service - OpenStack Image Service API
   Loaded: loaded (/lib/systemd/system/glance-api.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2016-11-26 19:11:47 JST; 1s ago
  Process: 17080 ExecStartPre=/bin/chown glance:adm /var/log/glance (code=exited, status=0/SUCCESS)
  Process: 17074 ExecStartPre=/bin/chown glance:glance /var/lock/glance /var/lib/glance (code=exited, status=
  Process: 17071 ExecStartPre=/bin/mkdir -p /var/lock/glance /var/log/glance /var/lib/glance (code=exited, st
 Main PID: 17084 (glance-api)
    Tasks: 5
```

### Verify operation
```
・準備
$ source ~/admin-openrc

・Cirros
$ wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
$ openstack image create "cirros" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6                     |
| container_format | bare                                                 |
| created_at       | 2016-11-26T10:13:22Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/dc894552-ee2a-4938-a4de-a5eb34323215/file |
| id               | dc894552-ee2a-4938-a4de-a5eb34323215                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | 7d691f1c7898412d9d9fe831ca484032                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13287936                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2016-11-26T10:13:22Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+

・Ubuntu
$ wget https://cloud-images.ubuntu.com/releases/16.04/release/ubuntu-16.04-server-cloudimg-amd64-disk1.img
$ $ openstack image create "ubuntu16.04" --file ubuntu-16.04-server-cloudimg-amd64-disk1.img --disk-format qcow2 --container-format bare --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | 2957423af77cd18f2bdb4c6ef47e4597                     |
| container_format | bare                                                 |
| created_at       | 2016-11-26T10:16:22Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/53640dba-24a1-4370-aa29-ff9c674e7495/file |
| id               | 53640dba-24a1-4370-aa29-ff9c674e7495                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | ubuntu16.04                                          |
| owner            | 7d691f1c7898412d9d9fe831ca484032                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 314179584                                            |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2016-11-26T10:16:23Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+

・確認
$ openstack image list
+--------------------------------------+-------------+--------+
| ID                                   | Name        | Status |
+--------------------------------------+-------------+--------+
| 53640dba-24a1-4370-aa29-ff9c674e7495 | ubuntu16.04 | active |
| dc894552-ee2a-4938-a4de-a5eb34323215 | cirros      | active |
+--------------------------------------+-------------+--------+
```


## Compute service
### Install and configure controller node
#### Prerequisites
```
MySQL準備
$ sudo mysql -u root

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

KeyStone設定
$ source ~/admin-openrc
$ openstack user create --domain default --password-prompt nova
User Password: password
Repeat User Password: password
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 47c29cb7f07d4eda8898235eb48493e3 |
| name                | nova                             |
| password_expires_at | None                             |
+---------------------+----------------------------------+

$ openstack role add --project service --user nova admin
$ openstack service create --name nova --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 925ad5204b244d0e8a09068706dd728c |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute public http://192.168.0.200:8774/v2.1/%\(tenant_id\)s
+--------------+----------------------------------------------+
| Field        | Value                                        |
+--------------+----------------------------------------------+
| enabled      | True                                         |
| id           | 33071b000b3a47bbaf5a4dfc35682c89             |
| interface    | public                                       |
| region       | RegionOne                                    |
| region_id    | RegionOne                                    |
| service_id   | 925ad5204b244d0e8a09068706dd728c             |
| service_name | nova                                         |
| service_type | compute                                      |
| url          | http://192.168.0.200:8774/v2.1/%(tenant_id)s |
+--------------+----------------------------------------------+

$ openstack endpoint create --region RegionOne compute internal http://192.168.0.200:8774/v2.1/%\(tenant_id\)s
+--------------+----------------------------------------------+
| Field        | Value                                        |
+--------------+----------------------------------------------+
| enabled      | True                                         |
| id           | 9dfbe0f9779241b89a8dc4495da8b030             |
| interface    | internal                                     |
| region       | RegionOne                                    |
| region_id    | RegionOne                                    |
| service_id   | 925ad5204b244d0e8a09068706dd728c             |
| service_name | nova                                         |
| service_type | compute                                      |
| url          | http://192.168.0.200:8774/v2.1/%(tenant_id)s |
+--------------+----------------------------------------------+

$ openstack endpoint create --region RegionOne compute admin http://192.168.0.200:8774/v2.1/%\(tenant_id\)s
+--------------+----------------------------------------------+
| Field        | Value                                        |
+--------------+----------------------------------------------+
| enabled      | True                                         |
| id           | 7934cf084ee2419a841d4dcbc3d10318             |
| interface    | admin                                        |
| region       | RegionOne                                    |
| region_id    | RegionOne                                    |
| service_id   | 925ad5204b244d0e8a09068706dd728c             |
| service_name | nova                                         |
| service_type | compute                                      |
| url          | http://192.168.0.200:8774/v2.1/%(tenant_id)s |
+--------------+----------------------------------------------+
```

#### Install and configure components
```
インストール
$ sudo apt -y install nova-api nova-conductor nova-consoleauth nova-novncproxy nova-scheduler

設定
$ sudo vi /etc/nova/nova.conf
----
[DEFAULT]
...
transport_url = rabbit://openstack:password@192.168.0.200
auth_strategy = keystone
my_ip = 192.168.0.200
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
...
[database]
##connection=sqlite:////var/lib/nova/nova.sqlite
connection = mysql+pymysql://nova:password@192.168.0.200/nova

[api_database]
##connection=sqlite:////var/lib/nova/nova.sqlite
connection = mysql+pymysql://nova:password@192.168.0.200/nova_api
...

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
...

[oslo_concurrency]
##lock_path=/var/lock/nova
lock_path = /var/lib/nova/tmp
...
[vnc]
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip
vnc_keymap=ja

[glance]
api_servers = http://192.168.0.200:9292
----

反映
$ sudo bash -c "nova-manage api_db sync" nova
$ sudo bash -c "nova-manage db sync" nova
```

※vnc_keymapで日本語キーボード対応する。  
http://qiita.com/kure/items/a005bf5f9ddd18c54ad1

#### Finalize installation
```
$ echo 'nova-api
nova-consoleauth
nova-scheduler
nova-conductor
nova-novncproxy' | awk '{print "sudo service "$1" restart"}' | bash -x
```

### Install and configure a compute node
#### Install and configure components
```
インストール
$ sudo apt -y install nova-compute

設定
$ sudo vi /etc/nova/nova.conf
----
[DEFAULT]
...
transport_url = rabbit://openstack:password@192.168.0.200
auth_strategy = keystone
my_ip = 192.168.0.200
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

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
----

[vnc]
enabled = True
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html

[glance]
api_servers = http://192.168.0.200:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

```
#### Finalize installation
```
確認
$ egrep -c '(vmx|svm)' /proc/cpuinfo
4
→ 1のため、特に設定しなくてもOK.
If this command returns a value of one or greater, your compute node supports hardware acceleration which typically requires no additional configuration.

$ sudo view /etc/nova/nova-compute.conf
----
...
[libvirt]
virt_type=kvm
----

反映
$ sudo service nova-compute restart
```


### Verify operation
```
$ source ~/admin-openrc
$ openstack compute service list
+----+------------------+-----------+----------+---------+-------+----------------------------+
| ID | Binary           | Host      | Zone     | Status  | State | Updated At                 |
+----+------------------+-----------+----------+---------+-------+----------------------------+
|  4 | nova-consoleauth | ryunosuke | internal | enabled | up    | 2016-11-26T10:54:14.000000 |
|  5 | nova-scheduler   | ryunosuke | internal | enabled | up    | 2016-11-26T10:54:06.000000 |
|  6 | nova-conductor   | ryunosuke | internal | enabled | up    | 2016-11-26T10:54:06.000000 |
|  7 | nova-compute     | ryunosuke | nova     | enabled | up    | 2016-11-26T10:54:07.000000 |
+----+------------------+-----------+----------+---------+-------+----------------------------+
```

## Networking service
### Install and configure controller node

#### Prerequisites
```
MySQL準備
$ sudo mysql -u root
MariaDB [(none)]> CREATE DATABASE neutron;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> quit;
Bye

KeyStone準備
$ source ~/admin-openrc
$ openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | e1503179ab0b49188a5355f8d39d6472 |
| name                | neutron                          |
| password_expires_at | None                             |
+---------------------+----------------------------------+

$ openstack role add --project service --user neutron admin
$ openstack service create --name neutron --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 47d3381f49a24ab2b94e588a1e735bf5 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne network public http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 89be3eebac28431886a60610b0a98cf0 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 47d3381f49a24ab2b94e588a1e735bf5 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network internal http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6a258ccb9d4e441489451ec487dde7a5 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 47d3381f49a24ab2b94e588a1e735bf5 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne  network admin http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 91769b0e6af44ad2aeaefa33faa7a795 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 47d3381f49a24ab2b94e588a1e735bf5 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+
```

#### Configure networking options
Option2を選ぶ。
Option1はAdminのみがIPアドレスなどを払い出す形となっており、自分自身でIPアドレスを使用したい用途と異なるため。
##### Networking Option 2: Self-service networks
###### Install the components
```
インストール
$ sudo apt -y install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent  neutron-metadata-agent
```

###### Configure the server component
```
設定
$sudo vi /etc/neutron/neutron.conf
----
[DEFAULT]
...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
transport_url = rabbit://openstack:password@192.168.0.200
auth_strategy = keystone
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True

[database]
...
## connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql+pymysql://neutron:password@192.168.0.200/neutron
...

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
...

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
```

###### Configure the Modular Layer 2 (ML2) plug-in
```
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

###### Configure the Linux bridge agent
```
$ sudo vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
----
[linux_bridge]
physical_interface_mappings = provider:enp3s0
...
[vxlan]
enable_vxlan = True
local_ip = 192.168.0.200
l2_population = True

[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
----
```

###### Configure the layer-3 agent
```
設定
$ sudo vi /etc/neutron/l3_agent.ini 
----
[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
----
```
###### Configure the DHCP agent
```
$ sudo vi /etc/neutron/dhcp_agent.ini
----
[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
----
```

#### Configure the metadata agent
```
$ sudo vi /etc/neutron/metadata_agent.ini
----
[DEFAULT]
...
nova_metadata_ip = 192.168.0.200
metadata_proxy_shared_secret = password
----
```

#### Configure the Compute service to use the Networking service
```
$ sudo vi /etc/nova/nova.conf
----
...
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

#### Finalize installation
```
設定
$ sudo bash -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

反映
$ echo 'nova-api
neutron-server
neutron-linuxbridge-agent
neutron-dhcp-agent
neutron-metadata-agent
neutron-l3-agent' | awk '{print "sudo service "$1" restart"}' |bash -x
```

### Install and configure compute node
以下、差分
```
echo 'nova-compute
neutron-linuxbridge-agent' | awk '{print "sudo service "$1 " restart"}' |bash -x
```

### Verify operation
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
| flavors                   | Neutron Service Flavors                       |
| net-mtu                   | Network MTU                                   |
| availability_zone         | Availability Zone                             |
| quotas                    | Quota management support                      |
| l3-ha                     | HA Router extension                           |
| provider                  | Provider Network                              |
| multi-provider            | Multi Provider Network                        |
| address-scope             | Address scope                                 |
| extraroute                | Neutron Extra Route                           |
| subnet-service-types      | Subnet service types                          |
| standard-attr-timestamp   | Resource timestamps                           |
| service-type              | Neutron Service Type Management               |
| l3-flavors                | Router Flavor Extension                       |
| port-security             | Port Security                                 |
| extra_dhcp_opt            | Neutron Extra DHCP opts                       |
| standard-attr-revisions   | Resource revision numbers                     |
| pagination                | Pagination support                            |
| sorting                   | Sorting support                               |
| security-group            | security-group                                |
| dhcp_agent_scheduler      | DHCP Agent Scheduler                          |
| router_availability_zone  | Router Availability Zone                      |
| rbac-policies             | RBAC Policies                                 |
| standard-attr-description | standard-attr-description                     |
| router                    | Neutron L3 Router                             |
| allowed-address-pairs     | Allowed Address Pairs                         |
| project-id                | project_id field enabled                      |
| dvr                       | Distributed Virtual Router                    |
+---------------------------+-----------------------------------------------+
```

#### Networking Option 2: Self-service networks
```
$ openstack network agent list
+----------------------+--------------------+-----------+-------------------+-------+-------+----------------------+
| ID                   | Agent Type         | Host      | Availability Zone | Alive | State | Binary               |
+----------------------+--------------------+-----------+-------------------+-------+-------+----------------------+
| 11bb11d7-60db-4ac8   | L3 agent           | ryunosuke | nova              | True  | UP    | neutron-l3-agent     |
| -ae6a-2c0c1dd9714d   |                    |           |                   |       |       |                      |
| 78e44279-cb2b-4097   | DHCP agent         | ryunosuke | nova              | True  | UP    | neutron-dhcp-agent   |
| -b53c-3e678b76b3af   |                    |           |                   |       |       |                      |
| a024d45d-568d-4553-a | Linux bridge agent | ryunosuke | None              | True  | UP    | neutron-linuxbridge- |
| 5d7-b91379903295     |                    |           |                   |       |       | agent                |
| b5a36da3-d4ee-40cc-b | Metadata agent     | ryunosuke | None              | True  | UP    | neutron-metadata-    |
| 811-3732d36e27ae     |                    |           |                   |       |       | agent                |
+----------------------+--------------------+-----------+-------------------+-------+-------+----------------------+
```

## Dashboard
### Install and configure
#### Install and configure components
```
インストール
$ sudo apt -y install openstack-dashboard

設定
$ sudo vi /etc/openstack-dashboard/local_settings.py
----
OPENSTACK_HOST = "192.168.0.200"
ALLOWED_HOSTS = ['*', ]
...
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': '192.168.0.200:11211',
    }
}
...
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "default"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
OPENSTACK_NEUTRON_NETWORK → defaultのまま
TIME_ZONE = "Asia/Tokyo"
----
```

#### Finalize installation
```
反映
$ sudo service apache2 reload
```

### Verify operation
OpenStack DashboardへログインできればOK.(admin, demoユーザ)  
http://192.168.0.200/horizon/

### Next steps
#### Launch an instance
##### Create virtual networks
Option2を選択したので、Provider network、Self-service networkの2つを作成する。
###### Provider network
```
$ source ~/admin-openrc

・Create Network
$ openstack network create --share --provider-physical-network provider --provider-network-type flat provider
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2016-11-26T12:17:58Z                 |
| description               |                                      |
| headers                   |                                      |
| id                        | b0ba4cc3-c71f-4fa6-98cf-9a516980fe66 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| mtu                       | 1500                                 |
| name                      | provider                             |
| port_security_enabled     | True                                 |
| project_id                | 7d691f1c7898412d9d9fe831ca484032     |
| project_id                | 7d691f1c7898412d9d9fe831ca484032     |
| provider:network_type     | flat                                 |
| provider:physical_network | provider                             |
| provider:segmentation_id  | None                                 |
| revision_number           | 3                                    |
| router:external           | Internal                             |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      | []                                   |
| updated_at                | 2016-11-26T12:17:59Z                 |
+---------------------------+--------------------------------------+

・Create Subnet
$ openstack subnet create --network provider --allocation-pool start=192.168.0.100,end=192.168.0.120 --dns-nameserver 192.168.0.254 --gateway 192.168.0.254 --subnet-range 192.168.0.0/24 provider
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 192.168.0.100-192.168.0.120          |
| cidr              | 192.168.0.0/24                       |
| created_at        | 2016-11-26T12:22:51Z                 |
| description       |                                      |
| dns_nameservers   | 192.168.0.254                        |
| enable_dhcp       | True                                 |
| gateway_ip        | 192.168.0.254                        |
| headers           |                                      |
| host_routes       |                                      |
| id                | 3adffac3-f23e-4ba8-ad7f-c820c27b2c6a |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | provider                             |
| network_id        | b0ba4cc3-c71f-4fa6-98cf-9a516980fe66 |
| project_id        | 7d691f1c7898412d9d9fe831ca484032     |
| project_id        | 7d691f1c7898412d9d9fe831ca484032     |
| revision_number   | 2                                    |
| service_types     | []                                   |
| subnetpool_id     | None                                 |
| updated_at        | 2016-11-26T12:22:51Z                 |
+-------------------+--------------------------------------+
```
###### Self-service network
```
$ source ~/demo-openrc
$ openstack network create selfservice
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | UP                                   |
| availability_zone_hints |                                      |
| availability_zones      |                                      |
| created_at              | 2016-11-26T12:23:59Z                 |
| description             |                                      |
| headers                 |                                      |
| id                      | a12d6bc6-7fc0-43e1-a08d-22a657cb007a |
| ipv4_address_scope      | None                                 |
| ipv6_address_scope      | None                                 |
| mtu                     | 1450                                 |
| name                    | selfservice                          |
| port_security_enabled   | True                                 |
| project_id              | f79866ddad734d739c7fd0511d284b55     |
| project_id              | f79866ddad734d739c7fd0511d284b55     |
| revision_number         | 3                                    |
| router:external         | Internal                             |
| shared                  | False                                |
| status                  | ACTIVE                               |
| subnets                 |                                      |
| tags                    | []                                   |
| updated_at              | 2016-11-26T12:23:59Z                 |
+-------------------------+--------------------------------------+

$ openstack subnet create --network selfservice --dns-nameserver 192.168.0.220 --gateway 10.0.0.1 --subnet-range 10.0.0.0/24 selfservice
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 10.0.0.2-10.0.0.254                  |
| cidr              | 10.0.0.0/24                          |
| created_at        | 2016-11-26T12:26:25Z                 |
| description       |                                      |
| dns_nameservers   | 192.168.0.220                        |
| enable_dhcp       | True                                 |
| gateway_ip        | 10.0.0.1                             |
| headers           |                                      |
| host_routes       |                                      |
| id                | 9dd4e12d-473b-4e09-a588-e6ef1c164864 |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | selfservice                          |
| network_id        | a12d6bc6-7fc0-43e1-a08d-22a657cb007a |
| project_id        | f79866ddad734d739c7fd0511d284b55     |
| project_id        | f79866ddad734d739c7fd0511d284b55     |
| revision_number   | 2                                    |
| service_types     | []                                   |
| subnetpool_id     | None                                 |
| updated_at        | 2016-11-26T12:26:25Z                 |
+-------------------+--------------------------------------+
```

###### Create a router
```
$ source ~/admin-openrc
$ neutron net-update provider --router:external
Updated network: provider

$ source ~/demo-openrc
$ openstack router create router
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | UP                                   |
| availability_zone_hints |                                      |
| availability_zones      |                                      |
| created_at              | 2016-11-26T12:28:34Z                 |
| description             |                                      |
| external_gateway_info   | null                                 |
| flavor_id               | None                                 |
| headers                 |                                      |
| id                      | 56574967-8a02-49c2-87e6-34598bb34846 |
| name                    | router                               |
| project_id              | f79866ddad734d739c7fd0511d284b55     |
| project_id              | f79866ddad734d739c7fd0511d284b55     |
| revision_number         | 2                                    |
| routes                  |                                      |
| status                  | ACTIVE                               |
| updated_at              | 2016-11-26T12:28:34Z                 |
+-------------------------+--------------------------------------+

$ neutron router-interface-add router selfservice
Added interface c685a59f-0a64-4a4e-ade4-f4d054ad97a8 to router router.

$ neutron router-gateway-set router provider
Set gateway for router router

確認
$ source ~/admin-openrc
$ ip netns
qrouter-56574967-8a02-49c2-87e6-34598bb34846 (id: 2)
qdhcp-a12d6bc6-7fc0-43e1-a08d-22a657cb007a (id: 1)
qdhcp-b0ba4cc3-c71f-4fa6-98cf-9a516980fe66 (id: 0)

$ neutron router-port-list router
+--------------------------------------+------+-------------------+----------------------------------------------+
| id                                   | name | mac_address       | fixed_ips                                    |
+--------------------------------------+------+-------------------+----------------------------------------------+
| 2c2f1878-611f-4a6e-9d32-f48741eada10 |      | fa:16:3e:99:e1:47 | {"subnet_id": "3adffac3-f23e-4ba8-ad7f-      |
|                                      |      |                   | c820c27b2c6a", "ip_address":                 |
|                                      |      |                   | "192.168.0.106"}                             |
| c685a59f-0a64-4a4e-ade4-f4d054ad97a8 |      | fa:16:3e:08:5a:a5 | {"subnet_id": "9dd4e12d-473b-                |
|                                      |      |                   | 4e09-a588-e6ef1c164864", "ip_address":       |
|                                      |      |                   | "10.0.0.1"}                                  |
+--------------------------------------+------+-------------------+----------------------------------------------+

$ ping -c 4 192.168.0.106
PING 192.168.0.106 (192.168.0.106) 56(84) bytes of data.
64 bytes from 192.168.0.106: icmp_seq=1 ttl=64 time=0.068 ms
64 bytes from 192.168.0.106: icmp_seq=2 ttl=64 time=0.036 ms
64 bytes from 192.168.0.106: icmp_seq=3 ttl=64 time=0.045 ms
64 bytes from 192.168.0.106: icmp_seq=4 ttl=64 time=0.043 ms

--- 192.168.0.106 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 2997ms
rtt min/avg/max/mdev = 0.036/0.048/0.068/0.012 ms
ogalush@ryunosuke:~$ 
```

##### Create m1.nano flavor
```
・Create Flavor
$ source ~/admin-openrc
$ openstack flavor create --id 0 --vcpus 1 --ram 1024 --disk 1 m1.nano
+----------------------------+---------+
| Field                      | Value   |
+----------------------------+---------+
| OS-FLV-DISABLED:disabled   | False   |
| OS-FLV-EXT-DATA:ephemeral  | 0       |
| disk                       | 1       |
| id                         | 0       |
| name                       | m1.nano |
| os-flavor-access:is_public | True    |
| properties                 |         |
| ram                        | 1024    |
| rxtx_factor                | 1.0     |
| swap                       |         |
| vcpus                      | 1       |
+----------------------------+---------+
```

##### Create m1.small flavor
普段利用するsizeのflavorを作成する.
```
$ openstack flavor create --id 1 --vcpus 2 --ram 4096 --disk 40 m1.small --swap 4096
+----------------------------+----------+
| Field                      | Value    |
+----------------------------+----------+
| OS-FLV-DISABLED:disabled   | False    |
| OS-FLV-EXT-DATA:ephemeral  | 0        |
| disk                       | 40       |
| id                         | 1        |
| name                       | m1.small |
| os-flavor-access:is_public | True     |
| properties                 |          |
| ram                        | 4096     |
| rxtx_factor                | 1.0      |
| swap                       | 4096     |
| vcpus                      | 2        |
+----------------------------+----------+
```

##### Generate a key pair
```
・Create keypair
$ source ~/demo-openrc
$ openstack keypair create --public-key ~/.ssh/id_rsa_chef.pub chefkey
+-------------+-------------------------------------------------+
| Field       | Value                                           |
+-------------+-------------------------------------------------+
| fingerprint | f2:70:ad:9c:93:32:14:c0:36:e3:4f:ac:19:45:82:d2 |
| name        | chefkey                                         |
| user_id     | 28e270123c6b467fb35d5c74550e6a26                |
+-------------+-------------------------------------------------+

$ openstack keypair list
+---------+-------------------------------------------------+
| Name    | Fingerprint                                     |
+---------+-------------------------------------------------+
| chefkey | f2:70:ad:9c:93:32:14:c0:36:e3:4f:ac:19:45:82:d2 |
+---------+-------------------------------------------------+
```

##### Add security group rules
```
$ openstack security group rule create --proto icmp default
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2016-11-26T12:35:30Z                 |
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| headers           |                                      |
| id                | e8dd1cf4-3db8-453b-8058-ed0fb2bf1c40 |
| port_range_max    | None                                 |
| port_range_min    | None                                 |
| project_id        | f79866ddad734d739c7fd0511d284b55     |
| project_id        | f79866ddad734d739c7fd0511d284b55     |
| protocol          | icmp                                 |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 1                                    |
| security_group_id | 8c8c4c7a-a3f6-4bb9-9bf2-7225adcf5f00 |
| updated_at        | 2016-11-26T12:35:30Z                 |
+-------------------+--------------------------------------+

$ openstack security group rule create --proto tcp --dst-port 22 default
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2016-11-26T12:36:08Z                 |
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| headers           |                                      |
| id                | 22d42f1b-0e14-4485-97cc-0285ef6cc85a |
| port_range_max    | 22                                   |
| port_range_min    | 22                                   |
| project_id        | f79866ddad734d739c7fd0511d284b55     |
| project_id        | f79866ddad734d739c7fd0511d284b55     |
| protocol          | tcp                                  |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 1                                    |
| security_group_id | 8c8c4c7a-a3f6-4bb9-9bf2-7225adcf5f00 |
| updated_at        | 2016-11-26T12:36:08Z                 |
+-------------------+--------------------------------------+
```

##### Launch an instance
###### Launch an instance on the self-service network
```
$ source ~/demo-openrc
$ openstack flavor list
+----+---------+-----+------+-----------+-------+-----------+
| ID | Name    | RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+---------+-----+------+-----------+-------+-----------+
| 0  | m1.nano |  64 |    1 |         0 |     1 | True      |
+----+---------+-----+------+-----------+-------+-----------+

$ openstack image list
+--------------------------------------+-------------+--------+
| ID                                   | Name        | Status |
+--------------------------------------+-------------+--------+
| 53640dba-24a1-4370-aa29-ff9c674e7495 | ubuntu16.04 | active |
| dc894552-ee2a-4938-a4de-a5eb34323215 | cirros      | active |
+--------------------------------------+-------------+--------+

$ openstack network list
+--------------------------------------+-------------+--------------------------------------+
| ID                                   | Name        | Subnets                              |
+--------------------------------------+-------------+--------------------------------------+
| a12d6bc6-7fc0-43e1-a08d-22a657cb007a | selfservice | 9dd4e12d-473b-4e09-a588-e6ef1c164864 |
| b0ba4cc3-c71f-4fa6-98cf-9a516980fe66 | provider    | 3adffac3-f23e-4ba8-ad7f-c820c27b2c6a |
+--------------------------------------+-------------+--------------------------------------+

$ openstack security group list
+--------------------------------------+---------+------------------------+----------------------------------+
| ID                                   | Name    | Description            | Project                          |
+--------------------------------------+---------+------------------------+----------------------------------+
| 8c8c4c7a-a3f6-4bb9-9bf2-7225adcf5f00 | default | Default security group | f79866ddad734d739c7fd0511d284b55 |
+--------------------------------------+---------+------------------------+----------------------------------+

$ openstack server create --flavor m1.nano --image cirros --nic net-id=`openstack network list |grep selfservice | cut -d' ' -f2` --security-group default  --key-name chefkey selfservice-instance
→ net-idはselfserviceのnetworkIDを入力する。
+--------------------------------------+-----------------------------------------------+
| Field                                | Value                                         |
+--------------------------------------+-----------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                        |
| OS-EXT-AZ:availability_zone          |                                               |
| OS-EXT-STS:power_state               | NOSTATE                                       |
| OS-EXT-STS:task_state                | scheduling                                    |
| OS-EXT-STS:vm_state                  | building                                      |
| OS-SRV-USG:launched_at               | None                                          |
| OS-SRV-USG:terminated_at             | None                                          |
| accessIPv4                           |                                               |
| accessIPv6                           |                                               |
| addresses                            |                                               |
| adminPass                            | 8BLAxPD4EecD                                  |
| config_drive                         |                                               |
| created                              | 2016-11-26T12:41:35Z                          |
| flavor                               | m1.nano (0)                                   |
| hostId                               |                                               |
| id                                   | de8207c9-5bc7-4d40-9d58-6d5fe8474992          |
| image                                | cirros (dc894552-ee2a-4938-a4de-a5eb34323215) |
| key_name                             | chefkey                                       |
| name                                 | selfservice-instance                          |
| os-extended-volumes:volumes_attached | []                                            |
| progress                             | 0                                             |
| project_id                           | f79866ddad734d739c7fd0511d284b55              |
| properties                           |                                               |
| security_groups                      | [{u'name': u'default'}]                       |
| status                               | BUILD                                         |
| updated                              | 2016-11-26T12:41:35Z                          |
| user_id                              | 28e270123c6b467fb35d5c74550e6a26              |
+--------------------------------------+-----------------------------------------------+

$ openstack server list
+--------------------------------------+----------------------+--------+----------------------+------------+
| ID                                   | Name                 | Status | Networks             | Image Name |
+--------------------------------------+----------------------+--------+----------------------+------------+
| de8207c9-5bc7-4d40-9d58-6d5fe8474992 | selfservice-instance | ACTIVE | selfservice=10.0.0.7 | cirros     |
+--------------------------------------+----------------------+--------+----------------------+------------+
~~~ 仮想Instanceが起動している状況.(ACTIVEになっているので)

$ openstack console url show selfservice-instance
+-------+------------------------------------------------------------------------------------+
| Field | Value                                                                              |
+-------+------------------------------------------------------------------------------------+
| type  | novnc                                                                              |
| url   | http://192.168.0.200:6080/vnc_auto.html?token=d0a6d3b7-df4b-426a-9224-453bf60ed08c |
+-------+------------------------------------------------------------------------------------+
~~~ 上記URLでコンソールを閲覧できる.
```

DashboardからUbuntu16.04のインスタンスを起動してみる。
```
$ nova list
+--------------------------------------+------+--------+------------+-------------+-------------------------------------+
| ID                                   | Name | Status | Task State | Power State | Networks                            |
+--------------------------------------+------+--------+------------+-------------+-------------------------------------+
| f1e8dbad-7b87-4879-a159-acfe0b695057 | test | ACTIVE | -          | Running     | selfservice=10.0.0.6, 192.168.0.101 |
+--------------------------------------+------+--------+------------+-------------+-------------------------------------+

$ ssh -i ~/.ssh/id_rsa_chef ubuntu@192.168.0.101
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-47-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.


Last login: Sun Nov 27 01:16:51 2016 from 192.168.0.220
ubuntu@test:~$ 
ubuntu@test:~$ ifconfig ens3
ens3      Link encap:Ethernet  HWaddr fa:16:3e:60:e9:41  
          inet addr:10.0.0.6  Bcast:10.0.0.255  Mask:255.255.255.0
          inet6 addr: fe80::f816:3eff:fe60:e941/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:12795 errors:0 dropped:0 overruns:0 frame:0
          TX packets:12852 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:109568534 (109.5 MB)  TX bytes:1228270 (1.2 MB)
→ 仮想インスタンスへのログイン完了。
```
