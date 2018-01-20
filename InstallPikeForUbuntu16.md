## Install OpenStack Pike on Ubuntu 16.04
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/openstack-services.html)  
[OpenStack Pike 構築手順書(Ubuntu 16.04LTS版)](https://www.gitbook.com/book/virtualtech/openstack-pike-docs/details)  
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
### Register Pike Repositry
```
$ sudo add-apt-repository cloud-archive:pike
```

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
...
##character-set-server  = utf8mb4
##collation-server      = utf8mb4_general_ci
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
character-set-server = utf8
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
$ sudo apt install -y keystone apache2 libapache2-mod-wsgi python-openstackclient
```
Config
```
$ sudo vim /etc/keystone/keystone.conf
----
[database]
##connection = sqlite:////var/lib/keystone/keystone.db
connection = mysql+pymysql://keystone:password@ryunosuke/keystone
...
[token]
provider = fernet
...
----

# DB反映
$ sudo bash -c "keystone-manage db_sync" keystone

# 設定
$ sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage bootstrap --bootstrap-password password --bootstrap-admin-url http://ryunosuke:35357/v3/ --bootstrap-internal-url http://ryunosuke:5000/v3/ --bootstrap-public-url http://ryunosuke:5000/v3/ --bootstrap-region-id RegionOne
```

### Configure the Apache HTTP server
install
```
$ sudo vim /etc/apache2/apache2.conf
----
ServerName ryunosuke
----
$ sudo apachectl configtest
Syntax OK
$ sudo service apache2 restart
$ sudo service apache2 status
```

Config
```
$ export OS_USERNAME=admin
$ export OS_PASSWORD=password
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=default
$ export OS_PROJECT_DOMAIN_NAME=default
$ export OS_AUTH_URL=http://ryunosuke:35357/v3
$ export OS_IDENTITY_API_VERSION=3
```

### Create a domain, projects, users, and roles
#### Service Project
```
$ openstack project create --domain default --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 450cd888168c4bcaa99255d70701e3bc |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
+-------------+----------------------------------+
```
#### Demo Project
```
$ openstack project create --domain default --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | b0a6faa402c24bc0ae8aeba6289cdcd5 |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | default                          |
+-------------+----------------------------------+
```

#### Demo User
```
$ openstack user create --domain default --password-prompt demo
User Password: (パスワードを入力)
Repeat User Password: (パスワードを入力)
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | aea8ca8d30c74b56b17a91b1e030588a |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+ 
```

#### User Role
```
$ openstack role create user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 73304461e01e48e28fb0bfa53b550391 |
| name      | user                             |
+-----------+----------------------------------+
```
#### Add the user role to the demo project and user
```
$ openstack role add --project demo --user demo user
```

### Verify operation
```
## Edit Configfiles.
$ sudo vim /etc/keystone/keystone-paste.ini
→ admin_token_authがあれば削除する. (今回はなかった)

## Admin User.
$ unset OS_AUTH_URL OS_PASSWORD
$ openstack --os-auth-url http://ryunosuke:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-01-20T16:20:15+0000                                                                                                                                                                |
| id         | gAAAAABaY14vP2CFrhDkXz1de3zjMs6VT8ZWzYunMn3OSLPsaR-qzaUVwOGCEHGsVuhUTl6hioDjgw1dcPiJ5fa8GFvFyv69jk2jrT1BiZqF39j87st4aWHvv5NL6nVwDRzcD_L8dQTTItMK3QZZ3m7V1fVZnafOYnmHVb9p6GdObn0WiLngS-4 |
| project_id | dbad62d177f4454f83fffa93accaaff4                                                                                                                                                        |
| user_id    | 20eb2d8d578b40f1a5f240a773c7085f                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
→ 表示がでればOK.

## Demo User
$ openstack --os-auth-url http://ryunosuke:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name demo --os-username demo token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-01-20T16:21:33+0000                                                                                                                                                                |
| id         | gAAAAABaY159xsJiMHmRfxfC9-VbMVaxVfYysUBcSCQak-Mh-d5aAld1jsx-Gtqa3Lm6j2b-goMbSbxJRI4LBWP7F6K2tWYw2jHh7o8eH89KdRqM0wnYkWZOGAuN7sf0fyFr0DnBPlhf9EY3K5zSd2-BwCUegHcdvdihe9zfq3udE_hJzjSPjL8 |
| project_id | b0a6faa402c24bc0ae8aeba6289cdcd5                                                                                                                                                        |
| user_id    | aea8ca8d30c74b56b17a91b1e030588a                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
→ 表示がでればOK.
```

### Create OpenStack client environment scripts
クライアントで使用するEnvironmentファイルを作成する.
#### admin-openrc
```
$ cat << _EOT_ > ~/admin-openrc.sh
#!/bin/bash
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://ryunosuke:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
_EOT_
$ chmod -v 755 ~/admin-openrc.sh
mode of '/home/ogalush/admin-openrc.sh' changed from 0664 (rw-rw-r--) to 0755 (rwxr-xr-x)
```

#### demo-openrc
```
$ cat << _EOT_ > ~/demo-openrc.sh
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=password
export OS_AUTH_URL=http://ryunosuke:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
_EOT_
$ chmod -v 755 ~/demo-openrc.sh
mode of '/home/ogalush/demo-openrc.sh' changed from 0664 (rw-rw-r--) to 0755 (rwxr-xr-x)
```

#### Using the scripts
表示がでればOK.
```
$ source ~/admin-openrc.sh
$ openstack token issue                                                                         +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+                              | Field      | Value                                                                                                                                                                                   |                              +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+                              | expires    | 2018-01-20T16:30:11+0000                                                                                                                                                                |                              | id         | gAAAAABaY2CDJ_FYY1bG8WpDR0JJoxLFvsNu4qr2IRRXYuIFnwuzPkiX8XDoPIK5P3vvuBGLTq1QCjS_esQPXHAaCxMl55q0eg1hHXFZlRKTtkAek1kUMXeqRFWSyyRgrnolRXq4FleB6wNxj5rvHxTP_OCRxYcxaxuYTTuBcp9grtAwUEI8Llk |                              | project_id | dbad62d177f4454f83fffa93accaaff4                                                                                                                                                        |                              | user_id    | 20eb2d8d578b40f1a5f240a773c7085f                                                                                                                                                        |                              +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+    

$ source ~/demo-openrc.sh
$ openstack token issue                                                                         +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+                              | Field      | Value                                                                                                                                                                                   |                              +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+                              | expires    | 2018-01-20T16:30:22+0000                                                                                                                                                                |                              | id         | gAAAAABaY2CO-LLhP3zfLsInbSH21a05J9Fr_LE0sp7DKzsQ5fZa9J6zYyc_V6SW6P0iy2VgqW8wIjFrPM4s9KnmFzAkcKo6bbVO_T-en5aqRIr3gq2Xn9soZiU1Rq5nh2Rhl3mm311xbXNR8o9SsIg7Qe9PLtYxQVyLWGPC_qo3LIfmEX3EF24 |                              | project_id | b0a6faa402c24bc0ae8aeba6289cdcd5                                                                                                                                                        |                              | user_id    | aea8ca8d30c74b56b17a91b1e030588a                                                                                                                                                        |                              +------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+                              ogalush@ryunosuke:~$ 
```

## Grance Install
[Document](https://docs.openstack.org/glance/pike/install/)

### Install and configure (Ubuntu)
#### Create Glance DB
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;
```

#### Create Glance User
```
$ source ~/admin-openrc.sh
$ openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | a8fa1ef195e04a38b91ac220b4fedf20 |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

#### Add the admin role to the glance user and service project
```
$ openstack role add --project service --user glance admin
```

#### Create the glance service entity
```
$ openstack service create --name glance --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | b30b73fba5e547ac9741c13f985a0cb7 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

#### Create the Image service API endpoints
```
$ openstack endpoint create --region RegionOne image public http://ryunosuke:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6f823cc8666941759dba6716aeb8fd90 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b30b73fba5e547ac9741c13f985a0cb7 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://ryunosuke:9292            |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image internal http://ryunosuke:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 4ff2ec8b04be4a909042a77078483257 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b30b73fba5e547ac9741c13f985a0cb7 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://ryunosuke:9292            |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image admin http://ryunosuke:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a17b8bc2eca040ae8f4604c0902c89cd |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b30b73fba5e547ac9741c13f985a0cb7 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://ryunosuke:9292            |
+--------------+----------------------------------+
```

### Install and configure components
#### Install the packages
```
$ sudo apt install -y glance
```

#### Config
```
$ sudo vim /etc/glance/glance-api.conf
----
[database]
connection = mysql+pymysql://glance:password@ryunosuke/glance
...
[keystone_authtoken]
auth_uri = http://ryunosuke:5000
auth_url = http://ryunosuke:35357
memcached_servers = ryunosuke:11211
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

$ sudo vim /etc/glance/glance-registry.conf
----
[database]
connection = mysql+pymysql://glance:password@ryunosuke/glance
...
[keystone_authtoken]
auth_uri = http://ryunosuke:5000
auth_url = http://ryunosuke:35357
memcached_servers = ryunosuke:11211
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

$ sudo bash -c "glance-manage db_sync" glance
...
Upgraded database to: pike01, current revision(s): pike01

$ sudo service glance-registry restart
$ sudo service glance-registry status
$ sudo service glance-api restart
$ sudo service glance-api status
```

### Verify operation
#### Craete OS Image.
```
$ source ~/admin-openrc.sh
$ wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
$ openstack image create "cirros" --file cirros-0.3.5-x86_64-disk.img --disk-format qcow2 --container-format bare --public
+------------------+------------------------------------------------------+                                        | Field            | Value                                                |                                        +------------------+------------------------------------------------------+                                        | checksum         | f8ab98ff5e73ebab884d80c9dc9c7290                     |                                        | container_format | bare                                                 |                                        | created_at       | 2018-01-20T15:58:19Z                                 |                                        | disk_format      | qcow2                                                |                                        | file             | /v2/images/351ec2ce-8cca-4a11-a007-0c6de84926ac/file |                                        | id               | 351ec2ce-8cca-4a11-a007-0c6de84926ac                 |                                        | min_disk         | 0                                                    |                                        | min_ram          | 0                                                    |                                        | name             | cirros                                               |                                        | owner            | dbad62d177f4454f83fffa93accaaff4                     |                                        | protected        | False                                                |                                        | schema           | /v2/schemas/image                                    |                                        | size             | 13267968                                             |                                        | status           | active                                               |                                        | tags             |                                                      |                                        | updated_at       | 2018-01-20T15:58:19Z                                 |                                        
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
→ 表示されればOK.
```

#### Confirm upload of the image and validate attributes
```
$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 351ec2ce-8cca-4a11-a007-0c6de84926ac | cirros | active |
+--------------------------------------+--------+--------+
```

