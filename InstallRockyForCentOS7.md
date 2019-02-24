# Install OpenStack Rocky for CentOS7
[OpenStack Installation Guide](https://docs.openstack.org/install-guide/)  
Ubuntu18.04版がSegmentation Failtエラーが頻繁に出て不安定なため、CentOS7で構築をしてみる.
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

$ cat << _EOS_ > ~/00-nova-placement-api.conf
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
_EOS_
$ sudo cp -v 00-nova-placement-api.conf /etc/httpd/conf.d/00-nova-placement-api.conf
$ sudo chmod -v 644 /etc/httpd/conf.d/00-nova-placement-api.conf
$ apachectl configtest
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
