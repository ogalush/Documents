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

### RabbitMQ
#### Install
```
$ sudo apt install -y rabbitmq-server
$ sudo rabbitmqctl add_user openstack password
Creating user "openstack" ...
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
Setting permissions for user "openstack" in vhost "/" ...
```
#### Config
```
$ sudo vim /etc/rabbitmq/rabbitmq-env.conf
NODE_IP_ADDRESS=0.0.0.0
$ sudo service rabbitmq-server restart
controller# service rabbitmq-server restart
```

### Memcached
#### Install
```
$ sudo apt install -y memcached python-memcache
$ sudo vim /etc/memcached.conf
----
-l 0.0.0.0
----
$ sudo service memcached restart
$ sudo service memcached status
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

$ sudo vim /etc/mysql/mariadb.conf.d/50-client.cnf
----
##default-character-set = utf8mb4
default-character-set = utf8
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

## Compute service
[Document](https://docs.openstack.org/nova/pike/install/)

### Create DataBase
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> quit;
```

### Create the Compute service credentials
#### Create Nove User
```
$ source ~/admin-openrc.sh
$ openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 43665f157dc54699a197c78461672eba |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

#### Add the admin role to the nova user
```
$ openstack role add --project service --user nova admin
```

#### Create the nova service entity
```
$ openstack service create --name nova --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 0f3b5cdb31b24f568e9c04a6345a1b9c |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

### Create the Compute API service endpoints
```
$ openstack endpoint create --region RegionOne compute public http://ryunosuke:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | c87e20a2192543f8bb55292668508384 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 0f3b5cdb31b24f568e9c04a6345a1b9c |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://ryunosuke:8774/v2.1       |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute internal http://ryunosuke:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 40c677547be0496d9f00fcb46cf0e974 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 0f3b5cdb31b24f568e9c04a6345a1b9c |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://ryunosuke:8774/v2.1       |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute admin http://ryunosuke:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 9976714db7054a03b2af175651696b33 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 0f3b5cdb31b24f568e9c04a6345a1b9c |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://ryunosuke:8774/v2.1       |
+--------------+----------------------------------+
```

### Create a Placement service user using your chosen PLACEMENT_PASS
```
$ openstack user create --domain default --password-prompt placement
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 7dee00c6f28c4a958eadbe787589637d |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

### Add the Placement user to the service project with the admin role:
```
$ openstack role add --project service --user placement admin
```

### Create the Placement API entry in the service catalog:
```
$ openstack service create --name placement --description "Placement API" placement
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | bc4acb202a5246a1bc70c8a0a42c5a5d |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+
```

### Create the Placement API service endpoints
```
$ openstack endpoint create --region RegionOne placement public http://ryunosuke:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6bb6a2557ed94e78aabefccd57f1eb90 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | bc4acb202a5246a1bc70c8a0a42c5a5d |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://ryunosuke:8778            |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement internal http://ryunosuke:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 9d8599220baa46c297bdbfd01c39570f |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | bc4acb202a5246a1bc70c8a0a42c5a5d |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://ryunosuke:8778            |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement admin http://ryunosuke:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 1b76a8bb643a42cd9dc411511113c6f6 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | bc4acb202a5246a1bc70c8a0a42c5a5d |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://ryunosuke:8778            |
+--------------+----------------------------------+
```

### Install and configure components
#### Install the packages
```
$ sudo apt install -y nova-api nova-conductor nova-consoleauth nova-novncproxy nova-scheduler nova-placement-api
```

#### Edit the /etc/nova/nova.conf file and complete the following actions:
```
$ sudo vim /etc/nova/nova.conf
----
[DEFAULT]
transport_url = rabbit://openstack:password@ryunosuke
my_ip = 192.168.0.200
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
...
[api_database]
##connection = sqlite:////var/lib/nova/nova_api.sqlite
connection = mysql+pymysql://nova:password@ryunosuke/nova_api
....

[database]
###connection = sqlite:////var/lib/nova/nova.sqlite
connection = mysql+pymysql://nova:password@ryunosuke/nova
...
[api]
auth_strategy = keystone
...
[keystone_authtoken]
auth_uri = http://ryunosuke:5000
auth_url = http://ryunosuke:35357
memcached_servers = ryunosuke:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = password
...
[vnc]
enabled = true
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip
...
[glance]
api_servers = http://ryunosuke:9292
...
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
...
[placement]
# ...
os_region_name = RegionOne
project_domain_name = default
project_name = service
auth_type = password
user_domain_name = default
auth_url = http://ryunosuke:35357/v3
username = placement
password = password
----
```

### Populate the nova-api database
```
$ sudo bash -c "nova-manage api_db sync" nova
```
### Register the cell0 database
```
$ sudo bash -c "nova-manage cell_v2 map_cell0" nova
```

### Create the cell1 cell
```
$ sudo bash -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
a8b57c68-460a-4795-bd2c-3aad1f4f20bd
```

### Populate the nova database
```
$ sudo bash -c "nova-manage db sync" nova
```

### Verify nova cell0 and cell1 are registered correctly
```
$ sudo nova-manage cell_v2 list_cells
+-------+--------------------------------------+-----------------------------------+------------------------------------------------+
|  Name |                 UUID                 |           Transport URL           |              Database Connection               |
+-------+--------------------------------------+-----------------------------------+------------------------------------------------+
| cell0 | 00000000-0000-0000-0000-000000000000 |               none:/              | mysql+pymysql://nova:****@ryunosuke/nova_cell0 |
| cell1 | a8b57c68-460a-4795-bd2c-3aad1f4f20bd | rabbit://openstack:****@ryunosuke |    mysql+pymysql://nova:****@ryunosuke/nova    |
+-------+--------------------------------------+-----------------------------------+------------------------------------------------+
→ 表示されればOK.
```

### Restart the Compute services
```
$ for i in 'nova-api' 'nova-consoleauth' 'nova-scheduler' 'nova-conductor'; do sudo service $i restart; done
$ for i in 'nova-api' 'nova-consoleauth' 'nova-scheduler' 'nova-conductor'; do sudo service $i status; done
```

### Install and configure a compute node for Ubuntu
```
$ sudo apt install -y nova-compute
```

### Edit the /etc/nova/nova.conf file and complete the following actions:
```
$ sudo vim /etc/nova/nova.conf
----
Controller設定分に加え.
[vnc]
novncproxy_base_url = http://ryunosuke:6080/vnc_auto.html
[scheduler]
discover_hosts_in_cells_interval = 300
----
```

#### Edit the /etc/nova/nova-compute.conf
```
$ sudo vim /etc/nova/nova-compute.conf
----
[libvirt]
virt_type=kvm
~~~~ KVMのまま.
----
```

### Restart Nova Compute
```
$ sudo service nova-compute restart
$ sudo service nova-compute status
```

### Add the compute node to the cell database
#### Source the admin credentials to enable admin-only CLI commands, then confirm there are compute hosts in the database:
```
$ source ~/admin-openrc.sh
$ openstack compute service list --service nova-compute
+----+--------------+-----------+------+---------+-------+----------------------------+
| ID | Binary       | Host      | Zone | Status  | State | Updated At                 |
+----+--------------+-----------+------+---------+-------+----------------------------+
|  9 | nova-compute | ryunosuke | nova | enabled | up    | 2018-01-20T16:47:23.000000 |
+----+--------------+-----------+------+---------+-------+----------------------------+
```

#### Discover compute hosts
```
$ sudo bash -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting compute nodes from cell 'cell1': a8b57c68-460a-4795-bd2c-3aad1f4f20bd
Found 1 unmapped computes in cell: a8b57c68-460a-4795-bd2c-3aad1f4f20bd
Checking host mapping for compute host 'ryunosuke': db1298ca-ca57-48c1-9de5-30f880befc8a
Creating host mapping for compute host 'ryunosuke': db1298ca-ca57-48c1-9de5-30f880befc8a
→ 見つかっているのでOK.
```

#### Restart Nova Services.
```
$ for i in 'nova-api' 'nova-consoleauth' 'nova-scheduler' 'nova-conductor' 'nova-compute'; do sudo systemctl restart $i ; done
$ for i in 'nova-api' 'nova-consoleauth' 'nova-scheduler' 'nova-conductor' 'nova-compute'; do sudo systemctl status $i ; done
```

#### Verify operation
```
$ source ~/admin-openrc.sh
$ openstack compute service list
+----+------------------+-----------+----------+---------+-------+----------------------------+
| ID | Binary           | Host      | Zone     | Status  | State | Updated At                 |
+----+------------------+-----------+----------+---------+-------+----------------------------+
|  1 | nova-scheduler   | ryunosuke | internal | enabled | up    | 2018-01-20T16:51:48.000000 |
|  5 | nova-consoleauth | ryunosuke | internal | enabled | up    | 2018-01-20T16:51:47.000000 |
|  6 | nova-conductor   | ryunosuke | internal | enabled | up    | 2018-01-20T16:51:48.000000 |
|  9 | nova-compute     | ryunosuke | nova     | enabled | up    | 2018-01-20T16:51:50.000000 |
+----+------------------+-----------+----------+---------+-------+----------------------------+

$ openstack catalog list
+-----------+-----------+----------------------------------------+
| Name      | Type      | Endpoints                              |
+-----------+-----------+----------------------------------------+
| nova      | compute   | RegionOne                              |
|           |           |   internal: http://ryunosuke:8774/v2.1 |
|           |           | RegionOne                              |
|           |           |   admin: http://ryunosuke:8774/v2.1    |
|           |           | RegionOne                              |
|           |           |   public: http://ryunosuke:8774/v2.1   |
|           |           |                                        |
| keystone  | identity  | RegionOne                              |
|           |           |   admin: http://ryunosuke:35357/v3/    |
|           |           | RegionOne                              |
|           |           |   internal: http://ryunosuke:5000/v3/  |
|           |           | RegionOne                              |
|           |           |   public: http://ryunosuke:5000/v3/    |
|           |           |                                        |
| glance    | image     | RegionOne                              |
|           |           |   internal: http://ryunosuke:9292      |
|           |           | RegionOne                              |
|           |           |   public: http://ryunosuke:9292        |
|           |           | RegionOne                              |
|           |           |   admin: http://ryunosuke:9292         |
|           |           |                                        |
| placement | placement | RegionOne                              |
|           |           |   admin: http://ryunosuke:8778         |
|           |           | RegionOne                              |
|           |           |   public: http://ryunosuke:8778        |
|           |           | RegionOne                              |
|           |           |   internal: http://ryunosuke:8778      |
|           |           |                                        |
+-----------+-----------+----------------------------------------+

$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 351ec2ce-8cca-4a11-a007-0c6de84926ac | cirros | active |
+--------------------------------------+--------+--------+

$ sudo nova-status upgrade check
+---------------------------+
| Upgrade Check Results     |
+---------------------------+
| Check: Cells v2           |
| Result: Success           |
| Details: None             |
+---------------------------+
| Check: Placement API      |
| Result: Success           |
| Details: None             |
+---------------------------+
| Check: Resource Providers |
| Result: Success           |
| Details: None             |
+---------------------------+
```

## Neutron
[Document](https://docs.openstack.org/neutron/pike/install/install-ubuntu.html)

### Create Neutron User
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE neutron;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.00 sec)
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;
```

### Source the admin credentials to gain access to admin-only CLI commands:
#### Create the neutron user
```
$ source ~/admin-openrc.sh 
$ openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | e93734590f77414f8deace4be183fc96 |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

#### Add the admin role to the neutron user
```
$ openstack role add --project service --user neutron admin
```

#### Create the neutron service entity
```
$ openstack service create --name neutron --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 8c0c4c02a596486cb75928e0c6ff7be7 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```

### Create the Networking service API endpoints
```
$ openstack endpoint create --region RegionOne network public http://ryunosuke:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 801c49161c924b1fbf13a63e3ba14048 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c0c4c02a596486cb75928e0c6ff7be7 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://ryunosuke:9696            |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network internal http://ryunosuke:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 648f037a5c234fb48a28c37d5950684c |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c0c4c02a596486cb75928e0c6ff7be7 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://ryunosuke:9696            |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network admin http://ryunosuke:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 9a64f156708b4f078ae63f7ea967badf |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c0c4c02a596486cb75928e0c6ff7be7 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://ryunosuke:9696            |
+--------------+----------------------------------+
```

### Networking Option 2: Self-service networks
[Document](https://docs.openstack.org/neutron/pike/install/controller-install-option2-ubuntu.html)
#### Install
```
$ sudo apt install -y neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```

#### /etc/neutron/neutron.conf
```
$ sudo vim /etc/neutron/neutron.conf
----
[DEFAULT]
...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:password@ryunosuke
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
...
[database]
##connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql+pymysql://neutron:password@ryunosuke/neutron
...
[keystone_authtoken]
auth_uri = http://ryunosuke:5000
auth_url = http://ryunosuke:35357
memcached_servers = ryunosuke:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password
...
[nova]
auth_url = http://ryunosuke:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = password
----
```

#### /etc/neutron/plugins/ml2/ml2_conf.ini
```
$ sudo vim /etc/neutron/plugins/ml2/ml2_conf.ini
----
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security
...
[ml2_type_flat]
flat_networks = provider
...
[ml2_type_vxlan]
vni_ranges = 1:1000
...
[securitygroup]
enable_ipset = true
----
```

#### Edit the /etc/neutron/plugins/ml2/linuxbridge_agent.ini file and complete the following actions
```
$ sudo vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
----
[linux_bridge]
physical_interface_mappings = provider:enp3s0
...
[vxlan]
enable_vxlan = true
local_ip = 192.168.0.200
l2_population = true
...
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
----
```
#### /etc/neutron/l3_agent.ini
```
$ sudo vim /etc/neutron/l3_agent.ini
----
[DEFAULT]
interface_driver = linuxbridge
----
```

#### /etc/neutron/dhcp_agent.ini
```
$ sudo vim /etc/neutron/dhcp_agent.ini
----
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
----
```
#### Edit the /etc/neutron/metadata_agent.ini file and complete the following actions
```
$ sudo vim /etc/neutron/metadata_agent.ini
----
[DEFAULT]
nova_metadata_host = ryunosuke
metadata_proxy_shared_secret = password
----
```

#### /etc/nova/nova.conf
```
$ sudo vim /etc/nova/nova.conf
----
[neutron]
url = http://ryunosuke:9696
auth_url = http://ryunosuke:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = password
service_metadata_proxy = true
metadata_proxy_shared_secret = password
----
```

#### Finalize installation¶
```
$ sudo bash -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
...
INFO  [alembic.runtime.migration] Running upgrade f83a0b2964d0 -> fd38cd995cc0, change shared attribute for firewall resource
  OK
```

#### Restart Services.
```
$ for i in 'nova-api' 'nova-compute' 'neutron-server' 'neutron-linuxbridge-agent' 'neutron-dhcp-agent' 'neutron-metadata-agent' 'neutron-l3-agent'; do sudo systemctl restart $i ; done
$ for i in 'nova-api' 'nova-compute' 'neutron-server' 'neutron-linuxbridge-agent' 'neutron-dhcp-agent' 'neutron-metadata-agent' 'neutron-l3-agent'; do sudo systemctl status $i ; done
```

#### 確認
```
$ source ~/admin-openrc.sh
$ neutron agent-list
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+--------------------+-----------+-------------------+-------+----------------+---------------------------+
| id                                   | agent_type         | host      | availability_zone | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-----------+-------------------+-------+----------------+---------------------------+
| 373df810-0011-459b-a14f-ad3989c25cdb | DHCP agent         | ryunosuke | nova              | :-)   | True           | neutron-dhcp-agent        |
| 7f0270c0-63fa-4b22-9adb-1007edb2a568 | Metadata agent     | ryunosuke |                   | :-)   | True           | neutron-metadata-agent    |
| b6251ea8-2b1b-4886-b1e9-22f262837495 | L3 agent           | ryunosuke | nova              | :-)   | True           | neutron-l3-agent          |
| e93e074e-8833-4f99-bb3b-aadf431535b5 | Linux bridge agent | ryunosuke |                   | :-)   | True           | neutron-linuxbridge-agent |
+--------------------------------------+--------------------+-----------+-------------------+-------+----------------+---------------------------+
```

## Horizon
[Document](https://docs.openstack.org/horizon/pike/install/install-ubuntu.html)

### Install and configure for Ubuntu
#### Install the packages
```
$ sudo apt install -y openstack-dashboard
```
#### Edit the /etc/openstack-dashboard/local_settings.py file and complete the following actions
```
$ sudo vim /etc/openstack-dashboard/local_settings.py
----
##OPENSTACK_HOST = "127.0.0.1"
OPENSTACK_HOST = "ryunosuke"
...
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = { 
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': 'ryunosuke:11211',
    },  
}
...
##OPENSTACK_KEYSTONE_URL = "http://%s:5000/v2.0" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
...
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
...
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
...
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'default'
...
##OPENSTACK_KEYSTONE_DEFAULT_ROLE = "_member_"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
...
##TIME_ZONE = "UTC"
TIME_ZONE = "Asia/Tokyo"
----
```
#### Restart
```
$ sudo systemctl restart apache2
$ sudo systemctl status apache2
```

#### アクセス確認
[http://192.168.0.200/horizon/](http://192.168.0.200/horizon/)
→ ログインできればOK.

## インスタンス作成準備
### ネットワーク作成
#### Ext-Net
```
$ source ~/admin-openrc.sh
$ neutron net-create --shared --provider:physical_network provider --provider:network_type flat provider
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2018-01-20T17:48:54Z                 |
| description               |                                      |
| id                        | 4d03b6f5-0312-44f9-99c2-41561f11ccd8 |
| ipv4_address_scope        |                                      |
| ipv6_address_scope        |                                      |
| is_default                | False                                |
| mtu                       | 1500                                 |
| name                      | provider                             |
| port_security_enabled     | True                                 |
| project_id                | dbad62d177f4454f83fffa93accaaff4     |
| provider:network_type     | flat                                 |
| provider:physical_network | provider                             |
| provider:segmentation_id  |                                      |
| revision_number           | 2                                    |
| router:external           | False                                |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| tenant_id                 | dbad62d177f4454f83fffa93accaaff4     |
| updated_at                | 2018-01-20T17:48:54Z                 |
+---------------------------+--------------------------------------+

$ neutron subnet-create --name provider --allocation-pool start=192.168.0.230,end=192.168.0.240 --dns-nameserver 192.168.0.254 --gateway 192.168.0.254 provider 192.168.0.0/24
+-------------------+----------------------------------------------------+
| Field             | Value                                              |
+-------------------+----------------------------------------------------+
| allocation_pools  | {"start": "192.168.0.230", "end": "192.168.0.240"} |
| cidr              | 192.168.0.0/24                                     |
| created_at        | 2018-01-20T17:49:43Z                               |
| description       |                                                    |
| dns_nameservers   | 192.168.0.254                                      |
| enable_dhcp       | True                                               |
| gateway_ip        | 192.168.0.254                                      |
| host_routes       |                                                    |
| id                | 358784b1-b69c-43a2-91c7-3eb74a0a3697               |
| ip_version        | 4                                                  |
| ipv6_address_mode |                                                    |
| ipv6_ra_mode      |                                                    |
| name              | provider                                           |
| network_id        | 4d03b6f5-0312-44f9-99c2-41561f11ccd8               |
| project_id        | dbad62d177f4454f83fffa93accaaff4                   |
| revision_number   | 0                                                  |
| service_types     |                                                    |
| subnetpool_id     |                                                    |
| tags              |                                                    |
| tenant_id         | dbad62d177f4454f83fffa93accaaff4                   |
| updated_at        | 2018-01-20T17:49:43Z                               |
+-------------------+----------------------------------------------------+
```

#### Internal-Net
```
$ source ~/demo-openrc.sh
$ neutron net-create selfservice
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | True                                 |
| availability_zone_hints |                                      |
| availability_zones      |                                      |
| created_at              | 2018-01-20T17:50:49Z                 |
| description             |                                      |
| id                      | 2de82445-e5a1-40bb-b738-0d98641d4132 |
| ipv4_address_scope      |                                      |
| ipv6_address_scope      |                                      |
| is_default              | False                                |
| mtu                     | 1450                                 |
| name                    | selfservice                          |
| port_security_enabled   | True                                 |
| project_id              | b0a6faa402c24bc0ae8aeba6289cdcd5     |
| revision_number         | 2                                    |
| router:external         | False                                |
| shared                  | False                                |
| status                  | ACTIVE                               |
| subnets                 |                                      |
| tags                    |                                      |
| tenant_id               | b0a6faa402c24bc0ae8aeba6289cdcd5     |
| updated_at              | 2018-01-20T17:50:49Z                 |
+-------------------------+--------------------------------------+

$ neutron subnet-create --name selfservice  --dns-nameserver 192.168.0.220 --gateway 10.0.0.1 selfservice 10.0.0.0/24
+-------------------+--------------------------------------------+
| Field             | Value                                      |
+-------------------+--------------------------------------------+
| allocation_pools  | {"start": "10.0.0.2", "end": "10.0.0.254"} |
| cidr              | 10.0.0.0/24                                |
| created_at        | 2018-01-20T17:51:47Z                       |
| description       |                                            |
| dns_nameservers   | 192.168.0.220                              |
| enable_dhcp       | True                                       |
| gateway_ip        | 10.0.0.1                                   |
| host_routes       |                                            |
| id                | 76180669-eca3-4560-8991-e7aab8780f60       |
| ip_version        | 4                                          |
| ipv6_address_mode |                                            |
| ipv6_ra_mode      |                                            |
| name              | selfservice                                |
| network_id        | 2de82445-e5a1-40bb-b738-0d98641d4132       |
| project_id        | b0a6faa402c24bc0ae8aeba6289cdcd5           |
| revision_number   | 0                                          |
| service_types     |                                            |
| subnetpool_id     |                                            |
| tags              |                                            |
| tenant_id         | b0a6faa402c24bc0ae8aeba6289cdcd5           |
| updated_at        | 2018-01-20T17:51:47Z                       |
+-------------------+--------------------------------------------+
```

### Router
```
$ source ~/admin-openrc.sh
$ neutron net-update provider --router:external
Updated network: provider
$ source ~/demo-openrc.sh
$ neutron router-create router
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | True                                 |
| availability_zone_hints |                                      |
| availability_zones      |                                      |
| created_at              | 2018-01-20T17:53:44Z                 |
| description             |                                      |
| external_gateway_info   |                                      |
| flavor_id               |                                      |
| id                      | d5bbf14b-a5c5-4c83-8126-75148e459433 |
| name                    | router                               |
| project_id              | b0a6faa402c24bc0ae8aeba6289cdcd5     |
| revision_number         | 1                                    |
| routes                  |                                      |
| status                  | ACTIVE                               |
| tags                    |                                      |
| tenant_id               | b0a6faa402c24bc0ae8aeba6289cdcd5     |
| updated_at              | 2018-01-20T17:53:44Z                 |
+-------------------------+--------------------------------------+

$ neutron router-interface-add router selfservice
Connection to neutron failed: Failed to connect Neutron server
$ neutron router-gateway-set router provider
Set gateway for router router
```

### Security Rules
```
$ source ~/admin-openrc.sh
```
