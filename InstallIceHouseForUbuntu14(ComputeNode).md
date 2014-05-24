<!--
************************************************************
OpenStack IceHouseをUbuntu14.04(x86_64)へインストールする手順
参照元: http://docs.openstack.org/icehouse/install-guide/install/apt/content/
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->

# ComputeNodeをインストールする方法(OpenStack IceHouse)

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

MySQLアクセス用クライアントのインストール
```
# apt-get -y install python-mysqldb
```

### novaインストール
```
# apt-get -y install nova-compute-kvm python-guestfs
```
nova設定
```
# cp -raf /etc/nova $BAK

セキュリティ対応。novaからカーネルを参照できるようにしてる模様。
# dpkg-statoverride  --update --add root root 0644 /boot/vmlinuz-$(uname -r)
# vi /etc/kernel/postinst.d/statoverride
----
#!/bin/sh
version="$1"
# passing the kernel version is required
[ -z "${version}" ] && exit 0
dpkg-statoverride --update --add root root 0644 /boot/vmlinuz-${version}
----
# chmod +x /etc/kernel/postinst.d/statoverride

nova.conf設定
# vi /etc/nova/nova.conf 
----
[DEFAULT]
...
auth_strategy = keystone
...
rpc_backend = rabbit
rabbit_host = controller
rabbit_password = RABBIT_PASS
...
my_ip = 192.168.0.210
vnc_enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = 192.168.0.210
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html
vnc_keymap=ja
...
glance_host = 192.168.0.200

[database]
# The SQLAlchemy connection string used to connect to the database
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

# /etc/nova/nova-compute.conf f
----
[libvirt]
virt_type=kvm
----

不要ファイル削除
# rm /var/lib/nova/nova.sqlite

再起動
# service nova-compute restart

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
# neutron subnet-create --tenant-id=461247d27c674fff9f1decb6330a2513 demo-net --name demo-subnet --dns_nameservers list=true 8.8.8.8  --gateway 10.0.0.1 10.0.0.0/24

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


