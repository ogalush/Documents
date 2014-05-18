<!--
************************************************************
OpenStack IceHouseをUbuntu14.04(x86_64)へインストールする手順
参照元: http://docs.openstack.org/icehouse/install-guide/install/apt/content/
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->

# ControllNodeをインストールする方法(OpenStack IceHouse)

### 準備
バックアップディレクトリ作成
```
mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

ntpインストール
```
# apt-get -y install ntp
```

MySQLインストール
```
# apt-get -y install python-mysqldb mysql-server
→ パスワード: admin!
```

MySQL設定
```
# cp -raf /etc/mysql $BAK
# vi /etc/mysql/my.cnf
----
[mysqld]
...
bind-address            = 192.168.0.200
...
default-storage-engine = innodb
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
----

#-- 設定の反映
# service mysql restart

#-- 初期化
# mysql_install_db
# mysql_secure_installation
→ rootのパスワード変更等聞かれるが、後でnovaユーザ等を作るのでひとまずNoで良い。
```

RabbitMQインストール
```
# apt-get install -y rabbitmq-server
# rabbitmqctl change_password guest admin!
```


### KeyStoneインストール
```
# apt-get -y install keystone
```

KeyStone設定
```
# cp -raf /etc/keystone $BAK
# vi /etc/keystone/keystone.conf
----
### connection = sqlite:////var/lib/keystone/keystone.db
connection = mysql://keystone:password@192.168.0.200/keystone
----

#-- sqliteファイル削除
# rm /var/lib/keystone/keystone.db

#-- keystoneユーザ作成
# mysql -u root -p

mysql> CREATE DATABASE keystone;
mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
mysql> FLUSH PRIVILEGES;
mysql> ¥q

#-- keystoneDB初期化
# su -s /bin/sh -c "keystone-manage db_sync" keystone

#-- keystoneパスワード反映
# vi /etc/keystone/keystone.conf
----
[default]
# admin_token=ADMIN
admin_token=password
...
log_dir = /var/log/keystone
----

#-- 設定反映
# service keystone restart
# service keystone status

#-- 有効期限が切れたtokenの掃除スクリプト
# (crontab -l 2>&1 | grep -q token_flush) || \
echo '@hourly /usr/bin/keystone-manage token_flush >/var/log/keystone/keystone-tokenflush.log 2>&1' >> /var/spool/cron/crontabs/root

# cat /var/spool/cron/crontabs/root 
→設定が追加されていること。

#-- テナント作成
# export OS_SERVICE_TOKEN=password
# export OS_SERVICE_ENDPOINT=http://192.168.0.200:35357/v2.0

#-- adminテナント
# keystone user-create --name=admin --pass=password --email=admin@192.168.0.200
# keystone role-create --name=admin
# keystone tenant-create --name=admin --description="Admin Tenant"
# keystone user-role-add --user=admin --tenant=admin --role=admin
# keystone user-role-add --user=admin --role=_member_ --tenant=admin

#-- demoテナント
# keystone user-create --name=demo --pass=password --email=demo@192.168.0.200
# keystone tenant-create --name=demo --description="Demo Tenant"
# keystone user-role-add --user=demo --role=_member_ --tenant=demo

#-- Serviceテナント
# keystone tenant-create --name=service --description="Service Tenant"

#-- サービス作成
# keystone service-create --name=keystone --type=identity --description="OpenStack Identity"
# keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ identity / {print $2}') \
  --publicurl=http://192.168.0.200:5000/v2.0 \
  --internalurl=http://192.168.0.200:5000/v2.0 \
  --adminurl=http://192.168.0.200:35357/v2.0

#-- KeyStoneインストール確認
# unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT
# keystone --os-username=admin --os-password=password --os-auth-url=http://192.168.0.200:35357/v2.0 token-get
→ tokenが表示されればOK

# keystone --os-username=admin --os-password=password --os-tenant-name=admin --os-auth-url=http://192.168.0.200:35357/v2.0 token-get
→ tokenが表示されればOK

#-- 環境変数設定
# vi ~/.admin-openrc.sh
----
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_TENANT_NAME=admin
export OS_AUTH_URL=http://192.168.0.200:35357/v2.0
----
# echo 'source ~/.admin-openrc.sh' >> ~/.bashrc
# source ~/.bashrc

#-- 環境変数設定後の確認
# keystone token-get
→ tokenが表示されればOK

# keystone user-list
→ admin,demoが表示されればOK

# keystone user-role-list --user admin --tenant admin
→ admin, _member_が表示されればOK
```

### Python Clientのインストール
```
# apt-get -y install python-pip
```


### Glanceインストール
```
# apt-get -y install glance python-glanceclient
```

Glance設定
```
# cp -raf /etc/glance $BAK
# vi /etc/glance/glance-api.conf
----
[DEFAULT]
rpc_backend = rabbit
rabbit_host = 192.168.0.200
rabbit_userid = guest
rabbit_password = admin!
...
[database]
connection = mysql://glance:password@192.168.0.200/glance
##sqlite_db = /var/lib/glance/glance.sqlite
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = password
...
[paste_deploy]
flavor = keystone
...
----

# vi /etc/glance/glance-registry.conf
----
[database]
connection = mysql://glance:password@192.168.0.200/glance
##sqlite_db = /var/lib/glance/glance.sqlite
----

#-- 使用しないDBファイル削除
# rm /var/lib/glance/glance.sqlite

#-- glanceユーザ作成
# mysql -u root -p
mysql> CREATE DATABASE glance;
mysql> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
mysql> FLUSH PRIVILEGES;
mysql> ¥q

#-- DB初期化
# su -s /bin/sh -c "glance-manage db_sync" glance

#-- KeyStoneユーザ作成
# keystone user-create --name=glance --pass=password --email=glance@192.168.0.200
# keystone user-role-add --user=glance --tenant=service --role=admin

#-- KeyStoneサービス作成
# keystone service-create --name=glance --type=image --description="OpenStack Image Service"
# keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ image / {print $2}') \
  --publicurl=http://192.168.0.200:9292 \
  --internalurl=http://192.168.0.200:9292 \
  --adminurl=http://192.168.0.200:9292

# service glance-registry restart
# service glance-api restart

#-- イメージの登録 (ubuntu14.04 cloud image)
# cd /usr/local/src
# wget http://cloud-images.ubuntu.com/releases/14.04/release/ubuntu-14.04-server-cloudimg-amd64-disk1.img
# glance image-create --name="ubuntu14.04" --disk-format=qcow2 --container-format=bare --is-public=true --progress < /usr/local/src/ubuntu-14.04-server-cloudimg-amd64-disk1.img
# glance image-list
→イメージが表示されればOK
```

### novaインストール
```
# apt-get -y install nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy nova-scheduler python-novaclient
```
nova設定
```
# cp -raf /etc/nova $BAK
# vi /etc/nova/nova.conf
----
[DEFAULT]
...
rpc_backend = rabbit
rabbit_host = 192.168.0.200
rabbit_password = admin!
...
my_ip = 192.168.0.200
novnc_enable = true
novncproxy_port = 6080
novncproxy_host = 192.168.0.200
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html
vncserver_listen = 192.168.0.200
vncserver_proxyclient_address = 192.168.0.200
vnc_keymap=ja
...
auth_strategy = keystone
glance_host = 192.168.0.200
...
[database]
connection = mysql://nova:password@192.168.0.200/nova
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = password
----

#-- 使用しないDBファイルの削除
# rm /var/lib/nova/nova.sqlite

#-- DBユーザ作成
$ mysql -u root -p
mysql> CREATE DATABASE nova;
mysql> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
mysql> FLUSH PRIVILEGES;
mysql> ¥q

#-- DB初期化
# su -s /bin/sh -c "nova-manage db sync" nova

#-- KeyStoneユーザ作成
# keystone user-create --name=nova --pass=password --email=nova@192.168.0.200
# keystone user-role-add --user=nova --tenant=service --role=admin

#-- KeyStoneサービス作成
$ keystone service-create --name=nova --type=compute --description="OpenStack Compute"
$ keystone endpoint-create  \
  --service-id=$(keystone service-list | awk '/ compute / {print $2}') \
  --publicurl=http://192.168.0.200:8774/v2/%\(tenant_id\)s \
  --internalurl=http://192.168.0.200:8774/v2/%\(tenant_id\)s \
  --adminurl=http://192.168.0.200:8774/v2/%\(tenant_id\)s

#-- サービス再起動
# service nova-api restart
# service nova-cert restart
# service nova-consoleauth restart
# service nova-scheduler restart
# service nova-conductor restart
# service nova-novncproxy restart

#-- 確認
# nova image-list
→ glance登録時のイメージ（ubuntu14.04）が表示されればOK
```

nova設定(2)
```
# apt-get -y install nova-compute-kvm python-guestfs

# dpkg-statoverride  --update --add root root 0644 /boot/vmlinuz-$(uname -r)
→ 一般ユーザがKernelファイルへアクセスできるよう、権限を緩くする.
  https://bugs.launchpad.net/ubuntu/+source/linux/+bug/759725

# touch /etc/kernel/postinst.d/statoverride
# chmod +x /etc/kernel/postinst.d/statoverride
# vi /etc/kernel/postinst.d/statoverride
----
#!/bin/sh
version="$1"
# passing the kernel version is required
[ -z "${version}" ] && exit 0
dpkg-statoverride --update --add root root 0644 /boot/vmlinuz-${version}
----
→ Kernel version up後も適用できるようにしている。

#-- compute設定
# vi /etc/nova/nova-compute.conf 
----
[libvirt]
virt_type=qemu
~~~★変更する
----

#-- compute再起動
# service nova-compute restart
# service nova-compute status
```


### neutronインストール
```
#-- MySQLユーザ追加
# mysql -u root -p
mysql> CREATE DATABASE neutron;
mysql> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';
mysql> FLUSH PRIVILEGES;
mysql> ¥q

#-- KeyStoneユーザ作成
# keystone user-create --name neutron --pass password --email neutron@192.168.0.200
# keystone user-role-add --user=neutron --tenant=service --role=admin
# keystone service-create --name neutron --type network --description "OpenStack Networking"
# keystone endpoint-create \
  --service-id $(keystone service-list | awk '/ network / {print $2}') \
  --publicurl http://192.168.0.200:9696 \
  --adminurl http://192.168.0.200:9696 \
  --internalurl http://192.168.0.200:9696

#-- パッケージインストール
# apt-get -y install neutron-server neutron-plugin-ml2
```

neutron設定
```
# cp -raf /etc/neutron $BAK
# vi /etc/neutron/neutron.conf
----
[DEFAULT]
...
auth_strategy = keystone
rpc_backend = neutron.openstack.common.rpc.impl_kombu
rabbit_host = 192.168.0.200
rabbit_user = guest
rabbit_password = admin!
...

notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
nova_url = http://192.168.0.200:8774/v2
nova_admin_username = nova
nova_admin_tenant_id = 5b22bf7cb4e34e23bc60e2f23248747b
~~~★ keystone tenant-get serviceのID
nova_admin_password = password
nova_admin_auth_url = http://192.168.0.200:35357/v2.0
...
###core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin
core_plugin = ml2
~~~★書き換える
service_plugins = router
allow_overlapping_ips = True
...
[database]
##connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql://neutron:password@192.168.0.200/neutron
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_host = 192.168.0.200
auth_protocol = http
auth_port = 35357
admin_tenant_name = service
admin_user = neutron
admin_password = password
signing_dir = $state_path/keystone-signing
----

#-- ml2 plugin設定
# vi /etc/neutron/plugins/ml2/ml2_conf.ini
----
[ml2]
type_drivers = gre
tenant_network_types = gre
mechanism_drivers = openvswitch
...
[ml2_type_gre]
tunnel_id_ranges = 1:1000
...
[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
...
----

#-- nova側の設定更新
# vi /etc/nova/nova.conf
----
[DEFAULT]
...
network_api_class = nova.network.neutronv2.api.API
neutron_url = http://192.168.0.200:9696
neutron_auth_strategy = keystone
neutron_admin_tenant_name = service
neutron_admin_username = neutron
neutron_admin_password = password
neutron_admin_auth_url = http://192.168.0.200:35357/v2.0
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
security_group_api = neutron
----

#-- サービス再起動
# service nova-api restart
# service nova-scheduler restart
# service nova-conductor restart
# service neutron-server restart
```

newtron設定2 (for Network node)
```
#-- sysctl変更
# cp -p /etc/sysctl.conf $BAK
# vi /etc/sysctl.conf $BAK
----
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
----
# sysctl -p

#-- Network node向けパッケージのインストール
# apt-get -y install neutron-plugin-ml2 neutron-plugin-openvswitch-agent openvswitch-datapath-dkms neutron-l3-agent neutron-dhcp-agent

#-- Plug-in設定
# vi /etc/neutron/l3_agent.ini 
----
[DEFAULT]
...
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True
...
----

# vi /etc/neutron/dhcp_agent.ini
----
[DEFAULT]
....
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
use_namespaces = True
----

# vi /etc/neutron/metadata_agent.ini
----
auth_url = http://192.168.0.200:5000/v2.0
~~~★置き換える
auth_region = regionOne
admin_tenant_name = service
admin_user = neutron
admin_password = password
nova_metadata_ip = 192.168.0.200
metadata_proxy_shared_secret = password
...
verbose = True
----

# vi /etc/nova/nova.conf
----
service_neutron_metadata_proxy = true
neutron_metadata_proxy_shared_secret = password
----

#-- nova-api反映
# service nova-api restart
```

ml2プラグイン設定
```
# vi /etc/neutron/plugins/ml2/ml2_conf.ini
----
...
[ovs]
local_ip = 192.168.0.200
tunnel_type = gre
enable_tunneling = True
...
----

#-- サービス再起動
# service openvswitch-switch restart
# service openvswitch-switch status
```

NIC設定
```
# ovs-vsctl add-br br-int
# ovs-vsctl add-br br-ex
# ovs-vsctl add-port br-ex p1p1
→ 外部ネットワーク向けの通信 192.168.0.0/24のセグメント
# ethtool -K INTERFACE_NAME gro off
```

neutronサービス反映
```
# service neutron-plugin-openvswitch-agent restart
# service neutron-l3-agent restart
# service neutron-dhcp-agent restart
# service neutron-metadata-agent restart
```

NIC設定
```
openvswitchを有効にすると物理NIC(p1p1)から接続できなくなるため、br-exにIPアドレスを当てる。
# vi /etc/network/interfaces
----
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
#-- for Physical Interface
auto p1p1
iface p1p1 inet manual
 up ip address add 0/0 dev $IFACE
 up ip link set $IFACE up
 down ip link set $IFACE down

#-- for Virtual Interface(OVS)
auto br-ex
iface br-ex inet static
 address 192.168.0.200
 netmask 255.255.255.0
 network 192.168.0.0
 gateway 192.168.0.254
 dns-nameservers 218.176.253.97
----

#-- IPアドレス設定反映
# sync;sync;sync; shutdown -r now
→ 再起動まで待つ。
```

外部/内部ネットワーク作成
```
#-- 外部ネットワーク
# neutron net-create ext-net --shared --router:external=True
# neutron net-list
→ 表示されればOK
# neutron subnet-create ext-net --name ext-subnet \
  --allocation-pool start=192.168.0.100,end=192.168.0.149 \
  --disable-dhcp --gateway 192.168.0.254 192.168.0.0/24

#-- 内部ネットワーク
# keystone tenant-list
+----------------------------------+---------+---------+
|                id                |   name  | enabled |
+----------------------------------+---------+---------+
| 461247d27c674fff9f1decb6330a2513 |   demo  |   True  |
...
+----------------------------------+---------+---------+

# neutron net-create --tenant-id=461247d27c674fff9f1decb6330a2513 demo-net
~~~★tenant-id=demoユーザのtenant-idとする。
# neutron subnet-create --tenant-id=461247d27c674fff9f1decb6330a2513 demo-net --name demo-subnet  --gateway 10.0.0.1 10.0.0.0/24

#-- 内部ネットワーク用Router
# neutron router-create --tenant-id=461247d27c674fff9f1decb6330a2513 demo-router
# neutron router-interface-add demo-router demo-subnet
# neutron router-gateway-set demo-router ext-net
```

### Horizonインストール
```
# apt-get -y install apache2 memcached libapache2-mod-wsgi openstack-dashboard
# apt-get -y remove --purge openstack-dashboard-ubuntu-theme
```

Horizon設定
```
# cp -raf /etc/openstack-dashboard $BAK
# vi /etc/openstack-dashboard/local_settings.py
----
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {   
  'default': {
  'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache'
  'LOCATION': '192.168.0.200:11211',
  }
} 
...
OPENSTACK_HOST = "192.168.0.200"
~~~★変更する
----
```

Horizon反映
```
# service apache2 restart
# service memcached restart

#-- アクセス確認
http://192.168.0.200/horizon/

#-- memcached設定
# cp -p /etc/memcached.conf $BAK
# vi /etc/memcached.conf
----
## -l 127.0.0.1
-l 0.0.0.0
----
# service memcached restart
```


