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
