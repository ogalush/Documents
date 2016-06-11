## Install OpenStack Mitaka on Ubuntu(ComputeNode) 16.04

ドキュメント: [OpenStack Docs](http://docs.openstack.org/mitaka/install-guide-ubuntu/environment.html)  
インストール先: hayao(192.168.0.210)  
```
ogalush@livaserver:~$ ssh -c aes128-ctr -A ogalush@192.168.0.210
Welcome to Ubuntu 16.04 LTS (GNU/Linux 4.4.0-21-generic x86_64)
ogalush@hayao:~$ uname -a
Linux hayao 4.4.0-21-generic #37-Ubuntu SMP Mon Apr 18 18:33:37 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
```

## Environment
### ntp
```
$ ntpq -p
→ 時刻同期していればOK.
$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 0.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 1.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 2.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 3.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp.ubuntu.com  .POOL.          16 p    -   64    0    0.000    0.000   0.000
*einzbern.turena 103.1.106.69     2 u  415 1024  357   17.522   -5.260   5.321
-ntp2.ig-haase.d 131.188.3.221    2 u  90m 1024  340  265.584   -1.025   1.401
+balthasar.gimas 181.170.187.161  3 u  171 1024  177   10.800   -0.993   3.920
+ec2-54-64-6-78. 133.243.238.164  2 u  92m 1024  340    6.157   -0.969   2.752
-usins.0x00.lv   194.29.130.252   2 u  99m 1024  340  307.197   -6.909   4.595
```

### OpenStack packages
```
$ sudo apt-get -y install software-properties-common
$ sudo add-apt-repository cloud-archive:mitaka
 Ubuntu Cloud Archive for OpenStack Mitaka
 More info: https://wiki.ubuntu.com/ServerTeam/CloudArchive
Press [ENTER] to continue or ctrl-c to cancel adding it

cloud-archive for Mitaka only supported on trusty
~~~ このコマンドはubuntu16.04では使用できないらしい。mitaka使えるん？

$ sudo apt-get -y update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade
```

### OpenStack Client
```
$ sudo apt-get -y install python-openstackclient
```

## Compute Service
### Install and configure a compute node
インストール
```
$ sudo apt-get -y install nova-compute
```

設定
```
$ sudo vi /etc/nova/nova.conf
----
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone
my_ip = 192.168.0.210
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[oslo_messaging_rabbit]
rabbit_host = 192.168.0.200
rabbit_userid = openstack
rabbit_password = password

## dhcp domain
dhcp_domain=localdomain

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

[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html

[glance]
api_servers = http://192.168.0.200:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp
----
```

確認
```
$ egrep -c '(vmx|svm)' /proc/cpuinfo
4 ・・・ 1 以上なら仮想化可能。

$ sudo vi /etc/nova/nova-compute.conf
----
[libvirt]
virt_type=kvm
----

$ sudo service nova-compute restart
$ sudo service nova-compute status
● nova-compute.service - OpenStack Compute
   Loaded: loaded (/lib/systemd/system/nova-compute.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2016-04-23 21:47:44 JST; 9s ago
  Process: 11729 ExecStartPre=/bin/chown nova:nova /var/lock/nova /var/log/nova /var/lib/nova (code=exited, status=0/SUCCESS)
  Process: 11726 ExecStartPre=/bin/mkdir -p /var/lock/nova /var/log/nova /var/lib/nova (code=exited, status=0/SUCCESS)
 Main PID: 11731 (nova-compute)
    Tasks: 22 (limit: 512)
   Memory: 130.8M
      CPU: 2.282s
   CGroup: /system.slice/nova-compute.service
           └─11731 /usr/bin/python /usr/bin/nova-compute --config-file=/etc/nova/nova.conf --config-file=/etc/nova/nova-compute.
```

### Verify operation
確認
```
$ scp 192.168.0.200:~/{admin,demo}-openrc ~/
$ source ~/admin-openrc
$ openstack compute service list
+----+------------------+-----------+----------+---------+-------+----------------------------+
| Id | Binary           | Host      | Zone     | Status  | State | Updated At                 |
+----+------------------+-----------+----------+---------+-------+----------------------------+
|  4 | nova-consoleauth | ryunosuke | internal | enabled | up    | 2016-04-23T12:48:59.000000 |
|  5 | nova-scheduler   | ryunosuke | internal | enabled | up    | 2016-04-23T12:48:53.000000 |
|  6 | nova-conductor   | ryunosuke | internal | enabled | up    | 2016-04-23T12:48:53.000000 |
|  7 | nova-compute     | ryunosuke | nova     | enabled | up    | 2016-04-23T12:48:57.000000 |
|  8 | nova-compute     | hayao     | nova     | enabled | up    | 2016-04-23T12:48:52.000000 |
+----+------------------+-----------+----------+---------+-------+----------------------------+
~~~ hayaoが認識されているのでOK.
```

## Networking service
### Install and configure compute node
インストール
```
$ sudo apt-get -y install neutron-linuxbridge-agent
```

設定
```
$ sudo vi /etc/neutron/neutron.conf
----
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone

[oslo_messaging_rabbit]
rabbit_host = 192.168.0.200
rabbit_userid = openstack
rabbit_password = password

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
----
```

### Networking Option 2: Self-service networks
```
$ sudo vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
----
[linux_bridge]
physical_interface_mappings = provider:enp3s0

[vxlan]
enable_vxlan = True
local_ip = 192.168.0.210
l2_population = True

[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
----
```

### Configure Compute to use Networking
設定
```
$ sudo vi /etc/nova/nova.conf
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

反映
```
$ sudo service nova-compute restart
$ sudo service neutron-linuxbridge-agent restart
```

### Verify operation
確認
```
$ source ~/admin-openrc
$ neutron ext-list
+---------------------------+-----------------------------------------------+
| alias                     | name                                          |
+---------------------------+-----------------------------------------------+
| default-subnetpools       | Default Subnetpools                           |
| network-ip-availability   | Network IP Availability                       |
| network_availability_zone | Network Availability Zone                     |
| auto-allocated-topology   | Auto Allocated Topology Services              |
| ext-gw-mode               | Neutron L3 Configurable external gateway mode |
| binding                   | Port Binding                                  |
| agent                     | agent                                         |
| subnet_allocation         | Subnet Allocation                             |
| l3_agent_scheduler        | L3 Agent Scheduler                            |
| tag                       | Tag support                                   |
| external-net              | Neutron external network                      |
| net-mtu                   | Network MTU                                   |
| availability_zone         | Availability Zone                             |
| quotas                    | Quota management support                      |
| l3-ha                     | HA Router extension                           |
| provider                  | Provider Network                              |
| multi-provider            | Multi Provider Network                        |
| address-scope             | Address scope                                 |
| extraroute                | Neutron Extra Route                           |
| timestamp_core            | Time Stamp Fields addition for core resources |
| router                    | Neutron L3 Router                             |
| extra_dhcp_opt            | Neutron Extra DHCP opts                       |
| dns-integration           | DNS Integration                               |
| security-group            | security-group                                |
| dhcp_agent_scheduler      | DHCP Agent Scheduler                          |
| router_availability_zone  | Router Availability Zone                      |
| rbac-policies             | RBAC Policies                                 |
| standard-attr-description | standard-attr-description                     |
| port-security             | Port Security                                 |
| allowed-address-pairs     | Allowed Address Pairs                         |
| dvr                       | Distributed Virtual Router                    |
+---------------------------+-----------------------------------------------+
~~~ 表示されればOK.
```

### Networking Option 2: Self-service networks
```
ogalush@hayao:~$ neutron agent-list
+--------------------------------------+--------------------+-----------+-------------------+-------+----------------+---------------------------+
| id                                   | agent_type         | host      | availability_zone | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-----------+-------------------+-------+----------------+---------------------------+
| 0bee23a5-f0b9-4ce4-9447-a01231d0844e | L3 agent           | ryunosuke | nova              | :-)   | True           | neutron-l3-agent          |
| 3f6287b9-6135-4562-a093-de902b913906 | Metadata agent     | ryunosuke |                   | :-)   | True           | neutron-metadata-agent    |
| 81819226-9d6d-4853-93e5-0a2adc592e2d | Linux bridge agent | ryunosuke |                   | :-)   | True           | neutron-linuxbridge-agent |
| b3b0a32a-7b18-4604-ab8c-e25384cabc59 | DHCP agent         | ryunosuke | nova              | :-)   | True           | neutron-dhcp-agent        |
| f6064e13-0761-4ec7-a2d3-2622f68228b3 | Linux bridge agent | hayao     |                   | :-)   | True           | neutron-linuxbridge-agent |
+--------------------------------------+--------------------+-----------+-------------------+-------+----------------+---------------------------+
~~~ hayaoが追加されているのでOK.
```
