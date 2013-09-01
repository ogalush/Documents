<!--
************************************************************
OpenStack GrizzyをUbuntu12(x86_64)へインストールする手順
参照元: http://docs.openstack.org/grizzly/basic-install/apt/content/basic-install_controller.html
Copyright (c) Takehiko OGASAWARA 2013 All Rights Reserved.
************************************************************
-->
<div id='title'>　</div>    

# Grizzlyをインストールする方法

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

バックアップ
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
sudo apt-get -y update && apt-get -y upgrade
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
````
cp -p /etc/sysctl.conf $BAK
vi /etc/sysctl.conf
---
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
---
````

サービス再起動
---
/etc/init.d/networking restart
sysctl -e -p /etc/sysctl.conf
→rp_filterが0と表示されればOK
---

ntpdインストール
```
---
apt-get install -y ntp
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
apt-get install -y rabbitmq-server
rabbitmqctl change_password guest password
```

### Keystone

keystone
```
apt-get install -y keystone python-keystone python-keystoneclient
```



------------------
mysql設定更新
```
cp -p /etc/my.cnf $BAK
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/my.cnf
service mysqld start
chkconfig --list |grep -i mysqld
chkconfig mysqld on
```
OpenStack用DB作成
```
mysql -u root -p <<EOF
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
IDENTIFIED BY 'password';
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
IDENTIFIED BY 'password';
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
IDENTIFIED BY 'password';
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'password';
CREATE DATABASE quantum;
GRANT ALL PRIVILEGES ON quantum.* TO 'quantum'@'localhost' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON quantum.* TO 'quantum'@'10.0.2.15' \
IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
EOF
→パスワードが聞かれるが、空白のままEnterしてOK.
```

### Queueing Service
```
yum -y install qpid-cpp-server
# echo auth=1 >> /etc/qpidd.conf
→1を入れない。qpidd.confにauth=YESが既に入っているため。
chkconfig qpidd on
service qpidd start
```

Openstackパッケージのパッケージ登録
```
yum -y install wget
cd /etc/yum.repos.d
rpm -i http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
→CentOSパッケージ用の拡張リポジトリを指定する。OpenStack-**をインストールできる。

```

### KeyStone
```
yum -y install openstack-keystone python-keystone python-keystoneclient
```

設定ファイル編集
```
cp -apr /etc/keystone $BAK
vi /etc/keystone/keystone.conf
```
以下の設定値を追記する
```
[DEFAULT]
admin_token = password
debug = True
verbose = True

[sql]
connection = mysql://keystone:password@localhost/keystone
```

KeyStone用の鍵を作成する
```
keystone-manage pki_setup
chown -R keystone:keystone /etc/keystone/*
```

設定値を反映する
```
service openstack-keystone restart
chkconifig --list |grep -i keystone
chkconfig openstack-keystone on
keystone-manage db_sync
~~~マニュアルはopenstack-dbコマンドであったが、keystone-manageでDBをInitializeする。
openstack-dbコマンドが無かったため。
```

openrcファイル作成
```
vi /root/.openrc
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL="http://localhost:5000/v2.0/"
export OS_SERVICE_ENDPOINT="http://localhost:35357/v2.0"
export OS_SERVICE_TOKEN=password
```
ログイン時に読込むよう設定
```
source /root/.openrc
echo "source /root/.openrc" >> /root/.bashrc
```

keystone初期データ作成
```
vi /usr/local/src/initkeystone.sh
※IPアドレスを確認すること。
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
KEYSTONE_HOST=10.0.2.15

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
~~~★keystoneコマンドを正常終了できること。
(warningはOK、Errorの場合はmmysql:keystoneDBを再作成、keystone db_sync後、shell再実行)
```


### glance
パッケージインストール
```
yum -y install openstack-glance
```

バックアップ
```
cp -apr /etc/glance $BAK
```

DB接続先設定
```
vi /etc/glance/glance-api.conf
vi /etc/glance/glance-registry.conf
glance-api.conf, glance-registry.confへ、下記設定値を適用する
---
[DEFAULT]
sql_connection = mysql://glance:password@localhost/glance
[keystone_authtoken]
admin_tenant_name = service
admin_user = glance
admin_password = password
[paste_deploy]
flavor=keystone
---
```

設定値の適用
```
chkconfig --list |grep -i glance
chkconfig openstack-glance-api on
chkconfig openstack-glance-registry on
chkconfig openstack-glance-scrubber on
service openstack-glance-api restart
service openstack-glance-registry restart
service openstack-glance-scrubber restart
```

DB初期化
```
glance-manage db_sync
```

OSイメージのimport
※OSの種類は任意。
※CentOSを仮想ゲストとする場合は、イメージの手動作成が必要そう。
```
ubuntu 13.04
cd /usr/local/src
wget http://cloud-images.ubuntu.com/releases/13.04/release/ubuntu-13.04-server-cloudimg-amd64-disk1.img
glance image-create --is-public true --disk-format qcow2 --container-format bare --name "Ubuntu 13.04" < ubuntu-13.04-server-cloudimg-amd64-disk1.img
[root@opencent src]# glance image-list
+--------------------------------------+--------------+-------------+------------------+-----------+--------+
| ID                                   | Name         | Disk Format | Container Format | Size      | Status |
+--------------------------------------+--------------+-------------+------------------+-----------+--------+
| 1b78576a-4cd9-4aba-90d5-bb811c845da1 | Ubuntu 13.04 | qcow2       | bare             | 235405312 | active |
+--------------------------------------+--------------+-------------+------------------+-----------+--------+
~~~~イメージをインポートできればOK

```

### nova
パッケージインストール
```
yum install -y openstack-nova-api openstack-nova-scheduler openstack-nova-cert \
  openstack-nova-console openstack-nova-doc genisoimage openstack-dashboard \
  openstack-nova-novncproxy openstack-nova-conductor novnc openstack-nova-compute
```

edit config file
```
cp -ap /etc/nova $BAK
vi /etc/nova/api-paste.ini 
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
service_protocol = http
service_host = 127.0.0.1
service_port = 5000
admin_tenant_name = service 
admin_user  = nova  
admin_password = password
# Workaround for https://bugs.launchpad.net/nova/+bug/1154809
auth_version = v2.0

vi /etc/nova/nova.conf
パスワードを以下へ変更する
sql_connection = mysql://nova:password@localhost/nova

default欄へ追記する
---
# General
verbose = True
qpid_username=guest
qpid_password=guest
rpc_backend = nova.openstack.common.rpc.impl_qpid

# Networking
network_api_class=nova.network.quantumv2.api.API
quantum_url=http://10.0.2.15:9696
quantum_auth_strategy=keystone
quantum_admin_tenant_name=service
quantum_admin_username=quantum
quantum_admin_password=password
quantum_admin_auth_url=http://10.0.2.15:35357/v2.0
libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver

# Security Groups
firewall_driver=nova.virt.firewall.NoopFirewallDriver
security_group_api=quantum

# Metadata
quantum_metadata_proxy_shared_secret=password
service_quantum_metadata_proxy=true
metadata_listen = 10.0.2.15
metadata_listen_port = 8775

# Cinder
volume_api_class=nova.volume.cinder.API

# Glance
glance_api_servers=10.0.2.15:9292
image_service=nova.image.glance.GlanceImageService

# novnc
novnc_enable=true
novncproxy_port=6080
novncproxy_host=10.0.2.15
vncserver_listen=0.0.0.0
---
```

db初期化
```
nova-manage db sync
```

サービス再起動
```
service openstack-nova-api restart
service openstack-nova-cert restart
service openstack-nova-consoleauth restart
service openstack-nova-scheduler restart
service openstack-nova-novncproxy restart
service openstack-nova-compute restart
chkconfig openstack-nova-api on
chkconfig openstack-nova-cert on
chkconfig openstack-nova-consoleauth on
chkconfig openstack-nova-scheduler on
chkconfig openstack-nova-novncproxy on
chkconfig openstack-nova-compute on
※openstack-nova-conductorがないため、conductorのみスキップする。
```

### cinder
install package
```
yum install -y openstack-cinder openstack-cinder-doc \
        iscsi-initiator-utils scsi-target-utils
```

setup iscsi
```
service tgtd start
service iscsi start
chkconfig tgtd on
chkconfig iscsi on
```

設定ファイル更新
```
cp -ap /etc/cinder $BAK
vi /etc/cinder/cinder.conf
---
設定変更、追記
sql_connection = mysql://cinder:password@localhost/cinder
qpid_user = guest
qpid_password = quest
---

vi /etc/cinder/api-paste.ini
---
[filter:authtoken]へ追記
admin_tenant_name = service
admin_user = cinder 
admin_password = password
---
```

cinderストレージセットアップ
```
pvcreate /dev/sdb
vgcreate cinder-volumes /dev/sdb
```

db初期化
```
cinder-manage db sync
```

サービス起動設定
```
service openstack-cinder-api restart
service openstack-cinder-scheduler restart
service openstack-cinder-volume restart
chkconfig openstack-cinder-api on
chkconfig openstack-cinder-scheduler on
chkconfig openstack-cinder-volume on
```

### quantum with Linux Bridge
OpenVswitchがイマイチなので、Linux Bridgeを使用する。

install Package
```
yum -y install openstack-quantum-linuxbridge openstack-quantum
```

設定ファイル更新
```
cp -ap /etc/quantum $BAK
vi /etc/quantum/quantum.conf
---
core_plugin = quantum.plugins.linuxbridge.lb_quantum_plugin.LinuxBridgePluginV2
...
auth_strategy = keystone
...
rpc_backend=quantum.openstack.common.rpc.impl_qpid
fake_rabbit = False
---

vi /etc/quantum/plugins/linuxbridge/linuxbridge_conf.ini 
---
tenant_network_type = vlan
network_vlan_ranges = physnet1:1:1000
sql_connection = mysql://quantum:password@127.0.0.1:3306/quantum
physical_interface_mappings = physnet1:eth0
---

vi /etc/quantum/api-paste.ini 
---
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
...
admin_tenant_name = service
admin_user = quantum
admin_password = password
---

ln -s /etc/quantum/plugins/linuxbridge/linuxbridge_conf.ini /etc/quantum/plugin.ini

```

サービス再起動
```
service quantum-server 
service quantum-linuxbridge-agent start
chkconfig quantum-server on
chkconfig quantum-linuxbridge-agent on
```

### horizon
パッケージインストール
```
yum -y install httpd memcached
```

初期設定
```
 ( cat | sudo tee -a /etc/openstack-dashboard/local_settings ) <<EOF
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'horizon',
        'USER': 'horizon',
        'PASSWORD': 'password',
        'HOST': '10.0.2.15',
        'PORT': '',
    }
}
EOF
```

サービス再起動
```
service httpd start
service memcached start
chkconfig httpd on
chkconfig memcached on
```
