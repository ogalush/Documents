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

```
