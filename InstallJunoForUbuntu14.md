<!--
************************************************************
OpenStack JunoをUbuntu14.04(x86_64)へインストールする手順
参照元: http://docs.openstack.org/juno/install-guide/install/apt/content/ 
Copyright (c) Takehiko OGASAWARA 2015 All Rights Reserved.
************************************************************
-->

# OpenStack(Juno)をインストールする

## Controller node
### 準備
バックアップディレクトリ作成
```
$ sudo mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
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

MariaDB(MySQL)インストール
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

RabbitMQインストール
```
$ sudo apt-get -y install rabbitmq-server
$ sudo rabbitmqctl change_password guest admin!
```


### KeyStoneインストール

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
export OS_SERVICE_TOKEN=token
export OS_SERVICE_ENDPOINT=http://192.168.0.200:35357/v2.0
---

$ vi ~/.bashrc
---
...
source ~/.keystonerc
---

$ source ~/.bashrc
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

確認
```
$ keystone token-get
$ keystone user-list
$ keystone role-list
$ keystone tenant-list
```
