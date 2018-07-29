# Install OpenStack Queens on Ubuntu 18.04
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/openstack-services.html)  
インストール先: 192.168.0.200(192.168.0.200)  
設定ファイル: [Queens-InstallConfigs](https://github.com/ogalush/Queens-InstallConfigs)
```
ogalush@ryunosuke:~$ uname -na
Linux ryunosuke 4.15.0-29-generic #31-Ubuntu SMP Tue Jul 17 15:39:52 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
ogalush@ryunosuke:~$
```

## Environment
### Network Time Protocol (NTP)
```
$ ntpq -p |grep '*'
*li1546-25.membe 10.84.87.146     2 u   44   64   17    4.045   -2.458   2.782
```
### NIC Settings
```
$ cat /etc/netplan/01-netcfg.yaml 
network:
    ethernets:
        enp3s0:
            dhcp4: no
            dhcp6: true
            addresses: [192.168.0.200/24]
            gateway4: 192.168.0.254
            nameservers:
              addresses: [192.168.0.254, 8.8.8.8, 8.8.4.4]
    version: 2
$
```

### hosts
```
$ sudo vim /etc/hosts
----
127.0.0.1       localhost
192.168.0.200       ryunosuke
##127.0.1.1     ryunosuke

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
----
```

### OpenStack packages
#### Enable the OpenStack repository
```
$ sudo apt install -y software-properties-common
```

#### Finalize the installation
```
$ sudo apt -y update && sudo apt -y upgrade && sudo apt -y dist-upgrade && sudo apt -y autoremove
$ sudo apt -y install python-openstackclient
$ sudo shutdown -r now
```

### SQL database for Ubuntu
#### Install and configure components
```
$ sudo apt -y install mariadb-server python-pymysql
$ sudo cp -pv /etc/mysql/mariadb.conf.d/50-server.cnf ~
$ sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf
----
[mysqld]
-bind-address           = 127.0.0.1
+bind-address           = 192.168.0.200

-character-set-server  = utf8mb4
-collation-server      = utf8mb4_general_ci
+character-set-server = utf8
+collation-server = utf8_general_ci
 → keystone db sync時の文字長エラー防止.

+default-storage-engine = innodb
+innodb_file_per_table = on
+max_connections = 4096
----
```

#### Finalize installation
```
$ sudo service mysql restart
$ sudo service mysql status |grep Active
   Active: active (running) since Sun 2018-07-29 19:57:26 JST; 4s ago
$ sudo mysql_secure_installation
----
Set root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
...
All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
----
```

### Message queue for Ubuntu
#### Install and configure components
```
$ sudo apt -y install rabbitmq-server
$ sudo rabbitmqctl add_user openstack password
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

### Memcached for Ubuntu
#### Install and configure components
```
$ sudo apt -y install memcached python-memcache
$ sudo cp -pv /etc/memcached.conf ~
$ sudo sed -i 's/-l 127.0.0.1/-l 192.168.0.200/' /etc/memcached.conf
$ grep '\-l' /etc/memcached.conf
-l 192.168.0.200
$ sudo service memcached restart
$ sudo service memcached status |grep Active
   Active: active (running) since Sun 2018-07-29 19:59:53 JST; 4s ago
$ sudo netstat -lnp |grep 11211
tcp        0      0 192.168.0.200:11211     0.0.0.0:*               LISTEN      5615/memcached      
```

### Etcd for Ubuntu
#### Install and configure components
```
$ sudo groupadd --system etcd
$ sudo useradd --home-dir "/var/lib/etcd" --system --shell /bin/false -g etcd etcd
$ sudo mkdir -pv /etc/etcd /var/lib/etcd
$ sudo chown -v etcd:etcd /etc/etcd /var/lib/etcd

$ ETCD_VER=v3.2.7
$ sudo rm -rfv /tmp/etcd && mkdir -pv /tmp/etcd
$ curl -L https://github.com/coreos/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
$ tar xvzf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd --strip-components=1
$ sudo cp -v /tmp/etcd/etcd /usr/bin/etcd
$ sudo cp -v /tmp/etcd/etcdctl /usr/bin/etcdctl
$ sudo vim /etc/etcd/etcd.conf.yml
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

※Etcd for Ubuntu in openstackinstallguide 
https://bugs.launchpad.net/openstack-manuals/+bug/1770984

$ sudo vim /lib/systemd/system/etcd.service
----
[Unit]
After=network.target
Description=etcd - highly-available key value store

[Service]
LimitNOFILE=65536
Restart=on-failure
Type=notify
ExecStart=/usr/bin/etcd --config-file /etc/etcd/etcd.conf.yml
User=etcd

[Install]
WantedBy=multi-user.target
----
```

#### Finalize installation
```
$ sudo systemctl enable etcd
$ sudo systemctl start etcd
$ sudo systemctl status etcd |grep Active
   Active: active (running) since Sun 2018-07-29 20:03:30 JST; 3s ago
```

## Keystone Installation Tutorial for Ubuntu
[Keystone Document](https://docs.openstack.org/keystone/queens/install/index-ubuntu.html)

### Prerequisites
```
$ sudo mysql
mysql> CREATE DATABASE keystone;
mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
mysql> FLUSH PRIVILEGES;
mysql> quit;
```

### Install and configure components
#### Install
```
$ sudo apt -y install keystone  apache2 libapache2-mod-wsgi
```

#### Configure
```
$ sudo vim /etc/keystone/keystone.conf
----
[database]
##connection = sqlite:////var/lib/keystone/keystone.db
connection = mysql+pymysql://keystone:password@192.168.0.200/keystone
...
[token]
provider = fernet
----

$ sudo bash -c "keystone-manage db_sync" keystone
$ sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
$ sudo keystone-manage bootstrap --bootstrap-password password --bootstrap-admin-url http://192.168.0.200:5000/v3/ --bootstrap-internal-url http://192.168.0.200:5000/v3/ --bootstrap-public-url http://192.168.0.200:5000/v3/ --bootstrap-region-id RegionOne

$ sudo vim /etc/apache2/apache2.conf
----
ServerName 192.168.0.200
----
$ sudo service apache2 restart
$ sudo service apache2 status |grep Active
   Active: active (running) since Sun 2018-07-29 20:07:45 JST; 4s ago

$ export OS_USERNAME=admin
$ export OS_PASSWORD=password
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=default
$ export OS_PROJECT_DOMAIN_NAME=default
$ export OS_AUTH_URL=http://192.168.0.200:5000/v3
$ export OS_IDENTITY_API_VERSION=3
```

### Create a domain, projects, users, and roles
[Document](https://docs.openstack.org/keystone/queens/install/keystone-users-ubuntu.html)
#### Create
```
$ openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | 33574bd3c28e4237af6509241a5e334e |
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
| id          | 8740e702e1114d469e0689631d1f7144 |
| is_domain   | False                            |
| name        | service                          |
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
| id          | a98599e472fb403385de764ba0725ba7 |
| is_domain   | False                            |
| name        | demo                             |
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
| id                  | 05104147cd024c0e96c9d66348c45673 |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

$ openstack role create user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 1db4466f4e454147a1a8ce2103139d47 |
| name      | user                             |
+-----------+----------------------------------+

$ openstack role add --project demo --user demo user
```

### Verify operation
```
$ unset OS_AUTH_URL OS_PASSWORD
$ openstack --os-auth-url http://192.168.0.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-07-29T12:10:48+0000                                                                                                                                                                |
| id         | gAAAAABbXaC4ST9U9s6m_04JAsoE6cymZCzmwcqwA6Bvpq5lVd8VVOGZ9399-UXvnoUcS63DX9Q-qxSnHwnLxpAju63b_b2QWGbu1n0NxQmB3tETOsWF_1Qb1uBgyka3wLZ4hACXdFamINdOV_8oG_zklpw3KBk08238UhWxFg10HkMXfhHkT-Q |
| project_id | f9bccfaa52c1467c8a1e43c2261f6570                                                                                                                                                        |
| user_id    | 7ae53fd57d894c3a8c0453902d355296                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack --os-auth-url http://192.168.0.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name demo --os-username demo token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-07-29T12:11:21+0000                                                                                                                                                                |
| id         | gAAAAABbXaDZXLwHqHx1ekEnZegzGKS1MvQwuAYflE8tpuc6R7Mpnh-rKieC1fF8eXji5-98Kv9c-JtQMvekvIrDyRiuw9nh1X5MuKYmJLb5snO3wN1G1k6vuxN9k51oabDiTf7UhGC1luj2wfIR5tbrkEJ-R34AqkcnrY6VERRvYXJFdLHdk58 |
| project_id | a98599e472fb403385de764ba0725ba7                                                                                                                                                        |
| user_id    | 05104147cd024c0e96c9d66348c45673                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

### Create OpenStack client environment scripts
#### Creating the scripts
```
$ cat > ~/admin-openrc << __EOF__
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.0.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
__EOF__

$ chmod -v 700 ~/admin-openrc

$ cat > ~/demo-openrc << __EOF__
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.0.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
__EOF__
$ chmod -v 700 ~/demo-openrc

$ source ~/admin-openrc
$ openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-07-29T12:12:30+0000                                                                                                                                                                |
| id         | gAAAAABbXaEeLF_GG0MJLtbfGMNOqB_N43SsEd2VuJRTHBzW13opgMsAZV_ORMvF44OLS6mvzCR2-FBBcZswFsqXHiiAX3QI3xeoR43OC4g-QxXkuR8lfID4qkdLXme-wWZZzExSvgKSxSBDjdX60bmDBdBinLw4-6NQk_m8doy9DJgbu3ZGCZA |
| project_id | f9bccfaa52c1467c8a1e43c2261f6570                                                                                                                                                        |
| user_id    | 7ae53fd57d894c3a8c0453902d355296                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ source ~/demo-openrc
$ openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-07-29T12:12:56+0000                                                                                                                                                                |
| id         | gAAAAABbXaE4K-PLtRrya_CN6FmhVMLtIEmTRqtCyvQU6m1YzFuQBMA4g9bycT8HnXVL6OtVetHc9nSfls8G5Z94QjPBTLagtVcczSiiQkbPAd9G3TgC80scJweD483c5GzGKKPZdGeSzKpbWWLrMRGs-EL36tKLlU3IRA87VXb43I3yQYnHF9A |
| project_id | a98599e472fb403385de764ba0725ba7                                                                                                                                                        |
| user_id    | 05104147cd024c0e96c9d66348c45673                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

## glance installation for Queens
[Document URL](https://docs.openstack.org/glance/queens/install/)

### Prerequisites
```
$ sudo mysql
mysql> CREATE DATABASE glance;
mysql> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
mysql> FLUSH PRIVILEGES;
mysql> quit;

$ source ~/admin-openrc
$ openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 8663cb4a11284e20bf94c6d7df88809c |
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
| id          | 892b7c8632224690ac911d3ff5f73246 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne image public http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | c8ea6ae24ebd498caa62f434acd8bd77 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 892b7c8632224690ac911d3ff5f73246 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image internal http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | d3395efd1cd14255851e77cb48a1cfd4 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 892b7c8632224690ac911d3ff5f73246 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image admin http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 9d2acb2565544970909363c8d258baa1 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 892b7c8632224690ac911d3ff5f73246 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+
```

### Install and configure
```
$ sudo apt -y install glance

$ sudo vim /etc/glance/glance-api.conf
----
[database]
connection = mysql+pymysql://glance:password@192.168.0.200/glance
##connection = sqlite:////var/lib/glance/glance.sqlite
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
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
##connection = sqlite:////var/lib/glance/glance.sqlite
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
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
$ sudo service glance-registry restart
$ sudo service glance-api restart
```

### Verify operation
```
$ source ~/admin-openrc
$ wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
$ openstack image create "cirros" --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe                     |
| container_format | bare                                                 |
| created_at       | 2018-07-29T11:20:27Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/e57a36b1-ea78-49ed-b477-71f0e827c330/file |
| id               | e57a36b1-ea78-49ed-b477-71f0e827c330                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | f9bccfaa52c1467c8a1e43c2261f6570                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 12716032                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2018-07-29T11:20:27Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+

$ openstack image create "Ubuntu16.04" --file /usr/local/src/ubuntu-16.04-server-cloudimg-amd64-disk1.img --disk-format qcow2 --container-format bare --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | dc8393f531122fcf9401650e8d2fd266                     |
| container_format | bare                                                 |
| created_at       | 2018-07-29T11:21:29Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/2cb9b464-a21a-4f5c-993e-9758d9466d53/file |
| id               | 2cb9b464-a21a-4f5c-993e-9758d9466d53                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | Ubuntu16.04                                          |
| owner            | f9bccfaa52c1467c8a1e43c2261f6570                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 292225024                                            |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2018-07-29T11:21:31Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+

$ openstack image list
+--------------------------------------+-------------+--------+
| ID                                   | Name        | Status |
+--------------------------------------+-------------+--------+
| 2cb9b464-a21a-4f5c-993e-9758d9466d53 | Ubuntu16.04 | active |
| e57a36b1-ea78-49ed-b477-71f0e827c330 | cirros      | active |
+--------------------------------------+-------------+--------+
```

### nova installation for Queens
[Nova Document](https://docs.openstack.org/nova/queens/install/)
#### Prerequisites
```
$ sudo mysql
mysql> CREATE DATABASE nova_api;
mysql> CREATE DATABASE nova;
mysql> CREATE DATABASE nova_cell0;
mysql> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'password';
mysql> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
mysql> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'password';
mysql> FLUSH PRIVILEGES;
mysql> quit;

$ source ~/admin-openrc
$ openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | c04bce92f7fa4165a0a28953345c8cf3 |
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
| id          | 4dae373f7c4d42f9a324fb0927416b2c |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute public http://192.168.0.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a0b9d160104c4764bb9c1cca77c35c24 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 4dae373f7c4d42f9a324fb0927416b2c |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.0.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute internal http://192.168.0.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6db39fc196f54416b0e52e3ca46f4a08 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 4dae373f7c4d42f9a324fb0927416b2c |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.0.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute admin http://192.168.0.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 7cf19a41788949b1b6391b4a22bfb901 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 4dae373f7c4d42f9a324fb0927416b2c |
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
| id                  | dd8bb2f19a7d42cd9d0469ff64994dc7 |
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
| id          | 42f73234ccd24cd4bc515aa5fc537035 |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement public http://192.168.0.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 901b52735ab14c9f98b063297fcf8266 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 42f73234ccd24cd4bc515aa5fc537035 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.0.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement internal http://192.168.0.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 525bd69092d141a2af8f60795f81125c |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 42f73234ccd24cd4bc515aa5fc537035 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.0.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement admin http://192.168.0.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e3806a6416fb4c06902d245d06dde704 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 42f73234ccd24cd4bc515aa5fc537035 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.0.200:8778        |
+--------------+----------------------------------+
```

#### Install and configure controller node
```
$ sudo apt -y install nova-api nova-conductor nova-consoleauth nova-novncproxy nova-scheduler nova-placement-api

$ sudo vim /etc/nova/nova.conf
----
[DEFAULT]
...
transport_url = rabbit://openstack:password@192.168.0.200
my_ip = 192.168.0.200
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
#connection = sqlite:////var/lib/nova/nova_api.sqlite
connection = mysql+pymysql://nova:password@192.168.0.200/nova_api
...
[database]
##connection = sqlite:////var/lib/nova/nova.sqlite
connection = mysql+pymysql://nova:password@192.168.0.200/nova
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
...
[glance]
api_servers = http://192.168.0.200:9292
...
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
...
[placement]
##os_region_name = openstack
os_region_name = RegionOne
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
2037b21d-d936-488a-8ada-4032ce6281d4

$ sudo bash -c "nova-manage db sync" nova
→ 1度segmentation faultが出たが、OS再起動後に再実施したら通った.
----
[sudo] password for ogalush:
/usr/lib/python2.7/dist-packages/pymysql/cursors.py:165: Warning: (1831, u'Duplicate index `uniq_instances0uuid`. This is deprecated and will be disallowed in a future release.')
  result = self._query(query)
/usr/lib/python2.7/dist-packages/pymysql/cursors.py:165: Warning: (1831, u'Duplicate index `block_device_mapping_instance_uuid_virtual_name_device_name_idx`. This is deprecated and will be disallowed in a future release.')
  result = self._query(query)
----

$ sudo nova-manage cell_v2 list_cells
+-------+--------------------------------------+---------------------------------------+----------------------------------------------------+
|  Name |                 UUID                 |             Transport URL             |                Database Connection                 |
+-------+--------------------------------------+---------------------------------------+----------------------------------------------------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                 none:/                | mysql+pymysql://nova:****@192.168.0.200/nova_cell0 |
| cell1 | 9224e550-418c-454b-93c8-7bf1fceaa069 | rabbit://openstack:****@192.168.0.200 |    mysql+pymysql://nova:****@192.168.0.200/nova    |
+-------+--------------------------------------+---------------------------------------+----------------------------------------------------+

$ for i in 'nova-api' 'nova-consoleauth' 'nova-scheduler' 'nova-conductor' 'nova-novncproxy' ; do sudo systemctl restart $i ; done
```

### Install and configure a compute node
#### Install and configure components
```
$ sudo apt -y install nova-compute
$ sudo vim /etc/nova/nova.conf
----
[vnc]
...
server_listen = 0.0.0.0
novncproxy_base_url = http://$my_ip:6080/vnc_auto.html
----

$ egrep -c '(vmx|svm)' /proc/cpuinfo
4
$ sudo vim /etc/nova/nova-compute.conf
----
[DEFAULT]
compute_driver=libvirt.LibvirtDriver
[libvirt]
virt_type=kvm
→ kvmのままにしておく.
----

$ sudo service nova-compute restart
```

#### Add the compute node to the cell database
```
$ source ~/admin-openrc
$ openstack compute service list --service nova-compute
→ 何故か表示が出てこない.

$ sudo bash -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': 9224e550-418c-454b-93c8-7bf1fceaa069
Found 0 unmapped computes in cell: 9224e550-418c-454b-93c8-7bf1fceaa069

$ sudo vim /etc/nova/nova.conf
----
...
[scheduler]
discover_hosts_in_cells_interval = 60
...
[placement]
os_region_name = RegionOne
project_domain_name = default
project_name = service
auth_type = password
user_domain_name = default
auth_url = http://192.168.0.200:5000/v3
username = placement
password = password
----

$ sudo service nova-compute restart
```

### Verify operation
```
$ source ~/admin-openrc
$ openstack compute service list 
+----+------------------+-----------+----------+---------+-------+------------+
| ID | Binary           | Host      | Zone     | Status  | State | Updated At |
+----+------------------+-----------+----------+---------+-------+------------+
|  3 | nova-consoleauth | ryunosuke | internal | enabled | up    | None       |
|  4 | nova-scheduler   | ryunosuke | internal | enabled | up    | None       |
|  5 | nova-conductor   | ryunosuke | internal | enabled | up    | None       |
+----+------------------+-----------+----------+---------+-------+------------+

$ openstack catalog list
+-----------+-----------+--------------------------------------------+
| Name      | Type      | Endpoints                                  |
+-----------+-----------+--------------------------------------------+
| keystone  | identity  | RegionOne                                  |
|           |           |   internal: http://192.168.0.200:5000/v3/  |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.0.200:5000/v3/     |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.0.200:5000/v3/    |
...
| glance    | image     | RegionOne                                  |
|           |           |   admin: http://192.168.0.200:9292         |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.0.200:9292        |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.0.200:9292      |
|           |           |                                            |
+-----------+-----------+--------------------------------------------+

$ openstack image list
+--------------------------------------+-------------+--------+
| ID                                   | Name        | Status |
+--------------------------------------+-------------+--------+
| 2cb9b464-a21a-4f5c-993e-9758d9466d53 | Ubuntu16.04 | active |
| e57a36b1-ea78-49ed-b477-71f0e827c330 | cirros      | active |
+--------------------------------------+-------------+--------+

$ sudo nova-status upgrade check
Option "os_region_name" from group "placement" is deprecated. Use option "region-name" from group "placement".
Option "enable" from group "cells" is deprecated for removal (Cells v1 is being replaced with Cells v2.).  Its val
ue may be silently ignored in the future.
+--------------------------------------------------------------------+
| Upgrade Check Results                                              |
+--------------------------------------------------------------------+
| Check: Cells v2                                                    |
| Result: Success                                                    |
| Details: No host mappings or compute nodes were found. Remember to |
|   run command 'nova-manage cell_v2 discover_hosts' when new        |
|   compute hosts are deployed.                                      |
...
+--------------------------------------------------------------------+
| Check: API Service Version                                         |
| Result: Success                                                    |
| Details: None                                                      |
+--------------------------------------------------------------------+
```

## neutron installation for Queens
[Neutron Document](https://docs.openstack.org/neutron/queens/install/)

### Prerequisites
```
$ sudo mysql
mysql> CREATE DATABASE neutron;
mysql> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';
mysql> FLUSH PRIVILEGES;
mysql> quit;

$ source ~/admin-openrc
$ openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 3f0240589a1144e88e4d3bfdddcab76c |
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
| id          | 21e2100c7e5045e3aebf3169cf449be7 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne network public http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6bed4bffb4924d0e9d236d66be88aed3 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 21e2100c7e5045e3aebf3169cf449be7 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network internal http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | aa028d40ee304ef38d4f923dcaf2763a |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 21e2100c7e5045e3aebf3169cf449be7 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network admin http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e0fcb3629ac1414e8af0dd09725839b5 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 21e2100c7e5045e3aebf3169cf449be7 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+
```

### Install and configure for Ubuntu
#### Networking Option 2: Self-service networks
```
$ sudo apt -y install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent

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
dns_domain = localdomain
...

[database]
##connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql+pymysql://neutron:password@192.168.0.200/neutron
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
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
```

#### Configure the Linux bridge agent
```
$ sudo vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
----
[linux_bridge]                                                                                                              physical_interface_mappings = provider:enp3s0
...
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
...
[vxlan]
enable_vxlan = true
local_ip = 192.168.0.200
l2_population = true
----

$ sudo vim /etc/sysctl.conf
----
...
## For OpenStack
net.ipv6.conf.all.disable_ipv6 = 1
net.bridge.bridge-nf-call-iptables = 1
#net.bridge.bridge-nf-call-ip6tables = 1
----
$ sudo sysctl -p

$ sudo vim /etc/neutron/l3_agent.ini
----
[DEFAULT]
interface_driver = linuxbridge
...
----

$ sudo vim /etc/neutron/dhcp_agent.ini
----
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
----
```

#### Networking Option 2: Self-service networks(続き)
```
$ sudo vim /etc/neutron/metadata_agent.ini
----
[DEFAULT]
nova_metadata_host = 192.168.0.200
metadata_proxy_shared_secret = password
...
----

$ sudo vim /etc/nova/nova.conf
----
...
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
...
----

$ sudo bash -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
$ for i in 'nova-api' 'neutron-server' 'neutron-linuxbridge-agent' 'neutron-dhcp-agent' 'neutron-metadata-agent' 'neutron-l3-agent' ; do sudo systemctl restart $i ; done
```

### Install and configure compute node
```
$ sudo apt -y install neutron-linuxbridge-agent
$ sudo service neutron-linuxbridge-agent restart
$ sudo service neutron-linuxbridge-agent status |grep Active
   Active: active (running) since Sun 2018-07-29 21:16:35 JST; 4s ago
→ Config変更に関してはControllerNodeと同じ内容であったので割愛.
```

### Verify operation
```
$ source ~/admin-openrc
$ openstack extension list --network
+----------------------------------------------------------------------------------------------+---------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Name                                                                                         | Alias                     | Description                                                                                                                                              |
+----------------------------------------------------------------------------------------------+---------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Default Subnetpools                                                                          | default-subnetpools       | Provides ability to mark and use a subnetpool as the default.                                                                                            |
| Availability Zone                                                                            | availability_zone         | The availability zone extension.                                                                                       
...
| project_id field enabled                                                                     | project-id                | Extension that indicates that project_id field is enabled.                                                                                               |
| Distributed Virtual Router                                                                   | dvr                       | Enables configuration of Distributed Virtual Routers.                                                                                                    |
+----------------------------------------------------------------------------------------------+---------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack network agent list

ogalush@ryunosuke:~$ 
→ 表示されるはずが表示されない.
```
