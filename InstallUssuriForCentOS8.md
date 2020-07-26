# Install OpenStack Ussuri on CentOS8
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.3.200  
設定ファイル: [URL](URL)
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
