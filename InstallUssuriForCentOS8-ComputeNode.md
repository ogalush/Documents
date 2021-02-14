# Install OpenStack Ussuri on CentOS8 (ComputeNode)
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.3.210  
設定ファイル: [Ussuri-InstallConfigsForCentOS8](hoge)
```
[ogalush@hayao ~]$ uname -n
hayao.localdomain
[ogalush@hayao ~]$ cat /etc/redhat-release 
CentOS Stream release 8
[ogalush@hayao ~]$ uname -a
Linux hayao.localdomain 4.18.0-277.el8.x86_64 #1 SMP Wed Feb 3 20:35:19 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
[ogalush@hayao ~]$
```
# Environment
## OS
```
[ogalush@hayao ~]$ uname -n
hayao.localdomain
[ogalush@hayao ~]$ cat /etc/hostname 
hayao.localdomain
[ogalush@hayao ~]$ cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.3.210 hayao hayao.localdomain ・・・追記
[ogalush@hayao ~]$ 
```
## ntp
https://docs.openstack.org/install-guide/environment-ntp-controller.html
```
[ogalush@hayao ~]$ chronyc sources
210 Number of sources = 4
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^+ ernie.gerger-net.de           3  10   377   268  -1198us[-1198us] +/-  131ms
^* 79.133.44.139                 1   9   377   729   -831us[-1889us] +/-  126ms
^+ ntp1.m-online.net             2  10   377   282  +1787us[+1787us] +/-  140ms
^+ ns1.blazing.de                3  10   377   282  +5364us[+5364us] +/-  136ms
[ogalush@hayao ~]$ 
→ 同期しているためOK.
```
## OpenStack packages for RHEL and CentOS
https://docs.openstack.org/install-guide/environment-packages-rdo.html
```
$ sudo yum -y install centos-release-openstack-ussuri
$ sudo yum config-manager --set-enabled powertools
$ sudo yum -y upgrade
$ sudo yum -y install python3-openstackclient
$ sudo yum -y install openstack-selinux
```
# Nova
## Install and configure a compute node for Red Hat Enterprise Linux and CentOS
https://docs.openstack.org/nova/ussuri/install/compute-install-rdo.html
### Install and configure components
```
$ sudo yum install -y openstack-nova-compute
$ sudo vim /etc/nova/nova.conf
---
+ [DEFAULT]
+ enabled_apis = osapi_compute,metadata
+ transport_url = rabbit://openstack:password@192.168.3.200
+ my_ip = 192.168.3.210
...
[api]
+ auth_strategy = keystone
...
[keystone_authtoken]
+ www_authenticate_uri = http://192.168.3.200:5000/
+ auth_url = http://192.168.3.200:5000/
+ memcached_servers = 192.168.3.200:11211
+ auth_type = password
+ project_domain_name = default
+ user_domain_name = default
+ project_name = service
+ username = nova
+ password = password
...
[vnc]
+ enabled = true
+ server_listen = 0.0.0.0
+ server_proxyclient_address = $my_ip
+ novncproxy_base_url = http://192.168.3.200:6080/vnc_auto.html
...
[glance]
+ api_servers = http://192.168.3.200:9292
...
[oslo_concurrency]
+ lock_path = /var/lib/nova/tmp
...
[placement]
+ region_name = RegionOne
+ project_domain_name = default
+ project_name = service
+ auth_type = password
+ user_domain_name = default
+ auth_url = http://192.168.3.200:5000/v3
+ username = placement
+ password = password
...
[libvirt]
+ virt_type = kvm
...
[scheduler]
+ discover_hosts_in_cells_interval = 300
---
```
### Finalize installation
```
$ egrep -c '(vmx|svm)' /proc/cpuinfo
4
$ sudo systemctl enable libvirtd.service openstack-nova-compute.service
Created symlink /etc/systemd/system/multi-user.target.wants/openstack-nova-compute.service → /usr/lib/systemd/system/openstack-nova-compute.service.
$ sudo systemctl start libvirtd.service openstack-nova-compute.service
```
### Add the compute node to the cell database
```
[ogalush@ryunosuke ~]$ source ~/admin-openrc 
[ogalush@ryunosuke ~]$ openstack compute service list --service nova-compute
+----+--------------+-----------------------+------+---------+-------+----------------------------+
| ID | Binary       | Host                  | Zone | Status  | State | Updated At                 |
+----+--------------+-----------------------+------+---------+-------+----------------------------+
|  7 | nova-compute | ryunosuke.localdomain | nova | enabled | up    | 2021-02-14T15:21:33.000000 |
|  8 | nova-compute | hayao.localdomain     | nova | enabled | up    | 2021-02-14T15:21:38.000000 |
+----+--------------+-----------------------+------+---------+-------+----------------------------+
[ogalush@ryunosuke ~]$
→ 追加したホスト(hayao)が入っているためOK.

[ogalush@ryunosuke ~]$ sudo -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
[sudo] password for ogalush: 
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': 373f5d0c-ac8f-4864-aa93-db34aead5187
Checking host mapping for compute host 'hayao.localdomain': 88361479-0cc7-40f6-9e1e-43ba2c7bf616
Creating host mapping for compute host 'hayao.localdomain': 88361479-0cc7-40f6-9e1e-43ba2c7bf616
Found 1 unmapped computes in cell: 373f5d0c-ac8f-4864-aa93-db34aead5187
[ogalush@ryunosuke ~]$
```

# Neutron
## Install and configure compute node
https://docs.openstack.org/neutron/ussuri/install/compute-install-rdo.html
### Install the components
```
$ sudo yum -y install openstack-neutron-linuxbridge ebtables ipset
```
### Configure the common component
```
$ sudo vim /etc/neutron/neutron.conf
----
[DEFAULT]
+ transport_url = rabbit://openstack:password@192.168.3.200
+ auth_strategy = keystone
...
[keystone_authtoken]
+ www_authenticate_uri = http://192.168.3.200:5000
+ auth_url = http://192.168.3.200:5000
+ memcached_servers = 192.168.3.200:11211
+ auth_type = password
+ project_domain_name = default
+ user_domain_name = default
+ project_name = service
+ username = neutron
+ password = password
[oslo_concurrency]
+ lock_path = /var/lib/neutron/tmp
----
```
### Configure networking options
Networking Option 2: Self-service networks
https://docs.openstack.org/neutron/ussuri/install/compute-install-option2-rdo.html
#### Configure the Linux bridge agent
```
$ sudo vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
---
+ [linux_bridge]
+ physical_interface_mappings = provider:enp3s0

+ [vxlan]
+ enable_vxlan = true
+ local_ip = 192.168.3.210
+ l2_population = true

+ [securitygroup]
+ enable_security_group = true
+ firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
---

[ogalush@hayao ~]$ grep 'net.bridge.bridge-nf-call-ip' /usr/lib/sysctl.d/99-neutron-linuxbridge-agent.conf 
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
[ogalush@hayao ~]$ 
→ マニュアルの設定値はありそう.

```
### Configure the Compute service to use the Networking service
```
$ sudo vim /etc/nova/nova.conf
---
[neutron]
+ auth_url = http://192.168.3.200:5000
+ auth_type = password
+ project_domain_name = default
+ user_domain_name = default
+ region_name = RegionOne
+ project_name = service
+ username = neutron
+ password = password
---
```
### Finalize installation
```
$ sudo systemctl restart openstack-nova-compute.service
$ sudo systemctl enable neutron-linuxbridge-agent.service
Created symlink /etc/systemd/system/multi-user.target.wants/neutron-linuxbridge-agent.service → /usr/lib/systemd/system/neutron-linuxbridge-agent.service.
$ sudo systemctl start neutron-linuxbridge-agent.service
$ sudo systemctl is-active neutron-linuxbridge-agent.service
active
$ sudo systemctl is-enabled neutron-linuxbridge-agent.service
enabled
```
### Verificaiton
```
[ogalush@ryunosuke ~]$ source ~/admin-openrc 
[ogalush@ryunosuke ~]$ neutron agent-list
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+--------------------+-----------------------+-------------------+-------+----------------+---------------------------+
| id                                   | agent_type         | host                  | availability_zone | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-----------------------+-------------------+-------+----------------+---------------------------+
| 33b9dedc-4ca4-4c85-ab58-4af65ff33f2c | Linux bridge agent | ryunosuke.localdomain |                   | :-)   | True           | neutron-linuxbridge-agent |
| 5416772c-ffd0-4d98-bdfa-0152d21f820e | Linux bridge agent | hayao.localdomain     |                   | :-)   | True           | neutron-linuxbridge-agent |
| 8e5ee9c9-ae94-4ba9-8b12-169b2945647e | DHCP agent         | ryunosuke.localdomain | nova              | :-)   | True           | neutron-dhcp-agent        |
| ba69e63d-65a0-4154-84bf-3412beaea557 | L3 agent           | ryunosuke.localdomain | nova              | :-)   | True           | neutron-l3-agent          |
| bae18d7b-c8a1-48a0-bb2f-3a43013002d6 | Metadata agent     | ryunosuke.localdomain |                   | :-)   | True           | neutron-metadata-agent    |
+--------------------------------------+--------------------+-----------------------+-------------------+-------+----------------+---------------------------+
[ogalush@ryunosuke ~]$
→ 追加したNode(hayao)のagentが入っていればOK.
```
