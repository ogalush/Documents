<!--
************************************************************
OpenStack IceHouseをUbuntu14.04(x86_64)へインストールする手順
参照元: http://docs.openstack.org/icehouse/install-guide/install/apt/content/
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->

# ControllNodeをインストールする方法(OpenStack IceHouse)

### 準備
バックアップディレクトリ作成
```
mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

ntpインストール
```
# apt-get -y install ntp
```

MySQLインストール
```
# apt-get -y install python-mysqldb mysql-server
→ パスワード: admin!
```

MySQL設定
```
# cp -raf /etc/mysql $BAK
# vi /etc/mysql/my.cnf
----
[mysqld]
...
bind-address            = 192.168.0.200
...
default-storage-engine = innodb
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
----

#-- 設定の反映
# service mysql restart

#-- 初期化
# mysql_install_db
# mysql_secure_installation
→ rootのパスワード変更等聞かれるが、後でnovaユーザ等を作るのでひとまずNoで良い。
```

RabbitMQインストール
```
# apt-get install -y rabbitmq-server
# rabbitmqctl change_password guest admin!
```


KeyStoneインストール
```
# apt-get -y install keystone
```

KeyStone設定
```
# cp -raf /etc/keystone $BAK
# vi /etc/keystone/keystone.conf
----
### connection = sqlite:////var/lib/keystone/keystone.db
connection = mysql://keystone:password@192.168.0.200/keystone
----

#-- sqliteファイル削除
# rm /var/lib/keystone/keystone.db

#-- keystoneユーザ作成
# mysql -u root -p

mysql> CREATE DATABASE keystone;
mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
mysql> FLUSH PRIVILEGES;
mysql> ¥q

#-- keystoneDB初期化
# su -s /bin/sh -c "keystone-manage db_sync" keystone

#-- keystoneパスワード反映
# vi /etc/keystone/keystone.conf
----
[default]
# admin_token=ADMIN
admin_token=password
...
log_dir = /var/log/keystone
----

#-- 設定反映
# service keystone restart
# service keystone status

#-- 有効期限が切れたtokenの掃除スクリプト
# (crontab -l 2>&1 | grep -q token_flush) || \
echo '@hourly /usr/bin/keystone-manage token_flush >/var/log/keystone/keystone-tokenflush.log 2>&1' >> /var/spool/cron/crontabs/root

# cat /var/spool/cron/crontabs/root 
→設定が追加されていること。

#-- テナント作成
# export OS_SERVICE_TOKEN=password
# export OS_SERVICE_ENDPOINT=http://192.168.0.200:35357/v2.0

#-- adminテナント
# keystone user-create --name=admin --pass=password --email=admin@192.168.0.200
# keystone role-create --name=admin
# keystone tenant-create --name=admin --description="Admin Tenant"
# keystone user-role-add --user=admin --tenant=admin --role=admin
# keystone user-role-add --user=admin --role=_member_ --tenant=admin

#-- demoテナント
# keystone user-create --name=demo --pass=password --email=demo@192.168.0.200
# keystone tenant-create --name=demo --description="Demo Tenant"
# keystone user-role-add --user=demo --role=_member_ --tenant=demo

#-- Serviceテナント
# keystone tenant-create --name=service --description="Service Tenant"

#-- サービス作成
# keystone service-create --name=keystone --type=identity --description="OpenStack Identity"
# keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ identity / {print $2}') \
  --publicurl=http://192.168.0.200:5000/v2.0 \
  --internalurl=http://192.168.0.200:5000/v2.0 \
  --adminurl=http://192.168.0.200:35357/v2.0

#-- KeyStoneインストール確認
# unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT
# keystone --os-username=admin --os-password=password --os-auth-url=http://192.168.0.200:35357/v2.0 token-get
→ tokenが表示されればOK

# keystone --os-username=admin --os-password=password --os-tenant-name=admin --os-auth-url=http://192.168.0.200:35357/v2.0 token-get
→ tokenが表示されればOK

#-- 環境変数設定
# vi ~/.admin-openrc.sh
----
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_TENANT_NAME=admin
export OS_AUTH_URL=http://192.168.0.200:35357/v2.0
----
# echo 'source ~/.admin-openrc.sh' >> ~/.bashrc
# source ~/.bashrc

#-- 環境変数設定後の確認
# keystone token-get
→ tokenが表示されればOK

# keystone user-list
→ admin,demoが表示されればOK

# keystone user-role-list --user admin --tenant admin
→ admin, _member_が表示されればOK
```


