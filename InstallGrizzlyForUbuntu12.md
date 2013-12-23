<!--
************************************************************
OpenStack GrizzyをUbuntu12(x86_64)へインストールする手順
参照元: http://docs.openstack.org/grizzly/basic-install/apt/content/basic-install_controller.html
Copyright (c) Takehiko OGASAWARA 2013 All Rights Reserved.
************************************************************
-->
<div id='title'>　</div>    

# ControllNodeをインストールする方法(OpenStack Grizzly)

### 準備
バックアップディレクトリ作成
```
mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

パッケージインストール  
```
apt-get install ubuntu-cloud-keyring
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
...追加...
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
 network 192.168.0.0
 gateway 192.168.0.254
 dns-nameservers 192.168.0.254
 

## Internal Network
auto eth1
iface eth1 inet manual
 up ip address add 0/0 dev $IFACE
 up ip link set $IFACE up
 down ip link set $IFACE down

auto br-eth1
iface br-eth1 inet static
 address 10.0.1.200
 netmask 255.255.255.0
 network 10.0.1.0
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
MySQLへアクセスを行うホストからの接続許可設定を追加する。
(nova-computeがcontrollnodeのほとんどのサービスへアクセスするため開けておく。)
mysql -u root -p <<EOF
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'password';
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
CREATE DATABASE quantum;
GRANT ALL PRIVILEGES ON quantum.* TO 'quantum'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
EOF
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
connection = mysql://keystone:password@192.168.0.200/keystone
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
export OS_AUTH_URL="http://192.168.0.200:5000/v2.0/"
export OS_SERVICE_ENDPOINT="http://192.168.0.200:35357/v2.0"
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
export OS_SERVICE_ENDPOINT="http://192.168.0.200:35357/v2.0"
SERVICE_TENANT_NAME=${SERVICE_TENANT_NAME:-service}
#
MYSQL_USER=keystone
MYSQL_DATABASE=keystone
MYSQL_HOST=192.168.0.200
MYSQL_PASSWORD=password
#
KEYSTONE_REGION=RegionOne
KEYSTONE_HOST=192.168.0.200

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
sql_connection = mysql://glance:password@192.168.0.200/glance
...
[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
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
sql_connection = mysql://glance:password@192.168.0.200/glance
...
[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
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
ls -1 /etc/init.d/glance-*| while read LINE; do service `basename ${LINE}` restart; done
 →glance-api, glance-registryが再起動されていること。
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
 →libjs-swfobjectがエラーになるため、手動でPackage追加

apt-get install -y nova-api nova-cert nova-common nova-compute nova-compute-kvm nova-conductor nova-consoleauth nova-network nova-novncproxy nova-scheduler python-nova python-novaclient novnc
apt-get -y update
apt-get -y upgrade
```

設定変更
```
cp -p /etc/nova/api-paste.ini $BAK
vi /etc/nova/api-paste.ini
---
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service 
admin_user = nova 
admin_password = password
auth_version = v2.0
---

cp -p /etc/nova/nova.conf $BAK
vi /etc/nova/nova.conf
以下をdefaultセクションへ記載する
---
[DEFAULT]
connection_type=libvirt
sql_connection=mysql://nova:password@192.168.0.200/nova
my_ip=192.168.0.200
rabbit_password=password
auth_strategy=keystone
rabbit_host=192.168.0.200
ec2_host=192.168.0.200
ec2_url=http://192.168.0.200:8773/services/Cloud


# Networking
# Networking
network_api_class=nova.network.quantumv2.api.API
quantum_url=http://192.168.0.200:9696
quantum_auth_strategy=keystone
quantum_admin_tenant_name=service
quantum_admin_username=quantum
quantum_admin_password=password
quantum_admin_auth_url=http://192.168.0.200:35357/v2.0
libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver

# Security Groups
firewall_driver=nova.virt.firewall.NoopFirewallDriver
security_group_api=quantum

# Metadata
quantum_metadata_proxy_shared_secret=password
service_quantum_metadata_proxy=true
metadata_listen = 192.168.0.200
metadata_listen_port = 8775

# Cinder
volume_api_class=nova.volume.cinder.API

# Glance
glance_api_servers=192.168.0.200:9292
image_service=nova.image.glance.GlanceImageService

# novnc
novnc_enable = true
novncproxy_port = 6080
novncproxy_host = 192.168.0.200
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html
vncserver_listen = 192.168.0.200
vncserver_proxyclient_address = 192.168.0.200
vnc_keymap=ja
---

# Compute
compute_driver=libvirt.LibvirtDriver
connection_type=libvirt
```

初期設定
```
nova-manage db sync
```

再起動
```
ls -1 /etc/init.d/nova-*| while read LINE; do service `basename ${LINE}` restart; done
 → nova-api, nova-cert, nova-compute, nova-conductor, nova-consoleauth, nova-network, nova-novncproxy, nova-schedulerが再起動すること。
```

サービス確認
```
nova-manage service list
 → Stateがsmile「:-)」であればOK
----
Binary           Host                                 Zone             Status     State Updated_At
nova-cert        ryunosuke                            internal         enabled    :-)   2013-12-22 17:59:05
nova-consoleauth ryunosuke                            internal         enabled    :-)   2013-12-22 17:59:05
nova-scheduler   ryunosuke                            internal         enabled    :-)   2013-12-22 17:59:05
nova-conductor   ryunosuke                            internal         enabled    :-)   2013-12-22 17:59:05
nova-compute     ryunosuke                            nova             enabled    :-)   2013-12-22 17:59:05
nova-network     ryunosuke                            internal         enabled    :-)   2013-12-22 17:59:05
----
```


### cinder
インストール
```
cd /usr/local/src
wget http://ftp.jaist.ac.jp/pub/Linux/ubuntu//pool/main/libr/librdmacm/librdmacm1_1.0.15-1_amd64.deb
wget http://ftp.jaist.ac.jp/pub/Linux/ubuntu//pool/main/libi/libibverbs/libibverbs1_1.1.5-1ubuntu1_amd64.deb
dpkg -i librdmacm1_1.0.15-1_amd64.deb libibverbs1_1.1.5-1ubuntu1_amd64.deb
 →cinder-volumeエラーが出るので、先にインストールする。

apt-get -y install cinder-api cinder-common cinder-scheduler cinder-volume ython-cinder python-cinderclient iscsitarget open-iscsi iscsitarget-dkms  linux-headers-`uname -r`
```

※iscsiは使用しないため、公式手順の以下はスキップする。
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
sql_connection = mysql://cinder:password@192.168.0.200/cinder
rabbit_password = password
---

cp -p /etc/cinder/api-paste.ini $BAK
vi /etc/cinder/api-paste.ini
---
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
service_protocol = http
service_host = 192.168.0.200
service_port = 5000
auth_host = 192.168.0.200
auth_port = 35357 
auth_protocol = http
admin_tenant_name = service 
admin_user = cinder  
admin_password = password
---
```

cinder-volume割当て
※事前にPertitionを取れるよう空き領域を確保しておく。
 →Pertitionを取得できなければ、USBメモリやSDCard等を使用する。iSCSIで新たに確保した領域でも可。
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
ls -1 /etc/init.d/cinder-*| while read LINE; do service `basename ${LINE}` restart; done
 → cinder-api, cinder-scheduler, cinder-volumeが再起動すること
```

### quantum
インストール  
※Controller nodeとNetwork nodeを分ける場合は、quantum-serverのみで良い(NetworkNodeへほかを入れる)
```
apt-get -y install quantum-server quantum-common quantum-dhcp-agent quantum-l3-agent quantum-metadata-agent  quantum-plugin-openvswitch quantum-plugin-openvswitch-agent python-quantum python-quantumclient quantum-lbaas-agent 
```

設定変更
```
cp -p /etc/quantum/quantum.conf $BAK
vi /etc/quantum/quantum.conf
---
[DEFAULT]
verbose = True
rabbit_password = password
rabbit_host = 192.168.0.200
...
service_plugins = quantum.plugins.services.agent_loadbalancer.plugin.LoadBalancerPlugin
※LoadBalancer用設定

[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357 
auth_protocol = http
admin_tenant_name = service 
admin_user = quantum 
admin_password = password
---

cp -p /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini $BAK
vi /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini
---
[DATABASE]
sql_connection = mysql://quantum:password@192.168.0.200/quantum
reconnect_interval = 2
...
[OVS]
tenant_network_type = vlan
network_vlan_ranges = physnet1,physnet2:2:4000
tunnel_id_ranges =
integration_bridge = br-int
bridge_mappings = physnet1:br-ex,physnet2:br-eth1
...
※vlan経由でcompute nodeと通信する。通信ポートがbr-eth1(inner)。パケット送出時に表向きポート(br-ex)へマッピングする。

[AGENT]
polling_interval = 2
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
ls -1 /etc/init.d/quantum-*| while read LINE; do service `basename ${LINE}` restart; done
 → quantum-dhcp-agent, quantum-l3-agent, quantum-lbaas-agent, quantum-metadata-agent, quantum-plugin-openvswitch-agent, quantum-serverを再起動できること。

```

### Horizon
インストール
```
apt-get -y install openstack-dashboard python-django-horizon memcached python-memcache
apt-get remove --purge openstack-dashboard-ubuntu-theme
→Ubuntu独自のhorizonメニューを非表示とするためにremoveする。
 (NetworkMapが見えるようになる)
```

ブラウザ確認
http://192.168.0.200/horizon
→Login: admin/password  
 ログインできればOK

## Network node
### quantum

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

OVSインストール
```
apt-get -y install quantum-plugin-openvswitch-agent quantum-dhcp-agent quantum-l3-agent
service openvswitch-switch restart
```

仮想NIC追加
```
ovs-vsctl add-br br-ex
ovs-vsctl add-br br-eth1
ovs-vsctl add-br br-int
ovs-vsctl add-br br-tun
ovs-vsctl add-port br-ex eth0
ovs-vsctl add-port br-eth1 eth1
 →br-ex(external), br-eth1(admin-network), br-int(internal(KVM))を作成し、物理NICを紐づける。

cp -p /etc/network/interfaces $BAK/interfaces.2
vi /etc/network/interfaces
→eth0にIPアドレスを持たせている場合は、br-exへ切替える
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
 network 192.168.0.0
 gateway 192.168.0.254
 dns-nameservers 192.168.0.254
 

## Internal Network
auto eth1
iface eth1 inet manual
 up ip address add 0/0 dev $IFACE
 up ip link set $IFACE up
 down ip link set $IFACE down

auto br-eth1
iface br-eth1 inet static
 address 10.0.1.200
 netmask 255.255.255.0
 network 10.0.1.0
---
```

対応インターフェイス変更
→グローバル側のIPアドレスをbr-exへ寄せる
```
ip addr del 192.168.0.200/24 dev eth0
ip addr add 192.168.0.200/24 dev br-ex
 →通信断になるので、コンソールで実施すること。
```

再起動
```
/etc/init.d/networking restart
 →可能であればrebootして/etc/network/interfacesの設定が反映されていることを確認する。
```

内部→外部通信許可設定
```
iptables -A FORWARD -i br-eth1 -o br-ex -s 10.0.0.0/24 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A POSTROUTING -s 10.0.0.0/24 -t nat -j MASQUERADE
iptables -L
 →追加した設定が入っていればOK。内部から外部へ通信させる必要がある場合のみ設定する。
```

quantum設定
```
vi /etc/quantum/quantum.conf
---
※Controll nodeの設定に合わせる
---

vi /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini
---
※Controll nodeの設定に合わせる
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
[DEFAULT]
auth_url = http://192.168.0.200:35357/v2.0
auth_region = RegionOne
admin_tenant_name = service
admin_user = quantum
admin_password = password
nova_metadata_ip = 192.168.0.200
metadata_proxy_shared_secret = password
---
```

再起動
```
ls -1 /etc/init.d/quantum-*| while read LINE; do service `basename ${LINE}` restart; done
 → quantum-dhcp-agent, quantum-l3-agent, quantum-lbaas-agent, quantum-metadata-agent, quantum-plugin-openvswitch-agent, quantum-serverを再起動できること。
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
DNS_SERVER="8.8.8.8"
TENANT_ID=$(keystone tenant-list | grep " $TENANT_NAME " | awk '{print $2}')

TENANT_NET_ID=$(quantum net-create --tenant_id $TENANT_ID $TENANT_NETWORK_NAME --provider:network_type vlan  --provider:segmentation_id 1 --provider:physical_network physnet2 | grep " id " | awk '{print $4}')
TENANT_SUBNET_ID=$(quantum subnet-create --tenant_id $TENANT_ID --ip_version 4 --name $TENANT_SUBNET_NAME $TENANT_NET_ID $FIXED_RANGE --gateway $NETWORK_GATEWAY --dns_nameservers list=true $DNS_SERVER | grep " id " | awk '{print $4}')
ROUTER_ID=$(quantum router-create --tenant_id $TENANT_ID $TENANT_ROUTER_NAME | grep " id " | awk '{print $4}')
quantum router-interface-add $ROUTER_ID $TENANT_SUBNET_ID
---
bash /usr/local/src/addvmnetwork.sh
 →Added Interface to router ...と出ればOK
```

外部Network設定
```
quantum net-create public --router:external=True --provider:network_type vlan --provider:physical_network physnet1
quantum subnet-create --name public-subnet --ip_version 4 --gateway 192.168.0.254 public 192.168.0.0/24 --allocation-pool start=192.168.0.100,end=192.168.0.150 --disable-dhcp
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
apt-get -y install nova-compute nova-common nova-compute-kvm nova-network python-nova python-novaclient
```

設定
```
cp -ap /etc/nova $BAK
vi /etc/nova/api-paste.ini
---
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 192.168.0.200
~~~★ControllerノードのIPアドレス
auth_port = 35357
auth_protocol = http
admin_tenant_name = service 
admin_user = nova 
admin_password = password
auth_version = v2.0
---

vi /etc/nova/nova.conf
---
[DEFAULT]
...
sql_connection = mysql://nova:password@192.168.0.200/nova
my_ip=192.168.0.210
connection_type=libvirt
libvirt_type = kvm
...
ec2_dmz_host = 192.168.0.200
s3_host = 192.168.0.200
enabled_apis = ec2,osapi_compute,metadata
rabbit_host=192.168.0.200
rabbit_password=password
image_service = nova.image.glance.GlanceImageService
glance_api_servers = 192.168.0.200:9292
network_api_class = nova.network.quantumv2.api.API
quantum_url = http://192.168.0.200:9696
quantum_auth_strategy = keystone
quantum_admin_tenant_name = service
quantum_admin_username = quantum
quantum_admin_password = password
quantum_admin_auth_url = http://192.168.0.200:35357/v2.0
libvirt_vif_driver = nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
novnc_enable=true
novncproxy_port=6080
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = 192.168.0.210
auth_strategy = keystone
...
[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = nova
signing_dirname = /tmp/keystone-signing-nova
---
```

再起動
```
ls -1 /etc/init.d/nova-*| while read LINE; do service `basename ${LINE}` restart; done
 → nova-*が再起動すること。
```

状態確認
```
root@kinder:/etc/quantum# nova-manage service list
Binary           Host                                 Zone             Status     State Updated_At
...
nova-compute     kinder                               nova             enabled    :-)   2013-12-23 03:23:08
nova-network     kinder                               internal         enabled    :-)   2013-12-23 03:23:03
→ compute nodeの nova-compute, nova-networkが「:-)」であればOK
```


Networkインストール
→OpenVswitch, br-exインストール済みのためスキップ

quantum.conf
```
vi /etc/quantum/quantum.conf
---
[DEFAULT]
verbose = True
rabbit_password = password
rabbit_host = 192.168.0.200
~~~~★Controller node
lock_path = $state_path/lock
bind_host = 0.0.0.0
bind_port = 9696
core_plugin = quantum.plugins.openvswitch.ovs_quantum_plugin.OVSQuantumPluginV2
service_plugins = quantum.plugins.services.agent_loadbalancer.plugin.LoadBalancerPlugin
api_paste_config = /etc/quantum/api-paste.ini
control_exchange = quantum
notification_driver = quantum.openstack.common.notifier.rpc_notifier
default_notification_level = INFO
notification_topics = notifications
[QUOTAS]
[DEFAULT_SERVICETYPE]
[AGENT]
root_helper = sudo quantum-rootwrap /etc/quantum/rootwrap.conf
[keystone_authtoken]
auth_host = 192.168.0.200
~~~~★Controller node
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = quantum 
admin_password = password
signing_dir = /var/lib/quantum/keystone-signing
---
```

ovs_quantum_plugin.ini(重要)
```
vi /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini
(vi /etc/quantum/plugin.ini)
---
[DATABASE]
sql_connection = mysql://quantum:password@192.168.0.200/quantum
~~~★Controller node
reconnect_interval = 2
[OVS]
tenant_network_type = vlan
network_vlan_ranges = physnet1,physnet2:2:4000
~~~使用するインターフェイスをのせる。
tunnel_id_ranges =
integration_bridge = br-int
bridge_mappings = physnet1:br-ex,physnet2:br-eth1
~~~★physnet1, physnet2(Inner)をマッピングする。quantum net-createした時の設定と合わせる。
[AGENT]
polling_interval = 2
[SECURITYGROUP]
firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
---
```

NIC設定
```
ovs-vsctl add-br br-eth1
ovs-vsctl add-br br-ex
ovs-vsctl add-br br-int
ovs-vsctl add-br br-tun

ovs-vsctl add-ports br-eth1 eth1
ovs-vsctl add-ports br-ex eth0
```

NIC設定(Interfaces)
```
vi /etc/network/interfaces
---
auto br-ex
iface br-ex inet static
 address 192.168.0.210
 netmask 255.255.255.0
 gateway 192.168.0.254
 dns-nameservers 192.168.0.254
 
## Internal Network
auto eth1
iface eth1 inet manual
 up ip address add 0/0 dev $IFACE
 up ip link set $IFACE up
 down ip link set $IFACE down

auto br-eth1
iface br-eth1 inet static
 address 10.0.1.210
 network 10.0.1.0
 netmask 255.255.255.0
 broadcast 10.0.1.255
---

/etc/init.d/networking restart
```

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
