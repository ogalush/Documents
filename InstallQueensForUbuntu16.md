# Install OpenStack Queens on Ubuntu 16.04
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/openstack-services.html)  
インストール先: 192.168.0.200(192.168.0.200)  
設定ファイル: [Queens-InstallConfigs](https://github.com/ogalush/Queens-InstallConfigs)
```
ogalush@ryunosuke:~$ uname -a
Linux ryunosuke 4.4.0-128-generic #154-Ubuntu SMP Fri May 25 14:15:18 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
ogalush@ryunosuke:~$
```

## Environment
### Network Time Protocol (NTP)
```
$ ntpq -p |grep '*'
*li1546-25.membe 10.84.87.146     2 u   44   64   17    4.045   -2.458   2.782
```

### OpenStack packages
#### Enable the OpenStack repository
```
$ sudo apt install -y software-properties-common
$ sudo add-apt-repository cloud-archive:queens
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
   Active: active (running) since Sun 2018-06-17 18:08:12 JST; 9s ago

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
   Active: active (running) since Sun 2018-06-17 18:13:54 JST; 4s ago
$ sudo netstat -lnp |grep 11211
tcp        0      0 192.168.0.200:11211     0.0.0.0:*               LISTEN      6372/memcached
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
$ tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd --strip-components=1
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
   Active: active (running) since Sun 2018-06-17 19:38:10 JST; 7s ago
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
   Active: active (running) since Sun 2018-06-17 19:50:52 JST; 6s ago

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
| id          | 4051db2e4993425a8d890c9fb7526383 |
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
| id          | 654660630bb6404d98f376f3da2b49e7 |
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
| id          | dc86d2e95fef4f56a70021e055f0c2cf |
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
| id                  | 2ca851b69cbe4ed5b328f6f17b373405 |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

$ openstack role create user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 39185f3995484cfdacdc70e04db6d3a7 |
| name      | user                             |
+-----------+----------------------------------+

$ openstack role add --project demo --user demo user
```

### Verify operation
```
$ unset OS_AUTH_URL OS_PASSWORD
$ openstack --os-auth-url http://192.168.0.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-06-17T11:56:27+0000                                                                                                                                                                |
| id         | gAAAAABbJj5bphIWrGNEi1fE8hyWtoKrGQWQE6-6fm0hM5jbum5JR9lfylh_15A5cmrVX81yOoKaJCvNH7or6YY4vJgvDsm6YOhcBb6j0KaZBovlw9GCMkCpZ8LHBB50JzpY3C7YOtClVWhzZASx6BBFETKpox8sZmK6vP2MHEd5plKvltltFac |
| project_id | a77a45043cc943c398841d12d5ddfa05                                                                                                                                                        |
| user_id    | 2bf6f2a4ae7e4a7cb59be2ffb7eb5815                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ openstack --os-auth-url http://192.168.0.200:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name demo --os-username demo token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-06-17T11:57:14+0000                                                                                                                                                                |
| id         | gAAAAABbJj6KnkDL_9xOCHUvc82lmuwVxqGCd0dbBjLF_Goz_wIPLZHvZpnfob7fH4N6Y2-xiuivoVPpr64s4y5saC8EjQjv7nuSEWuebqP8wTZUN1JxLE08klplSdEemwl_MGRxy8GvrFw1244UID4ONUnP30CXUftqCAmr5gTeGvIIl05ps7M |
| project_id | dc86d2e95fef4f56a70021e055f0c2cf                                                                                                                                                        |
| user_id    | 2ca851b69cbe4ed5b328f6f17b373405                                                                                                                                                        |
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
| expires    | 2018-06-17T12:01:02+0000                                                                                                                                                                |
| id         | gAAAAABbJj9uBa2mNBB4QqTxdUHo9ADIiRYCdwnb4x6DKZGjxpA3rDVatCS7VqW_ZBRhP-ugGh4GuUbSQUDwAqItwaYan4HSqB3gzjKrxhyXGo6KLLk4vvKLJAKqQIfM0VDJyhvUhvuf1TJODuBP4n-3KM8WiMivgw7f6nkf8e87ZfGIhjNCMJ4 |
| project_id | a77a45043cc943c398841d12d5ddfa05                                                                                                                                                        |
| user_id    | 2bf6f2a4ae7e4a7cb59be2ffb7eb5815                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

$ source ~/demo-openrc
$ openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-06-17T12:01:27+0000                                                                                                                                                                |
| id         | gAAAAABbJj-HJlaWB9NNsTMtpl9oIcZhqa-UVr3Pgvj-gatC9bcn9Hgsl8F3_LmcSdLlbIQkwkFYasdNeyWwCh1NswVTyqrSFq7aon1vO1Bvw48iuqrt-k0buqcHMo9x4zrb2gZEoNsdkMR-e6rGPm4xT-0J8GQL6WDN_0ufcA10V54WTPynE2E |
| project_id | dc86d2e95fef4f56a70021e055f0c2cf                                                                                                                                                        |
| user_id    | 2ca851b69cbe4ed5b328f6f17b373405                                                                                                                                                        |
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
| id                  | 0a0233491ef04ca58de2c590461f29b1 |
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
| id          | a420bb01c22045c692a1d686c5427ec5 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne image public http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 135c73eb4d284b5093d3273b09d18d4f |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a420bb01c22045c692a1d686c5427ec5 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image internal http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 15a24b4b60d24218bf5e20c2de333388 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a420bb01c22045c692a1d686c5427ec5 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.0.200:9292        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne image admin http://192.168.0.200:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 8d302ddc6dbf41e5b57ff77d3c012f62 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a420bb01c22045c692a1d686c5427ec5 |
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
| created_at       | 2018-06-17T11:16:20Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/f3e81692-4e7e-4167-a5d0-c683264ed265/file |
| id               | f3e81692-4e7e-4167-a5d0-c683264ed265                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | a77a45043cc943c398841d12d5ddfa05                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 12716032                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2018-06-17T11:16:20Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+

$ openstack image create "Ubuntu16.04" --file /usr/local/src/ubuntu-16.04-server-cloudimg-amd64-disk1.img --disk-format qcow2 --container-format bare --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | 027b3e9d219f0f6c17b5448ed67dc41e                     |
| container_format | bare                                                 |
| created_at       | 2018-06-23T12:29:30Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/bf5b90d4-b2ee-4ddd-8847-391e9d7e3d95/file |
| id               | bf5b90d4-b2ee-4ddd-8847-391e9d7e3d95                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | Ubuntu16.04                                          |
| owner            | 6e4c9ccbb5a24aa097a73b2ca4fe1349                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 289931264                                            |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2018-06-23T12:29:31Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+

$ openstack image list
+--------------------------------------+-------------+--------+
| ID                                   | Name        | Status |
+--------------------------------------+-------------+--------+
| bf5b90d4-b2ee-4ddd-8847-391e9d7e3d95 | Ubuntu16.04 | active |
| f3e81692-4e7e-4167-a5d0-c683264ed265 | cirros      | active |
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
| id                  | a1ff0d22c58345f588559482a24b3196 |
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
| id          | f317c0ace80a40388fefd3b14fef84cc |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute public http://192.168.0.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 91d421cfe7094b85a66333c61e475553 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f317c0ace80a40388fefd3b14fef84cc |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.0.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute internal http://192.168.0.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | b8272155cb5943f49bc97d5345df21bb |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f317c0ace80a40388fefd3b14fef84cc |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.0.200:8774/v2.1   |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne compute admin http://192.168.0.200:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | ac1241294b0b41bb884c5807bd2132dc |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f317c0ace80a40388fefd3b14fef84cc |
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
| id                  | b60d4171a63a428ebdb977f703ac188c |
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
| id          | bbe57b6e976a4605bf0f38f388fc4a83 |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement public http://192.168.0.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 2126372c662b48c7a161628ad69e0f56 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | bbe57b6e976a4605bf0f38f388fc4a83 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.0.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement internal http://192.168.0.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e1d5d91068a74352b82522c7a8e71d48 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | bbe57b6e976a4605bf0f38f388fc4a83 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.0.200:8778        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement admin http://192.168.0.200:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 8dad677c16ba41968c73e89474a8d319 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | bbe57b6e976a4605bf0f38f388fc4a83 |
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
464f0c50-1df5-4ca5-beef-684b1d6e401e

$ sudo bash -c "nova-manage db sync" nova
$ sudo nova-manage cell_v2 list_cells
+-------+--------------------------------------+---------------------------------------+----------------------------------------------------+
|  Name |                 UUID                 |             Transport URL             |                Database Connection                 |
+-------+--------------------------------------+---------------------------------------+----------------------------------------------------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                 none:/                | mysql+pymysql://nova:****@192.168.0.200/nova_cell0 |
| cell1 | 464f0c50-1df5-4ca5-beef-684b1d6e401e | rabbit://openstack:****@192.168.0.200 |    mysql+pymysql://nova:****@192.168.0.200/nova    |
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
...
[libvirt]
virt_type=kvm
→ kvmのまま.
----

$ sudo service nova-compute restart
```

#### Add the compute node to the cell database
```
$ source ~/admin-openrc
$ openstack compute service list --service nova-compute
+----+--------------+-----------+------+---------+-------+----------------------------+
| ID | Binary       | Host      | Zone | Status  | State | Updated At                 |
+----+--------------+-----------+------+---------+-------+----------------------------+
|  7 | nova-compute | ryunosuke | nova | enabled | up    | 2018-06-17T11:42:25.000000 |
+----+--------------+-----------+------+---------+-------+----------------------------+

$ sudo bash -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': 464f0c50-1df5-4ca5-beef-684b1d6e401e
Checking host mapping for compute host 'ryunosuke': 55cfb848-fc62-42b5-a335-2bb192e06cd4
Creating host mapping for compute host 'ryunosuke': 55cfb848-fc62-42b5-a335-2bb192e06cd4
Found 1 unmapped computes in cell: 464f0c50-1df5-4ca5-beef-684b1d6e401e

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
+----+------------------+-----------+----------+---------+-------+----------------------------+
| ID | Binary           | Host      | Zone     | Status  | State | Updated At                 |
+----+------------------+-----------+----------+---------+-------+----------------------------+
|  1 | nova-scheduler   | ryunosuke | internal | enabled | up    | 2018-06-17T11:52:11.000000 |
|  2 | nova-consoleauth | ryunosuke | internal | enabled | up    | 2018-06-17T11:52:10.000000 |
|  5 | nova-conductor   | ryunosuke | internal | enabled | up    | 2018-06-17T11:52:12.000000 |
|  7 | nova-compute     | ryunosuke | nova     | enabled | up    | 2018-06-17T11:52:04.000000 |
+----+------------------+-----------+----------+---------+-------+----------------------------+

$ openstack catalog list
+-----------+-----------+--------------------------------------------+                                                      
| Name      | Type      | Endpoints                                  |                                                      
+-----------+-----------+--------------------------------------------+                                                      
| glance    | image     | RegionOne                                  |                                                      
|           |           |   public: http://192.168.0.200:9292        |
|           |           | RegionOne                                  |                                                      |           |           |   internal: http://192.168.0.200:9292      |                                                      |           |           | RegionOne                                  |                                                      |           |           |   admin: http://192.168.0.200:9292         |
...
| nova      | compute   | RegionOne                                  |                                                      |           |           |   public: http://192.168.0.200:8774/v2.1   |                                                      |           |           | RegionOne                                  |                                                      |           |           |   admin: http://192.168.0.200:8774/v2.1    |                                                      |           |           | RegionOne                                  |                                                      |           |           |   internal: http://192.168.0.200:8774/v2.1 |                                                      |           |           |                                            |                                                      
+-----------+-----------+--------------------------------------------+

$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| f3e81692-4e7e-4167-a5d0-c683264ed265 | cirros | active |
+--------------------------------------+--------+--------+

$ sudo nova-status upgrade check
Option "os_region_name" from group "placement" is deprecated. Use option "region-name" from group "placement".
+---------------------------+
| Upgrade Check Results     |
+---------------------------+
| Check: Cells v2           |
| Result: Success           |
| Details: None             |
+---------------------------+
| Check: Placement API      |
| Result: Success           |
| Details: None             |
+---------------------------+
| Check: Resource Providers |
| Result: Success           |
| Details: None             |
+---------------------------+
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
| id                  | 62b9043451b04961b0d742a4664339ed |
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
| id          | 5c024bf352314c13a03202573020e1a1 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

$ openstack endpoint create --region RegionOne network public http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 60110c15aa0d4d6ca1dd0369446921c9 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 5c024bf352314c13a03202573020e1a1 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network internal http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | cee38df48b6a4df1a37871408f00a352 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 5c024bf352314c13a03202573020e1a1 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.0.200:9696        |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne network admin http://192.168.0.200:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 97d4f0635e8a4e4d85fec0690316b90f |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 5c024bf352314c13a03202573020e1a1 |
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
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
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
   Active: active (running) since Sun 2018-06-17 21:19:29 JST; 4s ago
→ Config変更に関してはControllerNodeと同じ内容であったので割愛.
```

### Verify operation
```
$ source ~/admin-openrc
$ openstack extension list --network
+----------------------------------------------------------------------------------------------+---------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Name                                                                                         | Alias                     | Description                                                                                                                                              |
+----------------------------------------------------------------------------------------------+---------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Default Subnetpools                                                                          | default-subnetpools       | Provides ability to mark and use a subnetpool as the default.  
...
----

$ openstack network agent list
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host      | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| 1946cd7a-5b20-4497-a5c4-a2c2beca20fd | Linux bridge agent | ryunosuke | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 54f80a59-a7c6-4c66-b6cd-9baf2de4c926 | Metadata agent     | ryunosuke | None              | :-)   | UP    | neutron-metadata-agent    |
| f5e66291-3fe6-461b-8274-b6e4cd932e26 | L3 agent           | ryunosuke | nova              | :-)   | UP    | neutron-l3-agent          |
| fa4d5c98-7a95-4f3c-b493-6705a12e7343 | DHCP agent         | ryunosuke | nova              | :-)   | UP    | neutron-dhcp-agent        |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
```

## horizon installation for Queens
[horizon Document](https://docs.openstack.org/horizon/queens/install/)

### Install and configure for Ubuntu
```
$ sudo apt -y install openstack-dashboard
$ sudo vim /etc/openstack-dashboard/local_settings.py
----
##OPENSTACK_HOST = "127.0.0.1"
OPENSTACK_HOST = "192.168.0.200"
...
ALLOWED_HOSTS = '*'
...
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
  'default': {
    'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
    'LOCATION': '192.168.0.200:11211',
  }
}
...
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
...
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
...
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
...
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "default"
...
##OPENSTACK_KEYSTONE_DEFAULT_ROLE = "_member_"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
...
##TIME_ZONE = "UTC"
TIME_ZONE = "Asia/Tokyo"
...
----

$ sudo vim /etc/apache2/conf-available/openstack-dashboard.conf
----
...
WSGIApplicationGroup %{GLOBAL}
...
----

$ sudo service apache2 restart
```

### Verify operation for Ubuntu
[http://192.168.0.200/horizon/](http://192.168.0.200/horizon/)

## Launch an instance
[Document](https://docs.openstack.org/install-guide/launch-instance.html)

### Create virtual networks
#### Provider network
```
$ source ~/admin-openrc
$ openstack network create  --share --external --provider-physical-network provider --provider-network-type flat provider
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2018-06-17T12:55:38Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 57a0b995-9bc6-4460-801b-df2d4d7692ba |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | provider                             |
| port_security_enabled     | True                                 |
| project_id                | a77a45043cc943c398841d12d5ddfa05     |
| provider:network_type     | flat                                 |
| provider:physical_network | provider                             |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 5                                    |
| router:external           | External                             |
| segments                  | None                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2018-06-17T12:55:38Z                 |
+---------------------------+--------------------------------------+

$ openstack subnet create --network provider --allocation-pool start=192.168.0.100,end=192.168.0.120 --dns-nameserver 192.168.0.254 --gateway 192.168.0.254 --subnet-range 192.168.0.254/24 provider
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 192.168.0.100-192.168.0.120          |
| cidr              | 192.168.0.0/24                       |
| created_at        | 2018-06-17T12:57:24Z                 |
| description       |                                      |
| dns_nameservers   | 192.168.0.254                        |
| enable_dhcp       | True                                 |
| gateway_ip        | 192.168.0.254                        |
| host_routes       |                                      |
| id                | 5d134298-00a5-4b57-9c68-36a17cce0fac |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | provider                             |
| network_id        | 57a0b995-9bc6-4460-801b-df2d4d7692ba |
| project_id        | a77a45043cc943c398841d12d5ddfa05     |
| revision_number   | 0                                    |
| segment_id        | None                                 |
| service_types     |                                      |
| subnetpool_id     | None                                 |
| tags              |                                      |
| updated_at        | 2018-06-17T12:57:24Z                 |
+-------------------+--------------------------------------+
```

#### Self-service network
```
$ source ~/demo-openrc 
$ openstack network create selfservice
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2018-06-17T12:51:05Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | ec4cb910-2cbb-4a49-9953-d1661e65e877 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1450                                 |
| name                      | selfservice                          |
| port_security_enabled     | True                                 |
| project_id                | dc86d2e95fef4f56a70021e055f0c2cf     |
| provider:network_type     | None                                 |
| provider:physical_network | None                                 |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 2                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2018-06-17T12:51:05Z                 |
+---------------------------+--------------------------------------+

$ openstack subnet create --network selfservice --dns-nameserver 192.168.0.220 --gateway 10.0.0.1 --subnet-range 10.0.0.0/24 selfservice
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 10.0.0.2-10.0.0.254                  |
| cidr              | 10.0.0.0/24                          |
| created_at        | 2018-06-17T12:53:03Z                 |
| description       |                                      |
| dns_nameservers   | 192.168.0.220                        |
| enable_dhcp       | True                                 |
| gateway_ip        | 10.0.0.1                             |
| host_routes       |                                      |
| id                | ab6a8b25-f43b-4562-ac81-549ec56401b4 |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | selfservice                          |
| network_id        | ec4cb910-2cbb-4a49-9953-d1661e65e877 |
| project_id        | dc86d2e95fef4f56a70021e055f0c2cf     |
| revision_number   | 0                                    |
| segment_id        | None                                 |
| service_types     |                                      |
| subnetpool_id     | None                                 |
| tags              |                                      |
| updated_at        | 2018-06-17T12:53:03Z                 |
+-------------------+--------------------------------------+

$ source ~/demo-openrc
$ openstack router create router
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | UP                                   |
| availability_zone_hints |                                      |
| availability_zones      |                                      |
| created_at              | 2018-06-17T12:53:54Z                 |
| description             |                                      |
| distributed             | False                                |
| external_gateway_info   | None                                 |
| flavor_id               | None                                 |
| ha                      | False                                |
| id                      | 94d108dc-114e-4d15-92cf-eddb7658e282 |
| name                    | router                               |
| project_id              | dc86d2e95fef4f56a70021e055f0c2cf     |
| revision_number         | 1                                    |
| routes                  |                                      |
| status                  | ACTIVE                               |
| tags                    |                                      |
| updated_at              | 2018-06-17T12:53:54Z                 |
+-------------------------+--------------------------------------+

$ neutron router-interface-add router selfservice
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
Added interface 864fbbbe-f45a-4e0b-a9a4-3793a7a937c2 to router router.

$ neutron router-gateway-set router provider
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
Set gateway for router router
```

### Verify operation
```
$ source ~/admin-openrc
$ ip netns
qdhcp-57a0b995-9bc6-4460-801b-df2d4d7692ba (id: 2)
qrouter-94d108dc-114e-4d15-92cf-eddb7658e282 (id: 1)
qdhcp-ec4cb910-2cbb-4a49-9953-d1661e65e877 (id: 0)

$ neutron router-port-list router
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+------+----------------------------------+-------------------+--------------------------------------------------------------------------------------+
| id                                   | name | tenant_id                        | mac_address       | fixed_ips                                                                            |
+--------------------------------------+------+----------------------------------+-------------------+--------------------------------------------------------------------------------------+
| 864fbbbe-f45a-4e0b-a9a4-3793a7a937c2 |      | dc86d2e95fef4f56a70021e055f0c2cf | fa:16:3e:1e:06:9f | {"subnet_id": "ab6a8b25-f43b-4562-ac81-549ec56401b4", "ip_address": "10.0.0.1"}      |
| f75d0fa6-fb4b-4139-830f-10472fbedc96 |      |                                  | fa:16:3e:bc:b3:56 | {"subnet_id": "5d134298-00a5-4b57-9c68-36a17cce0fac", "ip_address": "192.168.0.102"} |
+--------------------------------------+------+----------------------------------+-------------------+--------------------------------------------------------------------------------------+

$ ping -c 4 192.168.0.102
PING 192.168.0.102 (192.168.0.102) 56(84) bytes of data.
64 bytes from 192.168.0.102: icmp_seq=1 ttl=64 time=0.101 ms
64 bytes from 192.168.0.102: icmp_seq=2 ttl=64 time=0.076 ms
64 bytes from 192.168.0.102: icmp_seq=3 ttl=64 time=0.075 ms
64 bytes from 192.168.0.102: icmp_seq=4 ttl=64 time=0.069 ms

--- 192.168.0.102 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 2999ms
rtt min/avg/max/mdev = 0.069/0.080/0.101/0.013 ms
ogalush@ryunosuke:~$
```

### Create flavor
```
$ source ~/admin-openrc
$ openstack flavor create --id 0 --vcpus 1 --ram 1024 --disk 1 m1.nano
$ openstack flavor create --id 1 --vcpus 2 --ram 4096 --disk 40 m1.small --swap 4096
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
| swap                       | 4096     |
| vcpus                      | 2        |
+----------------------------+----------+

$ openstack flavor list
+----+----------+------+------+-----------+-------+-----------+
| ID | Name     |  RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+----------+------+------+-----------+-------+-----------+
| 0  | m1.nano  | 1024 |    1 |         0 |     1 | True      |
| 1  | m1.small | 4096 |   40 |         0 |     2 | True      |
+----+----------+------+------+-----------+-------+-----------+
```

### import keypair
```
$ source ~/demo-openrc
$ openstack keypair create --public-key ~/.ssh/id_rsa_chef.pub chefkey
+-------------+-------------------------------------------------+
| Field       | Value                                           |
+-------------+-------------------------------------------------+
| fingerprint | f2:70:ad:9c:93:32:14:c0:36:e3:4f:ac:19:45:82:d2 |
| name        | chefkey                                         |
| user_id     | 2ca851b69cbe4ed5b328f6f17b373405                |
+-------------+-------------------------------------------------+

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
| created_at        | 2018-06-17T13:04:37Z                 |
| description       |                                      |
| direction         | ingress                              |
| ether_type        | IPv4                                 |
| id                | cf2781fc-dc17-4551-aafd-8a30c9565c9c |
| name              | None                                 |
| port_range_max    | None                                 |
| port_range_min    | None                                 |
| project_id        | dc86d2e95fef4f56a70021e055f0c2cf     |
| protocol          | icmp                                 |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 0                                    |
| security_group_id | 4c74d885-5ff6-4a08-882f-a545b37a610f |
| updated_at        | 2018-06-17T13:04:37Z                 |
+-------------------+--------------------------------------+

$ openstack security group rule create --proto tcp --dst-port 22 default
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2018-06-17T13:04:53Z                 |
| description       |                                      |
| direction         | ingress                              |
| ether_type        | IPv4                                 |
| id                | a195504e-3d21-4757-ad3f-550d8686f5f5 |
| name              | None                                 |
| port_range_max    | 22                                   |
| port_range_min    | 22                                   |
| project_id        | dc86d2e95fef4f56a70021e055f0c2cf     |
| protocol          | tcp                                  |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 0                                    |
| security_group_id | 4c74d885-5ff6-4a08-882f-a545b37a610f |
| updated_at        | 2018-06-17T13:04:53Z                 |
+-------------------+--------------------------------------+
```

### Launch an instance
```
$ source ~/demo-openrc
$ openstack flavor list
+----+----------+------+------+-----------+-------+-----------+
| ID | Name     |  RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+----------+------+------+-----------+-------+-----------+
| 0  | m1.nano  | 1024 |    1 |         0 |     1 | True      |
| 1  | m1.small | 4096 |   40 |         0 |     2 | True      |
+----+----------+------+------+-----------+-------+-----------+

$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| f3e81692-4e7e-4167-a5d0-c683264ed265 | cirros | active |
+--------------------------------------+--------+--------+

$ openstack network list
+--------------------------------------+-------------+--------------------------------------+
| ID                                   | Name        | Subnets                              |
+--------------------------------------+-------------+--------------------------------------+
| 57a0b995-9bc6-4460-801b-df2d4d7692ba | provider    | 5d134298-00a5-4b57-9c68-36a17cce0fac |
| ec4cb910-2cbb-4a49-9953-d1661e65e877 | selfservice | ab6a8b25-f43b-4562-ac81-549ec56401b4 |
+--------------------------------------+-------------+--------------------------------------+

$ openstack security group list
+--------------------------------------+---------+------------------------+----------------------------------+
| ID                                   | Name    | Description            | Project                          |
+--------------------------------------+---------+------------------------+----------------------------------+
| 4c74d885-5ff6-4a08-882f-a545b37a610f | default | Default security group | dc86d2e95fef4f56a70021e055f0c2cf |
+--------------------------------------+---------+------------------------+----------------------------------+

$ openstack server create --flavor m1.nano --image cirros --nic net-id=`openstack network list |grep selfservice | cut -d' ' -f2` --security-group default  --key-name chefkey selfservice-instance
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
| adminPass                   | Py6HmhjgPRWc                                  |
| config_drive                |                                               |
| created                     | 2018-06-17T13:07:26Z                          |
| flavor                      | m1.nano (0)                                   |
| hostId                      |                                               |
| id                          | e7079e89-94ef-4309-ad0f-2aa9d894df11          |
| image                       | cirros (f3e81692-4e7e-4167-a5d0-c683264ed265) |
| key_name                    | chefkey                                       |
| name                        | selfservice-instance                          |
| progress                    | 0                                             |
| project_id                  | dc86d2e95fef4f56a70021e055f0c2cf              |
| properties                  |                                               |
| security_groups             | name='4c74d885-5ff6-4a08-882f-a545b37a610f'   |
| status                      | BUILD                                         |
| updated                     | 2018-06-17T13:07:26Z                          |
| user_id                     | 2ca851b69cbe4ed5b328f6f17b373405              |
| volumes_attached            |                                               |
+-----------------------------+-----------------------------------------------+

$ openstack server list
+--------------------------------------+----------------------+--------+----------+--------+---------+
| ID                                   | Name                 | Status | Networks | Image  | Flavor  |
+--------------------------------------+----------------------+--------+----------+--------+---------+
| e7079e89-94ef-4309-ad0f-2aa9d894df11 | selfservice-instance | ERROR  |          | cirros | m1.nano |
+--------------------------------------+----------------------+--------+----------+--------+---------+
  
$ openstack console url show selfservice-instance
```
