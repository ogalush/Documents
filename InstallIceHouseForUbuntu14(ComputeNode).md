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
# cp -p /etc/network/interfaces $BAK
# cp -p /etc/sysctl.conf $BAK
# vi /etc/sysctl.conf
----
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
----
# sysctl -p

パッケージインストール
# apt-get -y install neutron-common neutron-plugin-ml2 neutron-plugin-openvswitch-agent openvswitch-datapath-dkms
```

neutron設定
```
# cp -raf /etc/neutron $BAK
# vi /etc/neutron/neutron.conf
----
[DEFAULT]
...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
...
auth_strategy = keystone
...
rpc_backend = neutron.openstack.common.rpc.impl_kombu
rabbit_host = 192.168.0.200
rabbit_user = guest
rabbit_password = admin!
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

[service_providers]
###service_provider=LOADBALANCER:Haproxy:neutron.services.loadbalancer.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
###service_provider=VPN:openswan:neutron.services.vpn.service_drivers.ipsec.IPsecVPNDriver:default
~~~★ServiceProviderセクションの設定値をすべてコメントアウトする（重要。行っていないとcompute controller間で通信できない）
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
[ovs]
local_ip = 192.168.0.210
tunnel_type = gre
enable_tunneling = True
...
[database]
connection = mysql://neutron:password@192.168.0.200/neutron
----

neutron再起動
# service openvswitch-switch restart
# ovs-vsctl add-br br-int

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
# service nova-compute restart
# service neutron-plugin-openvswitch-agent restart
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

環境設定ファイル作成
```
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
```
