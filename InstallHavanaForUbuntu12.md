<!--
************************************************************
OpenStack (havana) をUbuntu12(x86_64)へインストールする手順
参照元: 
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->
<div id='title'>　</div>    

# havanaインストール手順

### 準備
バックアップディレクトリ作成
```
BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
mkdir -p $BAK
```

KVM確認
```
apt-get install cpu-checker
root@kinder:~# kvm-ok 
INFO: /dev/kvm exists
KVM acceleration can be used
↑が出ていればOK

# lsmod | grep kvm
kvm_intel             137899  0 
kvm                   455932  1 kvm_intel
```

ntpインストール
```
# apt-get install -y ntp
```


### MySQLインストール
```
# apt-get install -y python-mysqldb mysql-server
# cp -raf /etc/mysql $BAK
# vi /etc/mysql/my.cnf
---
bind-address            = 192.168.0.200
---
```

MySQLユーザ、DB作成
```
service mysql restart
mysql_install_db
```

### OpenStack Packages
```
# apt-get install -y python-software-properties
# add-apt-repository cloud-archive:havana
~~aptヘhavanaを追加する
# apt-get update && apt-get dist-upgrade
# sync;sync;sync;reboot
```

### rabbit MQ
```
# apt-get install -y rabbitmq-server
# rabbitmqctl change_password guest password
```

### keystone
```
# apt-get install -y keystone
# cp -raf /etc/keystone $BAK
# vi /etc/keystone/keystone.conf
----
[sql]
##connection = sqlite:////var/lib/keystone/keystone.db
connection = mysql://keystone:password@192.168.0.200/keystone
----

# mysql -u root -p
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
\q

# keystone-manage db_sync

# vi /etc/keystone/keystone.conf
---
admin_token = password
---
# service keystone restart
```

### 環境変数設定
```
vi /root/.openrc
---
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL="http://192.168.0.200:5000/v2.0/"
export OS_SERVICE_ENDPOINT="http://192.168.0.200:35357/v2.0"
export OS_SERVICE_TOKEN=password
---
source ~/.openrc
echo "source ~/.openrc" >> ~/.bashrc
```


### keystoneユーザ作成
```
(1) user, role
# keystone tenant-create --name=admin --description="Admin Tenant"
# keystone tenant-create --name=service --description="Service Tenant"
# keystone user-create --name=admin --pass=password --email=admin@example.com
# keystone role-create --name=admin
# keystone user-role-add --user=admin --tenant=admin --role=admin

(2) service, endpoints
# keystone service-create --name=keystone --type=identity --description="Keystone Identity Service"
# keystone endpoint-create  --service-id=6d15194238b4418aae9f25178eadc110  --publicurl=http://192.168.0.200:5000/v2.0 --internalurl=http://192.168.0.200:5000/v2.0 --adminurl=http://192.168.0.200:35357/v2.0
~~~サービスIDは、keystone service-createで生成されたIDを入力する

### インストール確認
unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT
keystone --os-username=admin --os-password=password --os-auth-url=http://192.168.0.200:35357/v2.0 token-get
~~tokenを画面表示できればOK

keystone --os-username=admin --os-password=password  --os-tenant-name=admin --os-auth-url=http://192.168.0.200:35357/v2.0 token-get
~~tokenを画面表示できればOK

source ~/.openrc
keystone user-list
~~tokenを画面表示できればOK

```

### glance
```
Packageインストール
# apt-get install -y glance python-glanceclient
# cp -raf /etc/glance $BAK

設定変更
# vi /etc/glance/glance-api.conf 
----
##sql_connection = sqlite:////var/lib/glance/glance.sqlite
sql_connection = mysql://glance:password@192.168.0.200/glance
----

# vi /etc/glance/glance-registry.conf 
----
##sql_connection = sqlite:////var/lib/glance/glance.sqlite
sql_connection = mysql://glance:password@192.168.0.200/glance
----

mysql登録
# mysql -u root -p
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
\q

DB初期設定
# glance-manage db_sync

keystoneユーザ登録
# keystone user-create --name=glance --pass=password  --email=glance@example.com
# keystone user-role-add --user=glance --tenant=service --role=admin

設定変更
# vi /etc/glance/glance-api.conf
---
[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = password
---

# vi /etc/glance/glance-registry.conf
---
[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_user=glance
admin_tenant_name=service
admin_password=password
flavor=keystone
---

# keystone service-create --name=glance --type=image --description="Glance Image Service"
# keystone endpoint-create  --service-id=f4beac5c9c2a419e9e587dabd676b141 --publicurl=http://192.168.0.200:9292 --internalurl=http://192.168.0.200:9292 --adminurl=http://192.168.0.200:9292
~~~service-idは、keystone service-createで出力されたIDを入力する

glance再起動
# ls -1 /etc/init.d/glance-*| while read LINE; do service `basename ${LINE}` restart; done

### OSイメージ登録
# cd /usr/local/src
# wget http://cloud-images.ubuntu.com/releases/quantal/release/ubuntu-12.10-server-cloudimg-amd64-disk1.img
# glance image-create --is-public true --disk-format qcow2 --container-format bare --name "Ubuntu12.10" < ubuntu-12.10-server-cloudimg-amd64-disk1.img
# glance image-list
---
+--------------------------------------+-------------+-------------+------------------+-----------+--------+
| ID                                   | Name        | Disk Format | Container Format | Size      | Status |
+--------------------------------------+-------------+-------------+------------------+-----------+--------+
| aac7e922-8adc-458e-adbd-346c6456722c | Ubuntu12.10 | qcow2       | bare             | 224002048 | active |
+--------------------------------------+-------------+-------------+------------------+-----------+--------+
---
~~~イメージを表示できればOK

```

### nova
```
# apt-get install -y nova-novncproxy novnc nova-api nova-ajax-console-proxy nova-cert nova-conductor nova-consoleauth nova-doc nova-scheduler python-novaclient

# cp -raf /etc/nova $BAK
# vi /etc/nova/nova.conf
---
[DEFAULT]
connection = mysql://nova:password@192.168.0.200/nova
rpc_backend = nova.rpc.impl_kombu
rabbit_host = 192.168.0.200
rabbit_password = password
...
[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = password
---

DBユーザ作成
# mysql -u root -p
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
\q

DB初期設定
# nova-manage db sync

vnc設定
# vi /etc/nova/nova.conf
---
[DEFAULT]
...
my_ip=192.168.0.200
vncserver_listen=192.168.0.200
vncserver_proxyclient_address=192.168.0.200
...
---

keystoneユーザ設定
# keystone user-create --name=nova --pass=password --email=nova@example.com
# keystone user-role-add --user=nova --tenant=service --role=admin

# vi /etc/nova/nova.conf
---
[DEFAULT]
...
auth_strategy=keystone
...
---

# vi /etc/nova/api-paste.ini 
---
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = password
---

keystone設定
# keystone service-create --name=nova --type=compute --description="Nova Compute service"
# keystone endpoint-create --service-id=bdf676442a544c19b4d4fe524fb2ac5e --publicurl=http://192.168.0.200:8774/v2/%\(tenant_id\)s --internalurl=http://192.168.0.200:8774/v2/%\(tenant_id\)s  --adminurl=http://192.168.0.200:8774/v2/%\(tenant_id\)s
~~~登録できればOK

サービス再起動
# ls -1 /etc/init.d/nova-*| while read LINE; do service `basename ${LINE}` restart; done

確認
# nova image-list
---
+--------------------------------------+-------------+--------+--------+
| ID                                   | Name        | Status | Server |
+--------------------------------------+-------------+--------+--------+
| aac7e922-8adc-458e-adbd-346c6456722c | Ubuntu12.10 | ACTIVE |        |
+--------------------------------------+-------------+--------+--------+
---
~~~表示されればOK

### install
# apt-get install -y nova-compute-kvm python-guestfs
~~~superminのインストールが促されたら、yesと答える。

# chmod 0644 /boot/vmlinuz*
~~~python-guestfsのbug対応

# vi /etc/nova/nova.conf
---
my_ip=192.168.0.200
vnc_enabled=True
vncserver_listen=192.168.0.200
vncserver_proxyclient_address=192.168.0.200
novncproxy_base_url=http://192.168.0.200:6080/vnc_auto.html
...
glance_host=192.168.0.200
---

# service nova-compute restart
# rm /var/lib/nova/nova.sqlite

### nova-network
# apt-get install -y nova-network

# vi /etc/nova/nova.conf
----
network_manager=nova.network.manager.FlatDHCPManager
firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
network_size=254
allow_same_net_traffic=False
multi_host=True
send_arp_for_ha=True
share_dhcp_address=True
force_dhcp_release=True
flat_network_bridge=br100
flat_interface=eth1
public_interface=eth1
----
service nova-network restart
nova network-create vmnet --fixed-range-v4=10.0.0.0/24 --bridge-interface=br100 --multi-host=T
~~~★br100は存在しないnetwork。あとで解析する。

### sshkey登録
# ssh-keygen
# cd ~/.ssh
# nova-manage db sync
# nova keypair-add --pub_key id_rsa.pub mykey
# nova keypair-list
---
+-------+-------------------------------------------------+
| Name  | Fingerprint                                     |
+-------+-------------------------------------------------+
| mykey | 54:ea:e9:c1:9d:ef:f8:09:10:0f:69:59:c9:fc:e8:8a |
+-------+-------------------------------------------------+
---

# nova flavor-list
----
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| 1  | m1.tiny   | 512       | 1    | 0         |      | 1     | 1.0         | True      |
| 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      |
| 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      |
| 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      |
| 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
----
~~~表示されればOK

# nova image-list
---
+--------------------------------------+-------------+--------+--------+
| ID                                   | Name        | Status | Server |
+--------------------------------------+-------------+--------+--------+
| aac7e922-8adc-458e-adbd-346c6456722c | Ubuntu12.10 | ACTIVE |        |
+--------------------------------------+-------------+--------+--------+
---

# nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
# nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
```

### horizon
```
# apt-get install -y memcached libapache2-mod-wsgi openstack-dashboard
# apt-get remove --purge -y openstack-dashboard-ubuntu-theme
~~~ubuntu独自のテーマを削除する

# cp -raf /etc/openstack-dashboard $BAK
# vi /etc/openstack-dashboard/local_settings.py 
---
CACHES = {
   'default': {
       'BACKEND' : 'django.core.cache.backends.memcached.MemcachedCache',
       'LOCATION' : '127.0.0.1:11211',
   }
}
...
OPENSTACK_HOST = "192.168.0.200"
---
~~~ /etc/memcached.confのhost, portと合っていることを確認する。
~~~ OPENSTACK_HOSTは、書き換える

# service apache2 restart
# service memcached restart
```

### cinder
```
# apt-get install -y cinder-api cinder-scheduler
# cp -raf /etc/cinder $BAK
# vi /etc/cinder/cinder.conf
---
...
sql_connection = mysql://cinder:password@192.168.0.200/cinder
...
---

# mysql -u root -p
 CREATE DATABASE cinder;
 GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'password';
 \q

# cinder-manage db sync
# keystone user-create --name=cinder --pass=password --email=cinder@example.com
# keystone user-role-add --user=cinder --tenant=service --role=admin
# vi /etc/cinder/api-paste.ini
---
...
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = cinder
admin_password = password
...
---

# vi /etc/cinder/cinder.conf
---
...
rpc_backend = cinder.openstack.common.rpc.impl_kombu
rabbit_host = 192.168.0.200
rabbit_port = 5672
rabbit_userid = guest
rabbit_password = password
...
---

# keystone service-create --name=cinder --type=volume --description="Cinder Volume Service"
# keystone endpoint-create  --service-id=61f1d98e0da54093a5d0dc3d300dd560  --publicurl=http://192.168.0.200:8776/v1/%\(tenant_id\)s  --internalurl=http://192.168.0.200:8776/v1/%\(tenant_id\)s  --adminurl=http://192.168.0.200:8776/v1/%\(tenant_id\)s
~~~ service-idは、keystone service-createのIDを入力する

# keystone service-create --name=cinderv2 --type=volumev2 --description="Cinder Volume Service V2"
# keystone endpoint-create  --service-id=d83270c3d807421081b8eb72a9607641 --publicurl=http://192.168.0.200:8776/v2/%\(tenant_id\)s  --internalurl=http://192.168.0.200:8776/v2/%\(tenant_id\)s  --adminurl=http://192.168.0.200:8776/v2/%\(tenant_id\)s
~~~ service-idは、keystone service-createのIDを入力する

# service cinder-scheduler restart
# service cinder-api restart

# apt-get install lvm2
# pvcreate /dev/sdb
# vgcreate cinder-volumes /dev/sdb
~~~事前にcinder-volumeを確保しておくこと。

# pvcreate /dev/sdb1
# vgcreate cinder-volumes /dev/sdb1

## Add a filter entry to the devices section /etc/lvm/lvm.conf to keep LVM from scanning devices used by virtual machines:
## これは保留で。

# apt-get install -y cinder-volume
# service cinder-volume restart
# service tgt restart
```

### 8. Add Object Storage
```
ひとまずスキップ
```

### neutron
```
# mysql -u root -p
mysql> CREATE DATABASE neutron;
mysql> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';
mysql> FLUSH PRIVILEGES;

# keystone tenant-list
# keystone role-list
~~~一覧をひとまず取得しておくだけでOK(admin, serviceがあるかどうかだけ)

# keystone user-create --name=neutron --pass=password --email=neutron@example.com
# keystone user-role-add --user=neutron --tenant=service --role=admin
# keystone service-create --name=neutron --type=network --description="OpenStack Networking Service"
# keystone endpoint-create  --service-id 90f117c0031149dd9454054ee8cdc729 --publicurl http://192.168.0.200:9696  --adminurl http://192.168.0.200:9696 --internalurl http://192.168.0.200:9696
~~~service_idへは、keystone service-list のneutronのidを入力する。


###Install Networking services on a dedicated network node
# apt-get install -y neutron-server neutron-dhcp-agent neutron-plugin-openvswitch-agent neutron-l3-agent
# apt-get install openvswitch-datapath-dkms

# BAK='/root/MAINTENANCE/20140113/bak'
# NEW='/root/MAINTENANCE/20140113/new'
# mkdir -p $BAK
# cp -p /etc/sysctl.conf $BAK
# vi /etc/sysctl.conf 
---
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
---
# sysctl -p
# service networking restart

# cp -raf /etc/neutron $BAK
# vi /etc/neutron/neutron.conf
---
[DEFAULT]
auth_strategy = keystone
rabbit_host = 192.168.0.200
rabbit_userid = guest
rabbit_password = password
...
[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = neutron
admin_password = password
signing_dir = $state_path/keystone-signing

[database]
connection = mysql://neutron:password@192.168.0.200/neutron
---

# vi /etc/neutron/api-paste.ini 
---
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 192.168.0.200
auth_uri = http://192.168.0.200:5000
admin_tenant_name = service
admin_user = neutron
admin_password = password
---

## install openvswitch plugin
# apt-get -y install neutron-plugin-openvswitch-agent openvswitch-switch
# vi /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini
---
[ovs]
tenant_network_type = gre
tunnel_id_ranges = 1:1000
enable_tunneling = True
integration_bridge = br-int
tunnel_bridge = br-tun
local_ip = 192.168.0.200
---

# ovs-vsctl add-br br-int
# ovs-vsctl add-br br-ex
# ovs-vsctl add-br br-tun
# ovs-vsctl add-port br-ex eth0
# cp -p /etc/network/interfaces $BAK
# vi /etc/network/interfaces
---
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet manual
 up ip address add 0/0 dev $IFACE
 up ip link set $IFACE up
 down ip link set $IFACE down

auto br-ex
iface br-ex inet static
 address 192.168.0.200
 netmask 255.255.255.0
 network 192.168.0.0
 gateway 192.168.0.254
 dns-nameservers 192.168.0.254
---

# vi /etc/neutron/l3_agent.ini
----
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True
----

# vi /etc/neutron/dhcp_agent.ini 
----
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True
----

# vi /etc/neutron/neutron.conf
----
core_plugin = neutron.plugins.openvswitch.ovs_neutron_plugin.OVSNeutronPluginV2
----

# vi /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini
----
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
----

# vi /etc/neutron/dhcp_agent.ini
----
[default]
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
----

# vi /etc/nova/nova.conf
----
neutron_metadata_proxy_shared_secret = password
service_neutron_metadata_proxy = true
----
# service nova-api restart

# vi /etc/neutron/metadata_agent.ini 
----
auth_url = http://192.168.0.200:5000/v2.0
auth_region = regionOne
admin_tenant_name = service
admin_user = neutron
admin_password = password
nova_metadata_ip = 192.168.0.200
metadata_proxy_shared_secret = password
----

# service neutron-server restart
# service neutron-dhcp-agent restart
# service neutron-l3-agent restart
# service neutron-metadata-agent restart
# service neutron-plugin-openvswitch-agent restart

#-- create subnet
# neutron net-create ext-net -- --router:external=True
# neutron subnet-create ext-net 192.168.0.0/24 --allocation-pool start=192.168.0.100,end=192.168.0.150  --gateway=192.168.0.254 --enable_dhcp=False

#-- for DEMO
# keystone tenant-create --name demo
# keystone tenant-list | grep demo | awk '{print $2;}'

# keystone user-create --name=demo --pass=password --email=demo@example.com
# keystone role-create --name=demo
# keystone user-role-add --user=demo --tenant=demo --role=demo

# neutron router-create ext-to-int --tenant-id 951d41b536124106958dad93ef99d319
~~~tenant-idは、keystone-tenant-listで出力したID

# neutron router-gateway-set 57c8f6a5-3c10-4c0b-82b7-de8913f291d4  be932ed7-63da-427b-a3e4-8836f4ba9d86
~~~ neutron router-createのid, neutron net-listのID

#-- create demo-net
# neutron net-create --tenant-id 951d41b536124106958dad93ef99d319 demo-net
~~~keystone tenant-listのID

# neutron subnet-create --tenant-id 951d41b536124106958dad93ef99d319 demo-net 10.5.5.0/24 --gateway 10.5.5.1 --dns_nameservers list=true 192.168.0.254

# neutron router-interface-add 57c8f6a5-3c10-4c0b-82b7-de8913f291d4 96edee0c-86d5-47e8-8188-49e4076a15ef 
~~~neutron router-createのid, neutron subnet-list のdemonetのID


#--  Install networking support on a dedicated controller node
# apt-get -y install neutron-server
# vi /etc/neutron/neutron.conf
----
auth_url = http://192.168.0.200:35357/v2.0
rpc_backend = neutron.openstack.common.rpc.impl_kombu
rabbit_host = 192.168.0.200
rabbit_port = 5672
rabbit_password = password
----

# vi /etc/nova/nova.conf
----
network_api_class=nova.network.neutronv2.api.API
neutron_url=http://192.168.0.200:9696
neutron_auth_strategy=keystone
neutron_admin_tenant_name=service
neutron_admin_username=neutron
neutron_admin_password=password
neutron_admin_auth_url=http://192.168.0.200:35357/v2.0
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver=nova.virt.firewall.NoopFirewallDriver
security_group_api=neutron
----
# service neutron-server restart

#--  GRE tunneling network options
# vi /etc/neutron/l3_agent.ini
---
gateway_external_network_id = be932ed7-63da-427b-a3e4-8836f4ba9d86
router_id = 57c8f6a5-3c10-4c0b-82b7-de8913f291d4
---
~~~ gateway_external_network_id  は、neutron net-listのext-netのid
~~~ router_idは、neutron router-list のid

# service neutron-l3-agent restart
```

