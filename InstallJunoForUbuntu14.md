<!--
************************************************************
OpenStack JunoをUbuntu14.04(x86_64)へインストールする手順
参照元: http://docs.openstack.org/juno/install-guide/install/apt/content/ 
Copyright (c) Takehiko OGASAWARA 2015 All Rights Reserved.
************************************************************
-->

# OpenStack(Juno)をインストールする
[公式インストール手順](http://docs.openstack.org/juno/install-guide/install/apt/content/)

## Controller node
### 準備
バックアップディレクトリ作成
```
$ sudo mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
$ BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

### Basicインストール
ntpインストール
```
$ sudo apt-get -y install ntp
```

OpenStackパッケージ
```
$ sudo apt-get -y install ubuntu-cloud-keyring
$ echo 'deb http://ubuntu-cloud.archive.canonical.com/ubuntu trusty-updates/juno main' | sudo tee /etc/apt/sources.list.d/cloudarchive-juno.list
$ cat /etc/apt/sources.list.d/cloudarchive-juno.list
$ sudo apt-get -y update && sudo apt-get -y dist-upgrade
```

MariaDB(MySQL)
```
$ sudo apt-get -y install mariadb-server python-mysqldb
→ パスワード: admin!
```

MariaDB設定
```
$ sudo cp -raf /etc/mysql $BAK
$ sudo vi /etc/mysql/my.cnf
----
[mysqld]
...
bind-address            = 0.0.0.0
...
#-- For OpenStack
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
----

#-- 設定の反映
$ sudo service mysql restart

#-- 初期化
$ sudo mysql_secure_installation
→ rootのパスワード変更等聞かれるが、後でnovaユーザ等を作るのでひとまずNoで良い。
---
Enter current password for root (enter for none): 
OK, successfully used password, moving on...
Change the root password? [Y/n] n
 ... skipping.

Remove anonymous users? [Y/n] y
 ... Success!

Disallow root login remotely? [Y/n] y
 ... Success!

Remove test database and access to it? [Y/n] y
 ... Success!

Reload privilege tables now? [Y/n] y
 ... Success!
---
```

RabbitMQ
```
$ sudo apt-get -y install rabbitmq-server
$ sudo rabbitmqctl change_password guest admin!
```


### KeyStone

DB作成
```
$ mysql -u root -p
MariaDB> CREATE DATABASE keystone;
MariaDB> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
MariaDB> FLUSH PRIVILEGES;
MariaDB> quit;
```

パッケージインストール
```
$ sudo apt-get -y install keystone python-keystoneclient
```

KyeStone設定
```
$ sudo cp -raf /etc/keystone $BAK
$ sudo vi /etc/keystone/keystone.conf
---
[DEFAULT]
...
admin_token = token
...
[database]
connection = mysql://keystone:password@192.168.0.200/keystone
[token]
provider = keystone.token.providers.uuid.Provider
driver = keystone.token.persistence.backends.sql.Token
---
```

DB反映
```
$ sudo su -s /bin/sh -c "keystone-manage db_sync" keystone
```

再起動
```
$ sudo service keystone restart
$ sudo rm -f /var/lib/keystone/keystone.db
```

Cron設定
```
$ (sudo crontab -l -u keystone 2>&1 | grep -q token_flush) || echo '@hourly /usr/bin/keystone-manage token_flush  /var/log/keystone/keystone-tokenflush.log 2>&1' |sudo tee /var/spool/cron/crontabs/keystone
$ sudo chown keystone:keystone /var/spool/cron/crontabs/keystone
$ sudo ls -al /var/spool/cron/crontabs/keystone
$ sudo cat /var/spool/cron/crontabs/keystone
```

環境変数（一時使用）
```
$ export OS_SERVICE_TOKEN=password
$ export OS_SERVICE_ENDPOINT=http://192.168.0.200:35357/v2.0
```

テナント・ユーザ作成
```
$ keystone tenant-create --name admin --description "Admin Tenant"
$ keystone user-create --name admin --pass password --email admin@192.168.0.200
$ keystone role-create --name admin
$ keystone user-role-add --user admin --tenant admin --role admin
$ keystone tenant-create --name demo --description "Demo Tenant"
$ keystone user-create --name demo --tenant demo --pass password --email demo@192.168.0.200
$ keystone tenant-create --name service --description "Service Tenant"
```

サービス作成
```
$ keystone service-create --name keystone --type identity --description "OpenStack Identity"
$ keystone endpoint-create --service-id $(keystone service-list | awk '/ identity / {print $2}') --publicurl http://192.168.0.200:5000/v2.0 --internalurl http://192.168.0.200:5000/v2.0 --adminurl http://192.168.0.200:35357/v2.0 --region regionOne
```

環境変数解除
```
$ unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT
```

.keystone設定
```
$ vi ~/.keystonerc
---
#!/bin/bash

export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.0.200:35357/v2.0
---

$ vi ~/.bashrc
---
...
source ~/.keystonerc
---
$ source ~/.bashrc
```

確認
```
$ keystone token-get
$ keystone user-list
$ keystone role-list
$ keystone tenant-list
```

### Glance
DB設定
```
$ mysql -u root -p
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> ¥q
```

keystone設定
```
$ keystone user-create --name glance --pass password
$ keystone user-role-add --user glance --tenant service --role admin
$ keystone service-create --name glance --type image --description "OpenStack Image Service"
$ keystone endpoint-create --service-id $(keystone service-list | awk '/ image / {print $2}') --publicurl http://192.168.0.200:9292 --internalurl http://192.168.0.200:9292 --adminurl http://192.168.0.200:9292 --region regionOne
```

glanceパッケージ
```
$ sudo apt-get -y install glance python-glanceclient
```

glance設定
```
$ sudo cp -raf /etc/glance $BAK
$ sudo vi /etc/glance/glance-api.conf
---
[database]
...
connection = mysql://glance:password@192.168.0.200/glance
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000/v2.0
identity_uri = http://192.168.0.200:35357
admin_tenant_name = service
admin_user = glance
admin_password = password
...
[paste_deploy]
...
flavor = keystone
...
[glance_store]
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
---

$ sudo vi /etc/glance/glance-registry.conf
---
[database]
connection = mysql://glance:password@192.168.0.200/glance
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000/v2.0
identity_uri = http://192.168.0.200:35357
admin_tenant_name = service
admin_user = glance
admin_password = password
...
[paste_deploy]
flavor = keystone
...
---

$ sudo su -s /bin/sh -c "glance-manage db_sync" glance
```

glance再起動
```
$ sudo service glance-registry restart
$ sudo service glance-api restart
$ rm -f /var/lib/glance/glance.sqlite
```

OSイメージインポート
```
$ sudo wget -P /usr/local/src http://cloud-images.ubuntu.com/releases/14.04/release/ubuntu-14.04-server-cloudimg-amd64-disk1.img

```
