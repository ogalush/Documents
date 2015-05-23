<!--
************************************************************
OpenStack JunoをUbuntu15.04(x86_64)へインストールする手順
参照元: http://docs.openstack.org/kilo/install-guide/install/apt/content/  
Copyright (c) Takehiko OGASAWARA 2015 All Rights Reserved.
************************************************************
-->

# OpenStack(Kilo)をインストールする
[公式インストール手順](http://docs.openstack.org/kilo/install-guide/install/apt/content/)

## Controller node
### 準備
バックアップディレクトリ作成
```
$ sudo mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
$ BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

### Basicインストール
ntpインストール
```
$ sudo apt-get -y install ntp
```

OpenStackパッケージ
```
$ sudo apt-get -y install ubuntu-cloud-keyring
$ echo 'deb http://ubuntu-cloud.archive.canonical.com/ubuntu trusty-updates/juno main' | sudo tee /etc/apt/sources.list.d/cloudarchive-juno.list
$ cat /etc/apt/sources.list.d/cloudarchive-juno.list
$ sudo apt-get -y update && sudo apt-get -y dist-upgrade
```

MariaDB(MySQL)
```
$ sudo apt-get -y install mariadb-server python-mysqldb
→ パスワード: admin!
```

MariaDB設定
```
$ sudo cp -raf /etc/mysql $BAK
$ sudo vi /etc/mysql/my.cnf
----
[mysqld]
...
bind-address            = 0.0.0.0
...
#-- For OpenStack
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
----

#-- 設定の反映
$ sudo service mysql restart

#-- 初期化
$ sudo mysql_secure_installation
→ rootのパスワード変更等聞かれるが、後でnovaユーザ等を作るのでひとまずNoで良い。
---
Enter current password for root (enter for none): 
OK, successfully used password, moving on...
Change the root password? [Y/n] n
 ... skipping.

Remove anonymous users? [Y/n] y
 ... Success!

Disallow root login remotely? [Y/n] y
 ... Success!

Remove test database and access to it? [Y/n] y
 ... Success!

Reload privilege tables now? [Y/n] y
 ... Success!
---
```

RabbitMQ
```
$ sudo apt-get -y install rabbitmq-server
$ sudo rabbitmqctl change_password guest admin!
```


### KeyStone

DB作成
```
$ mysql -u root -p
MariaDB> CREATE DATABASE keystone;
MariaDB> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
MariaDB> FLUSH PRIVILEGES;
MariaDB> quit;
```

パッケージインストール
```
$ sudo apt-get -y install keystone python-keystoneclient
```

KyeStone設定
```
$ sudo cp -raf /etc/keystone $BAK
$ sudo vi /etc/keystone/keystone.conf
---
[DEFAULT]
...
admin_token = token
...
[database]
connection = mysql://keystone:password@192.168.0.200/keystone
[token]
provider = keystone.token.providers.uuid.Provider
driver = keystone.token.persistence.backends.sql.Token
---
```

DB反映
```
$ sudo su -s /bin/sh -c "keystone-manage db_sync" keystone
```

再起動
```
$ sudo service keystone restart
$ sudo rm -f /var/lib/keystone/keystone.db
```

Cron設定
```
$ (sudo crontab -l -u keystone 2>&1 | grep -q token_flush) || echo '@hourly /usr/bin/keystone-manage token_flush  /var/log/keystone/keystone-tokenflush.log 2>&1' |sudo tee /var/spool/cron/crontabs/keystone
$ sudo chown keystone:keystone /var/spool/cron/crontabs/keystone
$ sudo ls -al /var/spool/cron/crontabs/keystone
$ sudo cat /var/spool/cron/crontabs/keystone
```

環境変数（一時使用）
```
$ export OS_SERVICE_TOKEN=password
$ export OS_SERVICE_ENDPOINT=http://192.168.0.200:35357/v2.0
```

テナント・ユーザ作成
```
$ keystone tenant-create --name admin --description "Admin Tenant"
$ keystone user-create --name admin --pass password --email admin@192.168.0.200
$ keystone role-create --name admin
$ keystone user-role-add --user admin --tenant admin --role admin
$ keystone tenant-create --name demo --description "Demo Tenant"
$ keystone user-create --name demo --tenant demo --pass password --email demo@192.168.0.200
$ keystone tenant-create --name service --description "Service Tenant"
```

サービス作成
```
$ keystone service-create --name keystone --type identity --description "OpenStack Identity"
$ keystone endpoint-create --service-id $(keystone service-list | awk '/ identity / {print $2}') --publicurl http://192.168.0.200:5000/v2.0 --internalurl http://192.168.0.200:5000/v2.0 --adminurl http://192.168.0.200:35357/v2.0 --region regionOne
```

環境変数解除
```
$ unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT
```

.keystone設定
```
$ vi ~/.keystonerc
---
#!/bin/bash

export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.0.200:35357/v2.0
---

$ vi ~/.bashrc
---
...
source ~/.keystonerc
---
$ source ~/.bashrc
```

確認
```
$ keystone token-get
$ keystone user-list
$ keystone role-list
$ keystone tenant-list
```

### Glance
DB設定
```
$ mysql -u root -p
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> ¥q
```

keystone設定
```
$ keystone user-create --name glance --pass password
$ keystone user-role-add --user glance --tenant service --role admin
$ keystone service-create --name glance --type image --description "OpenStack Image Service"
$ keystone endpoint-create --service-id $(keystone service-list | awk '/ image / {print $2}') --publicurl http://192.168.0.200:9292 --internalurl http://192.168.0.200:9292 --adminurl http://192.168.0.200:9292 --region regionOne
```

glanceパッケージ
```
$ sudo apt-get -y install glance python-glanceclient
```

glance設定
```
$ sudo cp -raf /etc/glance $BAK
$ sudo vi /etc/glance/glance-api.conf
---
[database]
...
connection = mysql://glance:password@192.168.0.200/glance
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000/v2.0
identity_uri = http://192.168.0.200:35357
admin_tenant_name = service
admin_user = glance
admin_password = password
...
[paste_deploy]
...
flavor = keystone
...
[glance_store]
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
---

$ sudo vi /etc/glance/glance-registry.conf
---
[database]
connection = mysql://glance:password@192.168.0.200/glance
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000/v2.0
identity_uri = http://192.168.0.200:35357
admin_tenant_name = service
admin_user = glance
admin_password = password
...
[paste_deploy]
flavor = keystone
...
---

$ sudo su -s /bin/sh -c "glance-manage db_sync" glance
```

glance再起動
```
$ sudo service glance-registry restart
$ sudo service glance-api restart
$ rm -f /var/lib/glance/glance.sqlite
```

OSイメージインポート
```
$ sudo wget -P /usr/local/src http://cloud-images.ubuntu.com/releases/14.04/release/ubuntu-14.04-server-cloudimg-amd64-disk1.img
$ glance image-create --name "ubuntu14.04" --file /usr/local/src/ubuntu-14.04-server-cloudimg-amd64-disk1.img --disk-format qcow2 --container-format bare --is-public True --progress
$ glance image-list
```

### nova
DB設定
```
$  mysql -u root -p
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> ¥q
```
keystone設定
```
$ keystone user-create --name nova --pass password
$ keystone user-role-add --user nova --tenant service --role admin
$ keystone service-create --name nova --type compute --description "OpenStack Compute"
$ keystone endpoint-create --service-id $(keystone service-list | awk '/ compute / {print $2}') --publicurl http://192.168.0.200:8774/v2/%\(tenant_id\)s --internalurl http://192.168.0.200:8774/v2/%\(tenant_id\)s --adminurl http://192.168.0.200:8774/v2/%\(tenant_id\)s --region regionOne
```

novaインストール
```
$ sudo apt-get -y install nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy nova-scheduler python-novaclient
```

nova設定
```
$ sudo cp -raf /etc/neutron $BAK
$ sudo vi /etc/nova/nova.conf
---
[DEFAULT]
...
rpc_backend = rabbit
rabbit_host = 192.168.0.200
rabbit_user = guest
rabbit_password = admin!
auth_strategy = keystone
my_ip = 192.168.0.200
vnc_enabled = True
vncserver_listen = 192.168.0.200
vncserver_proxyclient_address = 192.168.0.200
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html
...
[database]
connection = mysql://nova:password@192.168.0.200/nova

[keystone_authtoken]
auth_uri = http://192.168.0.200:5000/v2.0
identity_uri = http://192.168.0.200:35357
admin_tenant_name = service
admin_user = nova
admin_password = password

[glance]
host = 192.168.0.200
---

$ sudo su -s /bin/sh -c "nova-manage db sync" nova
```

nova再起動
```
$ initctl list |grep -i nova | awk '{print $1}' |sort | awk '{print "sudo service "$1" restart"}' | bash

※ 以下のサービスがrestartすればOK
---
nova-api
nova-cert
nova-consoleauth
nova-scheduler
nova-conductor
nova-novncproxy
---
$ rm -f /var/lib/nova/nova.sqlite
```

Hypervisor設定
```
$ sudo apt-get -y install nova-compute sysfsutils
$ sudo service nova-compute restart
$ sudo rm -f /var/lib/nova/nova.sqlite
```

nova確認
```
5つのサービスが起動していればOK.
$ nova service-list
---
+----+------------------+-----------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host      | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+-----------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-cert        | ryunosuke | internal | enabled | up    | 2015-01-31T14:57:33.000000 | -               |
| 2  | nova-consoleauth | ryunosuke | internal | enabled | up    | 2015-01-31T14:57:34.000000 | -               |
| 3  | nova-conductor   | ryunosuke | internal | enabled | up    | 2015-01-31T14:57:34.000000 | -               |
| 4  | nova-scheduler   | ryunosuke | internal | enabled | up    | 2015-01-31T14:57:34.000000 | -               |
| 5  | nova-compute     | ryunosuke | nova     | enabled | up    | 2015-01-31T14:57:34.000000 | -               |
+----+------------------+-----------+----------+---------+-------+----------------------------+-----------------+
---

glanceイメージが表示されればOK.
$ nova image-list
+--------------------------------------+-------------+--------+--------+
| ID                                   | Name        | Status | Server |
+--------------------------------------+-------------+--------+--------+
| a72549aa-2abb-4f90-82ed-53c39acf3d5a | ubuntu14.04 | ACTIVE |        |
+--------------------------------------+-------------+--------+--------+
```

### neutron
DB設定
```
$ mysql -u root -p
MariaDB [(none)]> CREATE DATABASE neutron;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> ¥q
```

keystone設定
```
$ keystone user-create --name neutron --pass password
$ keystone user-role-add --user neutron --tenant service --role admin
$ keystone service-create --name neutron --type network --description "OpenStack Networking"
$ keystone endpoint-create --service-id $(keystone service-list | awk '/ network / {print $2}') --publicurl  http://192.168.0.200:9696 --adminurl http://192.168.0.200:9696 --internalurl http://192.168.0.200:9696 --region regionOne
```

neutronパッケージ
```
$ sudo apt-get -y install neutron-server neutron-plugin-ml2 python-neutronclient
```

neutron設定
```
$ keystone tenant-get service
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |          Service Tenant          |
|   enabled   |               True               |
|      id     | 302f366fd65440eab9406a8f20317a93 |
|     name    |             service              |
+-------------+----------------------------------+

$ sudo cp -raf /etc/neutron $BAK
$ sudo vi /etc/neutron/neutron.conf
---
[DEFAULT]
...
rpc_backend = rabbit
rabbit_host = 192.168.0.200
rabbit_user = guest
rabbit_password = admin!
auth_strategy = keystone
...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
...
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
nova_url = http://192.168.0.200:8774/v2
nova_admin_auth_url = http://192.168.0.200:35357/v2.0
nova_region_name = regionOne
nova_admin_username = nova
nova_admin_tenant_id = 302f366fd65440eab9406a8f20317a93
nova_admin_password = password
...
[database]
...
connection = mysql://neutron:password@192.168.0.200/neutron
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000/v2.0
identity_uri = http://192.168.0.200:35357
admin_tenant_name = service
admin_user = neutron
admin_password = password
---
```

ml2設定
```
$ sudo vi /etc/neutron/plugins/ml2/ml2_conf.ini
---
[ml2]
type_drivers = flat,gre
tenant_network_types = gre 
mechanism_drivers = openvswitch
...
[ml2_type_gre]
tunnel_id_ranges = 1:1000
...
[securitygroup]
enable_security_group = True
enable_ipset = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
---
```

nova設定変更
```
$ sudo vi /etc/nova/nova.conf
---
[DEFAULT]
...
network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
...
[neutron]
url = http://192.168.0.200:9696
auth_strategy = keystone
admin_auth_url = http://192.168.0.200:35357/v2.0
admin_tenant_name = service
admin_username = neutron
admin_password = password
---
```

設定反映
```
$ sudo su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade juno" neutron
```

neutron, nova再起動
```
$ initctl list |grep -i nova | awk '{print $1}' |sort | awk '{print "sudo service "$1" restart"}' | bash
$ sudo service neutron-server restart
$ neutron ext-list
→ 表示されれば確認OK.
```

networknode追加
```
$ sudo cp -p /etc/sysctl.conf $BAK
$ sudo vi /etc/sysctl.conf
---
...
#-- For OpenStack
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
---
$ sudo sysctl -p
```

neutronパッケージ
```
$ sudo apt-get -y install neutron-plugin-ml2 neutron-plugin-openvswitch-agent neutron-l3-agent neutron-dhcp-agent
```

neutron設定
```
$ sudo vi /etc/neutron/plugins/ml2/ml2_conf.ini 
---
[ml2_type_flat]
flat_networks = external

[ml2_type_gre]
tunnel_id_ranges = 1:1000
...
[securitygroup]
enable_security_group = True
enable_ipset = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

[ovs]
local_ip = 192.168.0.200
enable_tunneling = True
bridge_mappings = external:br-ex

[agent]
tunnel_types = gre
---

$ sudo vi /etc/neutron/l3_agent.ini
---
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True
external_network_bridge = br-ex
...
---

$ sudo vi /etc/neutron/dhcp_agent.ini
---
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
use_namespaces = True
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
dhcp_domain = localdomain
...
---

$ sudo vi /etc/neutron/dnsmasq-neutron.conf
---
dhcp-option-force=26,1400
---
※ Enable the DHCP MTU option (26) and configure it to 1454 bytes:
※ GREを使用すると最大フレームサイズ(1500)だと、カプセル化で付与したフレームが通らない.
　そのためMTU値を低く設定する.

$ sudo pkill dnsmasq

$ sudo vi /etc/neutron/metadata_agent.ini
---
[DEFAULT]
auth_url = http://192.168.0.200:5000/v2.0
auth_region = regionOne
admin_tenant_name = service
admin_user = neutron
admin_password = password
nova_metadata_ip = 192.168.0.200
metadata_proxy_shared_secret = password
...
---

$ sudo vi /etc/nova/nova.conf
---
[neutron]
...
service_metadata_proxy = True
metadata_proxy_shared_secret = password
---

$ sudo service nova-api restart
```

ovs設定
```
$ sudo service openvswitch-switch restart
$ sudo ovs-vsctl add-br br-ex
$ sudo ovs-vsctl add-port br-ex p1p1
$ sudo ethtool -K p1p1 gro off
```

neutron再起動
```
$ sudo service neutron-plugin-openvswitch-agent restart
$ sudo service neutron-l3-agent restart
$ sudo service neutron-dhcp-agent restart
$ sudo service neutron-metadata-agent restart
```

確認
```
$ neutron agent-list
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| id                                   | agent_type         | host      | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| 00326010-3775-47cb-ba35-d53d74d5135d | DHCP agent         | ryunosuke | :-)   | True           | neutron-dhcp-agent        |
| 28135e73-d56e-4e82-a1ce-02ad37e4b2d8 | Metadata agent     | ryunosuke | :-)   | True           | neutron-metadata-agent    |
| 61e5dfcd-cf6f-4f8e-880f-60cdb32aff9e | L3 agent           | ryunosuke | :-)   | True           | neutron-l3-agent          |
| b63336e0-d292-433e-b8da-1688efef7a07 | Open vSwitch agent | ryunosuke | :-)   | True           | neutron-openvswitch-agent |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
```

### 仮装ネットワーク作成
#### 外部ネットワーク
マニュアルにSource the admin credentials to gain access to admin-only CLI commands:.  
と記載があるので、admin権限で作成する.  
```
$ keystone tenant-list
+----------------------------------+---------+---------+
|                id                |   name  | enabled |
+----------------------------------+---------+---------+
| 61dc76652fa14c58a4a72e0e9fad93e5 |  admin  |   True  |
~~~これを使用する.
| 3397a7d0cf9047f1b7c3a0ae149f5a3d |   demo  |   True  |
| 302f366fd65440eab9406a8f20317a93 | service |   True  |
+----------------------------------+---------+---------+

$ neutron net-create ext-net --tenant-id 61dc76652fa14c58a4a72e0e9fad93e5 --router:external True --provider:physical_network external --provider:network_type flat

Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | b46155c1-19f7-4bc5-992f-d910947d89bc |
| name                      | ext-net                              |
| provider:network_type     | flat                                 |
| provider:physical_network | external                             |
| provider:segmentation_id  |                                      |
| router:external           | True                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 61dc76652fa14c58a4a72e0e9fad93e5     |
+---------------------------+--------------------------------------+

$ neutron subnet-create ext-net --name ext-subnet --tenant-id 61dc76652fa14c58a4a72e0e9fad93e5 --allocation-pool start=192.168.0.100,end=192.168.0.150 --disable-dhcp --gateway 192.168.0.254 192.168.0.0/24

Created a new subnet:
+-------------------+----------------------------------------------------+
| Field             | Value                                              |
+-------------------+----------------------------------------------------+
| allocation_pools  | {"start": "192.168.0.100", "end": "192.168.0.150"} |
| cidr              | 192.168.0.0/24                                     |
| dns_nameservers   |                                                    |
| enable_dhcp       | False                                              |
| gateway_ip        | 192.168.0.254                                      |
| host_routes       |                                                    |
| id                | be761def-2eb4-47c8-9a93-a82e48724000               |
| ip_version        | 4                                                  |
| ipv6_address_mode |                                                    |
| ipv6_ra_mode      |                                                    |
| name              | ext-subnet                                         |
| network_id        | b46155c1-19f7-4bc5-992f-d910947d89bc               |
| tenant_id         | 61dc76652fa14c58a4a72e0e9fad93e5                   |
+-------------------+----------------------------------------------------+
```

#### 内部ネットワーク(テナント用)
Source the demo credentials to gain access to user-only CLI commands:  
とあるので、demo向けに作成する.  
```
$ keystone tenant-list
+----------------------------------+---------+---------+
|                id                |   name  | enabled |
+----------------------------------+---------+---------+
| 61dc76652fa14c58a4a72e0e9fad93e5 |  admin  |   True  |
| 3397a7d0cf9047f1b7c3a0ae149f5a3d |   demo  |   True  |
~~~これを使用する.
| 302f366fd65440eab9406a8f20317a93 | service |   True  |
+----------------------------------+---------+---------+

$ neutron net-create --tenant-id 3397a7d0cf9047f1b7c3a0ae149f5a3d  demo-net

Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 20d0e6b7-1906-4134-a3c5-4c6adfc2e186 |
| name                      | demo-net                             |
| provider:network_type     | gre                                  |
| provider:physical_network |                                      |
| provider:segmentation_id  | 1                                    |
| router:external           | False                                |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 3397a7d0cf9047f1b7c3a0ae149f5a3d     |
+---------------------------+--------------------------------------+


$ neutron subnet-create demo-net --name demo-subnet --tenant-id 3397a7d0cf9047f1b7c3a0ae149f5a3d --gateway 10.0.0.1 10.0.0.0/24  --dns-nameservers list=true 192.168.0.254

Created a new subnet:
+-------------------+--------------------------------------------+
| Field             | Value                                      |
+-------------------+--------------------------------------------+
| allocation_pools  | {"start": "10.0.0.2", "end": "10.0.0.254"} |
| cidr              | 10.0.0.0/24                                |
| dns_nameservers   | 192.168.0.254                              |
| enable_dhcp       | True                                       |
| gateway_ip        | 10.0.0.1                                   |
| host_routes       |                                            |
| id                | e6e4c91a-ac1e-4f49-963c-67059d6ddb0f       |
| ip_version        | 4                                          |
| ipv6_address_mode |                                            |
| ipv6_ra_mode      |                                            |
| name              | demo-subnet                                |
| network_id        | 20d0e6b7-1906-4134-a3c5-4c6adfc2e186       |
| tenant_id         | 3397a7d0cf9047f1b7c3a0ae149f5a3d           |
+-------------------+--------------------------------------------+
```

#### 仮想ルータ
```
$ neutron router-create --tenant-id 3397a7d0cf9047f1b7c3a0ae149f5a3d demo-router

Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| distributed           | False                                |
| external_gateway_info |                                      |
| ha                    | False                                |
| id                    | 4c8dfa17-9399-4c8e-8875-7e175737ca49 |
| name                  | demo-router                          |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | 3397a7d0cf9047f1b7c3a0ae149f5a3d     |
+-----------------------+--------------------------------------+

$ neutron router-interface-add demo-router demo-subnet
$ neutron router-gateway-set demo-router ext-net

$ sudo cp /etc/network/interfaces $BAK
$ sudo vi /etc/network/interfaces
---
auto lo
iface lo inet loopback

# for OpenStack
auto p1p1
iface p1p1 inet manual
  up ip link set dev $IFACE up
  down ip link set dev $IFACE down

# The primary network interface
auto br-ex
iface br-ex inet static
  address 192.168.0.200
  netmask 255.255.255.0
  network 192.168.0.0
  gateway 192.168.0.254
  dns-nameservers 192.168.0.254
---
$ sudo reboot
→ 再起動後、sshログインできればOK.
```

### horizon
horizon本体
```
$ sudo apt-get -y install openstack-dashboard apache2 libapache2-mod-wsgi memcached python-memcache
```

horizon設定
```
$ sudo cp -raf /etc/openstack-dashboard $BAK
$ sudo vi /etc/openstack-dashboard/local_settings.py 
---
...
OPENSTACK_HOST = "192.168.0.200"
...
ALLOWED_HOSTS = ['*']
...
TIME_ZONE = "Asia/Tokyo"
---
```

反映
```
$ sudo service apache2 restart
$ sudo service memcached restart
```

ブラウザアクセス
[horizon](http://192.168.0.200/horizon/)
