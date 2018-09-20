# Install OpenStack Rocky on Ubuntu 18.04
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.0.200(192.168.0.200)  
設定ファイル: [Rocky-InstallConfigs](https://github.com/ogalush/Rocky-InstallConfigs)
```
$ uname -a
Linux ryunosuke 4.15.0-34-generic #37-Ubuntu SMP Mon Aug 27 15:21:48 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

# Host networking
## Configure network interfaces
127.0.1.1 がホスト名に紐づいているので、実際のIPアドレスへ置換する.
```
$ sudo cp -pv /etc/hosts /tmp/hosts
[sudo] password for ogalush: 
'/etc/hosts' -> '/tmp/hosts'
$ sudo vim /etc/hosts
$ diff -u /tmp/hosts /etc/hosts |egrep '^(\+|\-)'
--- /tmp/hosts  2018-09-16 14:40:00.059247346 +0900
+++ /etc/hosts  2018-09-16 15:01:06.148425262 +0900
-127.0.1.1      ryunosuke
+192.168.0.200  ryunosuke
```

## Network Time Protocol (NTP)
```
$ sudo dpkg -r ntp
$ sudo apt -y install chrony
$ dpkg -l |grep chrony
ii  chrony 3.2-4ubuntu4.2 amd64 Versatile implementation of the Network Time Protocol
$ sudo cp -ar /etc/chrony ~
$ sudo vim /etc/chrony/chrony.conf
$ diff -urBb ~/chrony /etc/chrony 2> /dev/null |egrep '^(\+|\-)'
--- /home/ogalush/chrony/chrony.conf    2018-08-20 15:00:29.000000000 +0900
+++ /etc/chrony/chrony.conf     2018-09-16 15:19:47.082452675 +0900
+
+server ntp.nict.jp iburst
+allow 10.0.0.0/24
+allow 192.168.0.0/24
----

$ sudo service chrony restart
$ sudo service chrony status
● chrony.service - chrony, an NTP client/server
   Loaded: loaded (/lib/systemd/system/chrony.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-09-16 15:20:50 JST; 10s ago
---

$ chronyc sources
210 Number of sources = 9
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^- chilipepper.canonical.com     2   6    77    64  -1219us[-1276us] +/-  148ms
^- golem.canonical.com           2   6   177     1  +3375us[+3375us] +/-  147ms
^- pugot.canonical.com           2   6   177     1  +3079us[+3079us] +/-  146ms
^- chilipepper.canonical.com     2   6    77    64  +1327us[+1270us] +/-  141ms
^- ns1.alza.is                   2   6   177     0    -53us[  -53us] +/-  175ms
^- 78.140.251.2                  2   6    77    64    +12ms[  +12ms] +/-  150ms
^- 2001:bc8:30a3:100::1          2   6    77    63  +6799us[+6742us] +/-  154ms
^- ntp1.ams1.nl.leaseweb.net     2   6    77    64  +1787us[+1730us] +/-  243ms
^* ntp-a3.nict.go.jp             1   6   177     2  -2094ns[  -56us] +/- 2202us
```

# OpenStack packages for Ubuntu
## Enable the OpenStack repository
```
$ sudo apt -y install software-properties-common
$ sudo add-apt-repository cloud-archive:rocky
...
Get:2 http://jp.archive.ubuntu.com/ubuntu bionic-updates InRelease [88.7 kB]
Get:3 http://jp.archive.ubuntu.com/ubuntu bionic-backports InRelease [74.6 kB]
Hit:4 http://security.ubuntu.com/ubuntu bionic-security InRelease
Ign:5 http://ubuntu-cloud.archive.canonical.com/ubuntu bionic-updates/rocky InRelease
Get:6 http://ubuntu-cloud.archive.canonical.com/ubuntu bionic-updates/rocky Release [7,879 B]
Get:7 http://ubuntu-cloud.archive.canonical.com/ubuntu bionic-updates/rocky Release.gpg [543 B]
Get:8 http://ubuntu-cloud.archive.canonical.com/ubuntu bionic-updates/rocky/main amd64 Packages [112 kB]
Get:9 http://ubuntu-cloud.archive.canonical.com/ubuntu bionic-updates/rocky/main i386 Packages [112 kB]
Fetched 395 kB in 4s (105 kB/s)
Reading package lists... Done
$
```

## Finalize the installation
```
$ sudo apt -y install python-openstackclient
$ sudo apt-get -y update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade && sudo apt-get -y autoremove
$ sudo shutdown -r now
```

# SQL database for Ubuntu
## Install and configure components
```
$ sudo apt -y install mariadb-server python-pymysql
$ sudo cp -ra /etc/mysql ~
$ sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf
$ diff -urBb ~/mysql /etc/mysql 2> /dev/null |egrep '^(\+|\-)'
--- /home/ogalush/mysql/mariadb.conf.d/50-server.cnf    2018-07-08 17:14:42.000000000 +0900
+++ /etc/mysql/mariadb.conf.d/50-server.cnf     2018-09-16 15:46:26.879647819 +0900
-bind-address           = 127.0.0.1
+bind-address           = 192.168.0.200
-character-set-server  = utf8mb4
-collation-server      = utf8mb4_general_ci
+character-set-server = utf8
+collation-server = utf8_general_ci
+
+default-storage-engine = innodb
+innodb_file_per_table = on
+max_connections = 4096
----

$ sudo service mysql restart
$ sudo service mysql status
● mariadb.service - MariaDB 10.1.34 database server
   Loaded: loaded (/lib/systemd/system/mariadb.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-09-16 15:47:42 JST; 14s ago
$ sudo mysql_secure_installation
Set root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

# Message queue for Ubuntu
## Install and configure components
```
$ sudo apt -y install rabbitmq-server
$ sudo rabbitmqctl add_user openstack password
Creating user "openstack"
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
Setting permissions for user "openstack" in vhost "/"

$ sudo rabbitmq-plugins enable rabbitmq_management
→ 管理画面を有効化するオプション
https://qiita.com/AKB428/items/a2662fbb624ce7659025
```

# Memcached for Ubuntu
## Install and configure components
```
$ sudo apt -y install memcached python-memcache
$ sudo sed -i 's/-l 127.0.0.1/-l 192.168.0.200/' /etc/memcached.conf 
$ grep 192.168.0.200 /etc/memcached.conf 
-l 192.168.0.200
```

## Finalize installation
```
$ sudo service memcached restart
$ sudo service memcached status
● memcached.service - memcached daemon
   Loaded: loaded (/lib/systemd/system/memcached.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-09-16 15:52:39 JST; 4s ago

$ netstat -ln |grep 11211
tcp        0      0 192.168.0.200:11211     0.0.0.0:*               LISTEN   
→ 変更したIPアドレスでLISTENできているのでOK.
```

# Etcd for Ubuntu
## Install and configure components
```
$ sudo apt -y install etcd
$ sudo vim /etc/default/etcd
----
name: 'ryunosuke'
data-dir: /var/lib/etcd
initial-cluster-state: 'new'
initial-cluster-token: 'etcd-cluster-01'
initial-cluster: ryunosuke=http://192.168.0.200:2380
initial-advertise-peer-urls: http://192.168.0.200:2380
advertise-client-urls: http://192.168.0.200:2379
listen-peer-urls: http://0.0.0.0:2380
listen-client-urls: http://192.168.0.200:2379
----
```

## Finalize installation
```
$ sudo systemctl enable etcd
$ sudo systemctl restart etcd
$ sudo systemctl status etcd
● etcd.service - etcd - highly-available key value store
   Loaded: loaded (/lib/systemd/system/etcd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-09-16 16:05:15 JST; 11s ago
     Docs: https://github.com/coreos/etcd
```

# Install OpenStack services
[URL](https://docs.openstack.org/install-guide/openstack-services.html#minimal-deployment-for-rocky)

# Keystone Installation Tutorial
## Install and configure
[URL](https://docs.openstack.org/keystone/rocky/install/keystone-install-ubuntu.html)
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;

$ sudo apt -y install keystone  apache2 libapache2-mod-wsgi
$ sudo vim /etc/keystone/keystone.conf
----
[database]
##connection = sqlite:////var/lib/keystone/keystone.db
connection = mysql+pymysql://keystone:password@192.168.0.200/keystone
[token]
provider = fernet
----

$ sudo mkdir -pv /etc/keystone/fernet-keys
$ sudo chown -v keystone:keystone /etc/keystone/fernet-keys
→ fernet用の鍵を作成する.

$ sudo bash -c "keystone-manage db_sync" keystone
$ sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage bootstrap --bootstrap-password password --bootstrap-admin-url http://192.168.0.200:5000/v3/ --bootstrap-internal-url http://192.168.0.200:5000/v3/ --bootstrap-public-url http://192.168.0.200:5000/v3/ --bootstrap-region-id RegionOne
```

## Configure the Apache HTTP server
```
$ sudo vim /etc/apache2/apache2.conf
----
ServerName 192.168.0.200
----
$ sudo apachectl configtest
$ sudo service apache2 restart
$ sudo service apache2 status
```

## Finalize the installation
```
$ export OS_USERNAME=admin
$ export OS_PASSWORD=password
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=default
$ export OS_PROJECT_DOMAIN_NAME=default
$ export OS_AUTH_URL=http://192.168.0.200:5000/v3
$ export OS_IDENTITY_API_VERSION=3
```

## Create a domain, projects, users, and roles
### Create a domain, projects
```
$ openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | bbf901b93ea6457380bf2e8e1a738dea |
| name        | example                          |
| tags        | []                               |
+-------------+----------------------------------+

$ openstack project create --domain default --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 5f6c99d3ec464221aba547bb8c216131 |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+

$ openstack project create --domain default --description "Demo Project" myproject
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 178b524c1c90427c89f5e675bff55a89 |
| is_domain   | False                            |
| name        | myproject                        |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

### Create Users
```
$ openstack user create --domain default --password-prompt myuser
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | ae1b3cd81eaf4da289498f978d8d5e1b |
| name                | myuser                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

$ openstack role create myrole
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 716709b6c5674c959f2f88a4625805e4 |
| name      | myrole                           |
+-----------+----------------------------------+

$ openstack role add --project myproject --user myuser myrole
```

### Verify operation
```
$ unset OS_AUTH_URL OS_PASSWORD
$ openstack --os-auth-url http://192.168.0.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-09-16T17:03:58+0000                                                                                                                                                                |
| id         | gAAAAABbnn7udr3T9lEjlu3-_5L9OhbbSAtRhPkjSY08eHiz0LVZPSE8KVQyVwjJto-cBilWIBm-CDfU-UqnbjTQQ1fO_JYHL92HG2lucBAu6dZClHpmwuWTFM1x99tHbv4v8GzeVPTU3C9Dww2KyZcROZvSvAVuK9YgBfa8hiF44OUb5FyfvKU |
| project_id | 81f50b65567c45f0839d5492a347b677                                                                                                                                                        |
| user_id    | 662f774d636d47579009932e7b3a4b18                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack --os-auth-url http://192.168.0.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name myproject --os-username myuser token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-09-16T17:04:40+0000                                                                                                                                                                |
| id         | gAAAAABbnn8YWTl6KxEOQ0jCLKEQlkZ2ivc8L7fmPLdp0CfsISj2tfGLspKPYV6_W9fSmoiCEYnVEsG3Zi62AmcCyrzpNoTdHTIkFYM4Go0qkPjcmxLSilom02b6W6L3O9EUvic8j38PWJz6JXK8WF7P9i4d-DOKb1d3OmH3kxQA__aCW1s6-po |
| project_id | 178b524c1c90427c89f5e675bff55a89                                                                                                                                                        |
| user_id    | ae1b3cd81eaf4da289498f978d8d5e1b                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

## Create OpenStack client environment scripts
### admin_openrc
```
$ cat << _EOF_ > ~/admin_openrc.sh
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.0.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
_EOF_

$ chmod -v 700 ~/admin_openrc.sh
```

### demo_openrc
```
$ cat << _EOF_ > ~/demo_openrc.sh
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.0.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
_EOF_
$ chmod -v 700 ~/demo_openrc.sh
```

### Using the scripts
```
$ source ~/admin_openrc.sh
$ openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-09-16T17:09:24+0000                                                                                                                                                                |
| id         | gAAAAABbnoA0AalF0CfqQT5ALeewURvtcwCqR-MkTMwu6zJ0-aX3VLOhNZj9j6-3t9geoy_brZ7LNFOU-CLV8QZXV-ZRlc3vMccmyL2MK0tEsctiz3UWQyPwIsEGAwa9OUijMEMKGuVsQfXqvByQT6tRFXZx8lcymJ75Fq5xiulgSdJjUjfKrag |
| project_id | 81f50b65567c45f0839d5492a347b677                                                                                                                                                        |
| user_id    | 662f774d636d47579009932e7b3a4b18                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ source ~/demo_openrc.sh 
$ openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-09-16T17:09:50+0000                                                                                                                                                                |
| id         | gAAAAABbnoBOWyGXwDy_tjBnTLEsL64CYwdnJy7qtJBf1EVgBjdGiOfZgK7A6iF349uRvjbaHTpDtpo1griRhOCoFV1qVxcMHF7KzJUodvE52B7b1fIvDjk8x56GF8K_Iqwv36A9aOHSAIN6UBabWzIA-C8-fU27yFuC_C8yCh4ohsQAo4uI4gM |
| project_id | 178b524c1c90427c89f5e675bff55a89                                                                                                                                                        |
| user_id    | ae1b3cd81eaf4da289498f978d8d5e1b                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

# glance installation for Rocky
[URL](https://docs.openstack.org/glance/rocky/install/)

## Install and configure (Ubuntu)
### Prerequisites
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;

$ source ~/admin_openrc.sh
$ openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | d712b665171f4534a5d187ee1423d90b |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

$ openstack role add --project service --user glance admin
$ openstack service create --name glance --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 061b7a2c4307445ea8e2f86839053518 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne image public http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 004a9cbda56e4adfb29ef74e09dc2fd8 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 061b7a2c4307445ea8e2f86839053518 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image internal http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | ecef30ffaa4a4cf98385e4baa04d8671 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 061b7a2c4307445ea8e2f86839053518 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image admin http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 2afcd9002c7442edafda3bf47d63d162 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 061b7a2c4307445ea8e2f86839053518 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+
```

### Install and configure components
```
$ sudo apt -y install glance
$ sudo vim /etc/glance/glance-api.conf
----
[database]
##connection = sqlite:////var/lib/glance/glance.sqlite
connection = mysql+pymysql://glance:password@192.168.0.200/glance

[keystone_authtoken]
www_authenticate_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:5000
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = password

[paste_deploy]
flavor = keystone

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
----

$ sudo vim  /etc/glance/glance-registry.conf
----
[database]
##connection = sqlite:////var/lib/glance/glance.sqlite
connection = mysql+pymysql://glance:password@192.168.0.200/glance

[keystone_authtoken]
www_authenticate_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:5000
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = password

[paste_deploy]
flavor = keystone
----

$ sudo bash -c "glance-manage db_sync" glance
```

### Finalize installation
```
$ sudo service glance-registry restart
$ sudo service glance-registry status
$ sudo service glance-api restart
$ sudo service glance-api status
```

## Verify operation
### add OS Image
```
$ source ~/admin_openrc.sh
$ wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
$ openstack image create "cirros" --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format 
bare --public
+------------------+------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------+
| Field            | Value                                                                                                      
                                                                                |
+------------------+------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe
...
| updated_at       | 2018-09-16T16:28:38Z                                                                                                                                                                       |
| virtual_size     | None                                                                                                                                                                                       |
| visibility       | public                                                                                                                                                                                     |
+------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| c6aa4743-a5fd-4d34-95f4-93afc2578fac | cirros | active |
+--------------------------------------+--------+--------+
→ OS ImageのステータスがActiveで出力されればOK.
```


## nova installation for Rocky
[URL](https://docs.openstack.org/nova/rocky/install/)

### Install and configure controller node for Ubuntu
#### Prerequisites
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;
MariaDB [(none)]> CREATE DATABASE placement;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;

$ source ~/admin_openrc.sh
$ openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | e49730a3f18c48c483a8f19e472f8a80 |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

$ openstack role add --project service --user nova admin

$ openstack service create --name nova --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | e013634c96ff4dd682fe9d5afa8e6b7e |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute public http://192.168.0.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 8f0e4f6d1fbd420d9ccdb064b35d027d |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | e013634c96ff4dd682fe9d5afa8e6b7e |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.0.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute internal http://192.168.0.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | ba886f602e714770a7357f892ea75745 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | e013634c96ff4dd682fe9d5afa8e6b7e |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.0.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute admin http://192.168.0.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | d96adee92f0346279873a43e8b319279 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | e013634c96ff4dd682fe9d5afa8e6b7e |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.0.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack user create --domain default --password-prompt placement
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 3f9b8c5c3c644114910a682c8a97422c |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------

$ openstack role add --project service --user placement admin

$ openstack service create --name placement --description "Placement API" placement
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | d3083338ca2d446eaa5d0df6f206323e |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement public http://192.168.0.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 5188c5d76e63499b8558476c76bac23b |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d3083338ca2d446eaa5d0df6f206323e |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.0.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement internal http://192.168.0.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0a8a17ade6b4418489a407a315f1f3a7 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d3083338ca2d446eaa5d0df6f206323e |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.0.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement admin http://192.168.0.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 86c7c44679eb4983be7f83ade60cf0de |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d3083338ca2d446eaa5d0df6f206323e |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.0.200:8778        |
+--------------+----------------------------------+
```

#### Install and configure components
Contorollerでインスタンス起動もさせたいので、nova-computeも入れておく.
```
$ sudo apt install -y nova-api nova-conductor nova-consoleauth nova-novncproxy nova-scheduler nova-placement-api nova-compute
$ sudo cp -raf /etc/nova ~
$ sudo vim /etc/nova/nova.conf
----
[api_database]
##connection = sqlite:////var/lib/nova/nova_api.sqlite
connection = mysql+pymysql://nova:password@192.168.0.200/nova_api

[database]
##connection = sqlite:////var/lib/nova/nova.sqlite
connection = mysql+pymysql://nova:password@192.168.0.200/nova

[placement_database]
connection = mysql+pymysql://placement:password@192.168.0.200/placement

[DEFAULT]
transport_url = rabbit://openstack:password@192.168.0.200
my_ip = 192.168.0.200
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://192.168.0.200:5000/v3
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = password


[neutron]
url = http://192.168.0.200:9696
auth_url = http://192.168.0.200:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = password
service_metadata_proxy = true
metadata_proxy_shared_secret = password

[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
api_servers = http://192.168.0.200:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
os_region_name = openstack
region_name = RegionOne
project_domain_name = default
project_name = service
auth_type = password
user_domain_name = default
auth_url = http://192.168.0.200:5000/v3
username = placement
password = password
----

$ sudo bash -c "nova-manage api_db sync" nova
$ sudo bash -c "nova-manage cell_v2 map_cell0" nova
$ sudo bash -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
bdec6b4e-2216-479e-b76a-800e80b524e9

$ sudo bash -c "nova-manage db sync" nova
$ sudo bash -c "nova-manage cell_v2 list_cells" nova
+-------+--------------------------------------+---------------------------------------+----------------------------------------------------+----------+
|  Name |                 UUID                 |             Transport URL             |                Database Connection                 | Disabled |
+-------+--------------------------------------+---------------------------------------+----------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                 none:/                | mysql+pymysql://nova:****@192.168.0.200/nova_cell0 |  False   |
| cell1 | bdec6b4e-2216-479e-b76a-800e80b524e9 | rabbit://openstack:****@192.168.0.200 |    mysql+pymysql://nova:****@192.168.0.200/nova    |  False   |
+-------+--------------------------------------+---------------------------------------+----------------------------------------------------+----------+
```

#### Finalize installation
```
$ for i in 'nova-api' 'nova-consoleauth' 'nova-scheduler' 'nova-conductor' 'nova-novncproxy' ; do sudo systemctl restart $i ; done
```

### Install and configure a compute node for Ubuntu
#### Install and configure components
```
$ sudo apt -y install nova-compute
$ sudo vim /etc/nova/nova.conf
----
→[DEFAULT]セクションなどはControllerNodeの変更と同じ.

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html

[scheduler]
discover_hosts_in_cells_interval = 60
----
```

#### Finalize installation
```
$ egrep -c '(vmx|svm)' /proc/cpuinfo
4

$ sudo cat /etc/nova/nova-compute.conf
----
[DEFAULT]
compute_driver=libvirt.LibvirtDriver
[libvirt]
virt_type=kvm
→ kvmのまま.
----

$ sudo service nova-compute restart
$ sudo service nova-compute status

$ source ~/admin_openrc.sh 
$ openstack compute service list
$ openstack compute service list --service nova-compute
→ なぜか出てこない.

$ sudo bash -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': bdec6b4e-2216-479e-b76a-800e80b524e9
Found 0 unmapped computes in cell: bdec6b4e-2216-479e-b76a-800e80b524e9
```

#### Verify operation
```
$ source ~/admin_openrc.sh
$ openstack compute service list
+----+------------------+-----------+----------+---------+-------+----------------------------+
| ID | Binary           | Host      | Zone     | Status  | State | Updated At                 |
+----+------------------+-----------+----------+---------+-------+----------------------------+
|  1 | nova-scheduler   | ryunosuke | internal | enabled | up    | 2018-09-17T05:27:33.000000 |
|  3 | nova-conductor   | ryunosuke | internal | enabled | up    | 2018-09-17T05:27:24.000000 |
|  5 | nova-consoleauth | ryunosuke | internal | enabled | up    | 2018-09-17T05:27:33.000000 |
|  6 | nova-compute     | ryunosuke | nova     | enabled | up    | 2018-09-17T05:27:28.000000 |
+----+------------------+-----------+----------+---------+-------+----------------------------+
→ downの場合は、設定とrabbitmq-serverが起動しているかを確認する


$ openstack catalog list
+-----------+-----------+--------------------------------------------+
| Name      | Type      | Endpoints                                  |
+-----------+-----------+--------------------------------------------+
| glance    | image     | RegionOne                                  |
|           |           |   public: http://192.168.0.200:9292        |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.0.200:9292         |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.0.200:9292      |
|           |           |                                            |
| keystone  | identity  | RegionOne                                  |
|           |           |   admin: http://192.168.0.200:5000/v3/     |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.0.200:5000/v3/  |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.0.200:5000/v3/    |
|           |           |                                            |
| placement | placement | RegionOne                                  |
|           |           |   internal: http://192.168.0.200:8778      |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.0.200:8778        |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.0.200:8778         |
|           |           |                                            |
| nova      | compute   | RegionOne                                  |
|           |           |   public: http://192.168.0.200:8774/v2.1   |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.0.200:8774/v2.1 |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.0.200:8774/v2.1    |
|           |           |                                            |
+-----------+-----------+--------------------------------------------+

$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| c6aa4743-a5fd-4d34-95f4-93afc2578fac | cirros | active |
+--------------------------------------+--------+--------+

$ sudo nova-status upgrade check
[sudo] password for ogalush:
Deprecated: Option "enable" from group "cells" is deprecated for removal (Cells v1 is being replaced with Cells v2.).  Its value may be silently ignored in the future.
+--------------------------------------------------------------------+
| Upgrade Check Results                                              |
+--------------------------------------------------------------------+
| Check: Cells v2                                                    |
| Result: Success                                                    |
| Details: No host mappings or compute nodes were found. Remember to |
|   run command 'nova-manage cell_v2 discover_hosts' when new        |
|   compute hosts are deployed.                                      |
+--------------------------------------------------------------------+
| Check: Placement API                                               |
| Result: Success                                                    |
| Details: None                                                      |
+--------------------------------------------------------------------+
| Check: Resource Providers                                          |
| Result: Success                                                    |
| Details: There are no compute resource providers in the Placement  |
|   service nor are there compute nodes in the database.             |
|   Remember to configure new compute nodes to report into the       |
|   Placement service. See                                           |
|   https://docs.openstack.org/nova/latest/user/placement.html       |
|   for more details.                                                |
+--------------------------------------------------------------------+
| Check: Ironic Flavor Migration                                     |
| Result: Success                                                    |
| Details: None                                                      |
+--------------------------------------------------------------------+
| Check: API Service Version                                         |
| Result: Success                                                    |
| Details: None                                                      |
+--------------------------------------------------------------------+
| Check: Request Spec Migration                                      |
| Result: Success                                                    |
| Details: None                                                      |
+--------------------------------------------------------------------+
```

# neutron installation for Rocky
## Install and configure for Ubuntu
### Install and configure controller node
#### Prerequisites
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE neutron;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;

$ source ~/admin_openrc.sh 
$ openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 8583188c44544ead914e169ca96b1bf7 |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

$ openstack role add --project service --user neutron admin
$ openstack service create --name neutron --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 474d4416b5574c478bfa86041447ea88 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne network public http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | b5391d19f86044b79d60e2e69fb56636 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 474d4416b5574c478bfa86041447ea88 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network internal http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 31770b41ec8242bd9d1d2c0105a7f0ed |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 474d4416b5574c478bfa86041447ea88 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network admin http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | d8408ead83e74c5794b1c4c6c904ad5d |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 474d4416b5574c478bfa86041447ea88 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+
```

#### Configure networking options
##### Networking Option 2: Self-service networks
```
$ sudo apt -y install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent

$ sudo vim /etc/neutron/neutron.conf
----
[database]
##connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql+pymysql://neutron:password@192.168.0.200/neutron

[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:password@192.168.0.200
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[keystone_authtoken]
www_authenticate_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:5000
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password

[nova]
auth_url = http://192.168.0.200:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = password
----
```

##### Configure the Modular Layer 2 (ML2) plug-in
```
$ sudo vim /etc/neutron/plugins/ml2/ml2_conf.ini
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
enable_ipset = true
----
```

##### Configure the Linux bridge agent
```
$ sudo vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
----
[linux_bridge]
physical_interface_mappings = provider:enp3s0

[vxlan]
enable_vxlan = true
local_ip = 192.168.0.200
l2_population = true

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
----

$ sudo vim /etc/sysctl.d/90-neutron.conf
---
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
---

$ sudo sysctl -p
$ sudo sysctl -a 2> /dev/null |grep -e 'net.bridge.bridge-nf-call-iptables' -e 'net.bridge.bridge-nf-call-ip6tables' 
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
→ 1になっていればOK.
```

##### Configure the layer-3 agent
```
$ sudo vim /etc/neutron/l3_agent.ini
----
[DEFAULT]
interface_driver = linuxbridge
----
```

##### Configure the DHCP agent
```
$ sudo vim /etc/neutron/dhcp_agent.ini
----
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
----
```

#### Configure the metadata agent
```
$ sudo vim /etc/neutron/metadata_agent.ini
----
[DEFAULT]
nova_metadata_host = 192.168.0.200
metadata_proxy_shared_secret = password
----

$ sudo vim /etc/nova/nova.conf
----
[neutron]
url = http://192.168.0.200:9696
auth_url = http://192.168.0.200:5000
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

### Finalize installation
```
$ sudo bash -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
$ for i in 'nova-api' 'nova-consoleauth' 'nova-scheduler' 'nova-conductor' 'nova-novncproxy' 'nova-compute' 'neutron-server' 'neutron-linuxbridge-agent' 'neutron-dhcp-agent' 'neutron-metadata-agent' 'neutron-l3-agent' ; do sudo systemctl restart $i ; done
```

## Install and configure compute node
### Install the components
```
$ sudo apt -y install neutron-linuxbridge-agent
$ sudo grep -e '^transport_url' /etc/neutron/neutron.conf
transport_url = rabbit://openstack:password@192.168.0.200
→ [DEFAULT], [keystone_authtoken]セクションはControllerと同じ.
```
### Configure networking options
→ 設定内容はControllerと同じ.

### Finalize installation
```
$ sudo service nova-compute restart
$ sudo service neutron-linuxbridge-agent restart
```

## Verify operation
### Networking Option 2: Self-service networks
```
$ source ~/admin_openrc.sh 
ogalush@ryunosuke:~$ openstack network agent list
Unable to establish connection to http://192.168.0.200:9696/v2.0/agents: ('Connection aborted.', BadStatusLine("''",))
Segmentation fault (core dumped)
ogalush@ryunosuke:~$ 
→ 表示されない.
```
