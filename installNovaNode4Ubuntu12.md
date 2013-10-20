<!--
************************************************************
OpenStack (Compute node) をUbuntu12(x86_64)へインストールする手順
参照元: http://docs.openstack.org/grizzly/openstack-compute/install/apt/content/configuring-the-hypervisor.html
Copyright (c) Takehiko OGASAWARA 2013 All Rights Reserved.
************************************************************
-->
<div id='title'>　</div>    

# Compute nodeを追加する方法

### 準備
バックアップディレクトリ作成
```
mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
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

MySQLインストール
```
# apt-get -y install mysql-server mysql-client
↑インストール時にパスワードを設定する
```

MySQLユーザ、DB作成
```
# mysql -u root -p
CREATE DATABASE nova;
GRANT ALL ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
GRANT ALL ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'password';
quit
```

sourcelist追加
```
vi /etc/apt/sources.list.d/cloud-archive.list
---
deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main
---
```

```
vi /etc/apt/sources.list.d/grizzly.list
---
deb http://archive.gplhost.com/debian grizzly main
deb http://archive.gplhost.com/debian grizzly-backports main
---
```

GPGキーエラーとなるので、keyをimportする
```
apt-get -y update
→GPGエラーが出力される。
gpg --keyserver subkeys.pgp.net --recv 64AA94D00B849883
gpg --export --armor 64AA94D00B849883 | apt-key add -
```

最新版へアップデート
```
apt-get install gplhost-archive-keyring
apt-get -y update && apt-get -y upgrade
```

ネットワーク設定
```
cp -p /etc/network/interfaces $BAK
vi /etc/network/interfaces
---
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
 address 192.168.0.200
 netmask 255.255.255.0
 network 192.168.0.0
 broadcast 192.168.0.255
 gateway 192.168.0.254
 dns-nameservers 192.168.0.254
 

auto eth1
iface eth1 inet static
 address 10.0.0.10
 netmask 255.255.255.0
 network 10.0.0.0
 broadcast 10.0.0.255
# gateway 10.0.0.1
---
```

パケットフォワード設定
```
cp -p /etc/sysctl.conf $BAK
vi /etc/sysctl.conf
---
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
---
```

サービス再起動
```
sysctl -e -p /etc/sysctl.conf
→rp_filterが0と表示されればOK
/etc/init.d/networking restart
```

ntpdインストール
```
---
apt-get -y install ntp
---
```

MySQLインストール
```
apt-get install -y python-mysqldb mysql-server
→rootのDBパスワードはadmin!
```

mysql設定更新
```
cp -p /etc/mysql/my.cnf $BAK
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
→bind-addressが0.0.0.0へ変更できていればOK
service mysql restart
```

DB作成
```
mysql -u root -p <<EOF
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'password';
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'password';
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'password';
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'password';
CREATE DATABASE quantum;
GRANT ALL PRIVILEGES ON quantum.* TO 'quantum'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON quantum.* TO 'quantum'@'10.0.0.10' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
EOF
→ひとまず自ホストのみで設定。
```

rabbitMQインストール
```
apt-get -y install rabbitmq-server
rabbitmqctl change_password guest password
```

### Keystone

keystone
```
apt-get -y install keystone python-keystone python-keystoneclient
```

設定変更
```
cp -ap /etc/keystone $BAK
vi /etc/keystone/keystone.conf
---
[DEFAULT]
admin_token = password
debug = True
verbose = True
...
[sql]
connection = mysql://keystone:password@localhost/keystone
---
```

再起動
```
service keystone restart
service keystone status
→起動中であること
keystone-manage db_sync
```

起動script作成
```
vi /root/.openrc
---
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL="http://localhost:5000/v2.0/"
export OS_SERVICE_ENDPOINT="http://localhost:35357/v2.0"
export OS_SERVICE_TOKEN=password
---
source ~/.openrc
echo "source ~/.openrc" >> ~/.bashrc
```

keystone初期設定流し込み
```
vi /usr/local/src/initkeystone.sh
---
#!/bin/bash

# Modify these variables as needed
ADMIN_PASSWORD=${ADMIN_PASSWORD:-password}
SERVICE_PASSWORD=${SERVICE_PASSWORD:-$ADMIN_PASSWORD}
DEMO_PASSWORD=${DEMO_PASSWORD:-$ADMIN_PASSWORD}
export OS_SERVICE_TOKEN="password"
export OS_SERVICE_ENDPOINT="http://localhost:35357/v2.0"
SERVICE_TENANT_NAME=${SERVICE_TENANT_NAME:-service}
#
MYSQL_USER=keystone
MYSQL_DATABASE=keystone
MYSQL_HOST=localhost
MYSQL_PASSWORD=password
#
KEYSTONE_REGION=RegionOne
KEYSTONE_HOST=10.0.0.10

# Shortcut function to get a newly generated ID
function get_field() {
    while read data; do
        if [ "$1" -lt 0 ]; then
            field="(\$(NF$1))"
        else
            field="\$$(($1 + 1))"
        fi
        echo "$data" | awk -F'[ \t]*\\|[ \t]*' "{print $field}"
    done
}

# Tenants
ADMIN_TENANT=$(keystone tenant-create --name=admin | grep " id " | get_field 2)
DEMO_TENANT=$(keystone tenant-create --name=demo | grep " id " | get_field 2)
SERVICE_TENANT=$(keystone tenant-create --name=$SERVICE_TENANT_NAME | grep " id " | get_field 2)

# Users
ADMIN_USER=$(keystone user-create --name=admin --pass="$ADMIN_PASSWORD" --email=admin@domain.com | grep " id " | get_field 2)
DEMO_USER=$(keystone user-create --name=demo --pass="$DEMO_PASSWORD" --email=demo@domain.com --tenant-id=$DEMO_TENANT | grep " id " | get_field 2)
NOVA_USER=$(keystone user-create --name=nova --pass="$SERVICE_PASSWORD" --tenant-id $SERVICE_TENANT --email=nova@domain.com | grep " id " | get_field 2)
GLANCE_USER=$(keystone user-create --name=glance --pass="$SERVICE_PASSWORD" --tenant-id $SERVICE_TENANT --email=glance@domain.com | grep " id " | get_field 2)
QUANTUM_USER=$(keystone user-create --name=quantum --pass="$SERVICE_PASSWORD" --tenant-id $SERVICE_TENANT --email=quantum@domain.com | grep " id " | get_field 2)
CINDER_USER=$(keystone user-create --name=cinder --pass="$SERVICE_PASSWORD" --tenant-id $SERVICE_TENANT --email=cinder@domain.com | grep " id " | get_field 2)

# Roles
ADMIN_ROLE=$(keystone role-create --name=admin | grep " id " | get_field 2)
MEMBER_ROLE=$(keystone role-create --name=Member | grep " id " | get_field 2)

# Add Roles to Users in Tenants
keystone user-role-add --user-id $ADMIN_USER --role-id $ADMIN_ROLE --tenant-id $ADMIN_TENANT
keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $NOVA_USER --role-id $ADMIN_ROLE
keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $GLANCE_USER --role-id $ADMIN_ROLE
keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $QUANTUM_USER --role-id $ADMIN_ROLE
keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $CINDER_USER --role-id $ADMIN_ROLE
keystone user-role-add --tenant-id $DEMO_TENANT --user-id $DEMO_USER --role-id $MEMBER_ROLE

# Create services
COMPUTE_SERVICE=$(keystone service-create --name nova --type compute --description 'OpenStack Compute Service' | grep " id " | get_field 2)
VOLUME_SERVICE=$(keystone service-create --name cinder --type volume --description 'OpenStack Volume Service' | grep " id " | get_field 2)
IMAGE_SERVICE=$(keystone service-create --name glance --type image --description 'OpenStack Image Service' | grep " id " | get_field 2)
IDENTITY_SERVICE=$(keystone service-create --name keystone --type identity --description 'OpenStack Identity' | grep " id " | get_field 2)
EC2_SERVICE=$(keystone service-create --name ec2 --type ec2 --description 'OpenStack EC2 service' | grep " id " | get_field 2)
NETWORK_SERVICE=$(keystone service-create --name quantum --type network --description 'OpenStack Networking service' | grep " id " | get_field 2)

# Create endpoints
keystone endpoint-create --region $KEYSTONE_REGION --service-id $COMPUTE_SERVICE --publicurl 'http://'"$KEYSTONE_HOST"':8774/v2/$(tenant_id)s' --adminurl 'http://'"$KEYSTONE_HOST"':8774/v2/$(tenant_id)s' --internalurl 'http://'"$KEYSTONE_HOST"':8774/v2/$(tenant_id)s'
keystone endpoint-create --region $KEYSTONE_REGION --service-id $VOLUME_SERVICE --publicurl 'http://'"$KEYSTONE_HOST"':8776/v1/$(tenant_id)s' --adminurl 'http://'"$KEYSTONE_HOST"':8776/v1/$(tenant_id)s' --internalurl 'http://'"$KEYSTONE_HOST"':8776/v1/$(tenant_id)s'
keystone endpoint-create --region $KEYSTONE_REGION --service-id $IMAGE_SERVICE --publicurl 'http://'"$KEYSTONE_HOST"':9292' --adminurl 'http://'"$KEYSTONE_HOST"':9292' --internalurl 'http://'"$KEYSTONE_HOST"':9292'
keystone endpoint-create --region $KEYSTONE_REGION --service-id $IDENTITY_SERVICE --publicurl 'http://'"$KEYSTONE_HOST"':5000/v2.0' --adminurl 'http://'"$KEYSTONE_HOST"':35357/v2.0' --internalurl 'http://'"$KEYSTONE_HOST"':5000/v2.0'
keystone endpoint-create --region $KEYSTONE_REGION --service-id $EC2_SERVICE --publicurl 'http://'"$KEYSTONE_HOST"':8773/services/Cloud' --adminurl 'http://'"$KEYSTONE_HOST"':8773/services/Admin' --internalurl 'http://'"$KEYSTONE_HOST"':8773/services/Cloud'
keystone endpoint-create --region $KEYSTONE_REGION --service-id $NETWORK_SERVICE --publicurl 'http://'"$KEYSTONE_HOST"':9696/' --adminurl 'http://'"$KEYSTONE_HOST"':9696/' --internalurl 'http://'"$KEYSTONE_HOST"':9696/'
---

bash /usr/local/src/initkeystone.sh
→流し込みできればOK
 warningはひとまずそのまま。ERRORが出たらやり直すこと。
```


### glance

インストール
```
apt-get -y install glance
```

設定変更
```
cp -ap /etc/glance $BAK
vi /etc/glance/glance-api.conf
---
[DEFAULT]
sql_connection = mysql://glance:password@localhost/glance
...
[keystone_authtoken]
admin_tenant_name = service
admin_user = glance
admin_password = password
...
[paste_deploy]
flavor=keystone
---

vi /etc/glance/glance-registry.conf
---
[DEFAULT]
sql_connection = mysql://glance:password@localhost/glance
...
[keystone_authtoken]
admin_tenant_name = service
admin_user = glance
admin_password = password
...
[paste_deploy]
flavor=keystone
---
```

再起動
```
service glance-api restart
service glance-api status
service glance-registry restart
service glance-registry status
```

初期設定
```
glance-manage db_sync
```

OSイメージのダウンロード
```
cd /usr/local/src
wget http://cloud-images.ubuntu.com/releases/quantal/release/ubuntu-12.10-server-cloudimg-amd64-disk1.img
→ダウンロード完了後にimportする
glance image-create --is-public true --disk-format qcow2 --container-format bare --name "Ubuntu12.10" < ubuntu-12.10-server-cloudimg-amd64-disk1.img
```

OSイメージのimport確認
```
glance image-list
→importされていればOK
```

### nova
インストール
```
cd /usr/local/src
wget http://mirror.pnl.gov/ubuntu/pool/universe/libj/libjs-swfobject/libjs-swfobject_2.2+dfsg-1_all.deb
dpkg -i libjs-swfobject_2.2+dfsg-1_all.deb
→libjs-swfobjectがエラーになるので、手動でPackage追加

apt-get install -y nova-api nova-cert nova-common nova-conductor nova-scheduler python-nova python-novaclient nova-consoleauth novnc nova-novncproxy nova-compute
apt-get -y update
apt-get -y upgrade
```

設定変更
```
cp -p /etc/nova/api-paste.ini $BAK
vi /etc/nova/api-paste.ini
---
admin_tenant_name = service 
admin_user = nova 
admin_password = password
---

cp -p /etc/nova/nova.conf $BAK
vi /etc/nova/nova.conf
以下をdefaultセクションへ追記
---
[DEFAULT]
sql_connection=mysql://nova:password@localhost/nova
my_ip=10.0.0.10
rabbit_password=password
auth_strategy=keystone

# Networking
network_api_class=nova.network.quantumv2.api.API
quantum_url=http://10.0.0.10:9696
quantum_auth_strategy=keystone
quantum_admin_tenant_name=service
quantum_admin_username=quantum
quantum_admin_password=password
quantum_admin_auth_url=http://10.0.0.10:35357/v2.0
libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver

# Security Groups
firewall_driver=nova.virt.firewall.NoopFirewallDriver
security_group_api=quantum

# Metadata
quantum_metadata_proxy_shared_secret=password
service_quantum_metadata_proxy=true
metadata_listen = 10.0.0.10
metadata_listen_port = 8775

# Cinder
volume_api_class=nova.volume.cinder.API

# Glance
glance_api_servers=10.0.0.10:9292
image_service=nova.image.glance.GlanceImageService

# novnc
novnc_enable=true
novncproxy_port=6080
novncproxy_host=10.0.0.10
vncserver_listen=0.0.0.0
---
```

初期設定
```
nova-manage db sync
```

再起動
```
service nova-api restart
service nova-api status
service nova-cert restart
service nova-cert status
service nova-consoleauth restart
service nova-consoleauth status
service nova-scheduler restart
service nova-scheduler status
service nova-novncproxy restart
service nova-novncproxy status
service nova-compute restart
service nova-compute status
```

### cinder
インストール
```
cd /usr/local/src
wget http://ftp.jaist.ac.jp/pub/Linux/ubuntu//pool/main/libr/librdmacm/librdmacm1_1.0.15-1_amd64.deb
wget http://ftp.jaist.ac.jp/pub/Linux/ubuntu//pool/main/libi/libibverbs/libibverbs1_1.1.5-1ubuntu1_amd64.deb
dpkg -i librdmacm1_1.0.15-1_amd64.deb libibverbs1_1.1.5-1ubuntu1_amd64.deb
→cinder-volumeエラーが出るので、先にインストールする。

apt-get -y install cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms python-cinderclient linux-headers-`uname -r`
```

※iscsiは使用しないので、以下はスキップする。
```
## sed -i 's/false/true/g' /etc/default/iscsitarget
## service iscsitarget start
## service open-iscsi start
```

設定変更
```
cp -p /etc/cinder/cinder.conf $BAK
vi /etc/cinder/cinder.conf
---
[DEFAULT]
sql_connection = mysql://cinder:password@localhost/cinder
rabbit_password = password
---

cp -p /etc/cinder/api-paste.ini $BAK
vi /etc/cinder/api-paste.ini
---
admin_tenant_name = service
admin_user = cinder 
admin_password = password
---
```

cinderパーテーション作成
※事前にUSBメモリやSDカード、別のストレージを用意する。
```
pvcreate /dev/sdb1
vgcreate cinder-volumes /dev/sdb1
```

設定初期化
```
cinder-manage db sync
```

再起動
```
service cinder-api restart
service cinder-api status
service cinder-scheduler restart
service cinder-scheduler status
service cinder-volume restart
service cinder-volume status
```

### quantum
インストール
```
apt-get -y install quantum-server
```

設定変更
```
cp -p /etc/quantum/quantum.conf $BAK
vi /etc/quantum/quantum.conf
---
[DEFAULT]
verbose = True
rabbit_password = password
...
[keystone_authtoken]
admin_tenant_name = service
admin_user = quantum 
admin_password = password
---

cp -p /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini $BAK
vi /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini
---
[DATABASE]
sql_connection = mysql://quantum:password@localhost/quantum
...
[OVS]
tenant_network_type = gre 
tunnel_id_ranges = 1:1000
enable_tunneling = True
local_ip = 10.0.0.10
...
[SECURITYGROUP]
firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
---

```

OVS plugin有効化
```
ln -s /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini /etc/quantum/plugin.ini
```

再起動
```
service quantum-server restart
service quantum-server status
```

### Horizon
インストール
```
apt-get -y install openstack-dashboard memcached python-memcache
apt-get remove --purge openstack-dashboard-ubuntu-theme
→Ubuntu独自のhorizonメニューを非表示とするためにremoveする。
 (NetworkMapが見えるようになる)
```

ブラウザ確認
http://192.168.0.200/horizon
→Login: admin/password


## Network node
### quantum
インストール
→ sourcelistの登録はcontrollnodeで実施したのでスキップする。

設定変更
```
vi /etc/sysctl.conf
---
net.ipv4.ip_forward = 1
net.ipv4.conf.all.forwarding = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
→rp_filterはControllNodeと同じ設定
---
```

再起動
```
sysctl -e -p /etc/sysctl.conf
/etc/init.d/networking restart
```
※ntpインストール、hosts設定共に重複してるので、スキップ


OVSインストール
```
apt-get -y install quantum-plugin-openvswitch-agent quantum-dhcp-agent quantum-l3-agent
service openvswitch-switch restart
```

仮想NIC追加
```
ovs-vsctl add-br br-ex
ovs-vsctl add-port br-ex eth1
ovs-vsctl add-br br-int

cp -p /etc/network/interfaces $BAK/interfaces.2
vi /etc/network/interfaces
→eth0通信をbr-ex通信へ切替える
---
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
 address 192.168.0.200
 netmask 255.255.255.0
 gateway 192.168.0.254
 dns-nameservers 192.168.0.254

auto eth1
iface eth1 inet static
 address 10.0.0.10
 netmask 255.255.255.0
 network 10.0.0.0
 broadcast 10.0.0.255
# gateway 10.0.0.1

---
```

対応インターフェイス変更
→グローバル側のIPアドレスをbr-exへ寄せる
```
ip addr del 192.168.0.200/24 dev eth0
ip addr add 192.168.0.200/24 dev br-ex
→通信断になるので、コンソールで実施
```

再起動
```
/etc/init.d/networking restart
→可能であればrebootして通信設定できてる事を確認する
```

内部→外部通信許可設定
```
iptables -A FORWARD -i eth1 -o br-ex -s 10.0.0.0/24 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A POSTROUTING -s 10.0.0.0/24 -t nat -j MASQUERADE
iptables -L
→追加した設定が入っていればOK
```

quantum設定
```
vi /etc/quantum/quantum.conf
---
[DEFAULT]
verbose = True
rabbit_password = password
rabbit_host = 127.0.0.1
~~~★自ホストなので127.0.0.1とする。別ホストにするなら変更
[keystone_authtoken]
auth_host = 127.0.0.1
~~~★自ホストなので127.0.0.1とする。別ホストにするなら変更
admin_tenant_name = service
admin_user = quantum
admin_password = password
---

vi /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini
---
[DATABASE]
sql_connection = mysql://quantum:password@localhost/quantum
...
[OVS]
tenant_network_type = gre
tunnel_id_ranges = 1:1000
enable_tunneling = True
local_ip = 10.0.0.10
...
[securitygroup]
firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
---

cp -p /etc/quantum/dhcp_agent.ini $BAK
vi /etc/quantum/dhcp_agent.ini
---
enable_isolated_metadata = True
enable_metadata_network = True
---

cp -p /etc/quantum/metadata_agent.ini $BAK
vi /etc/quantum/metadata_agent.ini
---
# The Quantum user information for accessing the Quantum API.
auth_url = http://localhost:35357/v2.0
auth_region = RegionOne
admin_tenant_name = service
admin_user = quantum
admin_password = password
nova_metadata_ip = 127.0.0.1
metadata_proxy_shared_secret = password
---
```

再起動
```
service quantum-plugin-openvswitch-agent restart
service quantum-plugin-openvswitch-agent status
service quantum-dhcp-agent restart
service quantum-dhcp-agent status
service quantum-metadata-agent restart
service quantum-metadata-agent status
service quantum-l3-agent restart
service quantum-l3-agent status
```

仮想Router作成
→openrcは作成済みのためskip
```
vi /usr/local/src/addvmnetwork.sh
---
#!/bin/bash
TENANT_NAME="demo"
TENANT_NETWORK_NAME="demo-net"
TENANT_SUBNET_NAME="${TENANT_NETWORK_NAME}-subnet"
TENANT_ROUTER_NAME="demo-router"
FIXED_RANGE="10.0.0.0/24"
NETWORK_GATEWAY="10.0.0.1"
TENANT_ID=$(keystone tenant-list | grep " $TENANT_NAME " | awk '{print $2}')

TENANT_NET_ID=$(quantum net-create --tenant_id $TENANT_ID $TENANT_NETWORK_NAME --provider:network_type gre  --provider:segmentation_id 1 | grep " id " | awk '{print $4}')
TENANT_SUBNET_ID=$(quantum subnet-create --tenant_id $TENANT_ID --ip_version 4 --name $TENANT_SUBNET_NAME $TENANT_NET_ID $FIXED_RANGE --gateway $NETWORK_GATEWAY --dns_nameservers list=true 8.8.8.8 | grep " id " | awk '{print $4}')
ROUTER_ID=$(quantum router-create --tenant_id $TENANT_ID $TENANT_ROUTER_NAME | grep " id " | awk '{print $4}')
quantum router-interface-add $ROUTER_ID $TENANT_SUBNET_ID
---
bash /usr/local/src/addvmnetwork.sh
→Added Interface to router ...と出ればOK
```

外部Network設定
```
quantum net-create public --router:external=True
quantum subnet-create --ip_version 4 --gateway 192.168.0.254 public 192.168.0.0/24 --allocation-pool start=192.168.0.100,end=192.168.0.150 --disable-dhcp --name public-subnet
```

仮想Routerのゲートウェイ設定
```
quantum router-gateway-set demo-router public
→仮想Routerから外部ネットワークへの接続を設定
```

## Compute Node
### nova
※source.listへの追記, rp_filter, ntpは実施済みのためスキップ

インストール
```
apt-get -y install nova-compute-kvm
```

設定
```
cp -ap /etc/nova $BAK
vi /etc/nova/api-paste.ini
---
[filter:authtoken]
auth_host = 127.0.0.1
~~~★controller nodeのIP
admin_tenant_name = service 
admin_user = nova 
admin_password = password
---

vi /etc/nova/nova.conf
---
[DEFAULT]
sql_connection=mysql://nova:password@localhost/nova
my_ip=10.0.0.10
rabbit_password=password
auth_strategy=keystone
rabbit_host=10.0.0.10
ec2_host=10.0.0.10
ec2_url=http://10.0.0.10:8773/services/Cloud
...

↓URLと実機の設定差分を比べて追記する
rabbit_host=10.0.0.10
ec2_host=10.0.0.10
ec2_url=http://10.0.0.10:8773/services/Cloud
novncproxy_base_url=http://10.0.0.10:6080/vnc_auto.html

# Compute
compute_driver=libvirt.LibvirtDriver
connection_type=libvirt 
↑ここまで
---
```

再起動
```
service nova-compute restart
service nova-compute status
```

Networkインストール
→OpenVswitch, br-exインストール済みのためスキップ

## 初回起動前に

horizonネットワーク設定
```
http://192.168.0.200/horizon
[アクセスとセキュリティ]→[default]→[ルールの編集]
 ★anyで許可されていればOK
```

sshkey登録
```
nova keypair-add --pub_key ~/.ssh/id_rsa.pub default_key 
※horizonから登録しても良い
```
→horizonからインスタンスを起動させてみる
