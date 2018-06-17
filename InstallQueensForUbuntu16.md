# Install OpenStack Queens on Ubuntu 16.04
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/openstack-services.html)  
インストール先: 192.168.0.200(192.168.0.200)  
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
