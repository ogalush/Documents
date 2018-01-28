## Install OpenStack Pike on Ubuntu 16.04(ComputeNode)

ドキュメント:[OpenStack Docs](https://docs.openstack.org/nova/pike/install/), [Ubuntu CloudArchive](https://wiki.ubuntu.com/OpenStack/CloudArchive)  
インストール先: hayao(192.168.0.210)  
Repo: https://github.com/ogalush/Pike-InstallConfigs
```
ogalush@hayao:~$ uname -a
Linux hayao 4.4.0-112-generic #135-Ubuntu SMP Fri Jan 19 11:48:36 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
ogalush@hayao:~$
```

## Environment
### Register Pike Repositry
```
$ sudo add-apt-repository cloud-archive:pike
```

### OS Update
```
$ sudo apt-get -y update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade
```

### disable IPv6
```
$ sudo vim /etc/sysctl.conf
----
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
----
$ sudo sysctl -p
$ sudo shutdown -r now
```

### Network Time Protocol (NTP)
時刻同期しているので新たなインストールは不要。
```
ogalush@hayao:~$ ntpq -p |grep '*'
*li461-162.membe 103.1.106.69     2 u   78  128  377    4.188   -0.835   1.923
ogalush@hayao:~$ 
```

## Nova
[Document](https://docs.openstack.org/nova/pike/install/compute-install.html)
### Install and configure components
#### Install
```
$ sudo apt -y install nova-compute
```

#### Configs
```
$ sudo vim /etc/nova/nova.conf
----
[DEFAULT]
...
transport_url = rabbit://openstack:password@192.168.0.200
my_ip = 192.168.0.210
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
...
[api]
auth_strategy = keystone
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = password
...
[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html
...
[glance]
api_servers = http://192.168.0.200:9292
...
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
...
[placement]
os_region_name = RegionOne
project_domain_name = default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://192.168.0.200:35357/v3
username = placement
password = password
...
[scheduler]
discover_hosts_in_cells_interval = 300
----
```

### Finalize installation
#### Config
```
$ egrep -c '(vmx|svm)' /proc/cpuinfo
4
$ sudo vim /etc/nova/nova-compute.conf
----
...
virt_type=kvm ・・・KVMのまま.
----

$ sudo service nova-compute restart
$ sudo service nova-compute status
```

#### Add the compute node to the cell database
```
Controller$ source ~/admin-openrc.sh 
$ openstack compute service list --service nova-compute
+----+--------------+-----------+------+---------+-------+----------------------------+
| ID | Binary       | Host      | Zone | Status  | State | Updated At                 |
+----+--------------+-----------+------+---------+-------+----------------------------+
|  6 | nova-compute | ryunosuke | nova | enabled | up    | 2018-01-28T06:04:29.000000 |
|  7 | nova-compute | hayao     | nova | enabled | up    | 2018-01-28T06:04:33.000000 |
+----+--------------+-----------+------+---------+-------+----------------------------+
→ 新たにComputeNodeができていればOK.

Controller$ sudo bash -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting compute nodes from cell 'cell1': 146ace08-a6e3-49f8-9b53-1116dcb17bc0
Found 1 unmapped computes in cell: 146ace08-a6e3-49f8-9b53-1116dcb17bc0
Checking host mapping for compute host 'hayao': 5b0f2d34-4a88-4a49-8b4d-de69eb23b639
Creating host mapping for compute host 'hayao': 5b0f2d34-4a88-4a49-8b4d-de69eb23b639
→ 新たなComputeNodeを認識していればOK.
```

## Neutron
[Document](https://docs.openstack.org/neutron/pike/install/compute-install-ubuntu.html)
### Install
```
$ sudo apt -y install neutron-linuxbridge-agent
```

### Configure the common component
#### neutron.conf
```
$ sudo vim /etc/neutron/neutron.conf
----
[DEFAULT]
...
transport_url = rabbit://openstack:password@192.168.0.200
auth_strategy = keystone
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password
...
----
```
#### linuxbridge_agent.ini
```
$ sudo vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
----
[linux_bridge]
physical_interface_mappings = provider:enp3s0
...
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

[vxlan]
enable_vxlan = true
local_ip = 192.168.0.210
l2_population = true
----
```

#### Networking Option 2: Self-service networks
[Document](https://docs.openstack.org/neutron/pike/install/compute-install-ubuntu.html)
```
$ sudo vim /etc/nova/nova.conf
----
[neutron]
...
url = http://192.168.0.200:9696
auth_url = http://192.168.0.200:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = password
----
```

#### Finalize installation
```
$ sudo service nova-compute restart
$ sudo service neutron-linuxbridge-agent restart
```

### Verify operation
[Document](https://docs.openstack.org/neutron/pike/install/verify.html)
```
Controller$ source ~/admin-openrc.sh 
$ openstack network agent list
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host      | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| 31ef0da4-25aa-4234-94c3-8263dd43cfe3 | Metadata agent     | ryunosuke | None              | :-)   | UP    | neutron-metadata-agent    |
| 3ea12f80-fc02-4a35-b2d7-f186b6f11a32 | Linux bridge agent | hayao     | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 52f10a5f-49a5-4283-bc43-e8e098a6de93 | DHCP agent         | ryunosuke | nova              | :-)   | UP    | neutron-dhcp-agent        |
| 769d3833-89c4-4f80-93e4-3f4d1c112dd7 | L3 agent           | ryunosuke | nova              | :-)   | UP    | neutron-l3-agent          |
| b39bdc1f-a70f-405c-852a-107e9a141cc9 | Linux bridge agent | ryunosuke | None              | :-)   | UP    | neutron-linuxbridge-agent |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
→ 新ホストのレコードが表示されればOK. 
```
