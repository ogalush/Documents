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
