# Install OpenStack Rocky for CentOS7 Compute Node
[OpenStack Installation Guide](https://docs.openstack.org/install-guide/)  
Ubuntu18.04版がSegmentation Failtエラーが頻繁に出て不安定なため、CentOS7で構築をしてみる.  
設定ファイル: [Rocky-InstallConfigsForCentOS7](https://github.com/ogalush/Rocky-InstallConfigsForCentOS7)

# 環境
```
[ogalush@hayao ~]$ uname -n
hayao.localdomain
[ogalush@hayao ~]$ cat /etc/redhat-release
CentOS Linux release 7.6.1810 (Core) 
[ogalush@hayao ~]$ uname -a
Linux hayao.localdomain 3.10.0-957.5.1.el7.x86_64 #1 SMP Fri Feb 1 14:54:57 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
[ogalush@hayao ~]$
```

# Environment
## Configure network interfaces
```
$ grep -e 'DEVICE' -e 'TYPE' -e 'ONBOOT' -e 'BOOTPROTO' /etc/sysconfig/network-scripts/ifcfg-enp3s0
TYPE=Ethernet
BOOTPROTO=none
DEVICE=enp3s0
ONBOOT=yes
```

## Hosts
```
$ sudo cp -pv /etc/hosts ~
[sudo] password for ogalush: 
‘/etc/hosts’ -> ‘/home/ogalush/hosts’
$
$ sudo vim /etc/hosts
----
192.168.0.200 ryunosuke.localdomain ryunosuke
192.168.0.210 hayao.localdomain hayao
----

$ diff -u ~/hosts /etc/hosts
--- /home/ogalush/hosts 2013-06-07 23:31:32.000000000 +0900
+++ /etc/hosts  2019-03-02 21:10:13.528696983 +0900
@@ -1,2 +1,4 @@
 127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
 ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
+192.168.0.200 ryunosuke.localdomain ryunosuke
+192.168.0.210 hayao.localdomain hayao
$
```

## NTP
```
$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*tama.paina.net  203.178.138.38   2 u    2   64    1    3.814    1.399   0.160
 ntp-5.jonlight. 10.84.87.146     2 u    1   64    1    3.551    1.646   0.099
 chobi.paina.net 203.178.138.38   2 u    1   64    1   10.869    2.568   5.711
 ntp3.jst.mfeed. .INIT.          16 u    -   64    0    0.000    0.000   0.000
$
```

## OpenStack packages for RHEL and CentOS
[Doc](https://docs.openstack.org/install-guide/environment-packages-rdo.html)
```
$ sudo yum -y install centos-release-openstack-rocky
$ sudo yum -y update
$ sudo yum -y install python-openstackclient openstack-selinux
```

# Nova
[Doc](https://docs.openstack.org/nova/rocky/install/compute-install-rdo.html)

## Install and configure a compute node for Red Hat Enterprise Linux and CentOS
```
$ sudo yum -y install openstack-nova-compute
$ sudo cp -rafv /etc/nova ~
$ sudo vim /etc/nova/nova.conf
----
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:password@192.168.0.200
my_ip = 192.168.0.210
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
...
[api]
auth_strategy = keystone
...
[keystone_authtoken]
auth_url = http://192.168.0.200:5000/v3
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = password
...
[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html
...
[glance]
api_servers = http://192.168.0.200:9292
...
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
...
[placement]
region_name = RegionOne
project_domain_name = default
project_name = service
auth_type = password
user_domain_name = default
auth_url = http://192.168.0.200:5000/v3
username = placement
password = password
...
[libvirt]
virt_type = kvm 
----

$ egrep -c '(vmx|svm)' /proc/cpuinfo
4
→ 1以上なのでKVMのまま.

$ sudo systemctl enable libvirtd.service openstack-nova-compute.service
$ sudo systemctl start libvirtd.service openstack-nova-compute.service
```

## ポート開放(Controller側)
Controller側のFirewallで接続が繋がらないのでポート開放しておく.  
[Table B.1. Default ports that OpenStack components use](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/8/html/configuration_reference_guide/firewalls-default-ports)
```
$ ssh controller
$ sudo firewall-cmd --list-all
----
[sudo] password for ogalush: 
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp3s0
  sources: 
  services: ssh dhcpv6-client http https
  ports: 6080/tcp
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:
----

(1) ComputeNode用にZoneを作る.
$ sudo firewall-cmd --permanent --new-zone=ComputeZone
$ sudo firewall-cmd --reload

(2) 設定したルールを許可する形にする.
$ sudo firewall-cmd --permanent --zone=ComputeZone --set-target=ACCEPT

(3) 対象ホストの登録
→ 今回は一旦/24にしておく.
$ sudo firewall-cmd --permanent --zone=ComputeZone --add-source=192.168.0.0/24
$ sudo firewall-cmd --reload
$ sudo firewall-cmd --get-active-zones
ComputeZone
  sources: 192.168.0.0/24
public
  interfaces: enp3s0

(4) ポート開放する一覧を設定する.
$ ZONE="ComputeZone"
$ sudo firewall-cmd --permanent --zone=${ZONE} --add-port=8773-8775/tcp
→ nova
$ sudo firewall-cmd --permanent --zone=${ZONE} --add-port=5900-5999/tcp
→ Compute ports for access to virtual machine consoles
$ sudo firewall-cmd --permanent --zone=${ZONE} --add-port=8773/tcp
→ Compute API (nova-api) 
$ sudo firewall-cmd --permanent --zone=${ZONE} --add-port=6080-6082/tcp
→ Compute VNC proxy
$ sudo firewall-cmd --permanent --zone=${ZONE} --add-port=5000/tcp
→ Identity service public endpoint
$ sudo firewall-cmd --permanent --zone=${ZONE} --add-port=9292/tcp
→ Image service (glance) API
$ sudo firewall-cmd --permanent --zone=${ZONE} --add-port=9696/tcp
→ Networking (neutron)
$ sudo firewall-cmd --permanent --zone=${ZONE} --add-port=5672/tcp
→ RabbitMQ
$ sudo firewall-cmd --permanent --zone=${ZONE} --add-port=11211/tcp 
→ Memcached
$ sudo firewall-cmd --reload
→ 反映.
$ sudo firewall-cmd --list-all --zone=${ZONE}
----
ComputeZone (active)
  target: ACCEPT
  icmp-block-inversion: no
  interfaces: 
  sources: 192.168.0.0/24
  services: 
  ports: 8773-8775/tcp 5900-5999/tcp 8773/tcp 6080-6082/tcp 5000/tcp 9292/tcp 9696/tcp 5672/tcp 11211/tcp
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:
----
```

## ComputeNode参加確認
```
@Controller
$ source ~/admin-openrc.sh
$ openstack compute service list --service nova-compute
+----+--------------+-----------------------+------+---------+-------+----------------------------+
| ID | Binary       | Host                  | Zone | Status  | State | Updated At                 |
+----+--------------+-----------------------+------+---------+-------+----------------------------+
|  9 | nova-compute | ryunosuke.localdomain | nova | enabled | up    | 2019-03-02T13:16:56.000000 |
| 10 | nova-compute | hayao.localdomain     | nova | enabled | up    | 2019-03-02T13:17:01.000000 |
+----+--------------+-----------------------+------+---------+-------+----------------------------+
$
→ ComputeNodeが新たに追加されているのでOK.

$ sudo bash -c "nova-manage cell_v2 discover_hosts --verbose" nova
[sudo] password for ogalush: 
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': e687ffdf-f4f3-49f9-85a4-5e915dfd32ad
Found 0 unmapped computes in cell: e687ffdf-f4f3-49f9-85a4-5e915dfd32ad
```

# Neutron
[Doc](https://docs.openstack.org/neutron/rocky/install/compute-install-rdo.html)

## Install and configure compute node
```
$ sudo yum -y install openstack-neutron-linuxbridge ebtables ipset
$ sudo cp -rafv /etc/neutron ~
$ sudo vim /etc/neutron/neutron.conf
----
[DEFAULT]
transport_url = rabbit://openstack:password@192.168.0.200
auth_strategy = keystone
...
[keystone_authtoken]
www_authenticate_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:5000
memcached_servers = 192.168.0.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password
...
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
----

$ sudo vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
----
[linux_bridge]
physical_interface_mappings = provider:enp3s0
...
[vxlan]
enable_vxlan = true
local_ip = 192.168.0.210
l2_population = true
...
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
----

$ sudo vim /etc/sysctl.d/99-neutron.conf
----
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
----

$ sudo sysctl --system
...
* Applying /usr/lib/sysctl.d/99-neutron-linuxbridge-agent.conf ...
* Applying /etc/sysctl.d/99-neutron.conf ...
* Applying /etc/sysctl.d/99-sysctl.conf ...
* Applying /etc/sysctl.conf ...
$ 

$ sudo vim /etc/nova/nova.conf
----
...
[neutron]
url = http://192.168.0.200:9696
auth_url = http://192.168.0.200:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = password
...
----
```

## Finalize installation
```
$ sudo systemctl enable neutron-linuxbridge-agent.service
$ sudo systemctl restart openstack-nova-compute.service neutron-linuxbridge-agent.service
```
