# Install OpenStack Caracal on Ubuntu 24.04
OpenStack Document: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
Configs: TBD
TargetHost: 192.168.3.200
```
$ uname -na
Linux ryunosuke 6.8.0-31-generic #31-Ubuntu SMP PREEMPT_DYNAMIC Sat Apr 20 00:40:06 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```

# Install OS Settings.
個人アカウント、ntpdなどの基本パッケージをインストールする.
## Install BasePackages.
```
@admin
$ ansible-playbook -i "192.168.3.200," initialize_instance.yml --user=ubuntu -k -bK --list-hosts
$ ansible-playbook -i "192.168.3.200," initialize_instance.yml --user=ubuntu -k -bK
```
Ref: [initialize_instance.yml](https://github.com/ogalush/Anything/blob/master/initialize_instance.yml)  

## Disable systemd-resolved
/etc/resolve.confの問い合わせ先が `127.0.0.53` のため無効化する.  
※ [initialize_instance.yml](https://github.com/ogalush/Anything/blob/master/initialize_instance.yml#L185-L214)に含めた.


# Environment
## Host networking
## Controller node
https://docs.openstack.org/install-guide/environment-networking-controller.html  
127.0.1.1 がホスト名に紐づいているので、実際のIPアドレスへ置換する.
```
$ sudo cp -pv /etc/hosts /tmp/hosts
[sudo] password for ogalush: 
'/etc/hosts' -> '/tmp/hosts'
$ sudo vim /etc/hosts
$ diff --unified=0 /tmp/hosts /etc/hosts
--- /tmp/hosts  2024-04-27 23:23:53.199321083 +0900
+++ /etc/hosts  2024-04-29 17:16:20.353868090 +0900
@@ -2 +2 @@
-127.0.1.1 ryunosuke
+192.168.3.200 ryunosuke
```

## Network Time Protocol (NTP)
https://docs.openstack.org/install-guide/environment-ntp.html  
時刻同期できているのでOK.
```
$ ntpq -p |grep '*'
*ntp-a2.nict.go. .NICT.           1 u    5   64  377   4.3231   0.3415   0.5235
```


# OpenStack packages for Ubuntu
https://docs.openstack.org/install-guide/environment-packages-ubuntu.html  
## Archive Enablement
 導入するOpenStackバージョンに合わせて設定する.
```
$ sudo add-apt-repository cloud-archive:caracal
cloud-archive for Caracal only supported on Jammy
```
→ Caracal = `Jammy = Ubuntu 22.04` のみの対応らしい.  

[CloudArchive](https://wiki.ubuntu.com/OpenStack/CloudArchive) を見ると、Ubuntu 24.04のデフォルトが Caracal となっているようなので省略.
```
Ubuntu 22.04 LTS

On 22.04, OpenStack Zed is supported for 18 months, OpenStack Antelope for 36 months,
 and OpenStack Bobcat (non-SLURP) is supported for 9 months.
 When 24.04's default OpenStack version (Caracal) is released it will be added to the UCA with support for 36 months
 (i.e. until the end of the Ubuntu 22.04 LTS lifecycle). 
```

## Sample, Client Installation
インストール
```
$ sudo apt -y install python3-openstackclient nova-compute
$ sudo apt -y update
$ sudo apt -y upgrade
$ sudo apt -y dist-upgrade
$ sudo apt -y autoremove
$ sudo reboot
```

# SQL database for Ubuntu
https://docs.openstack.org/install-guide/environment-sql-database-ubuntu.html  
## Install and configure components
```
$ sudo apt -y install mariadb-server python3-pymysql
$ sudo cp -rafv /etc/mysql ~
$ sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf
$ diff --unified=0 ~/mysql/mariadb.conf.d/50-server.cnf /etc/mysql/mariadb.conf.d/50-server.cnf
--- /home/ogalush/mysql/mariadb.conf.d/50-server.cnf    2024-02-07 11:51:12.000000000 +0900
+++ /etc/mysql/mariadb.conf.d/50-server.cnf     2024-04-29 17:37:10.110107669 +0900
@@ -27 +27 @@
-bind-address            = 127.0.0.1
+bind-address            = 192.168.3.200
@@ -95,2 +95,2 @@
-character-set-server  = utf8mb4
-collation-server      = utf8mb4_general_ci
+character-set-server  = utf8
+collation-server      = utf8_general_ci
```

## Finalize installation
```
$ sudo service mysql restart
$ sudo service mysql status |grep Active
  Active: active (running) since Mon 2024-04-29 17:37:31 JST; 6s ago
$ sudo mysql_secure_installation
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
--- /home/ogalush/memcached.conf        2024-04-29 17:40:36.282084207 +0900
+++ /etc/memcached.conf 2024-04-29 17:40:51.007973505 +0900
@@ -35 +35 @@
--l 127.0.0.1
+-l 192.168.3.200
$ sudo systemctl is-active memcached
active
$ sudo systemctl is-enabled memcached
enabled
```

## Finalize installation
```
$ sudo service memcached restart
$ sudo service memcached status |grep Active
  Active: active (running) since Mon 2024-04-29 17:41:15 JST; 3s ago
```

# Etcd for Ubuntu
## Install and configure components
→ 「etcd-server, etcd-client」とそれぞれインストールする.  
Ubuntu 22.04までは「etcd」でインストール出来たが、Ubuntu 24.04には無いため.
```
$ sudo apt -y install etcd-server etcd-client
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
$ sudo systemctl is-enabled etcd
$ sudo systemctl restart etcd
$ sudo systemctl is-active etcd
active
$ sudo systemctl status etcd |grep Active
  Active: active (running) since Mon 2024-04-29 17:46:19 JST; 10s ago
```


# Install OpenStack services
# Keystone Installation Tutorial for Ubuntu
https://docs.openstack.org/keystone/2023.2/install/index-ubuntu.html
## Install and configure
https://docs.openstack.org/keystone/2023.2/install/keystone-install-ubuntu.html
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
--- /home/ogalush/keystone/keystone.conf        2024-04-05 22:28:04.000000000 +0900
+++ /etc/keystone/keystone.conf 2024-04-29 17:52:10.073399129 +0900
@@ -696 +696 @@
-connection = sqlite:////var/lib/keystone/keystone.db
+connection = mysql+pymysql://keystone:password@192.168.3.200/keystone
@@ -2576 +2576 @@
-
+provider = fernet
----

$ sudo -s /bin/sh -c "keystone-manage db_sync" keystone
$ sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage bootstrap --bootstrap-password password --bootstrap-admin-url http://192.168.3.200:5000/v3/ --bootstrap-internal-url http://192.168.3.200:5000/v3/ --bootstrap-public-url http://192.168.3.200:5000/v3/ --bootstrap-region-id RegionOne

$ sudo cp -rafv /etc/apache2 ~
$ sudo vim /etc/apache2/apache2.conf
----
+ # For OpenStack KeyStone
+ ServerName 192.168.3.200
----
```

## Finalize the installation
```
$ sudo service apache2 restart
$ sudo service apache2 status |grep Active
  Active: active (running) since Mon 2024-04-29 17:55:13 JST; 5s ago
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
https://docs.openstack.org/keystone/2023.2/install/keystone-users-ubuntu.html
```
$ openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | f0185628854c4ff8844a870c7a2130a0 |
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
| id          | b70e811465584312a80118b0ff9816d9 |
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
| id          | fb33f516c20d462aaa0b7ceba3297be3 |
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
| id                  | 223997b993684a5f9570b86ee460d044 |
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
| id          | 421cd8c1097a4e1ba0cadda8414fffd2 |
| name        | myrole                           |
| options     | {}                               |
+-------------+----------------------------------+


$ openstack role add --project myproject --user myuser myrole
$
```


## Verify operation
https://docs.openstack.org/keystone/2023.2/install/keystone-verify-ubuntu.html
```
$ unset OS_AUTH_URL OS_PASSWORD
$ openstack --os-auth-url http://192.168.3.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
Password: 
+------------+------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                              |
+------------+------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2024-04-29T10:02:23+0000                                                                                                           |
| id         | gAAAAABmL2Ifai0o4uSs9QgWHTx97pgUgAkBMY-CE-CDm78jiHiy-OCPNKQrFFi01qAG0dC5-51J8c_CgisyJT-                                            |
|            | IYhSGUU4O7syZ9wFlIqn_JV5_ODRdt8lYy_miLWDgvz2GYAqBOoxSKus354F5pKG0PBE3ckDEtBSxwNoTLJrP-1xLkS18N6U                                   |
| project_id | c6bca75875974f8697ddf9cd81d9222d                                                                                                   |
| user_id    | 09fae96027d34f66972c787d6ae28d39                                                                                                   |
+------------+------------------------------------------------------------------------------------------------------------------------------------+

$ openstack --os-auth-url http://192.168.3.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name myproject --os-username myuser token issue
Password: 
+------------+------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                              |
+------------+------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2024-04-29T10:02:52+0000                                                                                                           |
| id         | gAAAAABmL2I8wQSM3H2UX5z0y5ZwUq3Qp5rZg45bSVTESvBOarUfpjhS1a4pdqXAvcVpKYbxSgo5pd3BjDV8FMdCHxjFjSh8Jmid6h7ICEIBXyrsSCiSzHMW28S-       |
|            | KJi7QSfTvKd5D7ip0Ur5JCb2CEz3XZwdVY38PrkR_0DkagT0r4-_t-yBh68                                                                        |
| project_id | fb33f516c20d462aaa0b7ceba3297be3                                                                                                   |
| user_id    | 223997b993684a5f9570b86ee460d044                                                                                                   |
+------------+------------------------------------------------------------------------------------------------------------------------------------+
```

## Create OpenStack client environment scripts
https://docs.openstack.org/keystone/2023.2/install/keystone-openrc-ubuntu.html
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
+------------+------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                              |
+------------+------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2024-04-29T10:04:05+0000                                                                                                           |
| id         | gAAAAABmL2KFa0vV4dON959T5JXBLU8AVNmpwOcJDQIMUwTz0IaN4Tl5EyiUrvD5SAdQSNcAPXdAj7NUBBP9UgPojMrSX-                                     |
|            | tNjlEPQ2TJB5B0_dGSalx4L36tiGRphpQF8-4NRJEwCxJdc1UYCa8jz16gJj_MLlwFiDxfsgVMKXN2IZEkJFwRuik                                          |
| project_id | c6bca75875974f8697ddf9cd81d9222d                                                                                                   |
| user_id    | 09fae96027d34f66972c787d6ae28d39                                                                                                   |
+------------+------------------------------------------------------------------------------------------------------------------------------------+

$ source ~/demo-openrc
$ openstack token issue
+------------+------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                              |
+------------+------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2024-04-29T10:04:26+0000                                                                                                           |
| id         | gAAAAABmL2KaWBWlb5H4CDRdRlYPVi4DKcFGOGXBEfMkH5RfuLC9Oq6A299c8PBu5xs03lWEWcOsCYsP9zfJm9Ve6aThNd2i4bAwkwN41lfPW8Y4I_NAIclkfXZ0UbSSHC |
|            | 9k6fRcLdkWXY5izQBS5BqgztchE_HmMsmKMfMwmDVaCKbbIcZu-rc                                                                              |
| project_id | fb33f516c20d462aaa0b7ceba3297be3                                                                                                   |
| user_id    | 223997b993684a5f9570b86ee460d044                                                                                                   |
+------------+------------------------------------------------------------------------------------------------------------------------------------+
```

# Glance Installation
https://docs.openstack.org/glance/2023.2/install/
## Install and configure (Ubuntu)
https://docs.openstack.org/glance/2023.2/install/install-ubuntu.html
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
| id                  | ed9a072839cc4a6d99e7bd203180534a |
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
| id          | 3e63382c399b437f9ca75a5f4aef78b1 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+


$ openstack endpoint create --region RegionOne image public http://192.168.3.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | f583bed2a9634b40ac70454c2e10b894 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 3e63382c399b437f9ca75a5f4aef78b1 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.3.200:9292        |
+--------------+----------------------------------+


$ openstack endpoint create --region RegionOne image internal http://192.168.3.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | fc2624a1d339430fade272a1087e9947 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 3e63382c399b437f9ca75a5f4aef78b1 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.3.200:9292        |
+--------------+----------------------------------+


$ openstack endpoint create --region RegionOne image admin http://192.168.3.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 1cfa0e9757c34591aee0df5473fa0714 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 3e63382c399b437f9ca75a5f4aef78b1 |
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
...
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
[DEFAULT]
+ enabled_backends=fs:file
...
[glance_store]
+ default_backend = fs
...
+ [fs]
+ filesystem_store_datadir = /var/lib/glance/images/

[oslo_limit]
+ auth_url = http://192.168.3.200:5000
+ auth_type = password
+ user_domain_id = default
+ username = glance
+ system_scope = all
+ password = password
+ endpoint_id = f583bed2a9634b40ac70454c2e10b894
+ region_name = RegionOne
→ 「endpoint_id」は、glance image publicのIDを設定する.
...
→ 「use_keystone_limits」はクォータ制限のため省略.
-----

$ openstack role add --user glance --user-domain default --system all reader
$ sudo -s /bin/sh -c "glance-manage db_sync" glance
```
### Finalize installation
```
$ sudo service glance-api restart
$ sudo systemctl is-enabled glance-api
enabled
$ sudo systemctl is-active glance-api
active
$ sudo service glance-api status |grep Active
  Active: active (running) since Mon 2024-04-29 18:21:08 JST; 33s ago
```

## Verify operation
https://docs.openstack.org/glance/2023.2/install/verify.html
```
$ source ~/admin-openrc
$ wget https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img
$ sudo mv -v cirros-0.6.2-x86_64-disk.img /usr/local/src
$ glance image-create --name "cirros" --file /usr/local/src/cirros-0.6.2-x86_64-disk.img --disk-format qcow2 --container-format bare --visibility=public
+------------------+----------------------------------------------------------------------------------+
| Property         | Value                                                                            |
+------------------+----------------------------------------------------------------------------------+
| checksum         | c8fc807773e5354afe61636071771906                                                 |
| container_format | bare                                                                             |
| created_at       | 2024-04-29T09:24:35Z                                                             |
| disk_format      | qcow2                                                                            |
| id               | 047d4142-4a9d-401e-ab0e-ee6ba1a0ea94                                             |
| min_disk         | 0                                                                                |
| min_ram          | 0                                                                                |
| name             | cirros                                                                           |
| os_hash_algo     | sha512                                                                           |
| os_hash_value    | 1103b92ce8ad966e41235a4de260deb791ff571670c0342666c8582fbb9caefe6af07ebb11d34f44 |
|                  | f8414b609b29c1bdf1d72ffa6faa39c88e8721d09847952b                                 |
| os_hidden        | False                                                                            |
| owner            | c6bca75875974f8697ddf9cd81d9222d                                                 |
| protected        | False                                                                            |
| size             | 21430272                                                                         |
| status           | active                                                                           |
| stores           | fs                                                                               |
| tags             | []                                                                               |
| updated_at       | 2024-04-29T09:24:36Z                                                             |
| virtual_size     | 117440512                                                                        |
| visibility       | public                                                                           |
+------------------+----------------------------------------------------------------------------------+


$ glance image-list
+--------------------------------------+--------+
| ID                                   | Name   |
+--------------------------------------+--------+
| 047d4142-4a9d-401e-ab0e-ee6ba1a0ea94 | cirros |
+--------------------------------------+--------+
```


# placement installation
https://docs.openstack.org/placement/2023.2/install/
## Install and configure Placement for Ubuntu
https://docs.openstack.org/placement/2023.2/install/install-ubuntu.html
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
| id                  | cee34de1216e46518f73e6050ea39e2b |
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
| id          | 026a24fe85dc4b45839cc6b74577434a |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+


$ openstack endpoint create --region RegionOne placement public http://192.168.3.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 57ac6ee5e2cf4179bbe487a997a37342 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 026a24fe85dc4b45839cc6b74577434a |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.3.200:8778        |
+--------------+----------------------------------+


$ openstack endpoint create --region RegionOne placement internal http://192.168.3.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6f2b653d98234243b471734d590a7a2b |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 026a24fe85dc4b45839cc6b74577434a |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.3.200:8778        |
+--------------+----------------------------------+


$ openstack endpoint create --region RegionOne placement admin http://192.168.3.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | c6d4d760d00448a5987803ddf8aad66a |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 026a24fe85dc4b45839cc6b74577434a |
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
----
[placement_database]
- connection = sqlite:////var/lib/placement/placement.sqlite
+ connection = mysql+pymysql://placement:password@192.168.3.200/placement
...
[api]
+ auth_strategy = keystone
...
[keystone_authtoken]
+ auth_url = http://192.168.3.200:5000/v3
+ memcached_servers = 192.168.3.200:11211
+ auth_type = password
+ project_domain_name = default
+ user_domain_name = default
+ project_name = service
+ username = placement
+ password = password
----
-----

$ sudo -s /bin/sh -c "placement-manage db sync" placement
$ sudo service apache2 restart
$ sudo service apache2 status |grep Active
  Active: active (running) since Mon 2024-04-29 18:36:27 JST; 4s ago
```

## Verify Installation
https://docs.openstack.org/placement/2023.2/install/verify.html
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
$ sudo apt -y install python3-osc-placement

※ マニュアルに記載のpip3経由だとエラー終了するため、apt経由でインストールした.
----
$ sudo pip3 install osc-placement
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, try apt install
    python3-xyz, where xyz is the package you are trying to
    install.
    
    If you wish to install a non-Debian-packaged Python package,
    create a virtual environment using python3 -m venv path/to/venv.
    Then use path/to/venv/bin/python and path/to/venv/bin/pip. Make
    sure you have python3-full installed.
    
    If you wish to install a non-Debian packaged Python application,
    it may be easiest to use pipx install xyz, which will manage a
    virtual environment for you. Make sure you have pipx installed.
    
    See /usr/share/doc/python3.12/README.venv for more information.

note: If you believe this is a mistake, please contact your Python installation or OS distribution provider.
 You can override this, at the risk of breaking your Python installation or OS, by passing --break-system-packages.
hint: See PEP 668 for the detailed specification.
----


$ openstack --os-placement-api-version 1.2 resource class list --sort-column name
+----------------------------------------+
| name                                   |
+----------------------------------------+
| DISK_GB                                |
| FPGA                                   |
| IPV4_ADDRESS                           |
| MEMORY_MB                              |
| MEM_ENCRYPTION_CONTEXT                 |
| NET_BW_EGR_KILOBIT_PER_SEC             |
| NET_BW_IGR_KILOBIT_PER_SEC             |
| NET_PACKET_RATE_EGR_KILOPACKET_PER_SEC |
| NET_PACKET_RATE_IGR_KILOPACKET_PER_SEC |
| NET_PACKET_RATE_KILOPACKET_PER_SEC     |
| NUMA_CORE                              |
| NUMA_MEMORY_MB                         |
| NUMA_SOCKET                            |
| NUMA_THREAD                            |
| PCI_DEVICE                             |
| PCPU                                   |
| PGPU                                   |
| SRIOV_NET_VF                           |
| VCPU                                   |
| VGPU                                   |
| VGPU_DISPLAY_HEAD                      |
+----------------------------------------+


$ openstack --os-placement-api-version 1.6 trait list --sort-column name
+---------------------------------------+
| name                                  |
+---------------------------------------+
| COMPUTE_ACCELERATORS                  |
| COMPUTE_ADDRESS_SPACE_EMULATED        |
| COMPUTE_ADDRESS_SPACE_PASSTHROUGH     |
...(略)...
| STORAGE_DISK_HDD                      |
| STORAGE_DISK_SSD                      |
+---------------------------------------+
```


# Compute service
https://docs.openstack.org/nova/2023.2/install/
## Install and configure controller node for Ubuntu
https://docs.openstack.org/nova/2023.2/install/controller-install-ubuntu.html
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
| id                  | ded655a2d20d4966b575640d60ad7d12 |
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
| id          | 9054015268b646d2994cba137a990566 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+


$ openstack endpoint create --region RegionOne compute public http://192.168.3.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 274ea540295f48a18db5e310a43dbf6e |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 9054015268b646d2994cba137a990566 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.3.200:8774/v2.1   |
+--------------+----------------------------------+


$ openstack endpoint create --region RegionOne compute internal http://192.168.3.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 9781652833344443a0ab2403dab02b16 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 9054015268b646d2994cba137a990566 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.3.200:8774/v2.1   |
+--------------+----------------------------------+


$ openstack endpoint create --region RegionOne compute admin http://192.168.3.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 181765d78551482380f4dabafc50107a |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 9054015268b646d2994cba137a990566 |
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
----
[api_database]
- connection = sqlite:////var/lib/nova/nova_api.sqlite
+ connection = mysql+pymysql://nova:password@192.168.3.200/nova_api
...
[database]
- connection = sqlite:////var/lib/nova/nova.sqlite
+ connection = mysql+pymysql://nova:password@192.168.3.200/nova
...

[DEFAULT]
...
+ transport_url = rabbit://openstack:password@192.168.3.200:5672/
+ my_ip = 192.168.3.200
...

[api]
auth_strategy = keystone
...

[keystone_authtoken]
+ www_authenticate_uri = http://192.168.3.200:5000/
+ auth_url = http://192.168.3.200:5000/
+ memcached_servers = 192.168.3.200:11211
+ auth_type = password
+ project_domain_name = default
+ user_domain_name = default
+ project_name = service
+ username = nova
+ password = password
...

[service_user]
+ send_service_user_token = true
+ auth_url = https://192.168.3.200/identity
+ auth_strategy = keystone
+ auth_type = password
+ project_domain_name = default
+ project_name = service
+ user_domain_name = default
+ username = nova
+ password = password
...
[vnc]
+ enabled = true
+ server_listen = $my_ip
+ server_proxyclient_address = $my_ip
...
[glance]
+ api_servers = http://192.168.3.200:9292
...
[oslo_concurrency]
+ lock_path = /var/lib/nova/tmp
...
[placement]
+ region_name = RegionOne
+ project_domain_name = default
+ project_name = service
+ auth_type = password
+ user_domain_name = default
+ auth_url = http://192.168.3.200:5000/v3
+ username = placement
+ password = password
...
----

※追加
VM作成時に「ホスト名.novalocal」になるためDHCPドメインを設定する.
$ sudo vim /etc/nova/nova.conf
----
[DEFAULT]
...
dhcp_domain = localdomain
----


$ sudo -s /bin/sh -c "nova-manage api_db sync" nova
$ sudo -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
$ sudo -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
3 RLock(s) were not greened, to fix this error make sure you run eventlet.monkey_patch() before importing any other modules.
--transport-url not provided in the command line, using the value [DEFAULT]/transport_url from the configuration file
--database_connection not provided in the command line, using the value [database]/connection from the configuration file
cf3ceeb5-e40b-4be2-a90c-d7e02c5f49ac

$ sudo -s /bin/sh -c "nova-manage db sync" nova
$ sudo -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
3 RLock(s) were not greened, to fix this error make sure you run eventlet.monkey_patch() before importing any other modules.
+-------+--------------------------------------+---------------------------------------------+----------------------------------------------------+----------+
|  Name |                 UUID                 |                Transport URL                |                Database Connection                 | Disabled |
+-------+--------------------------------------+---------------------------------------------+----------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                    none:/                   | mysql+pymysql://nova:****@192.168.3.200/nova_cell0 |  False   |
| cell1 | cf3ceeb5-e40b-4be2-a90c-d7e02c5f49ac | rabbit://openstack:****@192.168.3.200:5672/ |    mysql+pymysql://nova:****@192.168.3.200/nova    |  False   |
+-------+--------------------------------------+---------------------------------------------+----------------------------------------------------+----------+
```

### Finalize installation
```
$ sudo systemctl restart nova-api nova-scheduler nova-conductor nova-novncproxy
$ sudo systemctl status nova-api nova-scheduler nova-conductor nova-novncproxy |grep Active
  Active: active (running) since Mon 2024-04-29 19:09:23 JST; 4s ago
  Active: active (running) since Mon 2024-04-29 19:09:23 JST; 5s ago
  Active: active (running) since Mon 2024-04-29 19:09:23 JST; 5s ago
  Active: active (running) since Mon 2024-04-29 19:09:23 JST; 4s ago
$ sudo systemctl is-active nova-api nova-scheduler nova-conductor nova-novncproxy
active
active
active
active
$ sudo systemctl is-enabled nova-api nova-scheduler nova-conductor nova-novncproxy
enabled
enabled
enabled
enabled
```

## Install and configure a compute node for Ubuntu
https://docs.openstack.org/nova/2023.2/install/compute-install-ubuntu.html
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
virt_type=kvm ・・・KVMのままでOK.(Intel VT)
----

$ sudo systemctl restart nova-compute
$ sudo systemctl status nova-compute |grep Active
  Active: active (running) since Mon 2024-04-29 19:14:18 JST; 4s ago
$ sudo systemctl is-active nova-compute
active
$ sudo systemctl is-enabled nova-compute
enabled
```

### Add the compute node to the cell database
```
$ source ~/admin-openrc
$ openstack compute service list --service nova-compute
+--------------------------------------+--------------+-----------+------+---------+-------+----------------------------+
| ID                                   | Binary       | Host      | Zone | Status  | State | Updated At                 |
+--------------------------------------+--------------+-----------+------+---------+-------+----------------------------+
| fedc9728-a77f-42dc-97b6-f370ff5e29a2 | nova-compute | ryunosuke | nova | enabled | up    | 2024-04-29T10:15:17.000000 |
+--------------------------------------+--------------+-----------+------+---------+-------+----------------------------+


$ sudo -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
3 RLock(s) were not greened, to fix this error make sure you run eventlet.monkey_patch() before importing any other modules.
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': cf3ceeb5-e40b-4be2-a90c-d7e02c5f49ac
Checking host mapping for compute host 'ryunosuke': a921624a-abec-451f-bdc2-7b5466d33b97
Creating host mapping for compute host 'ryunosuke': a921624a-abec-451f-bdc2-7b5466d33b97
Found 1 unmapped computes in cell: cf3ceeb5-e40b-4be2-a90c-d7e02c5f49ac
```

## Verify operation
https://docs.openstack.org/nova/2023.2/install/verify.html
```
$ source ~/admin-openrc
$ openstack compute service list
+--------------------------------------+----------------+-----------+----------+---------+-------+----------------------------+
| ID                                   | Binary         | Host      | Zone     | Status  | State | Updated At                 |
+--------------------------------------+----------------+-----------+----------+---------+-------+----------------------------+
| 6d64eeac-c86e-47b8-9f2d-8f31387b18b4 | nova-scheduler | ryunosuke | internal | enabled | up    | 2024-04-29T10:17:20.000000 |
| a2b41edf-344a-4ca5-befb-66f4218ff30d | nova-conductor | ryunosuke | internal | enabled | up    | 2024-04-29T10:17:20.000000 |
| fedc9728-a77f-42dc-97b6-f370ff5e29a2 | nova-compute   | ryunosuke | nova     | enabled | up    | 2024-04-29T10:17:20.000000 |
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
| placement | placement | RegionOne                                  |
|           |           |   public: http://192.168.3.200:8778        |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:8778      |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:8778         |
|           |           |                                            |
| glance    | image     | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:9292         |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.3.200:9292        |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:9292      |
|           |           |                                            |
| nova      | compute   | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:8774/v2.1    |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.3.200:8774/v2.1   |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:8774/v2.1 |
|           |           |                                            |
+-----------+-----------+--------------------------------------------+


$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 047d4142-4a9d-401e-ab0e-ee6ba1a0ea94 | cirros | active |
+--------------------------------------+--------+--------+


$ sudo nova-status upgrade check
3 RLock(s) were not greened, to fix this error make sure you run eventlet.monkey_patch() before importing any other modules.
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
| Check: Service User Token Configuration   |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
```


# Neutron
https://docs.openstack.org/neutron/2023.2/install/
## Install and configure for Ubuntu
https://docs.openstack.org/neutron/2023.2/install/install-ubuntu.html
### Install and configure controller node
https://docs.openstack.org/neutron/2023.2/install/controller-install-ubuntu.html
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
| id                  | f3d37658cdc64cdcb7c7a9da1c263271 |
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
| id          | d62dafb6b60e4e048e0b963ba6501b36 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+


$ openstack endpoint create --region RegionOne network public http://192.168.3.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | aa1be566abc04b6b9896de6d7a43edf2 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d62dafb6b60e4e048e0b963ba6501b36 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.3.200:9696        |
+--------------+----------------------------------+


$ openstack endpoint create --region RegionOne network internal http://192.168.3.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0b71365447884139b7d5e1e345c33172 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d62dafb6b60e4e048e0b963ba6501b36 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.3.200:9696        |
+--------------+----------------------------------+

  
$ openstack endpoint create --region RegionOne network admin http://192.168.3.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 17c975cf69894acf8e4f01e7baa47d1c |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d62dafb6b60e4e048e0b963ba6501b36 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.3.200:9696        |
+--------------+----------------------------------+
```

### Configure networking options
Networking Option 2: Self-service networks
https://docs.openstack.org/neutron/2023.2/install/controller-install-option2-ubuntu.html  
  
【注意】  
従来通りLinuxBridgeでインストールを進める。  
（マニュアルではOpenvSwitchになっているが、利便性を考慮して）
#### Install the components
```
$ sudo apt -y install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```
#### Configure the server component
```
$ sudo cp -rafv /etc/neutron ~
$ sudo vim /etc/neutron/neutron.conf
----
[database]
- connection = sqlite:////var/lib/neutron/neutron.sqlite
+ connection = mysql+pymysql://neutron:password@192.168.3.200/neutron
...
[DEFAULT]
...
core_plugin = ml2
+ service_plugins = router
+ transport_url = rabbit://openstack:password@192.168.3.200
+ auth_strategy = keystone
+ notify_nova_on_port_status_changes = true
+ notify_nova_on_port_data_changes = true
...
[keystone_authtoken]
+ www_authenticate_uri = http://192.168.3.200:5000
+ auth_url = http://192.168.3.200:5000
+ memcached_servers = 192.168.3.200:11211
+ auth_type = password
+ project_domain_name = default
+ user_domain_name = default
+ project_name = service
+ username = neutron
+ password = password
...
[nova]
+ auth_url = http://192.168.3.200:5000
+ auth_type = password
+ project_domain_name = default
+ user_domain_name = default
+ region_name = RegionOne
+ project_name = service
+ username = nova
+ password = password
...
[oslo_concurrency]
+ lock_path = /var/lib/neutron/tmp
...
[experimental]
+ linuxbridge = true
...
----
```

#### Configure the Modular Layer 2 (ML2) plug-in
```
$ sudo vim /etc/neutron/plugins/ml2/ml2_conf.ini
----
[ml2]
+type_drivers = flat,vlan,vxlan
+tenant_network_types = vxlan
+mechanism_drivers = linuxbridge,l2population
+extension_drivers = port_security
...
 [ml2_type_flat]
+flat_networks = provider
...
[ml2_type_vxlan]
+ vni_ranges = 1:1000
...
[securitygroup]
+ enable_ipset = true
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

$ sudo cp -pv /etc/sysctl.conf ~
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
https://docs.openstack.org/neutron/2023.2/install/controller-install-ubuntu.html
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
  Active: active (running) since Mon 2024-04-29 19:44:51 JST; 8s ago
  Active: active (running) since Mon 2024-04-29 19:44:51 JST; 8s ago
  Active: active (running) since Mon 2024-04-29 19:44:59 JST; 802ms ago
  Active: active (running) since Mon 2024-04-29 19:44:57 JST; 3s ago
  Active: active (running) since Mon 2024-04-29 19:44:51 JST; 8s ago
  Active: active (running) since Mon 2024-04-29 19:44:53 JST; 6s ago

$ sudo systemctl is-active nova-api neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
active
active
active
active
active
active

$ sudo systemctl is-enabled nova-api neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
enabled
enabled
enabled
enabled
enabled
enabled
```

## Verify operation
https://docs.openstack.org/neutron/2023.2/install/verify.html
```
$ source ~/admin-openrc
$ openstack extension list --network
+-------------------------------------------------+---------------------------------------------+-------------------------------------------------+
| Name                                            | Alias                                       | Description                                     |
+-------------------------------------------------+---------------------------------------------+-------------------------------------------------+
| Address group                                   | address-group                               | Support address group                           |
...(略)...
| Resource timestamps                             | standard-attr-timestamp                     | Adds created_at and updated_at fields to all    |
|                                                 |                                             | Neutron resources that have Neutron standard    |
|                                                 |                                             | attributes.                                     |
+-------------------------------------------------+---------------------------------------------+-------------------------------------------------+
```

## Networking Option 2: Self-service networks
```
$ openstack network agent list
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host      | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| 1fb2560b-e207-470e-841a-636e044b376c | DHCP agent         | ryunosuke | nova              | :-)   | UP    | neutron-dhcp-agent        |
| 4dcb8847-5536-4514-9766-2cedb2653df5 | Metadata agent     | ryunosuke | None              | :-)   | UP    | neutron-metadata-agent    |
| 7a536ba2-ea32-417c-bdd4-519728efd252 | L3 agent           | ryunosuke | nova              | :-)   | UP    | neutron-l3-agent          |
| f6726859-9c96-4848-b162-529e284230b5 | Linux bridge agent | ryunosuke | None              | :-)   | UP    | neutron-linuxbridge-agent |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
```

# Horizon
https://docs.openstack.org/horizon/2023.2/install/
## Install and configure for Ubuntu
https://docs.openstack.org/horizon/2023.2/install/install-ubuntu.html
### Install and configure components
```
$ sudo apt -y install openstack-dashboard
$ sudo cp -rfv /etc/openstack-dashboard ~
$ sudo vim /etc/openstack-dashboard/local_settings.py
----
- OPENSTACK_HOST = "127.0.0.1"
- OPENSTACK_KEYSTONE_URL = "http://%s/identity/v3" % OPENSTACK_HOST
+ OPENSTACK_HOST = "192.168.3.200"
+ OPENSTACK_KEYSTONE_URL = "http://%s:5000/identity/v3" % OPENSTACK_HOST
+ ALLOWED_HOSTS = ['*']
...
+ SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
...
- CACHES = {
-     'default': {
-         'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
-         'LOCATION': '127.0.0.1:11211',
-     },
- }
+ CACHES = {
+     'default': {
+          'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
+          'LOCATION': '192.168.3.200:11211',
+     }
+ }
...
+ ## HorizonSettings From Manual
+ OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = False
+ OPENSTACK_API_VERSIONS = {
+     "identity": 3,
+     "image": 2,
+     "volume": 3,
+ }
...
+ OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "default"
+ OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
...
- TIME_ZONE = "UTC"
+ TIME_ZONE = "Asia/Tokyo"
----


$ sudo cp -rafv /etc/apache2 ~
$ sudo grep 'WSGIApplicationGroup %{GLOBAL}' /etc/apache2/conf-available/openstack-dashboard.conf
WSGIApplicationGroup %{GLOBAL}
→ 元々設定が入っているので追加せず.

$ sudo systemctl restart apache2.service
$ sudo systemctl status apache2.service |grep Active
  Active: active (running) since Mon 2024-04-29 20:07:33 JST; 2s ago
$ sudo systemctl is-active apache2.service
active
$ sudo systemctl is-enabled apache2.service
enabled
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

# Disable virbr0
OpenStackインストール時にlibvirtdが入るため、 `virbr0` のInterfaceが有効化される.  
このインターフェイスは利用しないため無効化する.  
参考: [virbr0を削除する](https://tech.nosuz.jp/blog/2011/09/17-23/)

## Show virbr0
```
$ ip addr show virbr0
4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:83:d9:7e brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
$
ogalush@ryunosuke:~$ virsh net-list
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes

ogalush@ryunosuke:~$ virsh net-dumpxml default
<network>
  <name>default</name>
  <uuid>73c83bdd-9aa3-4c03-8aa9-4ccb74fcc9f2</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:83:d9:7e'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>

ogalush@ryunosuke:~$ 
```

## Delete And Disable virbr0
```
$ virsh net-destroy default
Network default destroyed

$ virsh net-autostart default --disable
Network default unmarked as autostarted
```

## virbr0削除確認
```
$ virsh net-list --all
 Name      State      Autostart   Persistent
----------------------------------------------
 default   inactive   no          yes

$
→ Autostart = No, State = inactiveになっていればOK.
```
