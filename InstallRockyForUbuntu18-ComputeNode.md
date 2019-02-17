# Install OpenStack Rocky on Ubuntu 18.04 ComputeNode
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.0.210  
設定ファイル: [Rocky-InstallConfigs](https://github.com/ogalush/Rocky-InstallConfigs)
```
$ uname -a
Linux hayao 4.15.0-45-generic #48-Ubuntu SMP Tue Jan 29 16:28:13 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
```

# Prepare
## resolv.conf
nameserver 127.0.0.53になった事象があったので、参照先が怪しい場合は対応する.
```
$ grep nameserver /etc/resolv.conf 
nameserver 127.0.0.53
$ sudo rm -f /etc/resolv.conf
$ sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
$ ls -l /etc/resolv.conf 
lrwxrwxrwx 1 root root 32 Feb 17 15:56 /etc/resolv.conf -> /run/systemd/resolve/resolv.conf
$ grep nameserver /etc/resolv.conf 
nameserver 192.168.0.254
```

# Host networking
## Configure network interfaces
127.0.1.1 がホスト名に紐づいているので、実際のIPアドレスへ置換する.
```
$ sudo cp -pv /etc/hosts /tmp/hosts
[sudo] password for ogalush: 
'/etc/hosts' -> '/tmp/hosts'
$ sudo vim /etc/hosts
$ diff -u /tmp/hosts /etc/hosts |egrep '^(\+|\-)'
--- /tmp/hosts  2019-02-16 01:44:10.191132478 +0900
+++ /etc/hosts  2019-02-17 15:57:30.640683535 +0900
-127.0.1.1      hayao
+192.168.0.210  hayao
$
```

## Network Time Protocol (NTP)
```
$ sudo dpkg -r ntp
$ sudo apt -y install chrony
$ dpkg -l |grep chrony
ii  chrony 3.2-4ubuntu4.2 amd64 Versatile implementation of the Network Time Protocol
$ sudo cp -ar /etc/chrony ~
$ sudo vim /etc/chrony/chrony.conf
$ diff -urBb ~/chrony /etc/chrony 2> /dev/null |egrep '^(\+|\-)'
--- /home/ogalush/chrony/chrony.conf    2018-08-20 15:00:29.000000000 +0900
+++ /etc/chrony/chrony.conf     2018-09-16 15:19:47.082452675 +0900
+
+server ntp.nict.jp iburst
+allow 10.0.0.0/24
+allow 192.168.0.0/24
----

$ sudo service chrony restart
$ sudo service chrony status
● chrony.service - chrony, an NTP client/server
   Loaded: loaded (/lib/systemd/system/chrony.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2019-02-17 15:59:29 JST; 4s ago
     Docs: man:chronyd(8)
...
---

$ chronyc sources
210 Number of sources = 9
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^- golem.canonical.com           2   6    33    17   +832us[ +832us] +/-  151ms
^- pugot.canonical.com           2   6    17    21   +811us[ +811us] +/-  156ms
^- chilipepper.canonical.com     2   6    17    21  +5329us[+5329us] +/-  145ms
^- alphyn.canonical.com          2   6    17    22  +2935us[+2956us] +/-  119ms
^- ntp1.jst.mfeed.ad.jp          2   6    17    22   -412us[ -391us] +/-   73ms
^- sv1.localdomain1.com          2   6    17    23   -484us[ -973us] +/-   44ms
^- y.ns.gin.ntt.net              2   6    17    23  -1762us[-1742us] +/-  101ms
^- ntp3.jst.mfeed.ad.jp          2   6    17    23   -235us[ -214us] +/-  109ms
^* ntp-a3.nict.go.jp             1   6    17    22    +35us[  +55us] +/- 2589us
```

## CPU Performance (任意)
CPUガバナーをPerformanceへ変更してレスポンスを上げておく
```
https://askubuntu.com/questions/1021748/set-cpu-governor-to-performance-in-18-04
$ sudo apt-get -y install cpufrequtils
$ echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
$ sudo systemctl disable ondemand
$ sudo reboot
$ cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
performance
performance
performance
performance
$
→ Performance表示となっていればOK.
```

# OpenStack packages for Ubuntu
## Enable the OpenStack repository
```
$ sudo apt -y install software-properties-common
$ sudo add-apt-repository cloud-archive:rocky
```

## Finalize the installation
```
$ sudo apt -y install python-openstackclient
$ sudo apt-get -y update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade && sudo apt-get -y autoremove
$ sudo shutdown -r now
```

## nova installation for Rocky
[URL](https://docs.openstack.org/nova/rocky/install/)

### Install and configure ComputeNode for Ubuntu
#### Prerequisites
```
$ sudo apt -y install nova-compute
```

#### Config
```
$ sudo cp -rav /etc/nova ~
$ sudo vim /etc/nova/nova.conf
----
[DEFAULT]
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
server_listen = $my_ip
server_proxyclient_address = $my_ip
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html
keymap=ja
...
[glance]
api_servers = http://192.168.0.200:9292
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
...
[placement]
os_region_name = openstack
region_name = RegionOne
project_domain_name = default
project_name = service
auth_type = password
user_domain_name = default
auth_url = http://192.168.0.200:5000/v3
username = placement
password = password
----
```

### Finalize installation
```
$ egrep -c '(vmx|svm)' /proc/cpuinfo
4
$ sudo cat /etc/nova/nova-compute.conf 
[DEFAULT]
compute_driver=libvirt.LibvirtDriver
[libvirt]
virt_type=kvm
$
→ virt_typeはkvmのまま.
(Intel Virtualization Technologyが入っているので)

$ sudo service nova-compute restart
$ sudo service nova-compute status
● nova-compute.service - OpenStack Compute
   Loaded: loaded (/lib/systemd/system/nova-compute.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2019-02-17 16:23:34 JST; 1s ago
 Main PID: 7258 (nova-compute)
...
```

## Add the compute node to the cell database
```
$ source ~/admin_openrc.sh 
$ openstack compute service list --service nova-compute
+----+--------------+-----------+------+---------+-------+----------------------------+
| ID | Binary       | Host      | Zone | Status  | State | Updated At                 |
+----+--------------+-----------+------+---------+-------+----------------------------+
|  7 | nova-compute | ryunosuke | nova | enabled | up    | 2019-02-17T07:25:01.000000 |
|  8 | nova-compute | hayao     | nova | enabled | up    | 2019-02-17T07:24:52.000000 |
+----+--------------+-----------+------+---------+-------+----------------------------+
→ 追加対象の[hayao]が入っているのでOK.

ogalush@ryunosuke:~$ source ~/admin_openrc.sh
ogalush@ryunosuke:~$ sudo bash -c "nova-manage cell_v2 discover_hosts --verbose" nova
An error has occurred:
Traceback (most recent call last):
  File "/usr/lib/python2.7/dist-packages/nova/cmd/manage.py", line 2310, in main
    ret = fn(*fn_args, **fn_kwargs)
  File "/usr/lib/python2.7/dist-packages/nova/cmd/manage.py", line 1426, in discover_hosts
→ なんか見つからない...

DB設定は入っているので他の原因.
https://ask.openstack.org/en/question/99862/mitaka-nova-manage-api_db-sync-error-no-sql_connection-parameter-is-established/
$ sudo egrep '^\[api_database\]' /etc/nova/nova.conf -A1
[api_database]
connection = mysql+pymysql://nova:password@192.168.0.200/nova_api
$
```

# Configure the Compute service to use the Networking service
[URL](https://docs.openstack.org/neutron/rocky/install/compute-install-ubuntu.html)

### Install and configure compute node
```
$ sudo apt -y install neutron-linuxbridge-agent
```
### Configure the common component
#### neutron.conf
```
$ sudo vim /etc/neutron/neutron.conf
----
[DEFAULT]
core_plugin = ml2
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
----
```

#### nova.conf
```
$ sudo vim /etc/nova/nova.conf
----
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
service_metadata_proxy = true
metadata_proxy_shared_secret = password
----
```

#### finish
```
$ sudo service nova-compute restart
$ sudo service neutron-linuxbridge-agent restart
```
