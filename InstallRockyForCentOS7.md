# Install OpenStack Rocky for CentOS7
[OpenStack Installation Guide](https://docs.openstack.org/install-guide/)  
Ubuntu18.04版がSegmentation Failtエラーが頻繁に出て不安定なため、CentOS7で構築をしてみる.  
設定ファイル: [Rocky-InstallConfigsForCentOS7](https://github.com/ogalush/Rocky-InstallConfigsForCentOS7)
## 環境
```
[ogalush@ryunosuke ~]$ uname -n
ryunosuke.localdomain
[ogalush@ryunosuke ~]$ cat /etc/redhat-release 
CentOS Linux release 7.6.1810 (Core) 
[ogalush@ryunosuke ~]$ uname -a
Linux ryunosuke.localdomain 3.10.0-957.el7.x86_64 #1 SMP Thu Nov 8 23:39:32 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
[ogalush@ryunosuke ~]$ ip addr show |grep 192.168.0.200
    inet 192.168.0.200/24 brd 192.168.0.255 scope global noprefixroute enp3s0
[ogalush@ryunosuke ~]$ 
```

# Environment
## Configure network interfaces
[Doc](https://docs.openstack.org/install-guide/environment-networking-controller.html)
```
[ogalush@ryunosuke ~]$ grep -e 'DEVICE' -e 'TYPE' -e 'ONBOOT' -e 'BOOTPROTO' /etc/sysconfig/network-scripts/ifcfg-enp3s0 
TYPE=Ethernet
BOOTPROTO=none
DEVICE=enp3s0
ONBOOT=yes
[ogalush@ryunosuke ~]$
→ マニュアル通り.
```

## Hosts
```
$ sudo cp -pv /etc/hosts ~
$ sudo vim /etc/hosts
---
192.168.0.200 ryunosuke.localdomain ryunosuke
→ 追記
---

$ diff -u ~/hosts /etc/hosts
--- /home/ogalush/hosts 2013-06-07 23:31:32.000000000 +0900
+++ /etc/hosts  2019-02-24 16:56:09.282865680 +0900
@@ -1,2 +1,3 @@
 127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
 ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
+192.168.0.200 ryunosuke.localdomain ryunosuke
```

## NTP
```
[ogalush@ryunosuke ~]$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
-sv1.localdomain 133.243.238.244  2 u   21   64  377    3.536  -21.530   4.919
+s97.GchibaFL4.v 133.243.238.244  2 u   29   64  377    6.664  -26.797  16.482
+chobi.paina.net 203.178.138.38   2 u   27   64  377    9.825  -24.327   7.463
*ntp-5.jonlight. 133.243.238.243  2 u   33   64  377    3.349  -19.459   4.138
[ogalush@ryunosuke ~]$
→ すでにインストールしてあるのでOK.
```

## OpenStack packages for RHEL and CentOS
```
$ sudo yum -y install subscription-manager
$ sudo subscription-manager repos --enable=rhel-7-server-optional-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms
System certificates corrupted. Please reregister.
→ なぜかうまく言ってない.
$ sudo yum -y install centos-release-openstack-rocky
→ なぜかインストールはうまく言った.
----
Installed:
  centos-release-openstack-rocky.noarch 0:1-1.el7.centos                                                             

Dependency Installed:
  centos-release-ceph-luminous.noarch 0:1.1-2.el7.centos      centos-release-qemu-ev.noarch 0:1.0-4.el7.centos       
  centos-release-storage-common.noarch 0:2-2.el7.centos       centos-release-virt-common.noarch 0:1-1.el7.centos     

Complete!
[ogalush@ryunosuke ~]$
----

##$ sudo yum install https://rdoproject.org/repos/rdo-release.rpm
→ RDOを入れようとするとqueenが入るのでやめる.

$ sudo yum -y update
$ sudo yum -y install python-openstackclient
$ sudo yum -y install openstack-selinux
→ SELinuxがデフォルトで有効になってるので、よしなにやってくれるらしい.
```

## SQL database for RHEL and CentOS
[Doc](https://docs.openstack.org/install-guide/environment-sql-database-rdo.html)
```
$ sudo yum -y install mariadb mariadb-server python2-PyMySQL
$ sudo cp -rafv /etc/my.cnf* ~
$ cat << _EOS_ > ~/openstack.cnf
[mysqld]
bind-address = 192.168.0.200
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
_EOS_
$ sudo cp -v ~/openstack.cnf /etc/my.cnf.d
$ sudo systemctl enable mariadb.service
$ sudo systemctl start mariadb.service

$ sudo mysql_secure_installation
Enter current password for root (enter for none): 
OK, successfully used password, moving on...
Set root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

## Message queue for RHEL and CentOS
[Doc](https://docs.openstack.org/install-guide/environment-messaging-rdo.html)
```
$ sudo yum -y install rabbitmq-server
$ sudo systemctl enable rabbitmq-server.service
$ sudo systemctl start rabbitmq-server.service

$ sudo rabbitmqctl add_user openstack password
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

## Memcached for RHEL and CentOS
[Doc](https://docs.openstack.org/install-guide/environment-memcached-rdo.html)
```
$ sudo yum -y install memcached python-memcached
$ sudo cp -pv /etc/sysconfig/memcached ~
$ sudo sed -i 's/127.0.0.1,::1/192.168.0.200/g' /etc/sysconfig/memcached
$ diff -u ~/memcached /etc/sysconfig/memcached
--- /home/ogalush/memcached     2018-03-01 18:49:35.000000000 +0900
+++ /etc/sysconfig/memcached    2019-02-24 16:54:13.502753139 +0900
@@ -2,4 +2,4 @@
 USER="memcached"
 MAXCONN="1024"
 CACHESIZE="64"
-OPTIONS="-l 127.0.0.1,::1"
+OPTIONS="-l 192.168.0.200"
$

$ sudo systemctl enable memcached.service
$ sudo systemctl start memcached.service
```

## Etcd for RHEL and CentOS
[Doc](https://docs.openstack.org/install-guide/environment-etcd-rdo.html)
```
$ sudo yum -y install etcd
$ sudo cp -pv /etc/etcd/etcd.conf ~
$ sudo vim /etc/etcd/etcd.conf
$ diff -u ~/etcd.conf /etc/etcd/etcd.conf |egrep '^(\+|\-)'
--- /home/ogalush/etcd.conf     2019-02-14 01:56:03.000000000 +0900
+++ /etc/etcd/etcd.conf 2019-02-24 17:01:06.974445818 +0900
-ETCD_LISTEN_CLIENT_URLS="http://localhost:2379"
+ETCD_LISTEN_PEER_URLS="http://192.168.0.200:2380"
+ETCD_LISTEN_CLIENT_URLS="http://192.168.0.200:2379"
-ETCD_NAME="default"
+ETCD_NAME="ryunosuke"
-#ETCD_INITIAL_ADVERTISE_PEER_URLS="http://localhost:2380"
-ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"
+ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.200:2380"
+ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.200:2379"
-#ETCD_INITIAL_CLUSTER="default=http://localhost:2380"
-#ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
-#ETCD_INITIAL_CLUSTER_STATE="new"
+ETCD_INITIAL_CLUSTER="ryunosuke=http://192.168.0.200:2380"
+ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
+ETCD_INITIAL_CLUSTER_STATE="new"
$

$ sudo systemctl enable etcd
$ sudo systemctl start etcd
```

# Minimal deployment for Rocky
[Doc](https://docs.openstack.org/install-guide/openstack-services.html#minimal-deployment-for-rocky)

# Keystone Installation Tutorial for Red Hat Enterprise Linux and CentOS
[Doc](https://docs.openstack.org/keystone/rocky/install/index-rdo.html)

## Install and configure
[Doc](https://docs.openstack.org/keystone/rocky/install/keystone-install-rdo.html)
### Install Keystone
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;

$ sudo yum -y install openstack-keystone httpd mod_wsgi
$ sudo cp -rafv /etc/keystone ~
$ sudo vim /etc/keystone/keystone.conf 
----
[database]
connection = mysql+pymysql://keystone:password@192.168.0.200/keystone
...
[token]
provider = fernet
...
----

$ sudo bash -c "keystone-manage db_sync" keystone
$ sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage bootstrap --bootstrap-password password --bootstrap-admin-url http://192.168.0.200:5000/v3/ --bootstrap-internal-url http://192.168.0.200:5000/v3/ --bootstrap-public-url http://192.168.0.200:5000/v3/ --bootstrap-region-id RegionOne
```

### Configure the Apache HTTP server
```
$ sudo cp -rafv /etc/httpd/conf ~
$ sudo vim /etc/httpd/conf/httpd.conf
----
ServerName 192.168.0.200
----

$ sudo ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
$ ls -al /etc/httpd/conf.d/wsgi-keystone.conf 
lrwxrwxrwx 1 root root 38 Feb 24 17:15 /etc/httpd/conf.d/wsgi-keystone.conf -> /usr/share/keystone/wsgi-keystone.conf
$

$ sudo systemctl enable httpd.service
$ sudo systemctl start httpd.service

$ export OS_USERNAME=admin
$ export OS_PASSWORD=password
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=default
$ export OS_PROJECT_DOMAIN_NAME=default
$ export OS_AUTH_URL=http://192.168.0.200:5000/v3
$ export OS_IDENTITY_API_VERSION=3
```

## Create a domain, projects, users, and roles
[Doc](https://docs.openstack.org/keystone/rocky/install/keystone-users-rdo.html)
```
$ openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | 419a3c8a5e0744f4b672b1887bf849eb |
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
| id          | 6539ad025170426a930b8b9284e971f2 |
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
| id          | 4e50a5edae324c558727054db01e157d |
| is_domain   | False                            |
| name        | myproject                        |
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
| id                  | 05fcd425e107475c89eaccbf9ba2a376 |
| name                | myuser                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

$ openstack role create myrole
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 74deff2b2ccd429ba380620ad5cf757f |
| name      | myrole                           |
+-----------+----------------------------------+

$ openstack role add --project myproject --user myuser myrole
```

## Verify operation
[Doc](https://docs.openstack.org/keystone/rocky/install/keystone-verify-rdo.html)
```
$ unset OS_AUTH_URL OS_PASSWORD
$ openstack --os-auth-url http://192.168.0.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2019-02-24T09:22:08+0000                                                                                                                                                                |
| id         | gAAAAABcclQwdKkQhCfpUT6LJXr7Trxk3VryAG1sYFegy52i5H-utF471I0g-6lALR93mmcdCJe1W6-yl2l4WmWFzyB0zW4qnu-IRVUgT6L5oxmQKV5Jhp6iRE33CcPWA9_tIVVPryqr8ctMMAAUXp0npNtmpdaSdeGS3lw6Lo1XNWJOKd14lxk |
| project_id | c702086339374c4cb53e396b98ccd2e0                                                                                                                                                        |
| user_id    | d4cc0e07f1334266bf0df13941742614                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack --os-auth-url http://192.168.0.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name myproject --os-username myuser token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2019-02-24T09:23:07+0000                                                                                                                                                                |
| id         | gAAAAABcclRrzav0MSmw2OaHGvVP5aOYcLNZTuCQnFa5vpWN3iMYD2_szoqc74vwhW7HtfCSCc2IUoprZSjzN11U5k6bEU2UFVc_V9a1faNc2s1EFu3A_5K0KiUSjTMABx372UFIXKNJicxYfJ9OvY79ICMa2alyGgQWbTxwFroQ4zalE-BZBZk |
| project_id | 4e50a5edae324c558727054db01e157d                                                                                                                                                        |
| user_id    | 05fcd425e107475c89eaccbf9ba2a376                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

## Create OpenStack client environment scripts
[Doc](https://docs.openstack.org/keystone/rocky/install/keystone-openrc-rdo.html)

### Create Script
```
$ cat << _EOS_ > ~/admin-openrc.sh
#!/bin/bash

export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.0.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
_EOS_

$ chmod -v 700 ~/admin-openrc.sh
mode of ‘/home/ogalush/admin-openrc.sh’ changed from 0664 (rw-rw-r--) to 0700 (rwx------)

$ cat << _EOS_ > ~/demo-openrc.sh
#!/bin/bash

export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.0.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
_EOS_

$ chmod -v 700 ~/demo-openrc.sh
mode of ‘/home/ogalush/demo-openrc.sh’ changed from 0664 (rw-rw-r--) to 0700 (rwx------)
```

### Using the scripts
```
$ source ~/admin-openrc.sh
$ openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2019-02-24T09:27:39+0000                                                                                                                                                                |
| id         | gAAAAABcclV7Hbnphfz-8eAAtbZflShSSMYenxUYxTKMRHKO6bAN9HjQjs4mddVggFtykbo7UGDHr0LawYYsi-fto9kIgVhCjRCsJmQvsKxbswV6hUj8b6URaK9WYHxfspCgUgj8HSLSfSE98bKIJWPiMY5ctGKc-d3j6F9mIcRnAeVZkT7HZdA |
| project_id | c702086339374c4cb53e396b98ccd2e0                                                                                                                                                        |
| user_id    | d4cc0e07f1334266bf0df13941742614                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+


$ source ~/demo-openrc.sh 
$ openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2019-02-24T09:28:04+0000                                                                                                                                                                |
| id         | gAAAAABcclWUyvLM7ZJI20tJM336d68jQMIjDbKU4P8qBlwbjpWCKaZdw2l0bZXGaPPJu-HGCH7CKECTygXiMa5tDlpkB6KHo1AQkJnkospTIswQUz-SgVw8eQWjyKoepmx-WLej5jrdffDFTWMjiIAyf_sNj-kN7Ne9R9ESPVB3qyaCrsRgEYo |
| project_id | 4e50a5edae324c558727054db01e157d                                                                                                                                                        |
| user_id    | 05fcd425e107475c89eaccbf9ba2a376                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```


# Glance Installation
[Doc](https://docs.openstack.org/glance/rocky/install/)

## Install and configure (Red Hat)
[Doc](https://docs.openstack.org/glance/rocky/install/install-rdo.html)

### Prerequisites
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;

$ source ~/admin-openrc.sh
$ openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 670edf7e9da64e6bad0e1798b60fbeb5 |
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
| id          | 6b74ec3a32c540cfb6c372f51cbdd423 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne image public http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0b2b96fb79ad4146a6e818962ba9dffc |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 6b74ec3a32c540cfb6c372f51cbdd423 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image internal http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 173ce00ccfcb4fcf985bdf9751b056a3 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 6b74ec3a32c540cfb6c372f51cbdd423 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image admin http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | b173b71be5a24f18b851f0611ab5f8c5 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 6b74ec3a32c540cfb6c372f51cbdd423 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+  
```

### Install and configure components
```
$ sudo yum -y install openstack-glance
$ sudo cp -rafv /etc/glance ~
$ sudo vim /etc/glance/glance-api.conf
----
[database]
connection = mysql+pymysql://glance:password@192.168.0.200/glance
...
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
connection = mysql+pymysql://glance:password@192.168.0.200/glance
...
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
...
[paste_deploy]
flavor = keystone
----

$ sudo bash -c "glance-manage db_sync" glance
$ sudo chown -v glance:glance /var/log/glance/api.log
→権限不足でデーモンが落ちるので、変更する.
---
$ journalctl -xe |less
Feb 24 18:04:04 ryunosuke.localdomain glance-api[29085]: ERROR: [Errno 13] Permission denied: '/var/log/glance/api.log'
$ sudo ls -al /var/log/glance
total 12
drwxr-x---   2 glance glance   41 Feb 24 17:55 .
drwxr-xr-x. 14 root   root   4096 Feb 24 17:48 ..
-rw-r--r--   1 root   root    378 Feb 24 17:54 api.log
-rw-r--r--   1 glance glance 2660 Feb 24 17:55 registry.log
---

$ sudo systemctl enable openstack-glance-api.service openstack-glance-registry.service
$ sudo systemctl start openstack-glance-api.service openstack-glance-registry.service
```

### Verify operation
```
$ source ~/admin-openrc.sh
$ sudo yum -y install wget
$ wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
$ sudo mv -v cirros-0.4.0-x86_64-disk.img /usr/local/src
$ openstack image create "cirros" --file /usr/local/src/cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --contain
er-format bare --public
+------------------+-------------------------------------------------------------------------------------------------------------------
-------------------------------------------------------------------------+
| Field            | Value                                                                                                             
                                                                         |
+------------------+-------------------------------------------------------------------------------------------------------------------
-------------------------------------------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe                                                                                  
                                                                         |
| container_format | bare                                                                                                              
                                                                         |
| created_at       | 2019-02-24T09:12:47Z                                     
...
----

$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| c089d67b-d07f-4bf4-b70d-2d0c30d0ab20 | cirros | active |
+--------------------------------------+--------+--------+
```

# Compute service
[Doc](https://docs.openstack.org/nova/rocky/install/)

## Install and configure controller node for Red Hat Enterprise Linux and CentOS
[Doc](https://docs.openstack.org/nova/rocky/install/controller-install-rdo.html)

### Prerequisites
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

$ source ~/admin-openrc.sh
$ openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | d9847616bda74d83aa9857ae9e26f893 |
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
| id          | 3b637df84b6a48a18c309614779899ff |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute public http://192.168.0.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6d5a8eefb4834ba2ae26a50dd9583e86 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 3b637df84b6a48a18c309614779899ff |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.0.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute internal http://192.168.0.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 29d45a2b65644405b92db679c5aa9c79 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 3b637df84b6a48a18c309614779899ff |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.0.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute admin http://192.168.0.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | f829a35563f54ca3bbf49cb67629373a |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 3b637df84b6a48a18c309614779899ff |
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
| id                  | 421b5b40f697490a8b7c7f454f8dcae5 |
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
| id          | 7c34abce6fc3429e8f588a843e8cb7dd |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement public http://192.168.0.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | c6fe5922fb3d417e980c525a16243680 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 7c34abce6fc3429e8f588a843e8cb7dd |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.0.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement internal http://192.168.0.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 771605351f924f59a64d8cd4a0c07574 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 7c34abce6fc3429e8f588a843e8cb7dd |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.0.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement admin http://192.168.0.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 78c8bae73c2040ab9d104f6fa0c4fcdd |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 7c34abce6fc3429e8f588a843e8cb7dd |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.0.200:8778        |
+--------------+----------------------------------+
```

### Install and configure components
```
$ sudo yum -y install openstack-nova-api openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler openstack-nova-placement-api

$ sudo cp -rafv /etc/nova ~
$ sudo vim /etc/nova/nova.conf
----
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:password@192.168.0.200
my_ip = 192.168.0.200
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
...
[api_database]
connection = mysql+pymysql://nova:password@192.168.0.200/nova_api
...
[database]
connection = mysql+pymysql://nova:password@192.168.0.200/nova
...
[placement_database]
connection = mysql+pymysql://placement:password@192.168.0.200/placement
...
[api]
auth_strategy = keystone
...
[keystone_authtoken]
auth_url = http://192.168.0.200:5000/v3
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = password
...
[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip
keymap=ja
...
[glance]
api_servers = http://192.168.0.200:9292
...
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
...
[placement]
region_name = RegionOne
project_domain_name = default
project_name = service
auth_type = password
user_domain_name = default
auth_url = http://192.168.0.200:5000/v3
username = placement
password = password
----

$ sudo vim /etc/httpd/conf.d/00-nova-placement-api.conf
以下を追記
----
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
----
$ sudo apachectl configtest
$ sudo systemctl restart httpd

$ sudo bash -c "nova-manage api_db sync" nova
$ sudo bash -c "nova-manage cell_v2 map_cell0" nova
$ sudo bash -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
e687ffdf-f4f3-49f9-85a4-5e915dfd32ad
$ sudo bash -c "nova-manage db sync" nova
$ sudo bash -c "nova-manage cell_v2 list_cells" nova
+-------+--------------------------------------+---------------------------------------+----------------------------------------------------+----------+
|  Name |                 UUID                 |             Transport URL             |                Database Connection                 | Disabled |
+-------+--------------------------------------+---------------------------------------+----------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                 none:/                | mysql+pymysql://nova:****@192.168.0.200/nova_cell0 |  False   |
| cell1 | e687ffdf-f4f3-49f9-85a4-5e915dfd32ad | rabbit://openstack:****@192.168.0.200 |    mysql+pymysql://nova:****@192.168.0.200/nova    |  False   |
+-------+--------------------------------------+---------------------------------------+----------------------------------------------------+----------+

$ sudo systemctl enable openstack-nova-api.service openstack-nova-consoleauth openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
$ sudo systemctl start openstack-nova-api.service openstack-nova-consoleauth openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
```

## Install and configure a compute node for Red Hat Enterprise Linux and CentOS
Controller上でインスタンスを起動させたいのでまとめてインストールする.
[Doc](https://docs.openstack.org/nova/rocky/install/compute-install-rdo.html)
```
$ sudo yum -y install openstack-nova-compute
$ sudo vim /etc/nova/nova.conf
----
[vnc]
...
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html
...
[libvirt]
virt_type = kvm
...
[scheduler]
discover_hosts_in_cells_interval = 300
----

$ egrep -c '(vmx|svm)' /proc/cpuinfo
4
→ IntelVTサポートなので「virt_type = kvm」でOK.
$ sudo systemctl enable libvirtd.service openstack-nova-compute.service
$ sudo systemctl start libvirtd.service openstack-nova-compute.service

$ source ~/admin-openrc.sh
$ openstack compute service list --service nova-compute
+----+--------------+-----------------------+------+---------+-------+----------------------------+
| ID | Binary       | Host                  | Zone | Status  | State | Updated At                 |
+----+--------------+-----------------------+------+---------+-------+----------------------------+
|  9 | nova-compute | ryunosuke.localdomain | nova | enabled | up    | 2019-02-24T09:55:29.000000 |
+----+--------------+-----------------------+------+---------+-------+----------------------------+

$ sudo bash -c "nova-manage cell_v2 discover_hosts --verbose" nova
----
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': e687ffdf-f4f3-49f9-85a4-5e915dfd32ad
Checking host mapping for compute host 'ryunosuke.localdomain': de8b8617-5209-4de0-8b3b-c115480e2bbe
Creating host mapping for compute host 'ryunosuke.localdomain': de8b8617-5209-4de0-8b3b-c115480e2bbe
Found 1 unmapped computes in cell: e687ffdf-f4f3-49f9-85a4-5e915dfd32ad
----
```

### Verify operation
[Doc](https://docs.openstack.org/nova/rocky/install/verify.html)
```
$ source ~/admin-openrc.sh
$ openstack compute service list
+----+------------------+-----------------------+----------+---------+-------+----------------------------+
| ID | Binary           | Host                  | Zone     | Status  | State | Updated At                 |
+----+------------------+-----------------------+----------+---------+-------+----------------------------+
|  1 | nova-conductor   | ryunosuke.localdomain | internal | enabled | up    | 2019-02-24T09:58:06.000000 |
|  2 | nova-scheduler   | ryunosuke.localdomain | internal | enabled | up    | 2019-02-24T09:58:07.000000 |
|  5 | nova-consoleauth | ryunosuke.localdomain | internal | enabled | up    | 2019-02-24T09:58:07.000000 |
|  9 | nova-compute     | ryunosuke.localdomain | nova     | enabled | up    | 2019-02-24T09:58:09.000000 |
+----+------------------+-----------------------+----------+---------+-------+----------------------------+

[ogalush@ryunosuke ~]$ openstack catalog list
+-----------+-----------+--------------------------------------------+
| Name      | Type      | Endpoints                                  |
+-----------+-----------+--------------------------------------------+
| nova      | compute   | RegionOne                                  |
|           |           |   internal: http://192.168.0.200:8774/v2.1 |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.0.200:8774/v2.1   |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.0.200:8774/v2.1    |
|           |           |                                            |
...
| placement | placement | RegionOne                                  |
|           |           |   internal: http://192.168.0.200:8778      |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.0.200:8778         |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.0.200:8778        |
|           |           |                                            |
+-----------+-----------+--------------------------------------------+

$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| c089d67b-d07f-4bf4-b70d-2d0c30d0ab20 | cirros | active |
+--------------------------------------+--------+--------+

$ sudo nova-status upgrade check
+--------------------------------+
| Upgrade Check Results          |
+--------------------------------+
| Check: Cells v2                |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Placement API           |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Resource Providers      |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Ironic Flavor Migration |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: API Service Version     |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Request Spec Migration  |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Console Auths           |
| Result: Success                |
| Details: None                  |
+--------------------------------+
```

# Neutron Install and configure for Red Hat Enterprise Linux and CentOS
[Doc](https://docs.openstack.org/neutron/rocky/install/install-rdo.html)
## Install and configure controller node
[Doc](https://docs.openstack.org/neutron/rocky/install/controller-install-rdo.html)
### Prerequisites
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE neutron;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> quit;

$ source ~/admin-openrc.sh
$ openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | fed1a3f4b2ae419ba7328d04ff95c88b |
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
| id          | 81a1c607ed6e45749dc8c6f92885068d |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne network public http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 89777c33b20d42178562a2646e18b252 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 81a1c607ed6e45749dc8c6f92885068d |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network internal http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 2f7cb9a75cbe45dea999a8b3ee48f9f8 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 81a1c607ed6e45749dc8c6f92885068d |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network admin http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | fe257c0a8623480890fa4f883e17461a |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 81a1c607ed6e45749dc8c6f92885068d |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+
```

### Networking Option 2: Self-service networks
[Doc](https://docs.openstack.org/neutron/rocky/install/controller-install-option2-rdo.html)
```
$ sudo yum -y install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables

$ sudo cp -rafv /etc/neutron ~
$ sudo vim /etc/neutron/neutron.conf
----
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:password@192.168.0.200
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
...
[database]
connection = mysql+pymysql://neutron:password@192.168.0.200/neutron
...
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
...
[nova]
auth_url = http://192.168.0.200:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = password
...
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
----

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

$ sudo vim /etc/sysctl.d/99-neutron.conf
----
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
----

$ sudo sysctl --system
→ /etc/sysctl.d以下の更新は --systemオプションで反映する.
https://tsunokawa.hatenablog.com/entry/2015/09/02/175619

$ sudo vim /etc/neutron/l3_agent.ini
----
[DEFAULT]
interface_driver = linuxbridge
----

$ sudo vim /etc/neutron/dhcp_agent.ini
----
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
----
```

### Install and configure controller node (戻る)
Configure the metadata agent
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

$ sudo ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
$ ls -l /etc/neutron/plugin.ini
lrwxrwxrwx 1 root root 37 Feb 24 19:49 /etc/neutron/plugin.ini -> /etc/neutron/plugins/ml2/ml2_conf.ini

$ sudo bash -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

$ sudo systemctl restart openstack-nova-api.service

$ sudo systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service neutron-l3-agent.service
$ sudo systemctl start neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service neutron-l3-agent.service
```

## Install and configure compute node
[Doc](https://docs.openstack.org/neutron/rocky/install/compute-install-rdo.html)
```
$ sudo yum -y install openstack-neutron-linuxbridge ebtables ipset
→ Config類の設定はContorollerと同じ.
$ sudo systemctl restart openstack-nova-compute.service
$ sudo systemctl enable neutron-linuxbridge-agent.service
$ sudo systemctl start neutron-linuxbridge-agent.service
```

## Verify operation
```
$ source ~/admin-openrc.sh
$ openstack extension list --network
+--------------------------------------------------------------------------------------------------------------------------------------
---+--------------------------------+--------------------------------------------------------------------------------------------------
--------------------------------------------------------+
| Name                                                                                                                                 
   | Alias                          | Description                                                                                      
                                                        |
+--------------------------------------------------------------------------------------------------------------------------------------
---+--------------------------------+--------------------------------------------------------------------------------------------------
--------------------------------------------------------+
| Default Subnetpools                                                                                                                     | default-subnetpools            | Provides ability to mark and use a subnetpool as the default.                                                                                            |
| Availability Zone                                                                                                                       | availability_zone              | The availability zone extension.      
...
----

$ openstack network agent list
+--------------------------------------+--------------------+-----------------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host                  | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+-----------------------+-------------------+-------+-------+---------------------------+
| 11156a4f-e96b-4bf7-ac65-8615e55d0359 | Metadata agent     | ryunosuke.localdomain | None              | :-)   | UP    | neutron-metadata-agent    |
| 57657461-0ed0-4a7e-abf0-2566f062b789 | Linux bridge agent | ryunosuke.localdomain | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 84e2f8f8-2ab7-48e3-a13c-fdb60ea3eca7 | DHCP agent         | ryunosuke.localdomain | nova              | :-)   | UP    | neutron-dhcp-agent        |
| db50608b-954f-4e7a-aaf1-2024ea3bb6a3 | L3 agent           | ryunosuke.localdomain | nova              | :-)   | UP    | neutron-l3-agent          |
+--------------------------------------+--------------------+-----------------------+-------------------+-------+-------+---------------------------+
```

# horizon installation for Rocky
## Install and configure for Red Hat Enterprise Linux and CentOS
[Doc](https://docs.openstack.org/horizon/rocky/install/install-rdo.html)
```
$ sudo yum -y install openstack-dashboard
$ sudo cp -rafv /etc/openstack-dashboard ~
$ sudo vim /etc/openstack-dashboard/local_settings
----
OPENSTACK_HOST = "192.168.0.200"
ALLOWED_HOSTS = ['*']
...
##CACHES = {
##    'default': {
##        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
##    },
##}
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': '192.168.0.200:11211',
    }
}
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'default'
##OPENSTACK_KEYSTONE_DEFAULT_ROLE = "_member_"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
TIME_ZONE = "Asia/Tokyo"
----

$ sudo vim /etc/httpd/conf.d/openstack-dashboard.conf
----
...
WSGIApplicationGroup %{GLOBAL}
→ 追記する.
----

$ sudo systemctl restart httpd.service memcached.service
```

CentOS7の場合、Firewall接続とSELinuxへの接続許可を与える必要がある.
[参考](https://www.server-world.info/query?os=CentOS_7&p=openstack_pike&f=14)
```
$ sudo setsebool -P httpd_can_network_connect on 
setsebool:  SELinux is disabled.
$ sudo firewall-cmd --add-service={http,https} --permanent
→ Horizon用
$ sudo firewall-cmd --add-port=6080/tcp --permanent
→ VNC用
$ sudo firewall-cmd --reload 
success
$
```

## Verify operation for Red Hat Enterprise Linux and CentOS
[Horizon](http://192.168.0.200/dashboard) へアクセスできればOK.  
追加プラグインページはあとで見る.  
[Plugin Registry](https://docs.openstack.org/horizon/rocky/install/plugin-registry.html)

# Launch an instance
[Doc](https://docs.openstack.org/install-guide/launch-instance.html)

## Create virtual networks
### Provider Network
[Doc](https://docs.openstack.org/install-guide/launch-instance-networks-provider.html)
```
$ source ~/admin-openrc.sh
$ openstack network create  --share --external --provider-physical-network provider --provider-network-type flat provider
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2019-02-24T11:45:21Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 7b24db18-92cc-41e0-81a0-c1956f86c332 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | provider                             |
| port_security_enabled     | True                                 |
| project_id                | c702086339374c4cb53e396b98ccd2e0     |
| provider:network_type     | flat                                 |
| provider:physical_network | provider                             |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 0                                    |
| router:external           | External                             |
| segments                  | None                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2019-02-24T11:45:21Z                 |
+---------------------------+--------------------------------------+


$ openstack subnet create --network provider --allocation-pool start=192.168.0.100,end=192.168.0.130 --dns-nameserver 192.168.0.220 --gateway 192.168.0.254 --subnet-range 192.168.0.0/24 --no-dhcp provider
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 192.168.0.100-192.168.0.130          |
| cidr              | 192.168.0.0/24                       |
| created_at        | 2019-02-24T11:48:50Z                 |
| description       |                                      |
| dns_nameservers   | 192.168.0.220                        |
| enable_dhcp       | False                                |
| gateway_ip        | 192.168.0.254                        |
| host_routes       |                                      |
| id                | a49f67b6-992e-4c2a-8ae3-bef3ba7b32e8 |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | provider                             |
| network_id        | 7b24db18-92cc-41e0-81a0-c1956f86c332 |
| project_id        | c702086339374c4cb53e396b98ccd2e0     |
| revision_number   | 0                                    |
| segment_id        | None                                 |
| service_types     |                                      |
| subnetpool_id     | None                                 |
| tags              |                                      |
| updated_at        | 2019-02-24T11:48:50Z                 |
+-------------------+--------------------------------------+
```

### Self-service network
[Doc](https://docs.openstack.org/install-guide/launch-instance-networks-selfservice.html)
```
$ source ~/demo-openrc.sh 
$ openstack network create selfservice
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2019-02-24T11:50:39Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | cc76783d-5828-4cde-97df-04cde5c3c649 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1450                                 |
| name                      | selfservice                          |
| port_security_enabled     | True                                 |
| project_id                | 4e50a5edae324c558727054db01e157d     |
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
| updated_at                | 2019-02-24T11:50:39Z                 |
+---------------------------+--------------------------------------+

$ openstack subnet create --network selfservice --dns-nameserver 192.168.0.220 --gateway 10.0.0.1 --subnet-range 10.0.0.0/24 selfservice
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 10.0.0.2-10.0.0.254                  |
| cidr              | 10.0.0.0/24                          |
| created_at        | 2019-02-24T11:52:47Z                 |
| description       |                                      |
| dns_nameservers   | 192.168.0.220                        |
| enable_dhcp       | True                                 |
| gateway_ip        | 10.0.0.1                             |
| host_routes       |                                      |
| id                | ce5991b8-2910-4687-8f84-af363272b823 |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | selfservice                          |
| network_id        | cc76783d-5828-4cde-97df-04cde5c3c649 |
| project_id        | 4e50a5edae324c558727054db01e157d     |
| revision_number   | 0                                    |
| segment_id        | None                                 |
| service_types     |                                      |
| subnetpool_id     | None                                 |
| tags              |                                      |
| updated_at        | 2019-02-24T11:52:47Z                 |
+-------------------+--------------------------------------+

$ source ~/demo-openrc.sh
$ openstack router create router
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | UP                                   |
| availability_zone_hints |                                      |
| availability_zones      |                                      |
| created_at              | 2019-02-24T11:53:28Z                 |
| description             |                                      |
| distributed             | None                                 |
| external_gateway_info   | None                                 |
| flavor_id               | None                                 |
| ha                      | None                                 |
| id                      | 2573bc99-4097-42f0-8885-2bd763e7edf6 |
| name                    | router                               |
| project_id              | 4e50a5edae324c558727054db01e157d     |
| revision_number         | 0                                    |
| routes                  |                                      |
| status                  | ACTIVE                               |
| tags                    |                                      |
| updated_at              | 2019-02-24T11:53:28Z                 |
+-------------------------+--------------------------------------+

$ openstack router add subnet router selfservice
$ openstack router set router --external-gateway provider
```

### Verify operation
```
$ source ~/admin-openrc.sh
$ ip netns
qrouter-2573bc99-4097-42f0-8885-2bd763e7edf6 (id: 1)
qdhcp-cc76783d-5828-4cde-97df-04cde5c3c649 (id: 0)
$ openstack port list --router router
+--------------------------------------+------+-------------------+------------------------------------------------------------------------------+--------+
| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                           | Status |
+--------------------------------------+------+-------------------+------------------------------------------------------------------------------+--------+
| 15d98612-b233-4fe7-b634-0bb7044676ad |      | fa:16:3e:78:0a:d2 | ip_address='192.168.0.103', subnet_id='a49f67b6-992e-4c2a-8ae3-bef3ba7b32e8' | ACTIVE |
| 520ec710-03e4-4d9a-acc9-f961a4ae4a2b |      | fa:16:3e:d9:c9:d8 | ip_address='10.0.0.1', subnet_id='ce5991b8-2910-4687-8f84-af363272b823'      | ACTIVE |
+--------------------------------------+------+-------------------+------------------------------------------------------------------------------+--------+

]$ ping -c 3 192.168.0.103
PING 192.168.0.103 (192.168.0.103) 56(84) bytes of data.
64 bytes from 192.168.0.103: icmp_seq=1 ttl=64 time=0.097 ms
64 bytes from 192.168.0.103: icmp_seq=2 ttl=64 time=0.056 ms
64 bytes from 192.168.0.103: icmp_seq=3 ttl=64 time=0.056 ms

--- 192.168.0.103 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2000ms
rtt min/avg/max/mdev = 0.056/0.069/0.097/0.021 ms
$
```

### Create m1.nano flavor
```
$ source ~/admin-openrc.sh
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

$ openstack flavor create --id 1 --vcpus 2 --ram 4096 --disk 40 m1.small
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
| swap                       |          |
| vcpus                      | 2        |
+----------------------------+----------+
```

### Generate a key pair
```
$ source ~/demo-openrc.sh
$ openstack keypair create --public-key ~/.ssh/id_rsa_chef.pub chefkey
$ openstack keypair list
+---------+-------------------------------------------------+
| Name    | Fingerprint                                     |
+---------+-------------------------------------------------+
| chefkey | f2:70:ad:9c:93:32:14:c0:36:e3:4f:ac:19:45:82:d2 |
+---------+-------------------------------------------------+
```

### Add security group rules
```
$ openstack security group rule create --proto icmp default
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2019-02-24T12:01:21Z                 |
| description       |                                      |
| direction         | ingress                              |
| ether_type        | IPv4                                 |
| id                | f2689822-f5ac-48d4-be05-e361db09b2ad |
| name              | None                                 |
| port_range_max    | None                                 |
| port_range_min    | None                                 |
| project_id        | 4e50a5edae324c558727054db01e157d     |
| protocol          | icmp                                 |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 0                                    |
| security_group_id | f7a7729d-d974-4fa0-8849-484395fe2322 |
| updated_at        | 2019-02-24T12:01:21Z                 |
+-------------------+--------------------------------------+

$ openstack security group rule create --proto tcp --dst-port 22 default
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2019-02-24T12:01:42Z                 |
| description       |                                      |
| direction         | ingress                              |
| ether_type        | IPv4                                 |
| id                | 91430d90-7617-4dd7-a38d-4adf901922d4 |
| name              | None                                 |
| port_range_max    | 22                                   |
| port_range_min    | 22                                   |
| project_id        | 4e50a5edae324c558727054db01e157d     |
| protocol          | tcp                                  |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 0                                    |
| security_group_id | f7a7729d-d974-4fa0-8849-484395fe2322 |
| updated_at        | 2019-02-24T12:01:42Z                 |
+-------------------+--------------------------------------+
```

### 仮想インスタンス起動
OpenStack Dashboardからインスタンスを起動できればOK.

### OSイメージ追加
手順ではCirrosのみなので、CentOS7, Ubuntu18イメージを追加しておく.
```
$ source ~/admin-openrc.sh
$ openstack image create "ubuntu18.04" --file /usr/local/src/ubuntu-18.04-server-cloudimg-amd64.img --disk-format qcow2 --container-format bare --public
$ openstack image create "CentOS7" --file /usr/local/src/CentOS-7-x86_64-GenericCloud.qcow2 --disk-format qcow2 --container-format bare --public
$ openstack image list
+--------------------------------------+-------------+--------+
| ID                                   | Name        | Status |
+--------------------------------------+-------------+--------+
| 8014433c-e4ad-4247-b184-0b260e4f324f | CentOS7     | active |
| c089d67b-d07f-4bf4-b70d-2d0c30d0ab20 | cirros      | active |
| d0fd8505-56cf-48eb-b588-af0e5f44491e | ubuntu18.04 | active |
+--------------------------------------+-------------+--------+
```
