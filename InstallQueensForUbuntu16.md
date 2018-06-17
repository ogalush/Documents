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
