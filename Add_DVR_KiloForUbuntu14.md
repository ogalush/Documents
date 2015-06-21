# OpenStack(Kilo)へDVR(分散ルータ)を入れる.
## どういうもの？
 NetworkNodeの単一障害点を極力解消するために、ComputeNodeにも仮想ルータを入れる機能.  
  
* [High Availability using Distributed Virtual Routing (DVR)](http://docs.openstack.org/networking-guide/deploy_scenario2.html)  
* [Neutronの新機能！DVRとL3HA](http://www.school.ctc-g.co.jp/columns/nakai/nakai57.html)  
* [DVR解説by HP](http://www.slideshare.net/ToruMakabe/20-openstack-neutron-deep-dive-dvr)  
* [DVR公式Doc](http://www.slideshare.net/ToruMakabe/20-openstack-neutron-deep-dive-dvr)  
* [DVR Configuration](http://docs.openstack.org/kilo/config-reference/content/networking-options-dvr.html)  
* [マルチキャストアドレス(VXLAN向け)](http://www.infraexpert.com/study/multicast2.htm)

## 前提
OpenStackのインストールを通常通り行っており、仮想インスタンスで通信ができていること.

#手順
 テナントネットワークは[OpenvSwitch + VXLANが必須](http://docs.openstack.org/networking-guide/deploy_scenario2.html)の模様.
```
Warning
Proper operation of DVR requires Open vSwitch 2.1 or newer and VXLAN requires kernel 3.13 or newer. Also, the Kilo release increases stability and reliability of DVR considerably over the Juno release.
```

## 既存ネットワーク・サブネットの削除
設定を変えるので既存設定を削除しておく.

### 仮想Router削除
```
$ neutron router-list
| cc603ad2-2b4a-40e1-9d95-f851cca9c79a | demo-router | {"network_id": "eeaab19f-2fc6-4a3b-a745-bb2fcd4923e6", "enable_snat": true, "external_fixed_ips": [{"subnet_id": "a7e78785-7600-4200-b993-7eb3af222990", "ip_address": "192.168.0.100"}]} | False       | False |

$ neutron router-gateway-clear cc603ad2-2b4a-40e1-9d95-f851cca9c79a
Removed gateway from router cc603ad2-2b4a-40e1-9d95-f851cca9c79a

$ neutron router-port-list demo-router
 →表示されたsubnetにInterfaceが残ってるので削除する.

$ neutron  router-interface-delete demo-router  demo-subnet
Removed interface from router demo-router.

$ neutron router-list
$ neutron router-delete demo-router
```

### 仮想サブネット削除
GRE設定になっているサブネットを削除する(VXLANへ変更するため)
```
$ neutron subnet-list
$ neutron subnet-delete demo-subnet
$ neutron net-delete demo-net
Deleted network: demo-net
```

#DVR設定
## バックアップ
```
$ BAK=/root/MAINTENANCE/`date '+%Y%m%d'`/bak
$ sudo cp -raf /etc/neutron $BAK
$ sudo cp -p /etc/sysctl.conf $BAK
$ sudo cp -p /etc/network/interfaces $BAK
```

## Controller/Network Node
### 環境変数
```
$ sudo vi /etc/sysctl.conf
---
net.ipv4.ip_forward=1
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.all.rp_filter=0
---
$ sudo sysctl -p
```

### 設定
#### neutron
```
$ sudo vi /etc/neutron/neutron.conf
---
[DEFAULT]
router_distributed = True
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
allow_automatic_l3agent_failover = True
---
```

#### ml2
```
$ sudo vi /etc/neutron/plugins/ml2/ml2_conf.ini
---
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population

[ml2_type_flat]
flat_networks = external

[ml2_type_vlan]
network_vlan_ranges = external:1001:2000

[ml2_type_vxlan]
vni_ranges = 1001:2000
vxlan_group = 239.1.1.1

[securitygroup]
enable_security_group = True
enable_ipset = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

[ovs]
local_ip = 192.168.0.200
bridge_mappings = external:br-ex

[agent]
l2_population = True
tunnel_types = vxlan
enable_distributed_routing = True
arp_responder = True
---
→ gre設定は無効化する.
```

#### l3_agent
```
$ sudo vi /etc/neutron/l3_agent.ini
---
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True
external_network_bridge =
router_delete_namespaces = True
agent_mode = dvr_snat
---
→ The external_network_bridge option intentionally contains no value.
```

#### dhcp
```
$ sudo vi /etc/neutron/dhcp_agent.ini
---
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
use_namespaces = True
dhcp_delete_namespaces = True
...
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
---

$ sudo vi /etc/neutron/dnsmasq-neutron.conf
---
dhcp-option-force=26,1450
---
→ VXLANでカプセル化する際のヘッダ長が50Byte. 1,500Bytes(EthernetのMTU) - 50Bytes = 1,450Bytes.
  外部へ出る際にはカプセル化されたヘッダは出ないので、ひとまずこれで.
  (参考: Flets 光Nextが1454だが影響しないということ.)
```

#### metaagent
```
$ sudo vi /etc/neutron/metadata_agent.ini 
---
[DEFAULT]
nova_metadata_ip = 192.168.0.200
metadata_proxy_shared_secret = password
---
```

#### reboot
設定反映のため、再起動する.
```
$ sudo shutdown -r now
```

## Compute Node
### 設定
#### sysctl
```
$ sudo vi /etc/sysctl.conf
---
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
---

$ sudo sysctl -p
```

#### neutron
```
$ sudo vi /etc/neutron/neutron.conf
---
[DEFAULT]
router_distributed = True
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
allow_automatic_l3agent_failover = True
---
```

#### ml2
```
$ sudo vi /etc/neutron/plugins/ml2/ml2_conf.ini
---
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population

[ml2_type_flat]
flat_networks = external

[ml2_type_vlan]
network_vlan_ranges = external:1001:2000

[ml2_type_vxlan]
vni_ranges = 1001:2000
vxlan_group = 239.1.1.1

[securitygroup]
enable_security_group = True
enable_ipset = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

[ovs]
local_ip = 192.168.0.210
bridge_mappings = external:br-ex

[agent]
l2_population = True
tunnel_types = vxlan
enable_distributed_routing = True
arp_responder = True

---
→ gre設定は無効化する.
```

#### l3_agent
```
$ sudo apt-get -y install neutron-l3-agent
$ sudo vi /etc/neutron/l3_agent.ini
---
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True
external_network_bridge =
router_delete_namespaces = True
agent_mode = dvr
---
```

#### metaagent
```
$ sudo apt-get -y install neutron-metadata-agent
$ sudo vi /etc/neutron/metadata_agent.ini
---
[DEFAULT]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
auth_region = RegionOne
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = password
nova_metadata_ip = 192.168.0.200
metadata_proxy_shared_secret = password
---
```

### NIC追加
```
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
  address 192.168.0.210
  netmask 255.255.255.0
  network 192.168.0.0
  gateway 192.168.0.254
  dns-nameservers 192.168.0.254
---

$ sudo ovs-vsctl add-br br-ex
$ sudo ovs-vsctl add-port br-ex p1p1
$ sudo ethtool -K INTERFACE_NAME gro off
$ sudo shutdown -r now
```

## 確認
ComputeNodeにL3 agent, metadata agent, OpenvSwitch agentがあればOK.
```
$ neutron agent-list
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| id                                   | agent_type         | host      | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| 5e91f71e-ef0a-40e0-8cc5-e7b20e47d430 | L3 agent           | hayao     | :-)   | True           | neutron-l3-agent          |
| 64b348ca-4c6d-4ae7-9831-5cf80b6b2e24 | Metadata agent     | ryunosuke | :-)   | True           | neutron-metadata-agent    |
| 7def7387-4de2-4997-8de3-8fbae4fb67d6 | Open vSwitch agent | hayao     | :-)   | True           | neutron-openvswitch-agent |
| 82fbf144-6806-467a-b53a-dff6ad71dac8 | Open vSwitch agent | ryunosuke | :-)   | True           | neutron-openvswitch-agent |
| b0455a8f-f728-4034-8e83-5161eb1545b1 | Metadata agent     | hayao     | :-)   | True           | neutron-metadata-agent    |
| c5bd9b07-87e5-4877-aa6d-56ad61d5c8ac | DHCP agent         | ryunosuke | :-)   | True           | neutron-dhcp-agent        |
| dfc034fb-b5e7-42eb-a08e-788c89219eb0 | L3 agent           | ryunosuke | :-)   | True           | neutron-l3-agent          |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
```

### 仮想ネットワーク作成
Externalはflatで作成済みなので、Internalのみを作っていく.

```
$ neutron net-create demo-net
---
$ neutron net-create demo-net
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 8e997def-3fe7-4b94-a2cd-85af248bec87 |
| mtu                       | 0                                    |
| name                      | demo-net                             |
| provider:network_type     | vxlan                                |
| provider:physical_network |                                      |
| provider:segmentation_id  | 1001                                 |
| router:external           | False                                |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 7460bfbafaba45cdac98542810a92ac9     |
+---------------------------+--------------------------------------+
---

$ neutron subnet-create demo-net 10.0.0.0/24 --name demo-subnet --gateway 10.0.0.1 --dns-nameservers list=true 192.168.0.254
---
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
| id                | 01da9805-83ea-4127-b100-b0b353acd8f5       |
| ip_version        | 4                                          |
| ipv6_address_mode |                                            |
| ipv6_ra_mode      |                                            |
| name              | demo-subnet                                |
| network_id        | 8e997def-3fe7-4b94-a2cd-85af248bec87       |
| subnetpool_id     |                                            |
| tenant_id         | 7460bfbafaba45cdac98542810a92ac9           |
+-------------------+--------------------------------------------+
---

$ neutron router-create demo-router
Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| distributed           | True                                 |
~~~~ここがTrueならOK.
| external_gateway_info |                                      |
| ha                    | False                                |
| id                    | b1f72298-a2a3-4bfc-9cd0-0187261e5972 |
| name                  | demo-router                          |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | 7460bfbafaba45cdac98542810a92ac9     |
+-----------------------+--------------------------------------+

$ neutron router-interface-add demo-router demo-subnet
Added interface 53ea1499-0f8f-44b0-a055-dca8688ee765 to router demo-router.

$ neutron router-gateway-set demo-router ext-net
Set gateway for router demo-router

$ neutron router-port-list demo-router
---
| 53ea1499-0f8f-44b0-a055-dca8688ee765 |      | fa:16:3e:d6:d6:11 | {"subnet_id": "01da9805-83ea-4127-b100-b0b353acd8f5", "ip_address": "10.0.0.1"}      |
| 267fd40c-3c8c-447d-afe3-87c88c19d5d1 |      | fa:16:3e:ed:34:54 | {"subnet_id": "a7e78785-7600-4200-b993-7eb3af222990", "ip_address": "192.168.0.103"} |
~~~External側にRouter-GatewayのIPアドレスができていればOK.
| ec9f5466-1573-4c9e-8502-47e10e042a6b |      | fa:16:3e:88:18:50 | {"subnet_id": "01da9805-83ea-4127-b100-b0b353acd8f5", "ip_address": "10.0.0.3"}      |
---

$ neutron router-show demo-router
---
| admin_state_up        | True  |
| distributed           | True  |
| external_gateway_info | {"network_id": "eeaab19f-2fc6-4a3b-a745-bb2fcd4923e6", "enable_snat": true, "external_fixed_ips": [{"subnet_id": "a7e78785-7600-4200-b993-7eb3af222990", "ip_address": "192.168.0.103"}]} |
| ha                    | False |
| id                    | b1f72298-a2a3-4bfc-9cd0-0187261e5972 |
| name                  | demo-router |
| routes                |             |
| status                | ACTIVE      |
| tenant_id             | 7460bfbafaba45cdac98542810a92ac9     |
```
