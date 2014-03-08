<!--
************************************************************
OpenStack (havana) をUbuntu12(x86_64)へインストールする手順
参照元: 
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->
<div id='title'>　</div>    

# havanaインストール手順

### 準備
バックアップディレクトリ作成
```
BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
mkdir -p $BAK
```

KVM確認
```
apt-get install cpu-checker
root@kinder:~# kvm-ok 
INFO: /dev/kvm exists
KVM acceleration can be used
↑が出ていればOK

# lsmod | grep kvm
kvm_intel             137899  0 
kvm                   455932  1 kvm_intel
```

ntpインストール
```
# apt-get install -y ntp
```


### MySQLインストール
```
# apt-get install -y python-mysqldb mysql-server
# cp -raf /etc/mysql $BAK
# vi /etc/mysql/my.cnf
---
bind-address            = 192.168.0.200
---
```

MySQLユーザ、DB作成
```
service mysql restart
mysql_install_db
```

### OpenStack Packages
```
# apt-get install -y python-software-properties
# add-apt-repository cloud-archive:havana
~~aptヘhavanaを追加する
# apt-get update && apt-get dist-upgrade
# sync;sync;sync;reboot
```

### rabbit MQ
```
# apt-get install -y rabbitmq-server
# rabbitmqctl change_password guest password
```

### keystone
```
# apt-get install -y keystone
# cp -raf /etc/keystone $BAK
# vi /etc/keystone/keystone.conf
----
[sql]
##connection = sqlite:////var/lib/keystone/keystone.db
connection = mysql://keystone:password@192.168.0.200/keystone
----

# mysql -u root -p
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
\q

# keystone-manage db_sync

# vi /etc/keystone/keystone.conf
---
admin_token = password
---
# service keystone restart
```

### 環境変数設定
```
vi /root/.openrc
---
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL="http://192.168.0.200:5000/v2.0/"
export OS_SERVICE_ENDPOINT="http://192.168.0.200:35357/v2.0"
export OS_SERVICE_TOKEN=password
---
source ~/.openrc
echo "source ~/.openrc" >> ~/.bashrc
```


### keystoneユーザ作成
```
(1) user, role
# keystone tenant-create --name=admin --description="Admin Tenant"
# keystone tenant-create --name=service --description="Service Tenant"
# keystone user-create --name=admin --pass=password --email=admin@example.com
# keystone role-create --name=admin
# keystone user-role-add --user=admin --tenant=admin --role=admin

(2) service, endpoints
# keystone service-create --name=keystone --type=identity --description="Keystone Identity Service"
# keystone endpoint-create  --service-id=6d15194238b4418aae9f25178eadc110  --publicurl=http://192.168.0.200:5000/v2.0 --internalurl=http://192.168.0.200:5000/v2.0 --adminurl=http://192.168.0.200:35357/v2.0
~~~サービスIDは、keystone service-createで生成されたIDを入力する

### インストール確認
unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT
keystone --os-username=admin --os-password=password --os-auth-url=http://192.168.0.200:35357/v2.0 token-get
~~tokenを画面表示できればOK

keystone --os-username=admin --os-password=password  --os-tenant-name=admin --os-auth-url=http://192.168.0.200:35357/v2.0 token-get
~~tokenを画面表示できればOK

source ~/.openrc
keystone user-list
~~tokenを画面表示できればOK

```

### glance
```
Packageインストール
# apt-get install -y glance python-glanceclient
# cp -raf /etc/glance $BAK

設定変更
# vi /etc/glance/glance-api.conf 
----
##sql_connection = sqlite:////var/lib/glance/glance.sqlite
sql_connection = mysql://glance:password@192.168.0.200/glance
----

# vi /etc/glance/glance-registry.conf 
----
##sql_connection = sqlite:////var/lib/glance/glance.sqlite
sql_connection = mysql://glance:password@192.168.0.200/glance
----

mysql登録
# mysql -u root -p
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
\q

DB初期設定
# glance-manage db_sync

keystoneユーザ登録
# keystone user-create --name=glance --pass=password  --email=glance@example.com
# keystone user-role-add --user=glance --tenant=service --role=admin

設定変更
# vi /etc/glance/glance-api.conf
---
[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = password
---

# vi /etc/glance/glance-registry.conf
---
[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_user=glance
admin_tenant_name=service
admin_password=password
flavor=keystone
---

# keystone service-create --name=glance --type=image --description="Glance Image Service"
# keystone endpoint-create  --service-id=f4beac5c9c2a419e9e587dabd676b141 --publicurl=http://192.168.0.200:9292 --internalurl=http://192.168.0.200:9292 --adminurl=http://192.168.0.200:9292
~~~service-idは、keystone service-createで出力されたIDを入力する

glance再起動
# ls -1 /etc/init.d/glance-*| while read LINE; do service `basename ${LINE}` restart; done

### OSイメージ登録
# cd /usr/local/src
# wget http://cloud-images.ubuntu.com/releases/quantal/release/ubuntu-12.10-server-cloudimg-amd64-disk1.img
# glance image-create --is-public true --disk-format qcow2 --container-format bare --name "Ubuntu12.10" < ubuntu-12.10-server-cloudimg-amd64-disk1.img
# glance image-list
---
+--------------------------------------+-------------+-------------+------------------+-----------+--------+
| ID                                   | Name        | Disk Format | Container Format | Size      | Status |
+--------------------------------------+-------------+-------------+------------------+-----------+--------+
| aac7e922-8adc-458e-adbd-346c6456722c | Ubuntu12.10 | qcow2       | bare             | 224002048 | active |
+--------------------------------------+-------------+-------------+------------------+-----------+--------+
---
~~~イメージを表示できればOK

```

### nova
```
# apt-get install -y nova-novncproxy novnc nova-api nova-ajax-console-proxy nova-cert nova-conductor nova-consoleauth nova-doc nova-scheduler python-novaclient

# cp -raf /etc/nova $BAK
# vi /etc/nova/nova.conf
---
[DEFAULT]
connection = mysql://nova:password@192.168.0.200/nova
rpc_backend = nova.rpc.impl_kombu
rabbit_host = 192.168.0.200
rabbit_password = password
...
[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = password
---

DBユーザ作成
# mysql -u root -p
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
\q

DB初期設定
# nova-manage db sync

vnc設定
# vi /etc/nova/nova.conf
---
[DEFAULT]
...
my_ip=192.168.0.200
vncserver_listen=192.168.0.200
vncserver_proxyclient_address=192.168.0.200
...
---

keystoneユーザ設定
# keystone user-create --name=nova --pass=password --email=nova@example.com
# keystone user-role-add --user=nova --tenant=service --role=admin

# vi /etc/nova/nova.conf
---
[DEFAULT]
...
auth_strategy=keystone
...
---

# vi /etc/nova/api-paste.ini 
---
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = password
---

keystone設定
keystone service-create --name=nova --type=compute --description="Nova Compute service"
keystone endpoint-create --service-id=bdf676442a544c19b4d4fe524fb2ac5e --publicurl=http://192.168.0.200:8774/v2/%\(tenant_id\)s --internalurl=http://192.168.0.200:8774/v2/%\(tenant_id\)s  --adminurl=http://192.168.0.200:8774/v2/%\(tenant_id\)s
~~~登録できればOK

ls -1 /etc/init.d/nova-*| while read LINE; do service `basename ${LINE}` restart; done

### 確認
nova image-list
---
+--------------------------------------+-------------+--------+--------+
| ID                                   | Name        | Status | Server |
+--------------------------------------+-------------+--------+--------+
| aac7e922-8adc-458e-adbd-346c6456722c | Ubuntu12.10 | ACTIVE |        |
+--------------------------------------+-------------+--------+--------+
---
~~~表示されればOK

### install
apt-get install -y nova-compute-kvm python-guestfs
~~~superminのインストールが促されたら、yesと答える。

chmod 0644 /boot/vmlinuz*
~~~python-guestfsのbug対応

vi /etc/nova/nova.conf
---
my_ip=192.168.0.200
vnc_enabled=True
vncserver_listen=192.168.0.200
vncserver_proxyclient_address=192.168.0.200
novncproxy_base_url=http://192.168.0.200:6080/vnc_auto.html
...
glance_host=192.168.0.200
---

service nova-compute restart
rm /var/lib/nova/nova.sqlite

### nova-network
apt-get install -y nova-network

vi /etc/nova/nova.conf
----
network_manager=nova.network.manager.FlatDHCPManager
firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
network_size=254
allow_same_net_traffic=False
multi_host=True
send_arp_for_ha=True
share_dhcp_address=True
force_dhcp_release=True
flat_network_bridge=br100
flat_interface=eth1
public_interface=eth1
----
service nova-network restart
nova network-create vmnet --fixed-range-v4=10.0.0.0/24 --bridge-interface=br100 --multi-host=T
~~~★br100は存在しないnetwork。あとで解析する。

### sshkey登録
ssh-keygen
cd ~/.ssh
nova-manage db sync
nova keypair-add --pub_key id_rsa.pub mykey
nova keypair-list
---
+-------+-------------------------------------------------+
| Name  | Fingerprint                                     |
+-------+-------------------------------------------------+
| mykey | 54:ea:e9:c1:9d:ef:f8:09:10:0f:69:59:c9:fc:e8:8a |
+-------+-------------------------------------------------+
---

nova flavor-list
----
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| 1  | m1.tiny   | 512       | 1    | 0         |      | 1     | 1.0         | True      |
| 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      |
| 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      |
| 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      |
| 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
----
~~~表示されればOK

nova image-list
---
+--------------------------------------+-------------+--------+--------+
| ID                                   | Name        | Status | Server |
+--------------------------------------+-------------+--------+--------+
| aac7e922-8adc-458e-adbd-346c6456722c | Ubuntu12.10 | ACTIVE |        |
+--------------------------------------+-------------+--------+--------+
---

nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0

```
