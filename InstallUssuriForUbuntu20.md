# Install OpenStack Ussuri on Ubuntu 20.04
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.3.200(192.168.3.200)  
設定ファイル: [URL](URL)
```
$ uname -a
Linux ryunosuke 5.4.0-40-generic #44-Ubuntu SMP Tue Jun 23 00:01:04 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
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


# Nova
https://docs.openstack.org/nova/ussuri/install/
## Install and configure controller node for Ubuntu
https://docs.openstack.org/nova/ussuri/install/controller-install-ubuntu.html
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
| id                  | 32b8e42f0e3c47408b7f6d7b5a95dee7 |
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
| id          | d9dae399acaf4cc38228dad82ed4a914 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute public http://192.168.3.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | cff2d0a1fe0f484f9593ed7b7f81f07e |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d9dae399acaf4cc38228dad82ed4a914 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.3.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute internal http://192.168.3.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0dc2324d711c43558d62a39607cc8f5d |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d9dae399acaf4cc38228dad82ed4a914 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.3.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute admin http://192.168.3.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | cc6194a958ff4b459d66eed81d8c305c |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d9dae399acaf4cc38228dad82ed4a914 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.3.200:8774/v2.1   |
+--------------+----------------------------------+
```

### Install and configure components
```
$ sudo apt -y install nova-api nova-conductor nova-novncproxy nova-scheduler

$ sudo cp -rafv /etc/nova /tmp
$ sudo vim /etc/nova/nova.conf
----
[DEFAULT]
...
+ transport_url = rabbit://openstack:password@192.168.3.200:5672/
+ my_ip = 192.168.3.200

...
[api_database]
- ##connection = sqlite:////var/lib/nova/nova_api.sqlite
+ connection = mysql+pymysql://nova:password@192.168.3.200/nova_api
...
[database]
- ##connection = sqlite:////var/lib/nova/nova.sqlite
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
328a17ce-1115-4395-b522-1c2c99d75d99
$ sudo -s /bin/sh -c "nova-manage db sync" nova
$ sudo -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
+-------+--------------------------------------+---------------------------------------------+----------------------------------------------------+----------+
|  Name |                 UUID                 |                Transport URL                |                Database Connection                 | Disabled |
+-------+--------------------------------------+---------------------------------------------+----------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                    none:/                   | mysql+pymysql://nova:****@192.168.3.200/nova_cell0 |  False   |
| cell1 | 328a17ce-1115-4395-b522-1c2c99d75d99 | rabbit://openstack:****@192.168.3.200:5672/ |    mysql+pymysql://nova:****@192.168.3.200/nova    |  False   |
+-------+--------------------------------------+---------------------------------------------+----------------------------------------------------+----------+
```
### Finalize installation
```
$ for i in "nova-api" "nova-scheduler" "nova-conductor" "nova-novncproxy"; do sudo systemctl restart ${i}; done
$ for i in "nova-api" "nova-scheduler" "nova-conductor" "nova-novncproxy"; do sudo systemctl status ${i}; done
```

## Install and configure a compute node
https://docs.openstack.org/nova/ussuri/install/compute-install.html
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
4
$ sudo vim /etc/nova/nova-compute.conf
----
...
[libvirt]
virt_type=kvm ・・・KVMであればOK.
----
$ sudo systemctl restart nova-compute
$ sudo systemctl status nova-compute
```

### Add the compute node to the cell database
```
$ source ~/admin-openrc
$ openstack compute service list --service nova-compute
+----+--------------+-----------+------+---------+-------+----------------------------+
| ID | Binary       | Host      | Zone | Status  | State | Updated At                 |
+----+--------------+-----------+------+---------+-------+----------------------------+
|  8 | nova-compute | ryunosuke | nova | enabled | up    | 2020-07-24T08:42:56.000000 |
+----+--------------+-----------+------+---------+-------+----------------------------+

$ sudo -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': 328a17ce-1115-4395-b522-1c2c99d75d99
Checking host mapping for compute host 'ryunosuke': b408731e-a777-48c3-827e-d7967b4d677c
Creating host mapping for compute host 'ryunosuke': b408731e-a777-48c3-827e-d7967b4d677c
Found 1 unmapped computes in cell: 328a17ce-1115-4395-b522-1c2c99d75d99
```

## Verify operation
https://docs.openstack.org/nova/ussuri/install/verify.html
```
$ source ~/admin-openrc
$ openstack compute service list
+----+----------------+-----------+----------+---------+-------+----------------------------+
| ID | Binary         | Host      | Zone     | Status  | State | Updated At                 |
+----+----------------+-----------+----------+---------+-------+----------------------------+
|  1 | nova-conductor | ryunosuke | internal | enabled | up    | 2020-07-24T08:45:48.000000 |
|  2 | nova-scheduler | ryunosuke | internal | enabled | up    | 2020-07-24T08:45:48.000000 |
|  8 | nova-compute   | ryunosuke | nova     | enabled | up    | 2020-07-24T08:45:46.000000 |
+----+----------------+-----------+----------+---------+-------+----------------------------+

$ openstack catalog list
+-----------+-----------+--------------------------------------------+
| Name      | Type      | Endpoints                                  |
+-----------+-----------+--------------------------------------------+
| placement | placement | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:8778         |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:8778      |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.3.200:8778        |
|           |           |                                            |
| glance    | image     | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:9292      |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.3.200:9292        |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:9292         |
|           |           |                                            |
| keystone  | identity  | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:5000/v3/     |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.3.200:5000/v3/    |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:5000/v3/  |
|           |           |                                            |
| nova      | compute   | RegionOne                                  |
|           |           |   internal: http://192.168.3.200:8774/v2.1 |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.3.200:8774/v2.1    |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.3.200:8774/v2.1   |
|           |           |                                            |
+-----------+-----------+--------------------------------------------+

$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| fdcafefe-c69e-4123-9966-718075127d80 | cirros | active |
+--------------------------------------+--------+--------+

$ sudo nova-status upgrade check
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

# Neutron
https://docs.openstack.org/neutron/ussuri/install/
## Install and configure for Ubuntu
https://docs.openstack.org/neutron/ussuri/install/install-ubuntu.html
### Install and configure controller node
https://docs.openstack.org/neutron/ussuri/install/controller-install-ubuntu.html
#### Prerequisites
```
$ sudo mysql
MariaDB [(none)]> CREATE DATABASE neutron;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';
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
| id                  | 32ae15e87fd148c8bdde73f8b820a45c |
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
| id          | eebd179a5836431da52a130840ff1867 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne network public http://192.168.3.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | b7242d2d2a4749a88163a4a9432f88e5 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | eebd179a5836431da52a130840ff1867 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.3.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network internal http://192.168.3.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | d6997820411d4de8a1df75c8fe3470dc |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | eebd179a5836431da52a130840ff1867 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.3.200:9696        |
+--------------+----------------------------------+
  
$ openstack endpoint create --region RegionOne network admin http://192.168.3.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 66f17111ac4241b5be203e03174bdf43 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | eebd179a5836431da52a130840ff1867 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.3.200:9696        |
+--------------+----------------------------------+
```

### Configure networking options
Networking Option 2: Self-service networks
https://docs.openstack.org/neutron/ussuri/install/controller-install-option2-ubuntu.html
#### Install the components
```
$ sudo apt -y install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```
#### Configure the server component
```
$ sudo cp -rafv /etc/neutron /tmp
$ sudo vim /etc/neutron/neutron.conf
----
[DEFAULT]
core_plugin = ml2
+ service_plugins = router
+ allow_overlapping_ips = true
+ transport_url = rabbit://openstack:password@192.168.3.200
+ auth_strategy = keystone
+ notify_nova_on_port_status_changes = true
+ notify_nova_on_port_data_changes = true
...
[database]
- ##connection = sqlite:////var/lib/neutron/neutron.sqlite
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
lock_path = /var/lib/neutron/tmp
----
```

#### Configure the Modular Layer 2 (ML2) plug-in
```
$ sudo vim /etc/neutron/plugins/ml2/ml2_conf.ini
----
...
[ml2]
+ type_drivers = flat,vlan,vxlan
+ tenant_network_types = vxlan
+ mechanism_drivers = linuxbridge,l2population
+ extension_drivers = port_security
[ml2_type_flat]
+ flat_networks = provider
[ml2_type_vxlan]
+ vni_ranges = 1:1000
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

$ sudo vim /etc/sysctl.conf
----
+ net.bridge.bridge-nf-call-iptables = 1
+ net.bridge.bridge-nf-call-ip6tables = 1
----
$ sudo sysctl -p
```
#### Configure the layer-3 agent¶
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
https://docs.openstack.org/neutron/ussuri/install/controller-install-ubuntu.html
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

$ for i in nova-api neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent; do sudo systemctl restart ${i}; done
```

## Verify operation
```
$ source ~/admin-openrc
$ openstack extension list --network
+-----------------------------------------------------------------------------------------------------------------------------------------------------
-----------+---------------------------------------+--------------------------------------------------------------------------------------------------
--------------------------------------------------------+
| Name                                                                                                                                                
           | Alias                                 | Description                                                                                      
                                                        |
+-----------------------------------------------------------------------------------------------------------------------------------------------------
-----------+---------------------------------------+--------------------------------------------------------------------------------------------------
--------------------------------------------------------+
| Address scope                                                                                                                                       
           | address-scope                         | Address scopes extension.    
...
```
## Networking Option 2: Self-service networks
```
$ source ~/admin-openrc 
$ openstack network agent list
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host      | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| 02adeabc-9f97-413c-b650-b6c183dd82cd | Linux bridge agent | ryunosuke | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 0bcf3857-0f3a-4180-9ac3-0c660edf2a21 | DHCP agent         | ryunosuke | nova              | :-)   | UP    | neutron-dhcp-agent        |
| 79bd0ab4-f79c-4fd1-8283-d6d9d6396e9e | L3 agent           | ryunosuke | nova              | :-)   | UP    | neutron-l3-agent          |
| 7b513fd2-f09c-4f1a-9bbc-2612aa235994 | Metadata agent     | ryunosuke | None              | :-)   | UP    | neutron-metadata-agent    |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
```

### Networking Option 2: Self-service networks
https://docs.openstack.org/neutron/ussuri/install/verify-option2.html
```
$ source ~/admin-openrc
$ openstack network agent list
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host      | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| 02adeabc-9f97-413c-b650-b6c183dd82cd | Linux bridge agent | ryunosuke | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 0bcf3857-0f3a-4180-9ac3-0c660edf2a21 | DHCP agent         | ryunosuke | nova              | :-)   | UP    | neutron-dhcp-agent        |
| 79bd0ab4-f79c-4fd1-8283-d6d9d6396e9e | L3 agent           | ryunosuke | nova              | :-)   | UP    | neutron-l3-agent          |
| 7b513fd2-f09c-4f1a-9bbc-2612aa235994 | Metadata agent     | ryunosuke | None              | :-)   | UP    | neutron-metadata-agent    |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
```

# Launch an instance
https://docs.openstack.org/install-guide/launch-instance.html
## Create virtual networks
Self-service network
https://docs.openstack.org/install-guide/launch-instance-networks-selfservice.html
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
| created_at                | 2020-07-25T15:07:56Z                                                                                                                                    |
| description               |                                                                                                                                                         |
| dns_domain                | None                                                                                                                                                    |
| id                        | 89c7593d-2bf5-4fa0-b605-a070d3d6f02b                                                                                                                    |
| ipv4_address_scope        | None                                                                                                                                                    |
| ipv6_address_scope        | None                                                                                                                                                    |
| is_default                | False                                                                                                                                                   |
| is_vlan_transparent       | None                                                                                                                                                    |
| location                  | cloud='', project.domain_id=, project.domain_name='default', project.id='859c92e29d48481dba0674de75b3b0dc', project.name='admin', region_name='', zone= |
| mtu                       | 1500                                                                                                                                                    |
| name                      | provider                                                                                                                                                |
| port_security_enabled     | True                                                                                                                                                    |
| project_id                | 859c92e29d48481dba0674de75b3b0dc                                                                                                                        |
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
| updated_at                | 2020-07-25T15:07:56Z                                                                                                                                    |
+---------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack subnet create --network provider --allocation-pool start=192.168.3.110,end=192.168.3.130 --dns-nameserver 192.168.3.220 --gateway 192.168.3.254 --subnet-range 192.168.3.0/24 provider
+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                | Value                                                                                                                                                   |
+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| allocation_pools     | 192.168.3.110-192.168.3.130                                                                                                                             |
| cidr                 | 192.168.3.0/24                                                                                                                                          |
| created_at           | 2020-07-25T15:09:53Z                                                                                                                                    |
| description          |                                                                                                                                                         |
| dns_nameservers      | 192.168.3.220                                                                                                                                           |
| dns_publish_fixed_ip | None                                                                                                                                                    |
| enable_dhcp          | True                                                                                                                                                    |
| gateway_ip           | 192.168.3.254                                                                                                                                           |
| host_routes          |                                                                                                                                                         |
| id                   | 622c7255-b466-44c4-a408-5ae4aa98abbf                                                                                                                    |
| ip_version           | 4                                                                                                                                                       |
| ipv6_address_mode    | None                                                                                                                                                    |
| ipv6_ra_mode         | None                                                                                                                                                    |
| location             | cloud='', project.domain_id=, project.domain_name='default', project.id='859c92e29d48481dba0674de75b3b0dc', project.name='admin', region_name='', zone= |
| name                 | provider                                                                                                                                                |
| network_id           | 89c7593d-2bf5-4fa0-b605-a070d3d6f02b                                                                                                                    |
| prefix_length        | None                                                                                                                                                    |
| project_id           | 859c92e29d48481dba0674de75b3b0dc                                                                                                                        |
| revision_number      | 0                                                                                                                                                       |
| segment_id           | None                                                                                                                                                    |
| service_types        |                                                                                                                                                         |
| subnetpool_id        | None                                                                                                                                                    |
| tags                 |                                                                                                                                                         |
| updated_at           | 2020-07-25T15:09:53Z                                                                                                                                    |
+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+


```
### Create the self-service network
```
$ source ~/demo-openrc
$ openstack network create selfservice
+---------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                     | Value                                                                                                                                                       |
+---------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up            | UP                                                                                                                                                          |
| availability_zone_hints   |                                                                                                                                                             |
| availability_zones        |                                                                                                                                                             |
| created_at                | 2020-07-25T15:01:38Z                                                                                                                                        |
| description               |                                                                                                                                                             |
| dns_domain                | None                                                                                                                                                        |
| id                        | 4397da2e-4002-4991-a238-0a38d08ddabf                                                                                                                        |
| ipv4_address_scope        | None                                                                                                                                                        |
| ipv6_address_scope        | None                                                                                                                                                        |
| is_default                | False                                                                                                                                                       |
| is_vlan_transparent       | None                                                                                                                                                        |
| location                  | cloud='', project.domain_id=, project.domain_name='default', project.id='2293c210ac4449c789c06f28529283db', project.name='myproject', region_name='', zone= |
| mtu                       | 1450                                                                                                                                                        |
| name                      | selfservice                                                                                                                                                 |
| port_security_enabled     | True                                                                                                                                                        |
| project_id                | 2293c210ac4449c789c06f28529283db                                                                                                                            |
| provider:network_type     | None                                                                                                                                                        |
| provider:physical_network | None                                                                                                                                                        |
| provider:segmentation_id  | None                                                                                                                                                        |
| qos_policy_id             | None                                                                                                                                                        |
| revision_number           | 1                                                                                                                                                           |
| router:external           | Internal                                                                                                                                                    |
| segments                  | None                                                                                                                                                        |
| shared                    | False                                                                                                                                                       |
| status                    | ACTIVE                                                                                                                                                      |
| subnets                   |                                                                                                                                                             |
| tags                      |                                                                                                                                                             |
| updated_at                | 2020-07-25T15:01:38Z                                                                                                                                        |
+---------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack subnet create --network selfservice --dns-nameserver 192.168.3.220 --gateway 10.0.0.1 --subnet-range 10.0.0.0/24 selfservice
+----------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                | Value                                                                                                                                                       |
+----------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------+
| allocation_pools     | 10.0.0.2-10.0.0.254                                                                                                                                         |
| cidr                 | 10.0.0.0/24                                                                                                                                                 |
| created_at           | 2020-07-25T15:04:50Z                                                                                                                                        |
| description          |                                                                                                                                                             |
| dns_nameservers      | 192.168.3.220                                                                                                                                               |
| dns_publish_fixed_ip | None                                                                                                                                                        |
| enable_dhcp          | True                                                                                                                                                        |
| gateway_ip           | 10.0.0.1                                                                                                                                                    |
| host_routes          |                                                                                                                                                             |
| id                   | 158e3875-afbb-42c9-9f53-bf2f4b40976a                                                                                                                        |
| ip_version           | 4                                                                                                                                                           |
| ipv6_address_mode    | None                                                                                                                                                        |
| ipv6_ra_mode         | None                                                                                                                                                        |
| location             | cloud='', project.domain_id=, project.domain_name='default', project.id='2293c210ac4449c789c06f28529283db', project.name='myproject', region_name='', zone= |
| name                 | selfservice                                                                                                                                                 |
| network_id           | 4397da2e-4002-4991-a238-0a38d08ddabf                                                                                                                        |
| prefix_length        | None                                                                                                                                                        |
| project_id           | 2293c210ac4449c789c06f28529283db                                                                                                                            |
| revision_number      | 0                                                                                                                                                           |
| segment_id           | None                                                                                                                                                        |
| service_types        |                                                                                                                                                             |
| subnetpool_id        | None                                                                                                                                                        |
| tags                 |                                                                                                                                                             |
| updated_at           | 2020-07-25T15:04:50Z                                                                                                                                        |
+----------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

### Create a router
```
$ source ~/demo-openrc
$ openstack router create router
+-------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                   | Value                                                                                                                                                       |
+-------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up          | UP                                                                                                                                                          |
| availability_zone_hints |                                                                                                                                                             |
| availability_zones      |                                                                                                                                                             |
| created_at              | 2020-07-25T15:05:56Z                                                                                                                                        |
| description             |                                                                                                                                                             |
| external_gateway_info   | null                                                                                                                                                        |
| flavor_id               | None                                                                                                                                                        |
| id                      | 8d5aa966-ee9b-48c8-b6d3-beb5a4c5a865                                                                                                                        |
| location                | cloud='', project.domain_id=, project.domain_name='default', project.id='2293c210ac4449c789c06f28529283db', project.name='myproject', region_name='', zone= |
| name                    | router                                                                                                                                                      |
| project_id              | 2293c210ac4449c789c06f28529283db                                                                                                                            |
| revision_number         | 1                                                                                                                                                           |
| routes                  |                                                                                                                                                             |
| status                  | ACTIVE                                                                                                                                                      |
| tags                    |                                                                                                                                                             |
| updated_at              | 2020-07-25T15:05:56Z                                                                                                                                        |
+-------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack router add subnet router selfservice
$ openstack router set router --external-gateway provider
Segmentation fault (core dumped)
→ 治るかと思ってRebootしたら、「VFS: Unable to mount root fs UUID=...」でKernelPanicを起こして治らず.
```
ToBe Continue
