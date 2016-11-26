## Install OpenStack Newton on Ubuntu 16.04(ComputeNode)

ドキュメント: [OpenStack Docs](http://docs.openstack.org/newton/install-guide-ubuntu/)  
インストール先: hayao(192.168.0.210)
```
ogalush@livaserver:~$ ssh -c aes128-ctr -A ogalush@192.168.0.210
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-47-generic x86_64)
ogalush@hayao:~$ uname -a
Linux hayao 4.4.0-47-generic #68-Ubuntu SMP Wed Oct 26 19:39:52 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
ogalush@hayao:~$ 
```

## Environment
### Network Time Protocol (NTP)
時刻同期しているので新たなインストールは不要。
```
ogalush@hayao:~$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 0.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 1.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 2.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 3.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp.ubuntu.com  .POOL.          16 p    -   64    0    0.000    0.000   0.000
+chilipepper.can 17.253.34.125    2 u  327 1024  357  241.069    7.374   4.651
-golem.canonical 17.253.34.253    2 u  788 1024  377  249.159    2.955   1.296
+alphyn.canonica 17.253.34.125    2 u  754 1024  377  204.071   -4.922   6.715
-juniperberry.ca 17.253.34.253    2 u  829 1024  377  237.363    9.041  10.556
*laika.paina.jp  131.113.192.40   2 u  917 1024  377    4.104   -1.678   4.743
```

### OpenStack packages
```
$ sudo apt -y install software-properties-common
$ sudo add-apt-repository cloud-archive:newton
$ sudo apt -y update && sudo apt -y dist-upgrade
$ sudo apt -y install python-openstackclient
```

## Compute service
### Install and configure a compute node
#### Install and configure components
```
インストール
$ sudo apt -y install nova-compute

設定
$ sudo vi /etc/nova/nova.conf
----
[DEFAULT]
...
transport_url = rabbit://openstack:password@192.168.0.200
auth_strategy = keystone
my_ip = 192.168.0.210
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[database]
connection = mysql+pymysql://nova:password@192.168.0.200/nova

[api_database]
connection = mysql+pymysql://nova:password@192.168.0.200/nova_api
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
----

[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html
vnc_keymap=ja

[glance]
api_servers = http://192.168.0.200:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

#### Finalize installation
```
確認
$ egrep -c '(vmx|svm)' /proc/cpuinfo
4
→ 1以上のため設定変更不要。

$ sudo grep 'virt_type' /etc/nova/nova-compute.conf
virt_type=kvm

反映
$ sudo service nova-compute restart
```
### Verify operation
ComputeNodeのサービスが追加されたのでOK.
```
ogalush@ryunosuke:~$ source ~/admin-openrc 
ogalush@ryunosuke:~$ openstack compute service list
+----+------------------+-----------+----------+---------+-------+----------------------------+
| ID | Binary           | Host      | Zone     | Status  | State | Updated At                 |
+----+------------------+-----------+----------+---------+-------+----------------------------+
|  4 | nova-consoleauth | ryunosuke | internal | enabled | up    | 2016-11-26T13:32:05.000000 |
|  5 | nova-scheduler   | ryunosuke | internal | enabled | up    | 2016-11-26T13:32:07.000000 |
|  6 | nova-conductor   | ryunosuke | internal | enabled | up    | 2016-11-26T13:32:08.000000 |
|  7 | nova-compute     | ryunosuke | nova     | enabled | up    | 2016-11-26T13:32:04.000000 |
|  8 | nova-compute     | hayao     | nova     | enabled | up    | 2016-11-26T13:32:02.000000 |
~~~ 今回追加したComputeNode.
+----+------------------+-----------+----------+---------+-------+----------------------------+
```

## Networking service
### Install and configure compute node
#### Install the components
```
$ sudo apt -y install neutron-linuxbridge-agent
```
#### Configure the common component
```
$ sudo vi /etc/neutron/neutron.conf
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
----
```

#### Configure networking options
「Networking Option 2: Self-service networks」を選ぶ.
##### Configure the Linux bridge agent
```
$ sudo vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
----
[linux_bridge]
physical_interface_mappings = provider:enp3s0

...

[vxlan]
enable_vxlan = True
local_ip = 192.168.0.210
l2_population = True

[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
----
```

#### Configure the Compute service to use the Networking service
```
$ sudo vi /etc/nova/nova.conf
----
...
[neutron]
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
$ echo 'nova-compute
neutron-linuxbridge-agent' | awk '{print "sudo service "$1" restart"}' | bash -x
```

### Verify operation
```
$ source ~/admin-openrc
$ openstack network agent list
+----------------------+--------------------+-----------+-------------------+-------+-------+----------------------+
| ID                   | Agent Type         | Host      | Availability Zone | Alive | State | Binary               |
+----------------------+--------------------+-----------+-------------------+-------+-------+----------------------+
| 11bb11d7-60db-4ac8   | L3 agent           | ryunosuke | nova              | True  | UP    | neutron-l3-agent     |
| -ae6a-2c0c1dd9714d   |                    |           |                   |       |       |                      |
| 5eb81530-7d5d-491d-  | Linux bridge agent | hayao     | None              | True  | UP    | neutron-linuxbridge- |
| bb04-31446233c362    |                    |           |                   |       |       | agent                |
~~~ ComputeNodeのAgentが追加できているのでOK.
| 78e44279-cb2b-4097   | DHCP agent         | ryunosuke | nova              | True  | UP    | neutron-dhcp-agent   |
| -b53c-3e678b76b3af   |                    |           |                   |       |       |                      |
| a024d45d-568d-4553-a | Linux bridge agent | ryunosuke | None              | True  | UP    | neutron-linuxbridge- |
| 5d7-b91379903295     |                    |           |                   |       |       | agent                |
| b5a36da3-d4ee-40cc-b | Metadata agent     | ryunosuke | None              | True  | UP    | neutron-metadata-    |
| 811-3732d36e27ae     |                    |           |                   |       |       | agent                |
+----------------------+--------------------+-----------+-------------------+-------+-------+----------------------+
ogalush@ryunosuke:~$ 
```
