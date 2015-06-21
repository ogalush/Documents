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

## 既存ネットワーク・サブネットの削除
設定を変えるので一旦既存設定を削除しておく.

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
type_drivers = flat,vlan,gre,vxlan
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
