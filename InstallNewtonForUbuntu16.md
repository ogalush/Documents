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

確認
$ openstack image list
+--------------------------------------+-------------+--------+
| ID                                   | Name        | Status |
+--------------------------------------+-------------+--------+
| 53640dba-24a1-4370-aa29-ff9c674e7495 | ubuntu16.04 | active |
| dc894552-ee2a-4938-a4de-a5eb34323215 | cirros      | active |
+--------------------------------------+-------------+--------+
```
