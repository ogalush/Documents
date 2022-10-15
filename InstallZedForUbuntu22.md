# Install OpenStack Zed on Ubuntu 22.04
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.3.200(192.168.3.200)  
設定ファイル: [URL](URL)
```
$ uname -na
Linux ryunosuke 5.15.0-50-generic #56-Ubuntu SMP Tue Sep 20 13:23:26 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```

# Base Installation
個人アカウント、ntpdなどの基本パッケージをインストールする.
## install Base Packages.
```
@admin
$ ansible-playbook -i "192.168.3.200," initialize_instance.yml --user=ubuntu -k -bK --list-hosts
$ ansible-playbook -i "192.168.3.200," initialize_instance.yml --user=ubuntu -k -bK
```
## disable systemd-resolved
/etc/resolve.confの問い合わせ先が `127.0.0.53` となっておりDNS正引きが不調だったため無効化する.  
https://blog.jicoman.info/2020/06/how-to-resolve-problem-of-name-resolution-to-local-on-ubuntu-2004/
```
$ sudo vim /etc/systemd/resolved.conf
----
DNSStubListener=no
----

$ sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
$ sudo systemctl restart systemd-resolved.service
$ grep -v '#' /etc/resolv.conf 
----
nameserver 8.8.8.8
search localdomain
----
$ dig +short www.google.com
172.217.31.164
```


# Environment
## Host networking
## Controller node
https://docs.openstack.org/install-guide/environment-networking-controller.html  
127.0.1.1 がホスト名に紐づいているので、実際のIPアドレスへ置換する.
```
$ sudo cp -pv /etc/hosts /tmp/hosts
'/etc/hosts' -> '/tmp/hosts'
$ sudo vim /etc/hosts
$ diff --unified=0 /tmp/hosts /etc/hosts
--- /tmp/hosts  2022-10-15 19:07:55.762188109 +0900
+++ /etc/hosts  2022-10-15 19:38:59.010190148 +0900
@@ -2 +2 @@
-127.0.1.1 ryunosuke
+192.168.3.200 ryunosuke
```

## Network Time Protocol (NTP)
https://docs.openstack.org/install-guide/environment-ntp.html  
時刻同期できているのでOK.
```
$ ntpq -p |grep '*'
*ntp-b2.nict.go. .NICT.           1 u   38   64  377    4.080  -44.095  23.571
```


# OpenStack packages for Ubuntu
https://docs.openstack.org/install-guide/environment-packages-ubuntu.html  
## Archive Enablement
 `OpenStack Yoga is available by default using Ubuntu 22.04 LTS.` となっているため変更する.
```
$ sudo add-apt-repository cloud-archive:zed
Repository: 'deb http://ubuntu-cloud.archive.canonical.com/ubuntu jammy-updates/zed main'
Description:
Ubuntu Cloud Archive for OpenStack Zed
More info: https://wiki.ubuntu.com/OpenStack/CloudArchive
Adding repository.
Press [ENTER] to continue or Ctrl-c to cancel.
```

## Sample, Client Installation
インストール
```
$ sudo apt -y install python3-openstackclient nova-compute
$ sudo apt -y update
$ sudo apt -y upgrade
$ sudo apt -y dist-upgrade
$ sudo apt -y autoremove
```

# SQL database for Ubuntu
https://docs.openstack.org/install-guide/environment-sql-database-ubuntu.html  
## Install and configure components
```
$ sudo apt -y install mariadb-server python3-pymysql
$ sudo cp -rafv /etc/mysql ~
$ sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf
$ diff --unified=0 ~/mysql/mariadb.conf.d/50-server.cnf /etc/mysql/mariadb.conf.d/50-server.cnf
--- /home/ogalush/mysql/mariadb.conf.d/50-server.cnf    2022-04-28 03:23:07.000000000 +0900
+++ /etc/mysql/mariadb.conf.d/50-server.cnf     2022-10-15 19:53:25.969862493 +0900
@@ -27 +27 @@
-bind-address            = 127.0.0.1
+bind-address            = 192.168.3.200
@@ -90,2 +90,2 @@
-character-set-server  = utf8mb4
-collation-server      = utf8mb4_general_ci
+character-set-server  = utf8
+collation-server      = utf8_general_ci
```

## Finalize installation
```
$ sudo service mysql restart
$ sudo service mysql status |grep Active
  Active: active (running) since Sat 2022-10-15 19:54:16 JST; 4s ago

$ sudo mysql_secure_installation
Enter current password for root (enter for none): [enter]
Switch to unix_socket authentication [Y/n] y
Change the root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

# Message queue for Ubuntu
## Install and configure components
https://docs.openstack.org/install-guide/environment-messaging-ubuntu.html  
```
$ sudo apt -y install rabbitmq-server
$ sudo rabbitmqctl add_user openstack password
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

# Memcached for Ubuntu
## Install and configure components
https://docs.openstack.org/install-guide/environment-memcached-ubuntu.html
```
$ sudo apt -y install memcached python3-memcache
$ sudo cp -pv /etc/memcached.conf ~
$ sudo sed -i 's/-l 127.0.0.1/-l 192.168.3.200/g' /etc/memcached.conf
$ diff --unified=0 ~/memcached.conf /etc/memcached.conf
--- /home/ogalush/memcached.conf        2022-10-15 20:04:11.108151064 +0900
+++ /etc/memcached.conf 2022-10-15 20:04:44.327436187 +0900
@@ -35 +35 @@
--l 127.0.0.1
+-l 192.168.3.200
```

## Finalize installation
```
$ sudo service memcached restart
$ sudo service memcached status |grep Active
  Active: active (running) since Sat 2022-10-15 20:05:17 JST; 4s ago
```

# Etcd for Ubuntu
## Install and configure components
```
$ sudo apt -y install etcd
$ sudo cp -pv /etc/default/etcd ~
'/etc/default/etcd' -> '/home/ogalush/etcd'
$ sudo vim /etc/default/etcd
----
## etcd(1) daemon options
## See "/usr/share/doc/etcd-server/op-guide/configuration.md.gz"

### Member flags
ETCD_NAME="ryunosuke"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER="ryunosuke=http://192.168.3.200:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.3.200:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.3.200:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.3.200:2379"
----
```

## Finalize installation
```
$ sudo systemctl enable etcd
$ sudo systemctl restart etcd
$ sudo systemctl is-active etcd
active
$ sudo systemctl status etcd |grep Active
 Active: active (running) since Sat 2022-10-15 20:15:05 JST; 34s ago
```


# Install OpenStack services
# Keystone Installation Tutorial for Ubuntu
https://docs.openstack.org/keystone/zed/install/index-ubuntu.html
## Install and configure
https://docs.openstack.org/keystone/zed/install/keystone-install-ubuntu.html
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;
Bye
```

## Install and configure components
```
$ sudo apt -y install keystone
$ sudo cp -rafv /etc/keystone ~
$ sudo vim /etc/keystone/keystone.conf
$ sudo diff --unified=0 ~/keystone/keystone.conf /etc/keystone/keystone.conf
----
--- /home/ogalush/keystone/keystone.conf        2022-10-07 08:07:22.000000000 +0900
+++ /etc/keystone/keystone.conf 2022-10-15 20:23:56.508750128 +0900
@@ -661 +661 @@
-connection = sqlite:////var/lib/keystone/keystone.db
+connection = mysql+pymysql://keystone:password@192.168.3.200/keystone
@@ -2610,0 +2611 @@
+provider = fernet
----

$ sudo -s /bin/sh -c "keystone-manage db_sync" keystone
$ sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage bootstrap --bootstrap-password password --bootstrap-admin-url http://192.168.3.200:5000/v3/ --bootstrap-internal-url http://192.168.3.200:5000/v3/ --bootstrap-public-url http://192.168.3.200:5000/v3/ --bootstrap-region-id RegionOne

$ sudo cp -rafv /etc/apache2 ~
$ sudo vim /etc/apache2/apache2.conf
----
+ ServerName 192.168.3.200
----
```

## Finalize the installation
```
$ sudo service apache2 restart
$ sudo service apache2 status |grep Active
 Active: active (running) since Sat 2022-10-15 20:34:18 JST; 3s ago
$ export OS_USERNAME=admin
$ export OS_PASSWORD=password
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=default
$ export OS_PROJECT_DOMAIN_NAME=default
$ export OS_AUTH_URL=http://192.168.3.200:5000/v3
$ export OS_IDENTITY_API_VERSION=3
```

## Create a domain, projects, users, and roles
https://docs.openstack.org/keystone/zed/install/keystone-users-obs.html
```
$ openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | 96df3f52240241bab7f561d155a56a73 |
| name        | example                          |
| options     | {}                               |
| tags        | []                               |
+-------------+----------------------------------+


$ openstack project create --domain default --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 136191f497334c9d8c49cb115d71fc2b |
| is_domain   | False                            |
| name        | service                          |
| options     | {}                               |
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
| id          | ce6550a158c64ea6b54883cf74064f10 |
| is_domain   | False                            |
| name        | myproject                        |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+


$ openstack user create --domain default --password-prompt myuser
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 27745669d8b54d46958cce0a913edb2e |
| name                | myuser                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

$ openstack role create myrole
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| domain_id   | None                             |
| id          | 75e8e41b5749485bab5e2e85c02912e8 |
| name        | myrole                           |
| options     | {}                               |
+-------------+----------------------------------+

$ openstack role add --project myproject --user myuser myrole
$
```


## Verify operation
https://docs.openstack.org/keystone/zed/install/keystone-verify-obs.html
```
$ unset OS_AUTH_URL OS_PASSWORD
$ openstack --os-auth-url http://192.168.3.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2022-10-15T12:40:12+0000                                                                                                                                                                |
| id         | gAAAAABjSpwcWVCsxGNUlfwQBNEOhkF_Adhi2GlPnv6sOpWeS-pWhQtQcVB6rMBXehKBTgEGRi3SbrSaGXT8hc4s60hYK7xdtUMZ79JPkQo7CxYDXxaOBC1uQQX6hl3vdUzDAaKxM-cx5QJIHjIPxiF2uX_IpWfSXvSjkFcYf3QJpWyMBwRA8Ws |
| project_id | 32b4bc8011454da3837a54d15e16d31e                                                                                                                                                        |
| user_id    | eca11f2567d34b8e932ac35aa99e242b                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+


$ openstack --os-auth-url http://192.168.3.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name myproject --os-username myuser token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2022-10-15T12:40:41+0000                                                                                                                                                                |
| id         | gAAAAABjSpw5lKUdegXK3-atZ7MDHrSk9YyRs-1TxPnSeJUu3qedhkAtfZa7hi3HqOMQthC1br2Wk5e9qEep09WQ86ddWEtQr4vzh5rnbCTXFgT1b-41jbuQ5tDNKhSktIyiJBdc0wzisgdr3CJB1w5Kgnr89q2QKP8C2oRkfD-Va4Rff2qFMVc |
| project_id | ce6550a158c64ea6b54883cf74064f10                                                                                                                                                        |
| user_id    | 27745669d8b54d46958cce0a913edb2e                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

## Create OpenStack client environment scripts
https://docs.openstack.org/keystone/zed/install/keystone-openrc-obs.html
```
$ cat > ~/admin-openrc << __EOF__
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.3.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
__EOF__

$ cat > ~/demo-openrc << __EOF__
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.3.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
__EOF__
```

## Using the scripts
```
$ source ~/admin-openrc
$ openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2022-10-15T12:42:14+0000                                                                                                                                                                |
| id         | gAAAAABjSpyW7eDPQnJZpCCB7iAHj5dSppEOyNsIIubB8C9Q5WPgJDxzMNq09lW6P864_otsZuo3cQTIgwIUI4iLhoTvhMg9mVvx_s2rgycyO3l2WiD78VTUPejiMlg9-jmsVFWtJoe1He-gJUc-4ZiIAsD_kb2HodKFcRyBMPHuo6ubKG9c0S8 |
| project_id | 32b4bc8011454da3837a54d15e16d31e                                                                                                                                                        |
| user_id    | eca11f2567d34b8e932ac35aa99e242b                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ source ~/demo-openrc
$ openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2022-10-15T12:42:47+0000                                                                                                                                                                |
| id         | gAAAAABjSpy3sW8-_tdeQLTO3U1d_OtWCcqgZjCtAuf48cbfNNm-VPVDm4J02NupMW1xHSNrCEUAmzkwnLy9GriQyFbTPFSpjc1bLDrSRqUelIL8jflSDwXANEp7Or8WSTjlIMtH4rP1zXxmCc2Zkn_tvEtBDwiFuLRvaGRH7ELU4ov9YXIhS4Y |
| project_id | ce6550a158c64ea6b54883cf74064f10                                                                                                                                                        |
| user_id    | 27745669d8b54d46958cce0a913edb2e                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

# Glance Installation
https://docs.openstack.org/glance/zed/install/
## Install and configure (Ubuntu)
https://docs.openstack.org/glance/zed/install/install-ubuntu.html
### Prerequisites
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;

$ source ~/admin-openrc
$ openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 6be53b8068404e1a86f33501b8fb1407 |
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
| id          | a4dc223f1eb84348bca163eba68b5f69 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne image public http://192.168.3.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | bfe8549bcd5a43179d39a1270bb64d0d |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a4dc223f1eb84348bca163eba68b5f69 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.3.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image internal http://192.168.3.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 1fa44fb0fe664ad5bee05262ae9893ac |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a4dc223f1eb84348bca163eba68b5f69 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.3.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image admin http://192.168.3.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 364dc2296b5442d8813b38721256143b |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a4dc223f1eb84348bca163eba68b5f69 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.3.200:9292        |
+--------------+----------------------------------+
```


### Install and configure components
```
$ sudo apt -y install glance
$ sudo cp -rafv /etc/glance ~
$ sudo vim /etc/glance/glance-api.conf
-----
[database]
- connection = sqlite:////var/lib/glance/glance.sqlite
+ connection = mysql+pymysql://glance:password@192.168.3.200/glance
...
[keystone_authtoken]
+ www_authenticate_uri = http://192.168.3.200:5000
+ auth_url = http://192.168.3.200:5000
+ memcached_servers = 192.168.3.200:11211
+ auth_type = password
+ project_domain_name = default
+ user_domain_name = default
+ project_name = service
+ username = glance
+ password = password
...
[paste_deploy]
+ flavor = keystone
...
[glance_store]
+ stores = file,http
+ default_store = file
+ filesystem_store_datadir = /var/lib/glance/images/

[oslo_limit] に関しては制限になるため設定を省略する.
...
[DEFAULT]
use_keystone_quotas = True
-----

$ sudo -s /bin/sh -c "glance-manage db_sync" glance
```
### Finalize installation
```
$ sudo service glance-api restart
$ sudo service glance-api status |grep Active
 Active: active (running) since Sat 2022-10-15 20:59:27 JST; 3s ago
```

## Verify operation
https://docs.openstack.org/glance/zed/install/verify.html
```
$ source ~/admin-openrc
$ wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
$ glance image-create --name "cirros" --file /usr/local/src/cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --visibility=public
+------------------+----------------------------------------------------------------------------------+
| Property         | Value                                                                            |
+------------------+----------------------------------------------------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe                                                 |
| container_format | bare                                                                             |
| created_at       | 2022-10-15T12:01:25Z                                                             |
| disk_format      | qcow2                                                                            |
| id               | 65933176-c232-460a-8c46-062ee5266a35                                             |
| min_disk         | 0                                                                                |
| min_ram          | 0                                                                                |
| name             | cirros                                                                           |
| os_hash_algo     | sha512                                                                           |
| os_hash_value    | 6513f21e44aa3da349f248188a44bc304a3653a04122d8fb4535423c8e1d14cd6a153f735bb0982e |
|                  | 2161b5b5186106570c17a9e58b64dd39390617cd5a350f78                                 |
| os_hidden        | False                                                                            |
| owner            | 32b4bc8011454da3837a54d15e16d31e                                                 |
| protected        | False                                                                            |
| size             | 12716032                                                                         |
| status           | active                                                                           |
| tags             | []                                                                               |
| updated_at       | 2022-10-15T12:01:25Z                                                             |
| virtual_size     | 46137344                                                                         |
| visibility       | public                                                                           |
+------------------+----------------------------------------------------------------------------------+

$ glance image-list
+--------------------------------------+--------+
| ID                                   | Name   |
+--------------------------------------+--------+
| 65933176-c232-460a-8c46-062ee5266a35 | cirros |
+--------------------------------------+--------+
```


# placement installation for Zed
https://docs.openstack.org/placement/zed/install/
## Install and configure Placement for Ubuntu
https://docs.openstack.org/placement/zed/install/install-ubuntu.html
### Create Database
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE placement;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;
```

### Configure User and Endpoints
```
$ source ~/admin-openrc
$ openstack user create --domain default --password-prompt placement
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | ed762d30c746492f90888f3566611a83 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

$ openstack role add --project service --user placement admin
$ openstack service create --name placement --description "Placement API" placement
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | b60fe2e8d5ba4bed803439cbd560575e |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement public http://192.168.3.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | c417ea178e6840cf9a340c0c41a418c3 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b60fe2e8d5ba4bed803439cbd560575e |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.3.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement internal http://192.168.3.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | eec38f0095cd40bbb3e4646fa0811b06 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b60fe2e8d5ba4bed803439cbd560575e |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.3.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement admin http://192.168.3.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | c232110eb52447bead6dd7ebd11a2621 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b60fe2e8d5ba4bed803439cbd560575e |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.3.200:8778        |
+--------------+----------------------------------+
```

### Install and configure components
```
$ sudo apt -y install placement-api
$ sudo cp -rafv /etc/placement ~
$ sudo vim /etc/placement/placement.conf
$ sudo diff -u ~/placement/placement.conf /etc/placement/placement.conf
-----
--- /home/ogalush/placement/placement.conf      2022-10-07 10:45:58.000000000 +0900
+++ /etc/placement/placement.conf       2022-10-15 21:09:22.430678076 +0900
@@ -189,6 +189,7 @@
 
 
 [api]
+auth_strategy = keystone
 #
 # Options under this group are used to define Placement API.
 
@@ -238,6 +239,14 @@
 
 
 [keystone_authtoken]
+auth_url = http://192.168.3.200:5000/v3
+memcached_servers = 192.168.3.200:11211
+auth_type = password
+project_domain_name = default
+user_domain_name = default
+project_name = service
+username = placement
+password = password
 
 #
 # From keystonemiddleware.auth_token
@@ -512,7 +521,7 @@
 
 
 [placement_database]
-connection = sqlite:////var/lib/placement/placement.sqlite
+connection = mysql+pymysql://placement:password@192.168.3.200/placement
 #
 # The *Placement API Database* is a the database used with the placement
 # service. If the connection option is not set, the placement service will
-----

$ sudo -s /bin/sh -c "placement-manage db sync" placement
$ sudo service apache2 restart
$ sudo service apache2 status |grep Active
 Active: active (running) since Sat 2022-10-15 21:11:25 JST; 4s ago
```

## Verify Installation
https://docs.openstack.org/placement/zed/install/verify.html
```
$ source ~/admin-openrc
$ sudo placement-status upgrade check
+-------------------------------------------+
| Upgrade Check Results                     |
+-------------------------------------------+
| Check: Missing Root Provider IDs          |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Incomplete Consumers               |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Policy File JSON to YAML Migration |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+


$ sudo apt -y install python3-pip
$ sudo pip3 install osc-placement
$ openstack --os-placement-api-version 1.2 resource class list --sort-column name
openstack: '--os-placement-api-version 1.2 resource class list --sort-column name' is not an openstack command. See 'openstack --help'.
Did you mean one of these?
  application credential create
  application credential delete
  application credential list
  application credential show
  configuration show
$ openstack --os-placement-api-version 1.6 trait list --sort-column name
openstack: '--os-placement-api-version 1.6 trait list --sort-column name' is not an openstack command. See 'openstack --help'.
Did you mean one of these?
  application credential create
  application credential delete
  application credential list
  application credential show
  configuration show
→ 確認コマンドが無くなった模様.
```




# Compute service
https://docs.openstack.org/nova/zed/install/
## Install and configure controller node for Ubuntu
https://docs.openstack.org/nova/zed/install/controller-install-ubuntu.html
### Prerequisites
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;


$ source ~/admin-openrc
$ openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 5bdbfca993b64f35b4e2507420bb6b73 |
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
| id          | eb69807631074078bb0916ceadb9371a |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute public http://192.168.3.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 5b5d96fd20304b6f88c6c69be0aa29d4 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | eb69807631074078bb0916ceadb9371a |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.3.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute internal http://192.168.3.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 423c6f7d2af44c2996507da1fa3c3d46 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | eb69807631074078bb0916ceadb9371a |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.3.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute admin http://192.168.3.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | c545dd51760041188035801416d17157 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | eb69807631074078bb0916ceadb9371a |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.3.200:8774/v2.1   |
+--------------+----------------------------------+
```

### Install and configure components
```
$ sudo apt -y install nova-api nova-conductor nova-novncproxy nova-scheduler
$ sudo cp -rafv /etc/nova ~
$ sudo vim /etc/nova/nova.conf
$ sudo diff -u ~/nova/nova.conf /etc/nova/nova.conf
----
--- /home/ogalush/nova/nova.conf        2022-10-07 07:35:50.000000000 +0900
+++ /etc/nova/nova.conf 2022-10-15 21:31:31.552012335 +0900
@@ -2,6 +2,8 @@
 log_dir = /var/log/nova
 lock_path = /var/lock/nova
 state_path = /var/lib/nova
+transport_url = rabbit://openstack:password@192.168.3.200:5672/
+my_ip = 192.168.3.200

 #
 # From nova.conf
@@ -884,6 +886,7 @@


 [api]
+auth_strategy = keystone
 #
 # Options under this group are used to define Nova API.

@@ -1095,7 +1098,7 @@


 [api_database]
-connection = sqlite:////var/lib/nova/nova_api.sqlite
+connection = mysql+pymysql://nova:password@192.168.3.200/nova_api
 #
 # The *Nova API Database* is a separate database which is used for information
 # which is used across *cells*. This database is mandatory since the Mitaka
@@ -1838,7 +1841,7 @@


 [database]
-connection = sqlite:////var/lib/nova/nova.sqlite
+connection = mysql+pymysql://nova:password@192.168.3.200/nova
 #
 # The *Nova Database* is the primary database which is used for information
 # local to a *cell*.
@@ -2136,6 +2139,7 @@


 [glance]
+api_servers = http://192.168.3.200:9292
 # Configuration options for the Image service

 #
@@ -2841,6 +2845,15 @@


 [keystone_authtoken]
+www_authenticate_uri = http://192.168.3.200:5000/
+auth_url = http://192.168.3.200:5000/
+memcached_servers = 192.168.3.200:11211
+auth_type = password
+project_domain_name = default
+user_domain_name = default
+project_name = service
+username = nova
+password = password

 #
 # From keystonemiddleware.auth_token
@@ -3891,6 +3904,7 @@


 [oslo_concurrency]
+lock_path = /var/lib/nova/tmp

 #
 # From oslo.concurrency
@@ -4641,6 +4655,14 @@


 [placement]
+region_name = RegionOne
+project_domain_name = default
+project_name = service
+auth_type = password
+user_domain_name = default
+auth_url = http://192.168.3.200:5000/v3
+username = placement
+password = password

 #
 # From nova.conf
@@ -5641,6 +5663,9 @@


 [vnc]
+enabled = true
+server_listen = $my_ip
+server_proxyclient_address = $my_ip
 #
 # Virtual Network Computer (VNC) can be used to provide remote desktop
 # console access to instances for tenants and/or administrators.
----

$ sudo -s /bin/sh -c "nova-manage api_db sync" nova
$ sudo -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
$ sudo -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
Modules with known eventlet monkey patching issues were imported prior to eventlet monkey patching: urllib3. This warning can usually be ignored if the caller is only importing and not executing nova code.
--transport-url not provided in the command line, using the value [DEFAULT]/transport_url from the configuration file
--database_connection not provided in the command line, using the value [database]/connection from the configuration file
77290bf0-22a9-43a4-ae8c-4373e25c4f05

$ sudo -s /bin/sh -c "nova-manage db sync" nova
$ sudo -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
Modules with known eventlet monkey patching issues were imported prior to eventlet monkey patching: urllib3. This warning can usually be ignored if the caller is only importing and not executing nova code.
+-------+--------------------------------------+---------------------------------------------+----------------------------------------------------+----------+
|  Name |                 UUID                 |                Transport URL                |                Database Connection                 | Disabled |
+-------+--------------------------------------+---------------------------------------------+----------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                    none:/                   | mysql+pymysql://nova:****@192.168.3.200/nova_cell0 |  False   |
| cell1 | 77290bf0-22a9-43a4-ae8c-4373e25c4f05 | rabbit://openstack:****@192.168.3.200:5672/ |    mysql+pymysql://nova:****@192.168.3.200/nova    |  False   |
+-------+--------------------------------------+---------------------------------------------+----------------------------------------------------+----------+
```

### Finalize installation
```
$ sudo systemctl restart nova-api nova-scheduler nova-conductor nova-novncproxy
$ sudo systemctl status nova-api nova-scheduler nova-conductor nova-novncproxy |grep Active
     Active: active (running) since Sat 2022-10-15 21:36:34 JST; 6s ago
     Active: active (running) since Sat 2022-10-15 21:36:34 JST; 6s ago
     Active: active (running) since Sat 2022-10-15 21:36:34 JST; 6s ago
     Active: active (running) since Sat 2022-10-15 21:36:34 JST; 6s ago
```

## Install and configure a compute node for Ubuntu
https://docs.openstack.org/nova/zed/install/compute-install-ubuntu.html
### Install and configure components
```
$ sudo apt -y install nova-compute
$ sudo vim /etc/nova/nova.conf
----
[vnc]
...
+ novncproxy_base_url = http://192.168.3.200:6080/vnc_auto.html
...
[scheduler]
+ discover_hosts_in_cells_interval = 300
----
```
### Finalize installation
```
$ egrep -c '(vmx|svm)' /proc/cpuinfo
8

$ sudo vim /etc/nova/nova-compute.conf
----
...
[libvirt]
virt_type=kvm ・・・KVMであればOK.
----
$ sudo systemctl restart nova-compute
$ sudo systemctl status nova-compute |grep Active
 Active: active (running) since Sat 2022-10-15 21:41:10 JST; 5s ago
```

### Add the compute node to the cell database
```
$ source ~/admin-openrc
$ openstack compute service list --service nova-compute
+--------------------------------------+--------------+-----------+------+---------+-------+----------------------------+
| ID                                   | Binary       | Host      | Zone | Status  | State | Updated At                 |
+--------------------------------------+--------------+-----------+------+---------+-------+----------------------------+
| 4d20a7b1-d08d-442f-b136-7ea67d7d7b85 | nova-compute | ryunosuke | nova | enabled | up    | 2022-10-15T12:41:58.000000 |
+--------------------------------------+--------------+-----------+------+---------+-------+----------------------------+


$ sudo -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
Modules with known eventlet monkey patching issues were imported prior to eventlet monkey patching: urllib3. This warning can usually be ignored if the caller is only importing and not executing nova code.
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': 77290bf0-22a9-43a4-ae8c-4373e25c4f05
Checking host mapping for compute host 'ryunosuke': 2a55b0ff-00f1-4691-8365-a6c8e1b289ab
Creating host mapping for compute host 'ryunosuke': 2a55b0ff-00f1-4691-8365-a6c8e1b289ab
Found 1 unmapped computes in cell: 77290bf0-22a9-43a4-ae8c-4373e25c4f05
```

## Verify operation
https://docs.openstack.org/nova/zed/install/verify.html
```
$ source ~/admin-openrc
$ openstack compute service list
+--------------------------------------+----------------+-----------+----------+---------+-------+----------------------------+
| ID                                   | Binary         | Host      | Zone     | Status  | State | Updated At                 |
+--------------------------------------+----------------+-----------+----------+---------+-------+----------------------------+
| efcfa3f5-ad1e-41b1-a0f5-7367d707e38a | nova-scheduler | ryunosuke | internal | enabled | up    | 2022-10-15T12:43:41.000000 |
| 5a0d036e-89db-4940-aa82-150acb40bc6b | nova-conductor | ryunosuke | internal | enabled | up    | 2022-10-15T12:43:41.000000 |
| 4d20a7b1-d08d-442f-b136-7ea67d7d7b85 | nova-compute   | ryunosuke | nova     | enabled | up    | 2022-10-15T12:43:48.000000 |
+--------------------------------------+----------------+-----------+----------+---------+-------+----------------------------+

$ openstack catalog list
+-----------+-----------+--------------------------------------------+
| Name      | Type      | Endpoints                                  |
+-----------+-----------+--------------------------------------------+
| keystone  | identity  | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:5000/v3/  |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:5000/v3/     |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.3.200:5000/v3/    |
|           |           |                                            |
| glance    | image     | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:9292      |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:9292         |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.3.200:9292        |
|           |           |                                            |
| placement | placement | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:8778         |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.3.200:8778        |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:8778      |
|           |           |                                            |
| nova      | compute   | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:8774/v2.1 |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.3.200:8774/v2.1   |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:8774/v2.1    |
|           |           |                                            |
+-----------+-----------+--------------------------------------------+

$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 65933176-c232-460a-8c46-062ee5266a35 | cirros | active |
+--------------------------------------+--------+--------+


$ sudo nova-status upgrade check
Modules with known eventlet monkey patching issues were imported prior to eventlet monkey patching: urllib3. This warning can usually be ignored if the caller is only importing and not executing nova code.
+-------------------------------------------+
| Upgrade Check Results                     |
+-------------------------------------------+
| Check: Cells v2                           |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Placement API                      |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Cinder API                         |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Policy File JSON to YAML Migration |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Older than N-1 computes            |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: hw_machine_type unset              |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
```


# Neutron
https://docs.openstack.org/neutron/zed/install/
## Install and configure for Ubuntu
https://docs.openstack.org/neutron/zed/install/install-ubuntu.html
### Install and configure controller node
https://docs.openstack.org/neutron/zed/install/controller-install-ubuntu.html
#### Prerequisites
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE neutron;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;

$ source ~/admin-openrc
$ openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 8649665c5e524e5ea4e03345cee830a7 |
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
| id          | 086adf14468a44d4a355b974c5681a24 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne network public http://192.168.3.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 457ef68776414aaa8ba5cae90030a811 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 086adf14468a44d4a355b974c5681a24 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.3.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network internal http://192.168.3.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 63872ce016bb4efebc8da20e23406785 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 086adf14468a44d4a355b974c5681a24 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.3.200:9696        |
+--------------+----------------------------------+
  
$ openstack endpoint create --region RegionOne network admin http://192.168.3.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 4445329e19184aeca9984a4e299f805a |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 086adf14468a44d4a355b974c5681a24 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.3.200:9696        |
+--------------+----------------------------------+
```

### Configure networking options
Networking Option 2: Self-service networks
https://docs.openstack.org/neutron/zed/install/controller-install-option2-ubuntu.html
#### Install the components
```
$ sudo apt -y install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```
#### Configure the server component
```
$ sudo cp -rafv /etc/neutron ~
$ sudo vim /etc/neutron/neutron.conf
$ sudo diff --unified=2 ~/neutron/neutron.conf /etc/neutron/neutron.conf
----
--- /home/ogalush/neutron/neutron.conf  2022-10-07 11:45:06.000000000 +0900
+++ /etc/neutron/neutron.conf   2022-10-15 23:21:36.671706296 +0900
@@ -1,4 +1,9 @@
 [DEFAULT]
 core_plugin = ml2
+service_plugins = router
+transport_url = rabbit://openstack:password@192.168.3.200
+auth_strategy = keystone
+notify_nova_on_port_status_changes = true
+notify_nova_on_port_data_changes = true

 #
@@ -873,6 +878,6 @@

 [database]
-connection = sqlite:////var/lib/neutron/neutron.sqlite
-
+connection = mysql+pymysql://neutron:password@192.168.3.200/neutron
+
 #
 # From neutron.db
@@ -980,4 +985,5 @@

 [experimental]
+linuxbridge = true

 #
@@ -1123,4 +1129,13 @@

 [keystone_authtoken]
+www_authenticate_uri = http://192.168.3.200:5000
+auth_url = http://192.168.3.200:5000
+memcached_servers = 192.168.3.200:11211
+auth_type = password
+project_domain_name = default
+user_domain_name = default
+project_name = service
+username = neutron
+password = password

 #
@@ -1285,4 +1300,12 @@

 [nova]
+auth_url = http://192.168.3.200:5000
+auth_type = password
+project_domain_name = default
+user_domain_name = default
+region_name = RegionOne
+project_name = service
+username = nova
+password = password

 #
@@ -1396,4 +1419,5 @@

 [oslo_concurrency]
+lock_path = /var/lib/neutron/tmp


※2022.10.15 neutron-server.logを見るとlinuxbridgeがExperimentalとのことでデフォルトだと有効化出来なかった.
experimentalで「linuxbridge = true」を追加して解決.
----
$ sudo less /var/log/neutron/neutron-server.log
2022-10-15 22:18:41.898 41199 ERROR neutron.common.experimental [-] Feature 'linuxbridge' is experimental and has to be explicitly enabled in 'cfg.CONF.experimental'
----
----
→  adapt ci to neutron making linux bridge experimental
https://bugs.launchpad.net/nova/+bug/1980948
```

#### Configure the Modular Layer 2 (ML2) plug-in
```
$ sudo diff --unified=2 ~/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini
----
--- /home/ogalush/neutron/plugins/ml2/ml2_conf.ini      2022-10-07 11:45:06.000000000 +0900
+++ /etc/neutron/plugins/ml2/ml2_conf.ini       2022-10-15 22:02:24.473880008 +0900
@@ -152,4 +152,8 @@
 
 [ml2]
+type_drivers = flat,vlan,vxlan
+tenant_network_types = vxlan
+mechanism_drivers = linuxbridge,l2population
+extension_drivers = port_security
 
 #
@@ -201,4 +205,5 @@
 
 [ml2_type_flat]
+flat_networks = provider
 
 #
@@ -257,4 +262,5 @@
 
 [ml2_type_vxlan]
+vni_ranges = 1:1000
 
 #
@@ -292,4 +298,5 @@
 
 [securitygroup]
+enable_ipset = true
----
```

#### Configure the Linux bridge agent
```
$ sudo vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
----
[linux_bridge]
+ physical_interface_mappings = provider:enp3s0
...
[vxlan]
+ enable_vxlan = true
+ local_ip = 192.168.3.200
+ l2_population = true
...
[securitygroup]
+ enable_security_group = true
+ firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
----

$ sudo vim /etc/sysctl.conf
$ tail -n 2 /etc/sysctl.conf
----
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
----

$ sudo sysctl -p
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```


#### Configure the layer-3 agent
```
$ sudo vim /etc/neutron/l3_agent.ini
----
[DEFAULT]
+ interface_driver = linuxbridge
----
```
#### Configure the DHCP agent
```
$ sudo vim /etc/neutron/dhcp_agent.ini
----
[DEFAULT]
+ interface_driver = linuxbridge
+ dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
+ enable_isolated_metadata = true
----
```

### Configure the metadata agent
https://docs.openstack.org/neutron/zed/install/controller-install-ubuntu.html
```
$ sudo vim /etc/neutron/metadata_agent.ini
----
[DEFAULT]
+ nova_metadata_host = 192.168.3.200
+ metadata_proxy_shared_secret = password
----
```

### Configure the Compute service to use the Networking service
```
$ sudo vim /etc/nova/nova.conf
----
...
[neutron]
+ auth_url = http://192.168.3.200:5000
+ auth_type = password
+ project_domain_name = default
+ user_domain_name = default
+ region_name = RegionOne
+ project_name = service
+ username = neutron
+ password = password
+ service_metadata_proxy = true
+ metadata_proxy_shared_secret = password
...
----
```

### Finalize installation
```
$ sudo -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
$ sudo systemctl restart nova-api neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
$ sudo systemctl status nova-api neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent |grep Active
 Active: active (running) since Sat 2022-10-15 22:10:01 JST; 6s ago
 Active: active (running) since Sat 2022-10-15 22:10:08 JST; 4ms ago
 Active: active (running) since Sat 2022-10-15 22:10:02 JST; 6s ago
 Active: active (running) since Sat 2022-10-15 22:10:02 JST; 5s ago
 Active: active (running) since Sat 2022-10-15 22:10:02 JST; 6s ago
 Active: active (running) since Sat 2022-10-15 22:10:01 JST; 6s ago
```

## Verify operation
https://docs.openstack.org/neutron/zed/install/verify.html
```
$ source ~/admin-openrc
ogalush@ryunosuke:~$ openstack extension list --network
+----------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------+-------------------------
---------------------------------------------------------------------------------------------------------------------------------+
| Name                                                                                                                                                           | Alias                                 | Description             
                                                                                                                                 |
+----------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------+-------------------------
---------------------------------------------------------------------------------------------------------------------------------+
| Address group                                                                                                                                                  | address-group                         | Support address group   
                                                                                                                                 |
| Address scope                                                                                                                                                  | address-scope                         | Address scopes extension
...(略)...
```

## Networking Option 2: Self-service networks
```
$ openstack network agent list
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host      | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| 2cb59b97-4db9-494e-abe0-7bf3e639fa36 | Metadata agent     | ryunosuke | None              | :-)   | UP    | neutron-metadata-agent    |
| 6dc989d6-0c4a-4dc5-ac13-ac9c2d1db53b | Linux bridge agent | ryunosuke | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 77c349b1-d9ab-46aa-9c8b-2d3a84dedd41 | DHCP agent         | ryunosuke | nova              | :-)   | UP    | neutron-dhcp-agent        |
| d5789a5d-96d2-455a-843f-3b29fe3938a4 | L3 agent           | ryunosuke | nova              | :-)   | UP    | neutron-l3-agent          |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
```

# Horizon
https://docs.openstack.org/horizon/zed/install/
## Install and configure for Ubuntu
https://docs.openstack.org/horizon/zed/install/install-ubuntu.html
### Install and configure components
```
$ sudo apt -y install openstack-dashboard
$ sudo cp -rafv /etc/openstack-dashboard ~
$ sudo vim /etc/openstack-dashboard/local_settings.py
$ sudo diff --unified=0 ~/openstack-dashboard/local_settings.py /etc/openstack-dashboard/local_settings.py
----
--- /home/ogalush/openstack-dashboard/local_settings.py 2022-10-07 11:12:02.000000000 +0900
+++ /etc/openstack-dashboard/local_settings.py  2022-10-15 23:46:52.593685906 +0900
@@ -112,0 +113,7 @@
+SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
+CACHES = {
+    'default': {
+         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
+         'LOCATION': '192.168.3.200:11211',
+    }
+}
@@ -126,2 +133,11 @@
-OPENSTACK_HOST = "127.0.0.1"
-OPENSTACK_KEYSTONE_URL = "http://%s/identity/v3" % OPENSTACK_HOST
+OPENSTACK_HOST = "192.168.3.200"
+OPENSTACK_KEYSTONE_URL = "http://%s:5000/identity/v3" % OPENSTACK_HOST
+OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = False
+OPENSTACK_API_VERSIONS = {
+    "identity": 3,
+    "image": 2,
+    "volume": 3,
+}
+OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "default"
+OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
+
@@ -131 +147 @@
-TIME_ZONE = "UTC"
+TIME_ZONE = "Asia/Tokyo"
----


$ sudo cp -rafv /etc/apache2 ~
$ sudo grep 'WSGIApplicationGroup %{GLOBAL}' /etc/apache2/conf-available/openstack-dashboard.conf
WSGIApplicationGroup %{GLOBAL}
→ 元々設定が入っているので追加せず.

$ sudo systemctl reload apache2.service
$ sudo systemctl status apache2.service |grep Active
 Active: active (running) since Sat 2022-10-15 23:27:59 JST; 16min ago
```

## Verify operation for Ubuntu
https://docs.openstack.org/horizon/zed/install/verify-ubuntu.html

http://192.168.3.200/horizon/  
ID: admin, myuser


# Launch an instance
https://docs.openstack.org/install-guide/launch-instance.html
## Provider network
https://docs.openstack.org/install-guide/launch-instance-networks-provider.html
```
$ source ~/admin-openrc
$ openstack network create --share --external --provider-physical-network provider --provider-network-type flat provider
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2022-10-15T14:53:15Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 5b1f0b2e-49eb-480a-86cb-8f05e6e8d4ff |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | provider                             |
| port_security_enabled     | True                                 |
| project_id                | 32b4bc8011454da3837a54d15e16d31e     |
| provider:network_type     | flat                                 |
| provider:physical_network | provider                             |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 1                                    |
| router:external           | External                             |
| segments                  | None                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2022-10-15T14:53:15Z                 |
+---------------------------+--------------------------------------+


$ openstack subnet create --network provider --allocation-pool start=192.168.3.130,end=192.168.3.140 --dns-nameserver 192.168.3.220 --gateway 192.168.3.254 --subnet-range 192.168.3.0/24 provider
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 192.168.3.130-192.168.3.140          |
| cidr                 | 192.168.3.0/24                       |
| created_at           | 2022-10-15T14:53:54Z                 |
| description          |                                      |
| dns_nameservers      | 192.168.3.220                        |
| dns_publish_fixed_ip | None                                 |
| enable_dhcp          | True                                 |
| gateway_ip           | 192.168.3.254                        |
| host_routes          |                                      |
| id                   | e2b1b5e4-4f12-4dc9-9d01-7e693769537b |
| ip_version           | 4                                    |
| ipv6_address_mode    | None                                 |
| ipv6_ra_mode         | None                                 |
| name                 | provider                             |
| network_id           | 5b1f0b2e-49eb-480a-86cb-8f05e6e8d4ff |
| project_id           | 32b4bc8011454da3837a54d15e16d31e     |
| revision_number      | 0                                    |
| segment_id           | None                                 |
| service_types        |                                      |
| subnetpool_id        | None                                 |
| tags                 |                                      |
| updated_at           | 2022-10-15T14:53:54Z                 |
+----------------------+--------------------------------------+
```

## Self-service network
```
$ source ~/demo-openrc
$ openstack network create selfservice
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2022-10-15T14:54:33Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | e8020321-d29c-4b87-97cb-66cddd15c77e |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1450                                 |
| name                      | selfservice                          |
| port_security_enabled     | True                                 |
| project_id                | ce6550a158c64ea6b54883cf74064f10     |
| provider:network_type     | None                                 |
| provider:physical_network | None                                 |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 1                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2022-10-15T14:54:33Z                 |
+---------------------------+--------------------------------------+


$ openstack subnet create --network selfservice --dns-nameserver 192.168.3.220 --gateway 10.0.0.1 --subnet-range 10.0.0.0/24 selfservice
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 10.0.0.2-10.0.0.254                  |
| cidr                 | 10.0.0.0/24                          |
| created_at           | 2022-10-15T14:54:55Z                 |
| description          |                                      |
| dns_nameservers      | 192.168.3.220                        |
| dns_publish_fixed_ip | None                                 |
| enable_dhcp          | True                                 |
| gateway_ip           | 10.0.0.1                             |
| host_routes          |                                      |
| id                   | ea4f2ebf-9853-4ec2-83d7-ce58398f51f1 |
| ip_version           | 4                                    |
| ipv6_address_mode    | None                                 |
| ipv6_ra_mode         | None                                 |
| name                 | selfservice                          |
| network_id           | e8020321-d29c-4b87-97cb-66cddd15c77e |
| project_id           | ce6550a158c64ea6b54883cf74064f10     |
| revision_number      | 0                                    |
| segment_id           | None                                 |
| service_types        |                                      |
| subnetpool_id        | None                                 |
| tags                 |                                      |
| updated_at           | 2022-10-15T14:54:55Z                 |
+----------------------+--------------------------------------+


$ openstack router create router
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | UP                                   |
| availability_zone_hints |                                      |
| availability_zones      |                                      |
| created_at              | 2022-10-15T14:55:18Z                 |
| description             |                                      |
| enable_ndp_proxy        | None                                 |
| external_gateway_info   | null                                 |
| flavor_id               | None                                 |
| id                      | f681a77f-7f61-4246-ad18-9ac870fc3a67 |
| name                    | router                               |
| project_id              | ce6550a158c64ea6b54883cf74064f10     |
| revision_number         | 1                                    |
| routes                  |                                      |
| status                  | ACTIVE                               |
| tags                    |                                      |
| tenant_id               | ce6550a158c64ea6b54883cf74064f10     |
| updated_at              | 2022-10-15T14:55:18Z                 |
+-------------------------+--------------------------------------+


$ openstack router add subnet router selfservice
$ openstack router set router --external-gateway provider
```

## Verify operation
```
$ source ~/admin-openrc
$ ip netns
qrouter-f681a77f-7f61-4246-ad18-9ac870fc3a67 (id: 2)
qdhcp-e8020321-d29c-4b87-97cb-66cddd15c77e (id: 1)
qdhcp-5b1f0b2e-49eb-480a-86cb-8f05e6e8d4ff (id: 0)

$ openstack port list --router router
+--------------------------------------+------+-------------------+------------------------------------------------------------------------------+--------+
| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                           | Status |
+--------------------------------------+------+-------------------+------------------------------------------------------------------------------+--------+
| 18fe0514-db65-4900-94f2-5697af82baad |      | fa:16:3e:a6:b3:0e | ip_address='10.0.0.1', subnet_id='ea4f2ebf-9853-4ec2-83d7-ce58398f51f1'      | ACTIVE |
| 6a65676d-53ee-498b-b122-57fc71b1f662 |      | fa:16:3e:97:88:37 | ip_address='192.168.3.136', subnet_id='e2b1b5e4-4f12-4dc9-9d01-7e693769537b' | ACTIVE |
+--------------------------------------+------+-------------------+------------------------------------------------------------------------------+--------+

$ ping -c 3 192.168.3.136
PING 192.168.3.136 (192.168.3.136) 56(84) bytes of data.
64 bytes from 192.168.3.136: icmp_seq=1 ttl=64 time=0.088 ms
64 bytes from 192.168.3.136: icmp_seq=2 ttl=64 time=0.053 ms
64 bytes from 192.168.3.136: icmp_seq=3 ttl=64 time=0.065 ms

--- 192.168.3.136 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2049ms
rtt min/avg/max/mdev = 0.053/0.068/0.088/0.014 ms
→ RouterのIPアドレスへping出来たためOK.
```

## Create m1.small flavor
```
$ source ~/admin-openrc 
$ openstack flavor create --id 0 --vcpus 1 --ram 2048 --disk 20 m1.small
+----------------------------+----------+
| Field                      | Value    |
+----------------------------+----------+
| OS-FLV-DISABLED:disabled   | False    |
| OS-FLV-EXT-DATA:ephemeral  | 0        |
| description                | None     |
| disk                       | 20       |
| id                         | 0        |
| name                       | m1.small |
| os-flavor-access:is_public | True     |
| properties                 |          |
| ram                        | 2048     |
| rxtx_factor                | 1.0      |
| swap                       |          |
| vcpus                      | 1        |
+----------------------------+----------+
```

## Generate a key pair
```
$ source ~/demo-openrc
$ openstack keypair create --public-key ~/.ssh/authorized_keys ogakey
+-------------+-------------------------------------------------+
| Field       | Value                                           |
+-------------+-------------------------------------------------+
| created_at  | None                                            |
| fingerprint | 95:93:ad:1a:4a:1c:41:7b:26:b7:c7:fc:3b:0a:91:df |
| id          | ogakey                                          |
| is_deleted  | None                                            |
| name        | ogakey                                          |
| type        | ssh                                             |
| user_id     | 27745669d8b54d46958cce0a913edb2e                |
+-------------+-------------------------------------------------+

$ openstack keypair list
+--------+-------------------------------------------------+------+
| Name   | Fingerprint                                     | Type |
+--------+-------------------------------------------------+------+
| ogakey | 95:93:ad:1a:4a:1c:41:7b:26:b7:c7:fc:3b:0a:91:df | ssh  |
+--------+-------------------------------------------------+------+
```

## Add security group rules
```
$ source ~/demo-openrc
$ openstack security group rule create --proto icmp default
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| created_at              | 2022-10-15T15:02:38Z                 |
| description             |                                      |
| direction               | ingress                              |
| ether_type              | IPv4                                 |
| id                      | b9a36df4-0362-46c0-bbbf-b7f52df8965b |
| name                    | None                                 |
| port_range_max          | None                                 |
| port_range_min          | None                                 |
| project_id              | ce6550a158c64ea6b54883cf74064f10     |
| protocol                | icmp                                 |
| remote_address_group_id | None                                 |
| remote_group_id         | None                                 |
| remote_ip_prefix        | 0.0.0.0/0                            |
| revision_number         | 0                                    |
| security_group_id       | 24ef7fc3-fdd3-40ed-8992-da3c3d8595ef |
| tags                    | []                                   |
| updated_at              | 2022-10-15T15:02:38Z                 |
+-------------------------+--------------------------------------+

・22/tcpをデフォルトで開けておく.
$ openstack security group rule create --proto tcp --dst-port 22 default
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| created_at              | 2022-10-15T15:02:58Z                 |
| description             |                                      |
| direction               | ingress                              |
| ether_type              | IPv4                                 |
| id                      | b398424d-93f8-4163-8045-e295d5fb79eb |
| name                    | None                                 |
| port_range_max          | 22                                   |
| port_range_min          | 22                                   |
| project_id              | ce6550a158c64ea6b54883cf74064f10     |
| protocol                | tcp                                  |
| remote_address_group_id | None                                 |
| remote_group_id         | None                                 |
| remote_ip_prefix        | 0.0.0.0/0                            |
| revision_number         | 0                                    |
| security_group_id       | 24ef7fc3-fdd3-40ed-8992-da3c3d8595ef |
| tags                    | []                                   |
| updated_at              | 2022-10-15T15:02:58Z                 |
+-------------------------+--------------------------------------+

・80/tcpをデフォルトで開けておく.
$ openstack security group rule create --proto tcp --dst-port 80 default
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| created_at              | 2022-10-15T15:55:56Z                 |
| description             |                                      |
| direction               | ingress                              |
| ether_type              | IPv4                                 |
| id                      | 31d57aba-d21c-492a-9cf9-236ebdf9dcc0 |
| name                    | None                                 |
| port_range_max          | 80                                   |
| port_range_min          | 80                                   |
| project_id              | ce6550a158c64ea6b54883cf74064f10     |
| protocol                | tcp                                  |
| remote_address_group_id | None                                 |
| remote_group_id         | None                                 |
| remote_ip_prefix        | 0.0.0.0/0                            |
| revision_number         | 0                                    |
| security_group_id       | 24ef7fc3-fdd3-40ed-8992-da3c3d8595ef |
| tags                    | []                                   |
| updated_at              | 2022-10-15T15:55:56Z                 |
+-------------------------+--------------------------------------+

・443/tcpをデフォルトで開けておく.
$ openstack security group rule create --proto tcp --dst-port 443 default
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| created_at              | 2022-10-15T15:56:01Z                 |
| description             |                                      |
| direction               | ingress                              |
| ether_type              | IPv4                                 |
| id                      | ab1446a1-5eed-4f27-a846-6b59694eb0e1 |
| name                    | None                                 |
| port_range_max          | 443                                  |
| port_range_min          | 443                                  |
| project_id              | ce6550a158c64ea6b54883cf74064f10     |
| protocol                | tcp                                  |
| remote_address_group_id | None                                 |
| remote_group_id         | None                                 |
| remote_ip_prefix        | 0.0.0.0/0                            |
| revision_number         | 0                                    |
| security_group_id       | 24ef7fc3-fdd3-40ed-8992-da3c3d8595ef |
| tags                    | []                                   |
| updated_at              | 2022-10-15T15:56:01Z                 |
+-------------------------+--------------------------------------+
```

## 仮想インスタンス起動
OpenStack Dashboardからインスタンスを起動できればOK.

## ubuntu22.04取得
### CloudImageの取得
https://cloud-images.ubuntu.com/releases/22.04/release/
```
$ wget https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-amd64-disk-kvm.img
$ sudo mv -v ~/ubuntu-22.04-server-cloudimg-amd64-disk-kvm.img /usr/local/src
```
## OSイメージのインポート
```
$ source ~/admin-openrc
$ glance image-create --name "ubuntu22.04" --file /usr/local/src/ubuntu-22.04-server-cloudimg-amd64-disk-kvm.img --disk-format qcow2 --container-format bare --visibility=public
+------------------+----------------------------------------------------------------------------------+
| Property         | Value                                                                            |
+------------------+----------------------------------------------------------------------------------+
| checksum         | bc5b2a699b0986680d0f4f30344f6b22                                                 |
| container_format | bare                                                                             |
| created_at       | 2022-10-15T15:40:42Z                                                             |
| disk_format      | qcow2                                                                            |
| id               | 667bd279-6a5c-4eb9-9840-b204fa7971c7                                             |
| min_disk         | 0                                                                                |
| min_ram          | 0                                                                                |
| name             | ubuntu22.04                                                                      |
| os_hash_algo     | sha512                                                                           |
| os_hash_value    | 0243f97d0a3bcf5260d31f173b166d024f6157ae9337ec339fda0dc2cca380c81194cbc7eeccd84b |
|                  | bc0e5f8e3ceba0c744f8d98f8b2632854c71904893c199b2                                 |
| os_hidden        | False                                                                            |
| owner            | 32b4bc8011454da3837a54d15e16d31e                                                 |
| protected        | False                                                                            |
| size             | 631701504                                                                        |
| status           | active                                                                           |
| tags             | []                                                                               |
| updated_at       | 2022-10-15T15:40:45Z                                                             |
| virtual_size     | 2361393152                                                                       |
| visibility       | public                                                                           |
+------------------+----------------------------------------------------------------------------------+

$ glance image-list
+--------------------------------------+-------------+
| ID                                   | Name        |
+--------------------------------------+-------------+
| 65933176-c232-460a-8c46-062ee5266a35 | cirros      |
| 5befb15d-7e69-416c-bc7a-1004742f487f | ubuntu22.04 |
+--------------------------------------+-------------+
```
