<!--
************************************************************
OpenStack JunoをUbuntu14.04(x86_64)へインストールする手順(ComputeNode)
参照元: http://docs.openstack.org/juno/install-guide/install/apt/content/ 
Copyright (c) Takehiko OGASAWARA 2015 All Rights Reserved.
************************************************************
-->

# OpenStack(Juno) ComputeNodeをインストールする
[公式インストール手順](http://docs.openstack.org/juno/install-guide/install/apt/content/)

## Compute node
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

MariaDB(MySQL)クライアント
```
$ sudo apt-get -y install python-mysqldb
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
rabbit_host = 192.168.0.200
rabbit_user = guest
rabbit_password = admin!
auth_strategy = keystone
my_ip = 192.168.0.210
vnc_enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = 192.168.0.210
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html

[database]
connection = mysql://nova:password@192.168.0.200/nova

[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = password

[glance]
host = 192.168.0.200
---

$ sudo vi /etc/nova/nova-compute.conf
---
[libvirt]
virt_type=kvm
---

$ sudo rm -f /var/lib/nova/nova.sqlite
```

nova再起動
```
$ sudo service nova-compute restart
```


nova確認
```
ComputeNodeのnova-computeがUPしていればOK.
controller$ nova service-list
---
ogalush@ryunosuke:~$ nova service-list
+----+------------------+-----------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host      | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+-----------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-cert        | ryunosuke | internal | enabled | up    | 2015-03-09T14:45:56.000000 | -               |
| 2  | nova-consoleauth | ryunosuke | internal | enabled | up    | 2015-03-09T14:45:53.000000 | -               |
| 3  | nova-conductor   | ryunosuke | internal | enabled | up    | 2015-03-09T14:45:52.000000 | -               |
| 4  | nova-scheduler   | ryunosuke | internal | enabled | up    | 2015-03-09T14:45:54.000000 | -               |
| 5  | nova-compute     | ryunosuke | nova     | enabled | up    | 2015-03-09T14:45:55.000000 | -               |
| 6  | nova-compute     | hayao     | nova     | enabled | up    | 2015-03-09T14:45:47.000000 | -               |
+----+------------------+-----------+----------+---------+-------+----------------------------+-----------------+
---


### neutron
バックアップ
```
$ sudo cp -p /etc/network/interfaces $BAK
$ sudo cp -p /etc/sysctl.conf $BAK
```

sysctl設定
```
$ sudo vi /etc/sysctl.conf
---
#-- for OpenStack
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
---
$ sudo sysctl -p
```

パッケージ
```
$ sudo apt-get -y install neutron-plugin-ml2 neutron-plugin-openvswitch-agent
```

neutron設定
```
$ sudo cp -raf /etc/neutron $BAK
$ sudo vi /etc/neutron/neutron.conf
---
[DEFAULT]
...
service_plugins = router
allow_overlapping_ips = True
rpc_backend = neutron.openstack.common.rpc.impl_kombu
rabbit_host = 192.168.0.200
rabbit_user = guest
rabbit_password = admin!
auth_strategy = keystone
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_host = 192.168.0.200
auth_protocol = http
auth_port = 35357
admin_tenant_name = service
admin_user = neutron
admin_password = password
...
[database]
connection = mysql://neutron:password@192.168.0.200/neutron
---


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
...
[ovs]
local_ip = 192.168.0.210
enable_tunneling = True

[agent]
tunnel_types = gre 
---

$ sudo service openvswitch-switch restart
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

neutron, nova再起動
```
$ sudo service nova-compute restart
$ sudo service openvswitch-switch restart
$ sudo service neutron-plugin-openvswitch-agent restart
```

確認
```
ComputeNodeのagentがUPしていればOK
controller$ neutron agent-list
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| id                                   | agent_type         | host      | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| 00326010-3775-47cb-ba35-d53d74d5135d | DHCP agent         | ryunosuke | :-)   | True           | neutron-dhcp-agent        |
| 28135e73-d56e-4e82-a1ce-02ad37e4b2d8 | Metadata agent     | ryunosuke | :-)   | True           | neutron-metadata-agent    |
| 61e5dfcd-cf6f-4f8e-880f-60cdb32aff9e | L3 agent           | ryunosuke | :-)   | True           | neutron-l3-agent          |
| b5b1bd30-109c-44fb-8a81-c6ca4bccf187 | Open vSwitch agent | hayao     | :-)   | True           | neutron-openvswitch-agent |
| b63336e0-d292-433e-b8da-1688efef7a07 | Open vSwitch agent | ryunosuke | :-)   | True           | neutron-openvswitch-agent |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+

controller$ nova service-list
+----+------------------+-----------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host      | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+-----------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-cert        | ryunosuke | internal | enabled | up    | 2015-03-09T15:16:17.000000 | -               |
| 2  | nova-consoleauth | ryunosuke | internal | enabled | up    | 2015-03-09T15:16:13.000000 | -               |
| 3  | nova-conductor   | ryunosuke | internal | enabled | up    | 2015-03-09T15:16:23.000000 | -               |
| 4  | nova-scheduler   | ryunosuke | internal | enabled | up    | 2015-03-09T15:16:15.000000 | -               |
| 5  | nova-compute     | ryunosuke | nova     | enabled | up    | 2015-03-09T15:16:15.000000 | -               |
| 6  | nova-compute     | hayao     | nova     | enabled | up    | 2015-03-09T15:16:18.000000 | -               |
+----+------------------+-----------+----------+---------+-------+----------------------------+-----------------+
```


ovs確認
```
$ sudo ovs-vsctl list-br
→ br-int, br-tunがあること
```

ブラウザアクセス
[horizon](http://192.168.0.200/horizon/)
※ComputeNodeでインスタンスを作成でき、sshで疎通できればOK.


◆今後の課題 2015.3◆
・DVRを使ってないLegacy手順なので、DVRを試してみる(ComputeNodeにもRouterを置いて効率よくする)
  https://wiki.openstack.org/wiki/Neutron/DVR
  http://www.slideshare.net/ToruMakabe/20-openstack-neutron-deep-dive-dvr
・Cinder Volumeを作る(Dashboardで軽微なエラー表示がでる)
