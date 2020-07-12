# Install OpenStack Ussuri on Ubuntu 20.04
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.3.200(192.168.3.200)  
設定ファイル: [URL](URL)
```
ogalush@ryunosuke:~$ uname -a
Linux ryunosuke 5.4.0-40-generic #44-Ubuntu SMP Tue Jun 23 00:01:04 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
ogalush@ryunosuke:~$
```

# Networking
## Configure network interfaces
127.0.1.1 がホスト名に紐づいているので、実際のIPアドレスへ置換する.
```
$ sudo cp -pv /etc/hosts /tmp/hosts
'/etc/hosts' -> '/tmp/hosts'
$ sudo vim /etc/hosts
$ diff --unified=0 /tmp/hosts /etc/hosts
--- /tmp/hosts  2020-05-05 00:38:36.503499586 +0900
+++ /etc/hosts  2020-07-05 19:52:04.440981711 +0900
@@ -2 +2 @@
-127.0.1.1 ryunosuke
+192.168.3.200 ryunosuke
$
```

## Network Time Protocol (NTP)
時刻同期できているのでOK.
```
$ dpkg -l ntp
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name           Version                  Architecture Description
+++-==============-========================-============-=================================================
ii  ntp            1:4.2.8p12+dfsg-3ubuntu4 amd64        Network Time Protocol daemon and utility programs
$ ntpq -p |grep '*'
*ntp-a2.nict.go. .NICT.           1 u   52   64  177    4.558    1.035   2.728
$
```

# OpenStack packages for Ubuntu
cloud-archive:UssuriはUbuntu18.04用となるのでスキップ.
```
$ sudo add-apt-repository cloud-archive:ussuri
 Ubuntu Cloud Archive for OpenStack Ussuri
 More info: https://wiki.ubuntu.com/OpenStack/CloudArchive
Press [ENTER] to continue or Ctrl-c to cancel adding it.

cloud-archive for Ussuri only supported on bionic
```
[OpenStackのUssuriリリースがUbuntu 18.04 LTSと20.04 LTSで利用可能に](https://jp.ubuntu.com/blog/openstack%E3%81%AEussuri%E3%83%AA%E3%83%AA%E3%83%BC%E3%82%B9%E3%81%8Cubuntu-18-04-lts%E3%81%A820-04-lts%E3%81%A7%E5%88%A9%E7%94%A8%E5%8F%AF%E8%83%BD%E3%81%AB)を見ると利用できるはずだが.

デフォルトでussuriパッケージをインストールできる模様.
```
$ apt show python3-openstackclient | head -n 5

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

Package: python3-openstackclient
Version: 5.2.0-0ubuntu1
Priority: optional
Section: python
Source: python-openstackclient
$ 
```
[python-openstackclient 5.2.0-0ubuntu1 source package in Ubuntu](https://launchpad.net/ubuntu/+source/python-openstackclient/5.2.0-0ubuntu1)
```
python-openstackclient 5.2.0-0ubuntu1 source package in Ubuntu
Changelog
python-openstackclient (5.2.0-0ubuntu1) focal; urgency=medium
  * New upstream release for OpenStack Ussuri. ・・・★Ussuri用のリリースと書いてある.
  * d/control: Align (Build-)Depends with upstream.
 -- James Page <email address hidden>  Mon, 30 Mar 2020 15:59:21 +0100
```

インストール
```
$ sudo apt -y install python3-openstackclient
$ sudo apt -y update
$ sudo apt -y upgrade
$ sudo apt -y dist-upgrade
$ sudo apt -y autoremove
```

# SQL database for Ubuntu
https://docs.openstack.org/install-guide/environment-sql-database-ubuntu.html
## Install and configure components¶
```
$ sudo apt -y install mariadb-server python3-pymysql
$ sudo cp -rafv /etc/mysql ~
$ sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf 
-----
-bind-address            = 127.0.0.1
+bind-address            = 192.168.3.200
+default-storage-engine = innodb
+innodb_file_per_table = on
+max_connections = 4096

-character-set-server  = utf8mb4
-collation-server      = utf8mb4_general_ci
+character-set-server = utf8
+collation-server = utf8_general_ci
-----
```

## Finalize installation
```
$ sudo service mysql restart
$ sudo mysql_secure_installation
Enter current password for root (enter for none): [enter]
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
Adding user "openstack" ...
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
Setting permissions for user "openstack" in vhost "/" ...
```

# Memcached for Ubuntu
## Install and configure components
ubuntu20.04はpython2系を入れようとすると`E: Package 'python-memcache' has no installation candidate`となるため、  
hoge-python3fooでインストールしていく.
```
$ sudo apt -y install memcached python3-memcache
$ sudo cp -pv /etc/memcached.conf ~
'/etc/memcached.conf' -> '/home/ogalush/memcached.conf'
$ sudo sed -i 's/-l 127.0.0.1/-l 192.168.3.200/g' /etc/memcached.conf 
$ diff --unified=0 ~/memcached.conf /etc/memcached.conf 
--- /home/ogalush/memcached.conf        2020-07-05 20:28:00.418229945 +0900
+++ /etc/memcached.conf 2020-07-05 20:30:45.475942429 +0900
@@ -35 +35 @@
--l 127.0.0.1
+-l 192.168.3.200
$
```

## Finalize installation
```
$ sudo service memcached restart
$ sudo service memcached status
● memcached.service - memcached daemon
 Loaded: loaded (/lib/systemd/system/memcached.service; enabled; vendor preset: enabled)
 Active: active (running) since Sun 2020-07-05 20:31:50 JST; 3s ago
...
```

# Etcd for Ubuntu
## Install and configure components
```
$ sudo apt -y install etcd
$ sudo cp -pv /etc/default/etcd ~
'/etc/default/etcd' -> '/home/ogalush/etcd'
$ sudo vim /etc/default/etcd
-----
## etcd(1) daemon options
## See "/usr/share/doc/etcd-server/op-guide/configuration.md.gz"

### Member flags
ETCD_NAME="ryunosuke"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER="ryunosuke=http://192.168.3.200:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.3.200:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.3.200:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.3.200:2379"
-----
```

## Finalize installation
```
$ sudo systemctl enable etcd
Synchronizing state of etcd.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable etcd
$ sudo systemctl restart etcd
```

# Install OpenStack services
# Keystone Installation Tutorial for Ubuntu
https://docs.openstack.org/keystone/ussuri/install/index-ubuntu.html
## Install and configure
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> quit;
```

## Install and configure components
```
$ sudo apt -y install keystone
$ sudo cp -rafv /etc/keystone ~
$ sudo vim /etc/keystone/keystone.conf
----
[database]
- connection = sqlite:////var/lib/keystone/keystone.db
+ connection = mysql+pymysql://keystone:password@192.168.3.200/keystone

[token]
+ provider = fernet
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

## Finalize the installation¶
```
$ sudo service apache2 restart
$ export OS_USERNAME=admin
$ export OS_PASSWORD=password
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=default
$ export OS_PROJECT_DOMAIN_NAME=default
$ export OS_AUTH_URL=http://192.168.3.200:5000/v3
$ export OS_IDENTITY_API_VERSION=3
```

## Create a domain, projects, users, and roles
https://docs.openstack.org/keystone/ussuri/install/keystone-users-ubuntu.html
```
$ openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | adf276b21c454713af49ca460bbe5a0f |
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
| id          | 04d435f12be74783b884a829e66f208e |
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
| id          | 2293c210ac4449c789c06f28529283db |
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
| id                  | 2692597b24734a8fa49603afe095a9da |
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
| id          | d090dd95f75c4d939777cfaed97a7965 |
| name        | myrole                           |
| options     | {}                               |
+-------------+----------------------------------+

$ openstack role add --project myproject --user myuser myrole
```

## Verify operation
https://docs.openstack.org/keystone/ussuri/install/keystone-verify-ubuntu.html
```
$ unset OS_AUTH_URL OS_PASSWORD
~$ openstack --os-auth-url http://192.168.3.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-07-05T12:57:56+0000                                                                                                                                                                |
| id         | gAAAAABfAcBEXM9ALGs9DnkRHLMELo8... |
| project_id | 859c92e29d48481dba0674de75b3b0dc                                                                                                                                                        |
| user_id    | c47d505202a9491cbd012e6f387f5da5                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack --os-auth-url http://192.168.3.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name myproject --os-username myuser token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-07-05T12:58:49+0000                                                                                                                                                                |
| id         | gAAAAABfAcB5m8pnOjqhKUG_z5OWhaYe... |
| project_id | 2293c210ac4449c789c06f28529283db                                                                                                                                                        |
| user_id    | 2692597b24734a8fa49603afe095a9da                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

## Create OpenStack client environment scripts
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
~$ openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-07-05T13:04:01+0000                                                                                                                                                                |
| id         | gAAAAABfAcGxE8xGBi08G-p... |
| project_id | 859c92e29d48481dba0674de75b3b0dc                                                                                                                                                        |
| user_id    | c47d505202a9491cbd012e6f387f5da5                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ source ~/demo-openrc 
$ openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-07-05T13:04:24+0000                                                                                                                                                                |
| id         | gAAAAABfAcHIFIAY_H7a5uLVV8DtJVZX5VcWu2EsXgu5OiAub1... |
| project_id | 2293c210ac4449c789c06f28529283db                                                                                                                                                        |
| user_id    | 2692597b24734a8fa49603afe095a9da                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

# Glance Installation
https://docs.openstack.org/glance/ussuri/install/
## Install and configure (Ubuntu)
https://docs.openstack.org/glance/ussuri/install/install-ubuntu.html
### Prerequisites
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
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
| id                  | 67549dc682a5428aa9eed4e9309a81e3 |
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
| id          | 872c153f8ec44cd6859201d97a8949dd |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne image public http://192.168.3.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 80a49b4bc79a4ad9aea9159aba6c1f1d |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 872c153f8ec44cd6859201d97a8949dd |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.3.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image internal http://192.168.3.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 73615ac4b9fd4944bde28e61ad5e2a84 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 872c153f8ec44cd6859201d97a8949dd |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.3.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image admin http://192.168.3.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 830bf82101314b1683563d997c3d74fb |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 872c153f8ec44cd6859201d97a8949dd |
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
-----

$ sudo -s /bin/sh -c "glance-manage db_sync" glance
```
### Finalize installation
```
$ sudo service glance-api restart
```

## Verify operation
https://docs.openstack.org/glance/ussuri/install/verify.html
```
$ source ~/admin-openrc
$ wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
$ sudo mv -v cirros-0.4.0-x86_64-disk.img /usr/local/src
$ glance image-create --name "cirros" --file /usr/local/src/cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --visibility=public
+------------------+----------------------------------------------------------------------------------+
| Property         | Value                                                                            |
+------------------+----------------------------------------------------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe                                                 |
| container_format | bare                                                                             |
| created_at       | 2020-07-05T12:48:39Z                                                             |
| disk_format      | qcow2                                                                            |
| id               | fdcafefe-c69e-4123-9966-718075127d80                                             |
| min_disk         | 0                                                                                |
| min_ram          | 0                                                                                |
| name             | cirros                                                                           |
| os_hash_algo     | sha512                                                                           |
| os_hash_value    | 6513f21e44aa3da349f248188a44bc304a3653a04122d8fb4535423c8e1d14cd6a153f735bb0982e |
|                  | 2161b5b5186106570c17a9e58b64dd39390617cd5a350f78                                 |
| os_hidden        | False                                                                            |
| owner            | 859c92e29d48481dba0674de75b3b0dc                                                 |
| protected        | False                                                                            |
| size             | 12716032                                                                         |
| status           | active                                                                           |
| tags             | []                                                                               |
| updated_at       | 2020-07-05T12:48:39Z                                                             |
| virtual_size     | Not available                                                                    |
| visibility       | public                                                                           |
+------------------+----------------------------------------------------------------------------------+

$ glance image-list
+--------------------------------------+--------+
| ID                                   | Name   |
+--------------------------------------+--------+
| fdcafefe-c69e-4123-9966-718075127d80 | cirros |
+--------------------------------------+--------+
```


# placement installation for Ussuri
https://docs.openstack.org/placement/ussuri/install/

## Install and configure Placement for Ubuntu
https://docs.openstack.org/placement/ussuri/install/install-ubuntu.html  

### Create Database¶
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
| id                  | 8f27d609f7be46fa93947eb8c497418f |
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
| id          | 2fa1d6b1943643c0875cc3556d0d0662 |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement public http://192.168.3.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | f7cd61929bc946cd9b281ec467e33ac7 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2fa1d6b1943643c0875cc3556d0d0662 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.3.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement internal http://192.168.3.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | c091eea7c6bf4e0cac9b33653b2d1d4c |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2fa1d6b1943643c0875cc3556d0d0662 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.3.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement admin http://192.168.3.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 2f896aaef792410c901c45e59a601894 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2fa1d6b1943643c0875cc3556d0d0662 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.3.200:8778        |
+--------------+----------------------------------+
```

### Install and configure components
```
$ sudo apt -y install placement-api
$ sudo vim /etc/placement/placement.conf
-----
[placement_database]
+ connection = mysql+pymysql://placement:password@192.168.3.200/placement
- connection = sqlite:////var/lib/placement/placement.sqlite

[api]
+ auth_strategy = keystone

[keystone_authtoken]
+ auth_url = http://192.168.3.200:5000/v3
+ memcached_servers = 192.168.3.200:11211
+ auth_type = password
+ project_domain_name = default
+ user_domain_name = default
+ project_name = service
+ username = placement
+ password = password
-----

$ sudo -s /bin/sh -c "placement-manage db sync" placement
$ sudo service apache2 restart
$ sudo service apache2 status
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/lib/systemd/system/apache2.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2020-07-13 00:26:25 JST; 2s ago
       Docs: https://httpd.apache.org/docs/2.4/
    Process: 4640 ExecStart=/usr/sbin/apachectl start (code=exited, status=0/SUCCESS)
```

## Verify Installation
https://docs.openstack.org/placement/ussuri/install/verify.html
```
$ source ~/admin-openrc
$ placement-status upgrade check
$ sudo placement-status upgrade check
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

$ sudo apt -y install python3-pip
$ sudo pip3 install osc-placement
$ openstack --os-placement-api-version 1.2 resource class list --sort-column name
+----------------------------+
| name                       |
+----------------------------+
| DISK_GB                    |
| FPGA                       |
| IPV4_ADDRESS               |
| MEMORY_MB                  |
| MEM_ENCRYPTION_CONTEXT     |
| NET_BW_EGR_KILOBIT_PER_SEC |
| NET_BW_IGR_KILOBIT_PER_SEC |
| NUMA_CORE                  |
| NUMA_MEMORY_MB             |
| NUMA_SOCKET                |
| NUMA_THREAD                |
| PCI_DEVICE                 |
| PCPU                       |
| PGPU                       |
| SRIOV_NET_VF               |
| VCPU                       |
| VGPU                       |
| VGPU_DISPLAY_HEAD          |
+----------------------------+

$ openstack --os-placement-api-version 1.6 trait list --sort-column name
+---------------------------------------+
| name                                  |
+---------------------------------------+
| COMPUTE_ACCELERATORS                  |
| COMPUTE_DEVICE_TAGGING                |
| COMPUTE_GRAPHICS_MODEL_CIRRUS         |
| COMPUTE_GRAPHICS_MODEL_GOP            |
| COMPUTE_GRAPHICS_MODEL_NONE           |
| COMPUTE_GRAPHICS_MODEL_QXL            |
| COMPUTE_GRAPHICS_MODEL_VGA            |
...
| HW_NUMA_ROOT                          |
| MISC_SHARES_VIA_AGGREGATE             |
| STORAGE_DISK_HDD                      |
| STORAGE_DISK_SSD                      |
+---------------------------------------+
```
To Be Continue.
