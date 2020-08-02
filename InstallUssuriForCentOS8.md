# Install OpenStack Ussuri on CentOS8
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.3.200  
設定ファイル: [Ussuri-InstallConfigsForCentOS8](https://github.com/ogalush/Ussuri-InstallConfigsForCentOS8)
```
$ uname -n
ryunosuke.localdomain
$ cat /etc/redhat-release 
CentOS Linux release 8.2.2004 (Core) 
[ogalush@ryunosuke ~]$ uname -a
Linux ryunosuke.localdomain 4.18.0-193.6.3.el8_2.x86_64 #1 SMP Wed Jun 10 11:09:32 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
```
# Environment
## OS
CentOS8でインストールを行う.  
CentOS7は、ussuri向けのRPMが無いため.
## Hosts
```
$ uname -n
ryunosuke.localdomain
$ cat /etc/hostname 
ryunosuke.localdomain
$ cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.3.200 ryunosuke ryunosuke.localdomain
$
```
## ntp
https://docs.openstack.org/install-guide/environment-ntp-controller.html
```
$ chronyc sources
210 Number of sources = 4
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* gpg.n1zyy.com                 2   6   377    25  -4076us[-7488us] +/-   89ms
^+ login-vlan194.budapest2.>     2   6   377    24  -7054us[-7054us] +/-  169ms
^+ 138.68.183.179                3   6   377    24  +9476us[+9476us] +/-  171ms
^+ www.bochum.solar              2   6   377    25  +1840us[+1840us] +/-  133ms
→ 同期してる.

$ chronyc tracking
Reference ID    : C0630208 (gpg.n1zyy.com)
Stratum         : 3
Ref time (UTC)  : Sun Jul 26 06:38:51 2020 ・・・最終確認時刻(UTC, JST=UTC+0900)
System time     : 0.002377091 seconds slow of NTP time・・・NTPサーバーと自端末時刻の誤差
Last offset     : -0.003412287 seconds
RMS offset      : 0.002153547 seconds
Frequency       : 25.055 ppm fast
Residual freq   : -3.757 ppm
Skew            : 12.161 ppm
Root delay      : 0.171737716 seconds
Root dispersion : 0.006856591 seconds
Update interval : 64.4 seconds
Leap status     : Normal

読み方.
https://hackers-high.com/linux/easy-chrony-settings/
```

## OpenStack packages for RHEL and CentOS
https://docs.openstack.org/install-guide/environment-packages-rdo.html
```
$ sudo yum -y install centos-release-openstack-ussuri
$ sudo yum config-manager --set-enabled PowerTools
$ sudo yum -y upgrade
$ sudo yum -y install python3-openstackclient
$ sudo yum -y install openstack-selinux
```

## SQL database for RHEL and CentOS
https://docs.openstack.org/install-guide/environment-sql-database-rdo.html
### Install and configure components
```
$ sudo yum -y install mariadb mariadb-server python3-PyMySQL
$ sudo cp -rafv /etc/my.cnf.d /tmp
$ sudo vim /etc/my.cnf.d/mariadb-server.cnf
----
[mysqld]
...
+ bind-address = 192.168.3.200
+ default-storage-engine = innodb
+ innodb_file_per_table = on
+ max_connections = 4096
+ collation-server = utf8_general_ci
+ character-set-server = utf8
----
```
### Finalize installation
```
$ sudo systemctl enable mariadb.service
$ sudo systemctl restart mariadb.service
$ sudo systemctl status mariadb.service
$ sudo mysql_secure_installation
Enter current password for root (enter for none):なし
Set root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

## Message queue for RHEL and CentOS
https://docs.openstack.org/install-guide/environment-messaging-rdo.html
### Install and configure components
```
$ sudo yum -y install rabbitmq-server
$ sudo systemctl enable rabbitmq-server.service
$ sudo systemctl restart rabbitmq-server.service
$ sudo rabbitmqctl add_user openstack password
Adding user "openstack" ...
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
Setting permissions for user "openstack" in vhost "/" ...

OS再起動時にrabbitmq-serverの起動に失敗するので自動再起動設定を入れておく. (IPv6があるとなるらしい)
$ ls -l /etc/systemd/system/multi-user.target.wants/rabbitmq-server.service
lrwxrwxrwx 1 root root 47 Jul 26 15:55 /etc/systemd/system/multi-user.target.wants/rabbitmq-server.service -> /usr/lib/systemd/system/rabbitmq-server.service
$ sudo cp -pv /usr/lib/systemd/system/rabbitmq-server.service ~
'/usr/lib/systemd/system/rabbitmq-server.service' -> '/home/ogalush/rabbitmq-server.service'
$ sudo vim /usr/lib/systemd/system/rabbitmq-server.service
----
+Restart=on-failure
+RestartSec=10
----
```

## Memcached for RHEL and CentOS
https://docs.openstack.org/install-guide/environment-memcached-rdo.html
### Install and configure components
```
$ sudo yum -y install memcached python3-memcached
$ sudo cp -pv /etc/sysconfig/memcached /tmp
$ sudo vim /etc/sysconfig/memcached
----
- OPTIONS="-l 127.0.0.1,::1"
+ OPTIONS="-l 127.0.0.1,::1,192.168.3.200"
----

$ sudo vim /usr/lib/systemd/system/memcached.service
----
[Service]
...
+ Restart=on-failure
+ RestartSec=10
----
→ OS起動時にデーモン起動に失敗するため入れておく.
```

### Finalize installation
```
$ sudo systemctl enable memcached.service
$ sudo systemctl restart memcached.service
$ sudo netstat -lnp |grep 11211
tcp        0      0 192.168.3.200:11211     0.0.0.0:*               LISTEN      9001/memcached      
tcp        0      0 127.0.0.1:11211         0.0.0.0:*               LISTEN      9001/memcached      
tcp6       0      0 ::1:11211               :::*                    LISTEN      9001/memcached    
```

## Etcd for RHEL and CentOS
https://docs.openstack.org/install-guide/environment-etcd-rdo.html
### Install and configure components
```
$ sudo yum -y install etcd
$ sudo cp -rafv /etc/etcd /tmp
$ sudo vim /etc/etcd/etcd.conf
-----
$ diff --unified=0 /tmp/etcd/etcd.conf /etc/etcd/etcd.conf |grep -v '^@@'
--- /tmp/etcd/etcd.conf 2020-01-29 01:51:46.000000000 +0900
+++ /etc/etcd/etcd.conf 2020-07-26 16:05:16.713981639 +0900
-#ETCD_LISTEN_PEER_URLS="http://localhost:2380"
-ETCD_LISTEN_CLIENT_URLS="http://localhost:2379"
+ETCD_LISTEN_PEER_URLS="http://192.168.3.200:2380"
+ETCD_LISTEN_CLIENT_URLS="http://192.168.3.200:2379"
-ETCD_NAME="default"
+ETCD_NAME="ryunosuke"
-#ETCD_INITIAL_ADVERTISE_PEER_URLS="http://localhost:2380"
-ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"
+ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.3.200:2380"
+ETCD_ADVERTISE_CLIENT_URLS="http://192.168.3.200:2379"
-#ETCD_INITIAL_CLUSTER="default=http://localhost:2380"
-#ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
-#ETCD_INITIAL_CLUSTER_STATE="new"
+ETCD_INITIAL_CLUSTER="ryunosuke=http://192.168.3.200:2380"
+ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
+ETCD_INITIAL_CLUSTER_STATE="new"
-----
```
### Finalize installation
```
$ sudo systemctl enable etcd
$ sudo systemctl restart etcd
$ sudo systemctl status etcd
```

# Keystone Installation Tutorial for Red Hat Enterprise Linux and CentOS
https://docs.openstack.org/keystone/ussuri/install/index-rdo.html
## Install and configure
https://docs.openstack.org/keystone/ussuri/install/keystone-install-rdo.html
### Prerequisites
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;
```
### Install and configure components
```
$ sudo yum -y install openstack-keystone httpd python3-mod_wsgi
$ sudo vim /etc/keystone/keystone.conf
----
[database]
+ connection = mysql+pymysql://keystone:password@192.168.3.200/keystone
...
[token]
+ provider = fernet
----

$ sudo -s /bin/sh -c "keystone-manage db_sync" keystone
$ sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage bootstrap --bootstrap-password password --bootstrap-admin-url http://192.168.3.200:5000/v3/ --bootstrap-internal-url http://192.168.3.200:5000/v3/ --bootstrap-public-url http://192.168.3.200:5000/v3/ --bootstrap-region-id RegionOne
```

### Configure the Apache HTTP server
```
$ sudo vim /etc/httpd/conf/httpd.conf
----
+ ServerName 192.168.3.200
----
$ sudo ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```
### SSL
省略.  
よしなにSSL構成にするかSSL終端して下さいの記載のみとなっているため.
### Finalize the installation
```
$ sudo apachectl configtest
Syntax OK
$ sudo systemctl enable httpd.service
$ sudo systemctl restart httpd.service
$ sudo systemctl status httpd.service

$ export OS_USERNAME=admin
$ export OS_PASSWORD=password
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=default
$ export OS_PROJECT_DOMAIN_NAME=default
$ export OS_AUTH_URL=http://192.168.3.200:5000/v3
$ export OS_IDENTITY_API_VERSION=3
→ この辺の変数は次の手順で利用する.
```

## Create a domain, projects, users, and roles
https://docs.openstack.org/keystone/ussuri/install/keystone-users-rdo.html
```
$ openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | fdd499b8c5a3423f8bd068c9d86724e7 |
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
| id          | 5fe4e13210ac4b28865b982aa6ce0385 |
| is_domain   | False                            |
| name        | service                          |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+

$ openstack project create --domain default --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 14b03537059c4f5ebade3a03a70219d4 |
| is_domain   | False                            |
| name        | demo                             |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+

$ openstack user create --domain default --password-prompt demo
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 251d7c850e994535be3b4f1a8b67750a |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

$ openstack role create demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| domain_id   | None                             |
| id          | 245a95fee8f74b00850452809cba5f5d |
| name        | demo                             |
| options     | {}                               |
+-------------+----------------------------------+

$ openstack role add --project demo --user demo demo
```

## Verify operation
https://docs.openstack.org/keystone/ussuri/install/keystone-verify-rdo.html
```
$ unset OS_AUTH_URL OS_PASSWORD
$ openstack --os-auth-url http://192.168.3.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-07-26T08:29:12+0000                                                                                                                                                                |
| id         | gAAAAABfHTDI2Cuu7wbjWIG2nAN7Wb5oyHNi7TtaI4GU3XCFizW1nyboTzpT461867WLxoL6xCF0RWv6NBPC0VQ6FCqwLgyqfkiYFjyb7ggXjbAwovTBj4MZDz8OwUbxnk-3aRXFdchl2BBpru53_n5OCuJax7VWBWdeNwQdWrfA9KX7Ulr_tgA |
| project_id | 2994f37f552943bfbf00e1deaf0b483e                                                                                                                                                        |
| user_id    | 2bf771765b114d47bf77f63a6a2e90e8                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack --os-auth-url http://192.168.3.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name demo --os-username demo token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-07-26T08:29:49+0000                                                                                                                                                                |
| id         | gAAAAABfHTDtEQQ2NC2CyNHUHSEoHC1cbPpGL5f4c6IHjxMGaMgrfqJ1M3nA6J9gaFmv9F-4vYLKalu0CZtHOELY96elWWJYzYishFhPmf1LGVaDm5Vrttv3RaYBmSHAXWOyYUs2t82ol9d2rnkQ0Ve7HMOzdf8_NIfyfRREp_fM6i2JBBJvdTI |
| project_id | 14b03537059c4f5ebade3a03a70219d4                                                                                                                                                        |
| user_id    | 251d7c850e994535be3b4f1a8b67750a                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

## Create OpenStack client environment scripts
https://docs.openstack.org/keystone/ussuri/install/keystone-openrc-rdo.html
### Creating the scripts
```
$ cat << _EOF_ > ~/admin-openrc
> export OS_PROJECT_DOMAIN_NAME=default
> export OS_USER_DOMAIN_NAME=default
> export OS_PROJECT_NAME=admin
> export OS_USERNAME=admin
> export OS_PASSWORD=password
> export OS_AUTH_URL=http://192.168.3.200:5000/v3
> export OS_IDENTITY_API_VERSION=3
> export OS_IMAGE_API_VERSION=2
> _EOF_

$ cat << _EOF_ > ~/demo-openrc
> export OS_PROJECT_DOMAIN_NAME=default
> export OS_USER_DOMAIN_NAME=default
> export OS_PROJECT_NAME=demo
> export OS_USERNAME=demo
> export OS_PASSWORD=password
> export OS_AUTH_URL=http://192.168.3.200:5000/v3
> export OS_IDENTITY_API_VERSION=3
> export OS_IMAGE_API_VERSION=2
> _EOF_

$ chmod -v 400 ~/{admin,demo}-openrc
mode of '/home/ogalush/admin-openrc' changed from 0664 (rw-rw-r--) to 0400 (r--------)
mode of '/home/ogalush/demo-openrc' changed from 0664 (rw-rw-r--) to 0400 (r--------)

$ ls -l ~/{admin,demo}-openrc
-r-------- 1 ogalush ogalush 266 Jul 26 16:31 /home/ogalush/admin-openrc
-r-------- 1 ogalush ogalush 264 Jul 26 16:32 /home/ogalush/demo-openrc

$ source ~/admin-openrc
$ openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-07-26T08:33:07+0000                                                                                                                                                                |
| id         | gAAAAABfHTGzMptbJWB1NKTTRIhGSQkTZYqlgGP6iD77EdVl_zqvfE0akDKxlrZKS7MS6nQBYa0ZSsDnDX_DYdksr2CyXvS4tp-s0gBRLBe-6OZ-9z3Nh8pxaXqhJF42shMs6dXPInOvgr6BCC4M9XmI1sMgYQ-FwejY0dRbbfwLKM-RCdCyL4E |
| project_id | 2994f37f552943bfbf00e1deaf0b483e                                                                                                                                                        |
| user_id    | 2bf771765b114d47bf77f63a6a2e90e8                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
→ 値を取得できているのでOK.

$ source ~/demo-openrc 
$ openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-07-26T08:33:13+0000                                                                                                                                                                |
| id         | gAAAAABfHTG5zo-KUsBnJVePNuDVsN0S2eRidsfxse8iTAaRP8Lb56L4z2rPUBWWGzExc1AX64qKkq7zI_by7aR2FuGawZpxYlK7UgLQuc8pBgPxeC9oGKX7UKXXpa7UDGR7NDVhApp7KfuJNtKLNS-bXlXoi_sRi763B5G1YbdpTFi-Eb63M4U |
| project_id | 14b03537059c4f5ebade3a03a70219d4                                                                                                                                                        |
| user_id    | 251d7c850e994535be3b4f1a8b67750a                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
→ 値を取得できているのでOK.
```


# Glance Installation
https://docs.openstack.org/glance/ussuri/install/

## Install and configure (Red Hat)
https://docs.openstack.org/glance/ussuri/install/install-rdo.html
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
| id                  | 476f0520032b4aa7a2786862e5e5e7be |
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
| id          | 2dd69f06143a41b1ae242dacccbb8171 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne image public http://192.168.3.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 64ee455db9c442a487fbf24be298bad5 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2dd69f06143a41b1ae242dacccbb8171 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.3.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image internal http://192.168.3.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 75e22f1cc5e1438e9ed909db88bb0dca |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2dd69f06143a41b1ae242dacccbb8171 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.3.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image admin http://192.168.3.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 8a8231d7c68c4fe886e7d734d1a925d8 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2dd69f06143a41b1ae242dacccbb8171 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.3.200:9292        |
+--------------+----------------------------------+
```

### Install and configure components
```
$ sudo yum -y install openstack-glance
$ sudo vim /etc/glance/glance-api.conf
----
[database]
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
----

$ sudo -s /bin/sh -c "glance-manage db_sync" glance

$ sudo chown -v glance:glance /var/log/glance
$ sudo chown -v glance:glance /var/log/glance/api.log
→ glance起動時にPermission DeniedでStopするため.
```

### Finalize installation
```
$ sudo systemctl enable openstack-glance-api.service
$ sudo systemctl restart openstack-glance-api.service
$ sudo systemctl status openstack-glance-api.service
```

## Verify operation
https://docs.openstack.org/glance/ussuri/install/verify.html
```
$ source ~/admin-openrc 
$ glance image-list
+----+------+
| ID | Name |
+----+------+
+----+------+
→ 作成前は空であることを確認.

$ wget --directory-prefix=/tmp http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
$ sudo mv -v /tmp/cirros-0.4.0-x86_64-disk.img /usr/local/src
$ ls -l /usr/local/src
→ OSイメージがあること.

$ glance image-create --name "cirros" --file /usr/local/src/cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --visibility=public
+------------------+----------------------------------------------------------------------------------+
| Property         | Value                                                                            |
+------------------+----------------------------------------------------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe                                                 |
| container_format | bare                                                                             |
| created_at       | 2020-07-26T07:54:22Z                                                             |
| disk_format      | qcow2                                                                            |
| id               | 64becb16-bd9a-41c0-b955-e8bf38fa1348                                             |
| min_disk         | 0                                                                                |
| min_ram          | 0                                                                                |
| name             | cirros                                                                           |
| os_hash_algo     | sha512                                                                           |
| os_hash_value    | 6513f21e44aa3da349f248188a44bc304a3653a04122d8fb4535423c8e1d14cd6a153f735bb0982e |
|                  | 2161b5b5186106570c17a9e58b64dd39390617cd5a350f78                                 |
| os_hidden        | False                                                                            |
| owner            | 2994f37f552943bfbf00e1deaf0b483e                                                 |
| protected        | False                                                                            |
| size             | 12716032                                                                         |
| status           | active                                                                           |
| tags             | []                                                                               |
| updated_at       | 2020-07-26T07:54:22Z                                                             |
| virtual_size     | Not available                                                                    |
| visibility       | public                                                                           |
+------------------+----------------------------------------------------------------------------------+

$ glance image-list
+--------------------------------------+--------+
| ID                                   | Name   |
+--------------------------------------+--------+
| 64becb16-bd9a-41c0-b955-e8bf38fa1348 | cirros |
+--------------------------------------+--------+
→ 登録できたのでOK.
```

# Install and configure Placement for Red Hat Enterprise Linux and CentOS
https://docs.openstack.org/placement/ussuri/install/install-rdo.html
## Prerequisites
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE placement;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'password';                                                                                          MariaDB [(none)]> FLUSH PRIVILEGES;                                                                                                                                                         
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
| id                  | 5d4a89ddfea045e1830685832e113f2d |
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
| id          | 8d7494d6a1b74fc9adf9ba2836c37c84 |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement public http://192.168.3.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | dd47589a2a2a448c961a1c889990042d |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8d7494d6a1b74fc9adf9ba2836c37c84 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.3.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement internal http://192.168.3.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 2d9542edcb544e6b85d5b5d34c52efe0 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8d7494d6a1b74fc9adf9ba2836c37c84 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.3.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement admin http://192.168.3.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 3e7c067c54364bec81ac61dc96937f38 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8d7494d6a1b74fc9adf9ba2836c37c84 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.3.200:8778        |
+--------------+----------------------------------+
```

### Install and configure components
```
$ sudo yum -y install openstack-placement-api
$ sudo vim /etc/placement/placement.conf
----
[placement_database]
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

$ sudo chmod -v 644 /etc/placement/placement.conf
$ sudo chmod -v 644 /usr/share/placement/placement-dist.conf
→ 後で動作確認する際にPermission Deniedとなるので権限を入れておく.

$ sudo -s /bin/sh -c "placement-manage db sync" placement
```

### Finalize installation
```
$ sudo systemctl restart httpd
$ sudo systemctl status httpd
```

## Verify Installation
https://docs.openstack.org/placement/ussuri/install/verify.html
```
$ source ~/admin-openrc 
$ placement-status upgrade check
+----------------------------------+
| Upgrade Check Results            |
+----------------------------------+
| Check: Missing Root Provider IDs |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
| Check: Incomplete Consumers      |
| Result: Success                  |
| Details: None                    |
+----------------------------------+

$ sudo pip3 install osc-placement
$ openstack --os-placement-api-version 1.2 resource class list
Expecting value: line 1 column 1 (char 0)
→ 未登録なので見えなくて良いのかな..?
```

# Nova
## Install and configure controller node for Red Hat Enterprise Linux and CentOS
https://docs.openstack.org/nova/ussuri/install/controller-install-rdo.html
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
| id                  | d9489c87f9c94aafb1e3964fab96b6ae |
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
| id          | 25e17533fa1a4366ad4bb4695d260239 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute public http://192.168.3.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | ec81d7902d10453585e8d5c452c438d0 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 25e17533fa1a4366ad4bb4695d260239 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.3.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute internal http://192.168.3.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e15b510a4043439da2ba73b6b03ee05b |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 25e17533fa1a4366ad4bb4695d260239 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.3.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute admin http://192.168.3.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 92e414416f7346c781fff784a298ea84 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 25e17533fa1a4366ad4bb4695d260239 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.3.200:8774/v2.1   |
+--------------+----------------------------------+
```

### Install and configure components
```
$ sudo yum -y install openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler
$ sudo vim /etc/nova/nova.conf
----
[DEFAULT]
+ enabled_apis = osapi_compute,metadata
+ transport_url = rabbit://openstack:password@192.168.3.200:5672/
+ my_ip = 192.168.3.200
...
[api_database]
+ connection = mysql+pymysql://nova:password@192.168.3.200/nova_api
...
[database]
+ connection = mysql+pymysql://nova:password@192.168.3.200/nova
...
[api]
+ auth_strategy = keystone
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
----

$ sudo -s /bin/sh -c "nova-manage api_db sync" nova
$ sudo -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
$ sudo -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
--transport-url not provided in the command line, using the value [DEFAULT]/transport_url from the configuration file
--database_connection not provided in the command line, using the value [database]/connection from the configuration file
373f5d0c-ac8f-4864-aa93-db34aead5187

$ sudo -s /bin/sh -c "nova-manage db sync" nova
$ sudo -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
+-------+--------------------------------------+---------------------------------------------+----------------------------------------------------+----------+
|  Name |                 UUID                 |                Transport URL                |                Database Connection                 | Disabled |
+-------+--------------------------------------+---------------------------------------------+----------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                    none:/                   | mysql+pymysql://nova:****@192.168.3.200/nova_cell0 |  False   |
| cell1 | 373f5d0c-ac8f-4864-aa93-db34aead5187 | rabbit://openstack:****@192.168.3.200:5672/ |    mysql+pymysql://nova:****@192.168.3.200/nova    |  False   |
+-------+--------------------------------------+---------------------------------------------+----------------------------------------------------+----------+
→ 表示されたのでOK.

※バグ対応(nova-status upgrade checkでエラーとなる対応)
・On Stein, "nova-status upgrade check" check failed [closed]
https://ask.openstack.org/en/question/122313/on-stein-nova-status-upgrade-check-check-failed/
----
$ sudo chmod -v 644 /etc/nova/nova.conf
mode of '/etc/nova/nova.conf' changed from 0640 (rw-r-----) to 0644 (rw-r--r--)
$ sudo chmod -v 644 /etc/nova/policy.json
mode of '/etc/nova/policy.json' changed from 0640 (rw-r-----) to 0644 (rw-r--r--)
$ sudo chmod -v 644 /usr/share/nova/nova-dist.conf
$ sudo cp -pv /etc/httpd/conf.d/00-placement-api.conf  /tmp
$ sudo diff --unified=0 /tmp/00-placement-api.conf /etc/httpd/conf.d/00-placement-api.conf 
--- /tmp/00-placement-api.conf  2020-05-13 21:20:33.000000000 +0900
+++ /etc/httpd/conf.d/00-placement-api.conf     2020-07-26 17:43:23.757368457 +0900
@@ -15,0 +16,9 @@
+  <Directory /usr/bin>
+    <IfVersion >= 2.4>
+      Require all granted
+    </IfVersion>
+    <IfVersion < 2.4>
+      Order allow,deny
+      Allow from all
+    </IfVersion>
+  </Directory>
$
→ VirtualHostの中にDirectory設定を入れる.
----
```

### Finalize installation
```
$ sudo systemctl enable openstack-nova-api.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
$ sudo systemctl restart openstack-nova-api.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
$ sudo systemctl status openstack-nova-api.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
```

## Install and configure a compute node for Red Hat Enterprise Linux and CentOS
https://docs.openstack.org/nova/ussuri/install/compute-install-rdo.html
### Install and configure components
```
$ sudo yum -y install openstack-nova-compute
$ sudo vim /etc/nova/nova.conf
----
[vnc]
...
novncproxy_base_url = http://192.168.3.200:6080/vnc_auto.html
----
```

### Finalize installation
```
$ egrep -c '(vmx|svm)' /proc/cpuinfo
4
$ sudo vim /etc/nova/nova.conf
----
[libvirt]
+ virt_type=kvm
----

$ sudo systemctl enable libvirtd.service openstack-nova-compute.service
$ sudo systemctl restart libvirtd.service openstack-nova-compute.service
$ sudo systemctl status libvirtd.service openstack-nova-compute.service
→ Status: AcitveであればOK.
```

### Add the compute node to the cell database
```
$ source ~/admin-openrc 
$ openstack compute service list --service nova-compute
+----+--------------+-----------------------+------+---------+-------+----------------------------+
| ID | Binary       | Host                  | Zone | Status  | State | Updated At                 |
+----+--------------+-----------------------+------+---------+-------+----------------------------+
|  7 | nova-compute | ryunosuke.localdomain | nova | enabled | up    | 2020-07-26T08:33:50.000000 |
+----+--------------+-----------------------+------+---------+-------+----------------------------+

$ sudo -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': 373f5d0c-ac8f-4864-aa93-db34aead5187
Checking host mapping for compute host 'ryunosuke.localdomain': a5c71ed5-c970-4a83-918d-eb1e632d664f
Creating host mapping for compute host 'ryunosuke.localdomain': a5c71ed5-c970-4a83-918d-eb1e632d664f
Found 1 unmapped computes in cell: 373f5d0c-ac8f-4864-aa93-db34aead5187

$ sudo vim /etc/nova/nova.conf
----
[scheduler]
+ discover_hosts_in_cells_interval = 300
----

$ sudo systemctl restart openstack-nova-scheduler.service
$ sudo systemctl status openstack-nova-scheduler.service
```

## Verify operation
https://docs.openstack.org/nova/ussuri/install/verify.html
```
$ source ~/admin-openrc
$ openstack compute service list
+----+----------------+-----------------------+----------+---------+-------+----------------------------+
| ID | Binary         | Host                  | Zone     | Status  | State | Updated At                 |
+----+----------------+-----------------------+----------+---------+-------+----------------------------+
|  1 | nova-conductor | ryunosuke.localdomain | internal | enabled | up    | 2020-07-26T08:36:20.000000 |
|  3 | nova-scheduler | ryunosuke.localdomain | internal | enabled | up    | 2020-07-26T08:36:14.000000 |
|  7 | nova-compute   | ryunosuke.localdomain | nova     | enabled | up    | 2020-07-26T08:36:10.000000 |
+----+----------------+-----------------------+----------+---------+-------+----------------------------+

$ openstack catalog list
+-----------+-----------+--------------------------------------------+
| Name      | Type      | Endpoints                                  |
+-----------+-----------+--------------------------------------------+
| keystone  | identity  | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:5000/v3/     |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.3.200:5000/v3/    |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:5000/v3/  |
|           |           |                                            |
| nova      | compute   | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:8774/v2.1    |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:8774/v2.1 |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.3.200:8774/v2.1   |
|           |           |                                            |
| glance    | image     | RegionOne                                  |
|           |           |   public: http://192.168.3.200:9292        |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:9292      |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:9292         |
|           |           |                                            |
| placement | placement | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:8778      |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:8778         |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.3.200:8778        |
|           |           |                                            |
+-----------+-----------+--------------------------------------------+

$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 64becb16-bd9a-41c0-b955-e8bf38fa1348 | cirros | active |
+--------------------------------------+--------+--------+

$ nova-status upgrade check
+------------------------------------+
| Upgrade Check Results              |
+------------------------------------+
| Check: Cells v2                    |
| Result: Success                    |
| Details: None                      |
+------------------------------------+
| Check: Placement API               |
| Result: Success                    |
| Details: None                      |
+------------------------------------+
| Check: Ironic Flavor Migration     |
| Result: Success                    |
| Details: None                      |
+------------------------------------+
| Check: Cinder API                  |
| Result: Success                    |
| Details: None                      |
+------------------------------------+
| Check: Policy Scope-based Defaults |
| Result: Success                    |
| Details: None                      |
+------------------------------------+
```

# Neutron Install and configure for Red Hat Enterprise Linux and CentOS
https://docs.openstack.org/neutron/ussuri/install/install-rdo.html
## Install and configure controller node
https://docs.openstack.org/neutron/ussuri/install/controller-install-rdo.html
### Prerequisites
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
| id                  | 294e4a0c912d4e0a9a612fb1e0f5ffb0 |
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
| id          | 0cfe6ad2726e46018c08accf186302e3 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne network public http://192.168.3.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a0d855296a954b9cbd94e891364a4050 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 0cfe6ad2726e46018c08accf186302e3 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.3.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network internal http://192.168.3.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | aa8925d888eb483b8160d8387061c131 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 0cfe6ad2726e46018c08accf186302e3 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.3.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network admin http://192.168.3.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 10196c77383a4c5e94a32922b88db15c |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 0cfe6ad2726e46018c08accf186302e3 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.3.200:9696        |
+--------------+----------------------------------+
```

### Configure networking options
Networking Option 2: Self-service networks
### Networking Option 2: Self-service networks
https://docs.openstack.org/neutron/ussuri/install/controller-install-option2-rdo.html
#### Install the components
```
$ sudo yum -y install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables
```

#### Configure the server component
```
$ sudo vim /etc/neutron/neutron.conf
----
[DEFAULT]
+ core_plugin = ml2
+ service_plugins = router
+ allow_overlapping_ips = true
+ transport_url = rabbit://openstack:password@192.168.3.200
+ auth_strategy = keystone
+ notify_nova_on_port_status_changes = true
+ notify_nova_on_port_data_changes = true
+ dns_domain = localdomain
...
[database]
+ connection = mysql+pymysql://neutron:password@192.168.3.200/neutron
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
+ [nova]
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
----
```

#### Configure the Modular Layer 2 (ML2) plug-in¶
```
$ sudo vim /etc/neutron/plugins/ml2/ml2_conf.ini
----
...
+ [ml2]
+ type_drivers = flat,vlan,vxlan
+ tenant_network_types = vxlan
+ mechanism_drivers = linuxbridge,l2population
+ extension_drivers = port_security

+ [ml2_type_flat]
+ flat_networks = provider

+ [ml2_type_vxlan]
+ vni_ranges = 1:1000

+ [securitygroup]
+ enable_ipset = true
----
```

#### Configure the Linux bridge agent
```
$ sudo vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
----
+ [linux_bridge]
+ physical_interface_mappings = provider:enp3s0

+ [vxlan]
+ enable_vxlan = true
+ local_ip = 192.168.3.200
+ l2_population = true

+ [securitygroup]
+ enable_security_group = true
+ firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
----

$ grep 'net.bridge.bridge-nf-call-ip' /usr/lib/sysctl.d/99-neutron-linuxbridge-agent.conf 
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
→ マニュアルにあるKernelParameterは入っていそう.
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
----
```

### Finalize installation¶
```
$ sudo ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
$ ls -l /etc/neutron/plugin.ini
lrwxrwxrwx 1 root root 37 Jul 26 18:47 /etc/neutron/plugin.ini -> /etc/neutron/plugins/ml2/ml2_conf.ini

$ sudo -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

$ sudo systemctl restart openstack-nova-api.service
$ sudo systemctl status openstack-nova-api.service
→ active (runnning)ならOK.

$ sudo systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service neutron-l3-agent.service

$ sudo systemctl start neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service neutron-l3-agent.service
```

## Verify operation
https://docs.openstack.org/neutron/ussuri/install/verify.html
```
$ source ~/admin-openrc
$ openstack extension list --network
+----------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Name                                                                                                                                                           | Alias                                 | Description                                                                                                                                              |
+----------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Address scope                                                                                                                                                  | address-scope                         | Address scopes extension.                                                                                                                                |
| Enforce Router's Admin State Down Before Update Extension                                                                                                      | router-admin-state-down-before-update | Ensure that the admin state of a router is down (admin_state_up=False) before updating the distributed attribute                                         |
| agent                                                                                                                                                          | agent                                 | The agent management extension.                                                                                                                          |
| Agent's Resource View Synced to Placement                                                                                                                      | agent-resources-synced                | Stores success/failure of last sync to Placement                                                                                                         |
...(略)...
| Resource timestamps                                                                                                                                            | standard-attr-timestamp               | Adds created_at and updated_at fields to all Neutron resources that have Neutron standard attributes.                                                    |
+----------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
$

$ openstack network agent list
+--------------------------------------+--------------------+-----------------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host                  | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+-----------------------+-------------------+-------+-------+---------------------------+
| 33b9dedc-4ca4-4c85-ab58-4af65ff33f2c | Linux bridge agent | ryunosuke.localdomain | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 8e5ee9c9-ae94-4ba9-8b12-169b2945647e | DHCP agent         | ryunosuke.localdomain | nova              | :-)   | UP    | neutron-dhcp-agent        |
| ba69e63d-65a0-4154-84bf-3412beaea557 | L3 agent           | ryunosuke.localdomain | nova              | :-)   | UP    | neutron-l3-agent          |
| bae18d7b-c8a1-48a0-bb2f-3a43013002d6 | Metadata agent     | ryunosuke.localdomain | None              | :-)   | UP    | neutron-metadata-agent    |
+--------------------------------------+--------------------+-----------------------+-------------------+-------+-------+---------------------------+
→ 全てのagentがUPしているのでOK.
```


# Launch an instance
https://docs.openstack.org/install-guide/launch-instance.html
## Create virtual networks
### Provider network
https://docs.openstack.org/install-guide/launch-instance-networks-provider.html
```
$ source ~/admin-openrc
$ openstack network create --share --external --provider-physical-network provider --provider-network-type flat provider
+---------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                     | Value                                                                                                                                                   |
+---------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up            | UP                                                                                                                                                      |
| availability_zone_hints   |                                                                                                                                                         |
| availability_zones        |                                                                                                                                                         |
| created_at                | 2020-07-26T09:57:46Z                                                                                                                                    |
| description               |                                                                                                                                                         |
| dns_domain                | None                                                                                                                                                    |
| id                        | d14b13e6-a8c2-4d19-8161-8d7b4f6a8722                                                                                                                    |
| ipv4_address_scope        | None                                                                                                                                                    |
| ipv6_address_scope        | None                                                                                                                                                    |
| is_default                | False                                                                                                                                                   |
| is_vlan_transparent       | None                                                                                                                                                    |
| location                  | cloud='', project.domain_id=, project.domain_name='default', project.id='2994f37f552943bfbf00e1deaf0b483e', project.name='admin', region_name='', zone= |
| mtu                       | 1500                                                                                                                                                    |
| name                      | provider                                                                                                                                                |
| port_security_enabled     | True                                                                                                                                                    |
| project_id                | 2994f37f552943bfbf00e1deaf0b483e                                                                                                                        |
| provider:network_type     | flat                                                                                                                                                    |
| provider:physical_network | provider                                                                                                                                                |
| provider:segmentation_id  | None                                                                                                                                                    |
| qos_policy_id             | None                                                                                                                                                    |
| revision_number           | 1                                                                                                                                                       |
| router:external           | External                                                                                                                                                |
| segments                  | None                                                                                                                                                    |
| shared                    | True                                                                                                                                                    |
| status                    | ACTIVE                                                                                                                                                  |
| subnets                   |                                                                                                                                                         |
| tags                      |                                                                                                                                                         |
| updated_at                | 2020-07-26T09:57:46Z                                                                                                                                    |
+---------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack subnet create --network provider --allocation-pool start=192.168.3.130,end=192.168.3.150 --dns-nameserver 192.168.3.220 --gateway 192.168.3.254 --subnet-range 192.168.3.0/24 provider
+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                | Value                                                                                                                                                   |
+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| allocation_pools     | 192.168.3.130-192.168.3.150                                                                                                                             |
| cidr                 | 192.168.3.0/24                                                                                                                                          |
| created_at           | 2020-07-26T09:59:27Z                                                                                                                                    |
| description          |                                                                                                                                                         |
| dns_nameservers      | 192.168.3.220                                                                                                                                           |
| dns_publish_fixed_ip | None                                                                                                                                                    |
| enable_dhcp          | True                                                                                                                                                    |
| gateway_ip           | 192.168.3.254                                                                                                                                           |
| host_routes          |                                                                                                                                                         |
| id                   | 509f5eea-b007-4431-88a0-9117637636c1                                                                                                                    |
| ip_version           | 4                                                                                                                                                       |
| ipv6_address_mode    | None                                                                                                                                                    |
| ipv6_ra_mode         | None                                                                                                                                                    |
| location             | cloud='', project.domain_id=, project.domain_name='default', project.id='2994f37f552943bfbf00e1deaf0b483e', project.name='admin', region_name='', zone= |
| name                 | provider                                                                                                                                                |
| network_id           | d14b13e6-a8c2-4d19-8161-8d7b4f6a8722                                                                                                                    |
| prefix_length        | None                                                                                                                                                    |
| project_id           | 2994f37f552943bfbf00e1deaf0b483e                                                                                                                        |
| revision_number      | 0                                                                                                                                                       |
| segment_id           | None                                                                                                                                                    |
| service_types        |                                                                                                                                                         |
| subnetpool_id        | None                                                                                                                                                    |
| tags                 |                                                                                                                                                         |
| updated_at           | 2020-07-26T09:59:27Z                                                                                                                                    |
+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
```

### Self-service network
```
$ source ~/demo-openrc
$ openstack network create selfservice
+---------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                     | Value                                                                                                                                                  |
+---------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up            | UP                                                                                                                                                     |
| availability_zone_hints   |                                                                                                                                                        |
| availability_zones        |                                                                                                                                                        |
| created_at                | 2020-07-26T10:00:23Z                                                                                                                                   |
| description               |                                                                                                                                                        |
| dns_domain                | None                                                                                                                                                   |
| id                        | ccc1e57e-483f-4a30-bf13-beaafbae9517                                                                                                                   |
| ipv4_address_scope        | None                                                                                                                                                   |
| ipv6_address_scope        | None                                                                                                                                                   |
| is_default                | False                                                                                                                                                  |
| is_vlan_transparent       | None                                                                                                                                                   |
| location                  | cloud='', project.domain_id=, project.domain_name='default', project.id='14b03537059c4f5ebade3a03a70219d4', project.name='demo', region_name='', zone= |
| mtu                       | 1450                                                                                                                                                   |
| name                      | selfservice                                                                                                                                            |
| port_security_enabled     | True                                                                                                                                                   |
| project_id                | 14b03537059c4f5ebade3a03a70219d4                                                                                                                       |
| provider:network_type     | None                                                                                                                                                   |
| provider:physical_network | None                                                                                                                                                   |
| provider:segmentation_id  | None                                                                                                                                                   |
| qos_policy_id             | None                                                                                                                                                   |
| revision_number           | 1                                                                                                                                                      |
| router:external           | Internal                                                                                                                                               |
| segments                  | None                                                                                                                                                   |
| shared                    | False                                                                                                                                                  |
| status                    | ACTIVE                                                                                                                                                 |
| subnets                   |                                                                                                                                                        |
| tags                      |                                                                                                                                                        |
| updated_at                | 2020-07-26T10:00:23Z                                                                                                                                   |
+---------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack subnet create --network selfservice --dns-nameserver 192.168.3.220 --gateway 10.0.0.1 --subnet-range 10.0.0.0/24 selfservice
+----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                | Value                                                                                                                                                  |
+----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
| allocation_pools     | 10.0.0.2-10.0.0.254                                                                                                                                    |
| cidr                 | 10.0.0.0/24                                                                                                                                            |
| created_at           | 2020-07-26T10:02:00Z                                                                                                                                   |
| description          |                                                                                                                                                        |
| dns_nameservers      | 192.168.3.220                                                                                                                                          |
| dns_publish_fixed_ip | None                                                                                                                                                   |
| enable_dhcp          | True                                                                                                                                                   |
| gateway_ip           | 10.0.0.1                                                                                                                                               |
| host_routes          |                                                                                                                                                        |
| id                   | 9e74a535-6bdf-4991-b12e-00f2b918520e                                                                                                                   |
| ip_version           | 4                                                                                                                                                      |
| ipv6_address_mode    | None                                                                                                                                                   |
| ipv6_ra_mode         | None                                                                                                                                                   |
| location             | cloud='', project.domain_id=, project.domain_name='default', project.id='14b03537059c4f5ebade3a03a70219d4', project.name='demo', region_name='', zone= |
| name                 | selfservice                                                                                                                                            |
| network_id           | ccc1e57e-483f-4a30-bf13-beaafbae9517                                                                                                                   |
| prefix_length        | None                                                                                                                                                   |
| project_id           | 14b03537059c4f5ebade3a03a70219d4                                                                                                                       |
| revision_number      | 0                                                                                                                                                      |
| segment_id           | None                                                                                                                                                   |
| service_types        |                                                                                                                                                        |
| subnetpool_id        | None                                                                                                                                                   |
| tags                 |                                                                                                                                                        |
| updated_at           | 2020-07-26T10:02:00Z                                                                                                                                   |
+----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack router create router
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                   | Value                                                                                                                                                  |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up          | UP                                                                                                                                                     |
| availability_zone_hints |                                                                                                                                                        |
| availability_zones      |                                                                                                                                                        |
| created_at              | 2020-07-26T10:02:19Z                                                                                                                                   |
| description             |                                                                                                                                                        |
| external_gateway_info   | null                                                                                                                                                   |
| flavor_id               | None                                                                                                                                                   |
| id                      | 64b976b9-9456-4ad5-a86f-33055a5b5740                                                                                                                   |
| location                | cloud='', project.domain_id=, project.domain_name='default', project.id='14b03537059c4f5ebade3a03a70219d4', project.name='demo', region_name='', zone= |
| name                    | router                                                                                                                                                 |
| project_id              | 14b03537059c4f5ebade3a03a70219d4                                                                                                                       |
| revision_number         | 1                                                                                                                                                      |
| routes                  |                                                                                                                                                        |
| status                  | ACTIVE                                                                                                                                                 |
| tags                    |                                                                                                                                                        |
| updated_at              | 2020-07-26T10:02:19Z                                                                                                                                   |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack router add subnet router selfservice
$ openstack router set router --external-gateway provider
```

## Verify operation¶
```
$ source ~/admin-openrc
$ ip netns
qrouter-64b976b9-9456-4ad5-a86f-33055a5b5740 (id: 2)
qdhcp-ccc1e57e-483f-4a30-bf13-beaafbae9517 (id: 1)
qdhcp-d14b13e6-a8c2-4d19-8161-8d7b4f6a8722 (id: 0)

$ openstack port list --router router
+--------------------------------------+------+-------------------+------------------------------------------------------------------------------+--------+
| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                           | Status |
+--------------------------------------+------+-------------------+------------------------------------------------------------------------------+--------+
| 506ec5ab-85d3-4b95-8087-a5a63b3aad15 |      | fa:16:3e:f7:b7:9b | ip_address='10.0.0.1', subnet_id='9e74a535-6bdf-4991-b12e-00f2b918520e'      | ACTIVE |
| e88b3d08-502e-440c-bb30-933098dfaa90 |      | fa:16:3e:37:2e:f9 | ip_address='192.168.3.148', subnet_id='509f5eea-b007-4431-88a0-9117637636c1' | ACTIVE |
+--------------------------------------+------+-------------------+------------------------------------------------------------------------------+--------+

$ ping -c 4 192.168.3.148
PING 192.168.3.148 (192.168.3.148) 56(84) bytes of data.
64 bytes from 192.168.3.148: icmp_seq=1 ttl=64 time=0.102 ms
64 bytes from 192.168.3.148: icmp_seq=2 ttl=64 time=0.055 ms
64 bytes from 192.168.3.148: icmp_seq=3 ttl=64 time=0.052 ms
64 bytes from 192.168.3.148: icmp_seq=4 ttl=64 time=0.054 ms

--- 192.168.3.148 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 68ms
rtt min/avg/max/mdev = 0.052/0.065/0.102/0.023 ms
[ogalush@ryunosuke ~]$
→ 上記RouterのIP(192.168.3.148)へpingを通せたのでOK.
```

## Create m1.nano flavor
https://docs.openstack.org/install-guide/launch-instance.html#launch-instance-networks
```
$ source ~/admin-openrc 
$ openstack flavor create --id 0 --vcpus 1 --ram 64 --disk 1 m1.nano
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
| ram                        | 64      |
| rxtx_factor                | 1.0     |
| swap                       |         |
| vcpus                      | 1       |
+----------------------------+---------+
```

## Generate a key pair
```
$ source ~/demo-openrc
$ openstack keypair create --public-key ~/.ssh/authorized_keys ogalush_key
+-------------+-------------------------------------------------+
| Field       | Value                                           |
+-------------+-------------------------------------------------+
| fingerprint | 95:93:ad:1a:4a:1c:41:7b:26:b7:c7:fc:3b:0a:91:df |
| name        | ogalush_key                                     |
| user_id     | 251d7c850e994535be3b4f1a8b67750a                |
+-------------+-------------------------------------------------+

$ openstack keypair list
+-------------+-------------------------------------------------+
| Name        | Fingerprint                                     |
+-------------+-------------------------------------------------+
| ogalush_key | 95:93:ad:1a:4a:1c:41:7b:26:b7:c7:fc:3b:0a:91:df |
+-------------+-------------------------------------------------+
```

## Add security group rules
```
$ source ~/demo-openrc
$ openstack security group rule create --proto icmp default
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field             | Value                                                                                                                                                  |
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
| created_at        | 2020-07-26T10:10:07Z                                                                                                                                   |
| description       |                                                                                                                                                        |
| direction         | ingress                                                                                                                                                |
| ether_type        | IPv4                                                                                                                                                   |
| id                | 5e27a211-50c3-40f2-8625-a27fb66bb534                                                                                                                   |
| location          | cloud='', project.domain_id=, project.domain_name='default', project.id='14b03537059c4f5ebade3a03a70219d4', project.name='demo', region_name='', zone= |
| name              | None                                                                                                                                                   |
| port_range_max    | None                                                                                                                                                   |
| port_range_min    | None                                                                                                                                                   |
| project_id        | 14b03537059c4f5ebade3a03a70219d4                                                                                                                       |
| protocol          | icmp                                                                                                                                                   |
| remote_group_id   | None                                                                                                                                                   |
| remote_ip_prefix  | 0.0.0.0/0                                                                                                                                              |
| revision_number   | 0                                                                                                                                                      |
| security_group_id | ccabec2b-283a-4c70-8ec8-de07b72f4fc4                                                                                                                   |
| tags              | []                                                                                                                                                     |
| updated_at        | 2020-07-26T10:10:07Z                                                                                                                                   |
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack security group rule create --proto tcp --dst-port 22 default
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field             | Value                                                                                                                                                  |
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
| created_at        | 2020-07-26T10:10:25Z                                                                                                                                   |
| description       |                                                                                                                                                        |
| direction         | ingress                                                                                                                                                |
| ether_type        | IPv4                                                                                                                                                   |
| id                | 104d141d-e96e-4c6b-87fe-37624c50a79e                                                                                                                   |
| location          | cloud='', project.domain_id=, project.domain_name='default', project.id='14b03537059c4f5ebade3a03a70219d4', project.name='demo', region_name='', zone= |
| name              | None                                                                                                                                                   |
| port_range_max    | 22                                                                                                                                                     |
| port_range_min    | 22                                                                                                                                                     |
| project_id        | 14b03537059c4f5ebade3a03a70219d4                                                                                                                       |
| protocol          | tcp                                                                                                                                                    |
| remote_group_id   | None                                                                                                                                                   |
| remote_ip_prefix  | 0.0.0.0/0                                                                                                                                              |
| revision_number   | 0                                                                                                                                                      |
| security_group_id | ccabec2b-283a-4c70-8ec8-de07b72f4fc4                                                                                                                   |
| tags              | []                                                                                                                                                     |
| updated_at        | 2020-07-26T10:10:25Z                                                                                                                                   |
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
```

## Launch an instance on the self-service network
https://docs.openstack.org/install-guide/launch-instance-selfservice.html
```
$ source ~/demo-openrc
$ openstack flavor list
+----+---------+-----+------+-----------+-------+-----------+
| ID | Name    | RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+---------+-----+------+-----------+-------+-----------+
| 0  | m1.nano |  64 |    1 |         0 |     1 | True      |
+----+---------+-----+------+-----------+-------+-----------+

$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 64becb16-bd9a-41c0-b955-e8bf38fa1348 | cirros | active |
+--------------------------------------+--------+--------+

$ openstack network list
+--------------------------------------+-------------+--------------------------------------+
| ID                                   | Name        | Subnets                              |
+--------------------------------------+-------------+--------------------------------------+
| ccc1e57e-483f-4a30-bf13-beaafbae9517 | selfservice | 9e74a535-6bdf-4991-b12e-00f2b918520e |
| d14b13e6-a8c2-4d19-8161-8d7b4f6a8722 | provider    | 509f5eea-b007-4431-88a0-9117637636c1 |
+--------------------------------------+-------------+--------------------------------------+

$ openstack security group list
+--------------------------------------+---------+------------------------+----------------------------------+------+
| ID                                   | Name    | Description            | Project                          | Tags |
+--------------------------------------+---------+------------------------+----------------------------------+------+
| ccabec2b-283a-4c70-8ec8-de07b72f4fc4 | default | Default security group | 14b03537059c4f5ebade3a03a70219d4 | []   |
+--------------------------------------+---------+------------------------+----------------------------------+------+

$ openstack server create --flavor m1.nano --image cirros --nic net-id=ccc1e57e-483f-4a30-bf13-beaafbae9517 --security-group default --key-name ogalush_key selfservice-instance
+-----------------------------+-----------------------------------------------+
| Field                       | Value                                         |
+-----------------------------+-----------------------------------------------+
| OS-DCF:diskConfig           | MANUAL                                        |
| OS-EXT-AZ:availability_zone |                                               |
| OS-EXT-STS:power_state      | NOSTATE                                       |
| OS-EXT-STS:task_state       | scheduling                                    |
| OS-EXT-STS:vm_state         | building                                      |
| OS-SRV-USG:launched_at      | None                                          |
| OS-SRV-USG:terminated_at    | None                                          |
| accessIPv4                  |                                               |
| accessIPv6                  |                                               |
| addresses                   |                                               |
| adminPass                   | 8i56sY6dFYB9                                  |
| config_drive                |                                               |
| created                     | 2020-07-26T10:27:57Z                          |
| flavor                      | m1.nano (0)                                   |
| hostId                      |                                               |
| id                          | 735f59fa-4a72-4787-8768-1ab4d6824a4e          |
| image                       | cirros (64becb16-bd9a-41c0-b955-e8bf38fa1348) |
| key_name                    | ogalush_key                                   |
| name                        | selfservice-instance                          |
| progress                    | 0                                             |
| project_id                  | 14b03537059c4f5ebade3a03a70219d4              |
| properties                  |                                               |
| security_groups             | name='ccabec2b-283a-4c70-8ec8-de07b72f4fc4'   |
| status                      | BUILD                                         |
| updated                     | 2020-07-26T10:27:57Z                          |
| user_id                     | 251d7c850e994535be3b4f1a8b67750a              |
| volumes_attached            |                                               |
+-----------------------------+-----------------------------------------------+

$ openstack server list
+--------------------------------------+----------------------+--------+------------------------+--------+---------+
| ID                                   | Name                 | Status | Networks               | Image  | Flavor  |
+--------------------------------------+----------------------+--------+------------------------+--------+---------+
| 735f59fa-4a72-4787-8768-1ab4d6824a4e | selfservice-instance | ACTIVE | selfservice=10.0.0.228 | cirros | m1.nano |
+--------------------------------------+----------------------+--------+------------------------+--------+---------+
→ ActiveとなったのでOK.
```

### Access the instance using a virtual console
```
$ openstack console url show selfservice-instance
+-------+----------------------------------------------------------------------------------------------+
| Field | Value                                                                                        |
+-------+----------------------------------------------------------------------------------------------+
| type  | novnc                                                                                        |
| url   | http://192.168.3.200:6080/vnc_auto.html?path=%3Ftoken%3D0993a177-3377-4004-bb58-70ba47601967 |
+-------+----------------------------------------------------------------------------------------------+
→ 接続を確立できず.
とりあえず続ける.
```

### Access the instance remotely
```
$ openstack floating ip create provider
+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field               | Value                                                                                                                                                                            |
+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| created_at          | 2020-07-26T10:50:34Z                                                                                                                                                             |
| description         |                                                                                                                                                                                  |
| dns_domain          | None                                                                                                                                                                             |
| dns_name            | None                                                                                                                                                                             |
| fixed_ip_address    | None                                                                                                                                                                             |
| floating_ip_address | 192.168.3.144                                                                                                                                                                    |
| floating_network_id | d14b13e6-a8c2-4d19-8161-8d7b4f6a8722                                                                                                                                             |
| id                  | 52942075-d49e-4b85-b37b-76b051d9c141                                                                                                                                             |
| location            | Munch({'cloud': '', 'region_name': '', 'zone': None, 'project': Munch({'id': '14b03537059c4f5ebade3a03a70219d4', 'name': 'demo', 'domain_id': None, 'domain_name': 'default'})}) |
| name                | 192.168.3.144                                                                                                                                                                    |
| port_details        | None                                                                                                                                                                             |
| port_id             | None                                                                                                                                                                             |
| project_id          | 14b03537059c4f5ebade3a03a70219d4                                                                                                                                                 |
| qos_policy_id       | None                                                                                                                                                                             |
| revision_number     | 0                                                                                                                                                                                |
| router_id           | None                                                                                                                                                                             |
| status              | DOWN                                                                                                                                                                             |
| subnet_id           | None                                                                                                                                                                             |
| tags                | []                                                                                                                                                                               |
| updated_at          | 2020-07-26T10:50:34Z                                                                                                                                                             |
+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack server add floating ip selfservice-instance 192.168.3.144
$ openstack server list
+--------------------------------------+----------------------+--------+---------------------------------------+--------+---------+
| ID                                   | Name                 | Status | Networks                              | Image  | Flavor  |
+--------------------------------------+----------------------+--------+---------------------------------------+--------+---------+
| 735f59fa-4a72-4787-8768-1ab4d6824a4e | selfservice-instance | ACTIVE | selfservice=10.0.0.228, 192.168.3.144 | cirros | m1.nano |
+--------------------------------------+----------------------+--------+---------------------------------------+--------+---------+
→ インスタンスにFloatingIPが付いたのでOK.

[ogalush@ryunosuke ~]$ ping -c 3 192.168.3.144
PING 192.168.3.144 (192.168.3.144) 56(84) bytes of data.
From 192.168.3.144 icmp_seq=1 Destination Host Unreachable
From 192.168.3.144 icmp_seq=2 Destination Host Unreachable
From 192.168.3.144 icmp_seq=3 Destination Host Unreachable

--- 192.168.3.144 ping statistics ---
3 packets transmitted, 0 received, +3 errors, 100% packet loss, time 60ms
pipe 3
[ogalush@ryunosuke ~]$ ping -c 3 10.0.0.228
PING 10.0.0.228 (10.0.0.228) 56(84) bytes of data.

--- 10.0.0.228 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 52ms

[ogalush@ryunosuke ~]$
→ これまた繋がらない.
```

## firewalld
Firewalldが起動している状態の場合、vncなどの接続が切れるので無効化しておく.  
参考: http://www.oss-note.com/centos/centos76/stein2  
https://docs.openstack.org/install-guide/firewalls-default-ports.html
```
$ sudo systemctl is-enabled firewalld
enabled
$ sudo systemctl is-active firewalld
active
$ sudo systemctl stop firewalld
$ sudo systemctl disable firewalld
Removed /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
$ sudo systemctl is-enabled firewalld
disabled
$ sudo systemctl is-active firewalld
inactive
```
確認
```
$ ping -c 4 192.168.3.144
PING 192.168.3.144 (192.168.3.144) 56(84) bytes of data.
64 bytes from 192.168.3.144: icmp_seq=1 ttl=63 time=0.242 ms
64 bytes from 192.168.3.144: icmp_seq=2 ttl=63 time=0.218 ms
64 bytes from 192.168.3.144: icmp_seq=3 ttl=63 time=0.244 ms
64 bytes from 192.168.3.144: icmp_seq=4 ttl=63 time=0.235 ms

--- 192.168.3.144 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 73ms
rtt min/avg/max/mdev = 0.218/0.234/0.244/0.021 ms
→ FloatingIPへのpingが通ったのでOK.
```

# Horizon
https://docs.openstack.org/horizon/ussuri/install/
## Install and configure for Red Hat Enterprise Linux and CentOS
https://docs.openstack.org/horizon/ussuri/install/install-rdo.html
### Install and configure components
```
$ sudo yum install -y openstack-dashboard
$ sudo vim /etc/openstack-dashboard/local_settings 
----
- OPENSTACK_HOST = "127.0.0.1"
+ OPENSTACK_HOST = "192.168.3.200"
- OPENSTACK_KEYSTONE_URL = "http://%s/identity/v3" % OPENSTACK_HOST
+ OPENSTACK_KEYSTONE_URL = "http://%s:5000/identity/v3" % OPENSTACK_HOST
- ALLOWED_HOSTS = ['horizon.example.com', 'localhost']
+ ALLOWED_HOSTS = ['*']
+ SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
+ CACHES = {
+     'default': {
+          'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
+          'LOCATION': '192.168.3.200:11211',
+     }
+ }
...
+ OPENSTACK_API_VERSIONS = {
+     "identity": 3,
+     "image": 2,
+     "volume": 3,
+ }
+ OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "default"
+ OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
- TIME_ZONE = "UTC"
+ TIME_ZONE = "Asia/Tokyo"
+ WEBROOT = '/dashboard'
----
WEBROOTを入れたのは以下のバグ対応のため.
[Horizon Install Guide - missing 'WEBROOT' directive in horizon config file](https://bugs.launchpad.net/horizon/+bug/1853651)  

$ sudo vim /etc/httpd/conf.d/openstack-dashboard.conf
----
+ WSGIApplicationGroup %{GLOBAL}
----
```

### Finalize installation
```
$ sudo systemctl restart httpd.service memcached.service
$ sudo systemctl is-active httpd.service memcached.service
active
active
```

## Verify operation for Red Hat Enterprise Linux and CentOS
https://docs.openstack.org/horizon/ussuri/install/verify-rdo.html  
以下へアクセスしてみる.
[http://192.168.3.200/dashboard](http://192.168.3.200/dashboard)
```
http://192.168.3.200/auth/login/?next=/dashboard/
Not Found
The requested URL /auth/login/ was not found on this server.
```
