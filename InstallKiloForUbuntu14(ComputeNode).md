<!--
************************************************************
OpenStack KiloをUbuntu14.04(x86_64)へインストールする手順
参照元: http://docs.openstack.org/kilo/install-guide/install/apt/content/  
Copyright (c) Takehiko OGASAWARA 2015 All Rights Reserved.
************************************************************
-->

# OpenStack(Kilo)のComputeNodeをインストールする
[公式インストール手順](http://docs.openstack.org/kilo/install-guide/install/apt/content/)

## Controller node
### 準備
バックアップディレクトリ作成
```
$ sudo mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
$ BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

### Basicインストール
CPU GOVERNER変更  
[Performanceへ変更](https://github.com/ogalush/documents/blob/master/setCPUPerformance.md)  

ntpインストール
```
$ sudo apt-get -y install ntp
$ ntpq -p
→ 同期され始めているようであればOK.
```

OpenStackパッケージ
```
$ sudo apt-get -y install ubuntu-cloud-keyring
$ echo 'deb http://ubuntu-cloud.archive.canonical.com/ubuntu trusty-updates/kilo main' | sudo tee /etc/apt/sources.list.d/cloudarchive-kilo.list
$ cat /etc/apt/sources.list.d/cloudarchive-kilo.list
$ sudo apt-get -y update && sudo apt-get -y update && sudo apt-get -y dist-upgrade
```

### nova

novaインストール
```
$ sudo apt-get -y install nova-compute sysfsutils
```

nova設定
```
$ sudo cp -raf /etc/nova $BAK
$ sudo vi /etc/nova/nova.conf
---
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone
my_ip = 192.168.0.210
vnc_enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = 192.168.0.210
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html

[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = nova
password = password

[glance]
host = 192.168.0.200

[oslo_messaging_rabbit]
rabbit_host = 192.168.0.200
rabbit_userid = openstack
rabbit_password = password

[oslo_concurrency]
lock_path = /var/lib/nova/tmp
---

$ sudo /etc/nova/nova-compute.conf
---
[libvirt]
virt_type=kvm
---

$ sudo service nova-compute restart
$ sudo rm -f /var/lib/nova/nova.sqlite
```

確認
```
$ sudo nova-manage service list
[sudo] password for ogalush: 
No handlers could be found for logger "oslo_config.cfg"
Binary           Host                                 Zone             Status     State Updated_At
nova-cert        ryunosuke                            internal         enabled    :-)   2015-06-07 08:24:26
nova-conductor   ryunosuke                            internal         enabled    :-)   2015-06-07 08:24:27
nova-consoleauth ryunosuke                            internal         enabled    :-)   2015-06-07 08:24:27
nova-scheduler   ryunosuke                            internal         enabled    :-)   2015-06-07 08:24:27
nova-compute     ryunosuke                            nova             enabled    :-)   2015-06-07 08:24:20
nova-compute     hayao                                nova             enabled    :-)   2015-06-07 08:24:22
~~~ComputeNode(hayao)が追加されればOK.
```


### neutron

neutronパッケージ
```
$ sudo cp /etc/sysctl.conf $BAK
$ sudo vi /etc/sysctl.conf
---
...
#-- for OpenStack
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
---

$ sudo sysctl -p
$ sudo apt-get -y install neutron-plugin-ml2 neutron-plugin-openvswitch-agent
```

neutron設定
```
$ sudo cp -raf /etc/neutron $BAK
$ sudo vi /etc/neutron/neutron.conf
---
[DEFAULT]
rpc_backend = rabbit
auth_strategy = keystone
...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True

[oslo_messaging_rabbit]
rabbit_host = 192.168.0.200
rabbit_userid = openstack
rabbit_password = password

[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = password
---
```

ml2設定
```
$ sudo vi /etc/neutron/plugins/ml2/ml2_conf.ini
---
[ml2]
type_drivers = flat,vlan,gre,vxlan
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

[ovs]
local_ip = 192.168.0.210

[agent]
tunnel_types = gre
---

$ sudo service openvswitch-switch restart
```

nova設定追加
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
$ sudo service nova-compute restart
$ sudo service neutron-plugin-openvswitch-agent restart
```

反映確認
```
$ source ~/admin-openrc.sh
$ neutron agent-list
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| id                                   | agent_type         | host      | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| 64b348ca-4c6d-4ae7-9831-5cf80b6b2e24 | Metadata agent     | ryunosuke | :-)   | True           | neutron-metadata-agent    |
| 7def7387-4de2-4997-8de3-8fbae4fb67d6 | Open vSwitch agent | hayao     | :-)   | True           | neutron-openvswitch-agent |
| 82fbf144-6806-467a-b53a-dff6ad71dac8 | Open vSwitch agent | ryunosuke | :-)   | True           | neutron-openvswitch-agent |
| c5bd9b07-87e5-4877-aa6d-56ad61d5c8ac | DHCP agent         | ryunosuke | :-)   | True           | neutron-dhcp-agent        |
| dfc034fb-b5e7-42eb-a08e-788c89219eb0 | L3 agent           | ryunosuke | :-)   | True           | neutron-l3-agent          |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
→ ComputeNode(hayao)のOpen vSwitch agentが入っていればOK.
```
