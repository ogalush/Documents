<!--
************************************************************
OpenStack (Compute node) をUbuntu12(x86_64)へインストールする手順
参照元: http://docs.openstack.org/havana/install-guide/install/apt/content/nova-compute.html
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->
<div id='title'>　</div>    

# Compute nodeを追加する方法

### 準備
バックアップディレクトリ作成
```
# mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
# BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

### 準備
```
# ntp
# apt-get -y install ntp

# mysql
# apt-get -y install python-mysqldb mysql-client
```

### openstack packages
```
# apt-get -y install python-software-properties
# add-apt-repository cloud-archive:havana
# apt-get -y update && apt-get -y dist-upgrade
# reboot
```

### nova
```
# apt-get -y install nova-compute-kvm python-guestfs
→When prompted to create a supermin appliance, respond yes.

#  for BUG
# dpkg-statoverride  --update --add root root 0644 /boot/vmlinuz-$(uname -r)

# vi /etc/kernel/postinst.d/statoverride
----
#!/bin/sh
version="$1"
# passing the kernel version is required
[ -z "${version}" ] && exit 0
dpkg-statoverride --update --add root root 0644 /boot/vmlinuz-${version}
-----
# chmod +x /etc/kernel/postinst.d/statoverride


## edit nova.conf
# mkdir -p /root/MAINTENANCE/20140201/bak
# cp -raf /etc/nova /root/MAINTENANCE/20140201/bak/
# vi /etc/nova/nova.conf 
-----
[DEFAULT]
auth_strategy=keystone
rpc_backend = nova.rpc.impl_kombu
rabbit_host = 192.168.0.200
rabbit_password = password
my_ip=192.168.0.210
vnc_enabled=True
vncserver_listen=0.0.0.0
vncserver_proxyclient_address=192.168.0.210
novncproxy_base_url=http://192.168.0.200:6080/vnc_auto.html

glance_host=192.168.0.200

[database]
# The SQLAlchemy connection string used to connect to the database
connection = mysql://nova:password@192.168.0.200/nova
----- 

# vi /etc/nova/api-paste.ini
----
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = password
----

# service nova-compute restart
# service nova-compute status
# ls -al /var/lib/nova/nova.sqlite 
# mv /var/lib/nova/nova.sqlite /root/MAINTENANCE/20140201/bak/

# apt-get install -y nova-api-metadata

```

### neutron
```
### neutron
# apt-get -y install neutron-plugin-openvswitch-agent

# cp -p /etc/sysctl.conf /root/MAINTENANCE/20140201/bak
# vi /etc/sysctl.conf
---
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
---
# sysctl -p

# /etc/init.d/networking restart
# cp -p /etc/neutron /root/MAINTENANCE/20140201/bak
# vi /etc/neutron/neutron.conf
----
auth_strategy = keystone
...
rabbit_host = 192.168.0.200
rabbit_userid = guest
rabbit_password = password

[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = neutron
admin_password = password
...
[database]
connection = mysql://neutron:password@192.168.0.200/neutron
----

# vi /etc/neutron/api-paste.ini
----
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 192.168.0.200
auth_uri = http://192.168.0.200:5000
admin_tenant_name = service
admin_user = neutron
admin_password = password
----

# vi /etc/nova/nova.conf
----
[DEFAULT]
neutron_metadata_proxy_shared_secret = METADATA_PASS
service_neutron_metadata_proxy = true
----

# service nova-api restart
# vi /etc/neutron/metadata_agent.ini
----
auth_url = http://192.168.0.200:5000/v2.0
auth_region = regionOne
admin_tenant_name = service
admin_user = neutron
admin_password = password
nova_metadata_ip = 192.168.0.200
metadata_proxy_shared_secret = password
----
```

### ネットワーク設定
```
# cp -p /etc/network/interfaces /root/MAINTENANCE/20140201/bak
# vi /etc/network/interfaces
----
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet manual
 up ip address add 0/0 dev $IFACE
 up ip link set $IFACE up
 down ip link set $IFACE down

auto br-ex
iface br-ex inet static
 address 192.168.0.210
 netmask 255.255.255.0
 network 192.168.0.0
 gateway 192.168.0.254
 dns-nameservers 192.168.0.254
----

# ovs-vsctl add-br br-int
# ovs-vsctl add-br br-ex
# ovs-vsctl add-br br-tun
# ovs-vsctl add-port br-ex eth0
# sync; sync; sync; reboot

# service neutron-metadata-agent restart
# service neutron-plugin-openvswitch-agent restart

# openrc
# vi /root/.openrc
---
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL="http://192.168.0.200:5000/v2.0/"
export OS_SERVICE_ENDPOINT="http://192.168.0.200:35357/v2.0"
export OS_SERVICE_TOKEN=password
---

# echo 'source ~/.openrc' >> /root/.bashrc

## GREインストール時にエラーになる対応
# apt-get install openvswitch-datapath-dkms
# reboot
```
