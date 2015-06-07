<!--
************************************************************
OpenStack KiloをUbuntu14.04(x86_64)へインストールする手順
参照元: http://docs.openstack.org/kilo/install-guide/install/apt/content/  
Copyright (c) Takehiko OGASAWARA 2015 All Rights Reserved.
************************************************************
-->

# OpenStack(Kilo)をインストールする
[公式インストール手順](http://docs.openstack.org/kilo/install-guide/install/apt/content/)

## Controller node
### 準備
バックアップディレクトリ作成
```
$ sudo mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
$ BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

### Basicインストール
CPU GOVERNER変更  
[Performanceへ変更](https://github.com/ogalush/documents/blob/master/setCPUPerformance.md)  

ntpインストール
```
$ sudo apt-get -y install ntp
$ ntpq -p
→ 同期され始めているようであればOK.
```

OpenStackパッケージ
```
$ sudo apt-get -y install ubuntu-cloud-keyring
$ echo 'deb http://ubuntu-cloud.archive.canonical.com/ubuntu trusty-updates/kilo main' | sudo tee /etc/apt/sources.list.d/cloudarchive-kilo.list
$ cat /etc/apt/sources.list.d/cloudarchive-kilo.list
$ sudo apt-get -y update && sudo apt-get -y update && sudo apt-get -y dist-upgrade
```

MariaDB(MySQL)
```
$ sudo apt-get -y install mariadb-server python-mysqldb
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

Reload privilege tables now? [Y/n] y
 ... Success!
---
```

RabbitMQ
```
$ sudo apt-get -y install rabbitmq-server
$ sudo rabbitmqctl add_user openstack password
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```


### KeyStone

DB作成
```
$ sudo mysql -u root -ppassword
MariaDB> CREATE DATABASE keystone;
MariaDB> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
MariaDB> FLUSH PRIVILEGES;
MariaDB> quit;
```

パッケージインストール
```
変更点の説明
---
By default, the keystone service listens on ports 5000 and 35357.
However, this guide configures the Apache HTTP server to listen on those ports.
To avoid port conflicts, disable the keystone service from starting automatically after installation:
→ 今回はPort 5000/35357をApacheで受けさせるために、KeyStoneのPort番号を変更するとのこと.
---

$ echo 'manual' | sudo tee /etc/init/keystone.override
$ sudo apt-get -y install keystone python-openstackclient apache2 libapache2-mod-wsgi memcached python-memcache
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
...
[memcache]
...
servers = localhost:11211
...
[token]
provider = keystone.token.providers.uuid.Provider
driver = keystone.token.persistence.backends.memcache.Token
...
[revoke]
driver = keystone.contrib.revoke.backends.sql.Revoke
---
```

DB反映
```
$ sudo su -s /bin/sh -c 'keystone-manage db_sync' keystone
```

apache2編集
```
$ sudo cp -raf /etc/apache2 $BAK
$ sudo vi /etc/apache2/apache2.conf 
---
...
ServerName ryunosuke.local
...
---

$ sudo vi /etc/apache2/sites-available/wsgi-keystone.conf
---
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /var/www/cgi-bin/keystone/main
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    LogLevel info
    ErrorLog /var/log/apache2/keystone-error.log
    CustomLog /var/log/apache2/keystone-access.log combined
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /var/www/cgi-bin/keystone/admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    LogLevel info
    ErrorLog /var/log/apache2/keystone-error.log
    CustomLog /var/log/apache2/keystone-access.log combined
</VirtualHost>
---

$ sudo ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled
$ sudo mkdir -p /var/www/cgi-bin/keystone
$ curl http://git.openstack.org/cgit/openstack/keystone/plain/httpd/keystone.py?h=stable/kilo | sudo tee /var/www/cgi-bin/keystone/main /var/www/cgi-bin/keystone/admin
$ sudo chown -R keystone:keystone /var/www/cgi-bin/keystone
$ sudo chmod 755 /var/www/cgi-bin/keystone/*
$ sudo service apache2 restart
$ sudo rm -f /var/lib/keystone/keystone.db
$ sudo service keystone restart
```

KeyStone設定
```
$ export OS_TOKEN=token
$ export OS_URL=http://192.168.0.200:35357/v2.0
$ export OS_AUTH_URL=http://192.168.0.200:35357/v3
$ openstack service create --name keystone --description "OpenStack Identity" identity

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Identity               |
| enabled     | True                             |
| id          | 148ddb1eff5c4eeebd16892f04173ac9 |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+

$ openstack endpoint create --publicurl http://192.168.0.200:5000/v2.0 --internalurl http://192.168.0.200:5000/v2.0  --adminurl http://192.168.0.200:35357/v2.0 --region RegionOne identity

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| adminurl     | http://192.168.0.200:35357/v2.0  |
| id           | 593c069dc6d24920b2320c472652c16b |
| internalurl  | http://192.168.0.200:5000/v2.0   |
| publicurl    | http://192.168.0.200:5000/v2.0   |
| region       | RegionOne                        |
| service_id   | 148ddb1eff5c4eeebd16892f04173ac9 |
| service_name | keystone                         |
| service_type | identity                         |
+--------------+----------------------------------+

$ openstack project create --description "Admin Project" admin
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Admin Project                    |
| enabled     | True                             |
| id          | 7460bfbafaba45cdac98542810a92ac9 |
| name        | admin                            |
+-------------+----------------------------------+

$ openstack user create --password-prompt admin

User Password: password
Repeat User Password: password
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | 859115906b1c429181dee2de5729478b |
| name     | admin                            |
| username | admin                            |
+----------+----------------------------------+

$ openstack role create admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 167be3dc6eeb468a986f701f6c9986c6 |
| name  | admin                            |
+-------+----------------------------------+

$ openstack role add --project admin --user admin admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 167be3dc6eeb468a986f701f6c9986c6 |
| name  | admin                            |
+-------+----------------------------------+

$ openstack project create --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| enabled     | True                             |
| id          | f8cecc6e1f264d3eb4ffbf9e90f555ee |
| name        | service                          |
+-------------+----------------------------------+

$ openstack project create --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| enabled     | True                             |
| id          | 5cc062a3bb494d96ba860b7c6c319f34 |
| name        | demo                             |
+-------------+----------------------------------+

$ openstack user create --password-prompt demo
User Password: password
Repeat User Password: password
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | adb2761beba44ff6bc9bf7f38d709eae |
| name     | demo                             |
| username | demo                             |
+----------+----------------------------------+

$ openstack role create user
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 2d14b106ba44431eabc6a7913fe6ed3c |
| name  | user                             |
+-------+----------------------------------+

$ openstack role add --project demo --user demo user
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 2d14b106ba44431eabc6a7913fe6ed3c |
| name  | user                             |
+-------+----------------------------------+
```

セキュリティ的な理由で削除する設定
```
$ sudo vi /etc/keystone/keystone-paste.ini
---
[pipeline:public_api]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension user_crud_extension public_service
※ admin_token_authを削除

[pipeline:admin_api]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension s3_extension crud_extension admin_service
※ admin_token_authを削除

[pipeline:api_v3]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension_v3 s3_extension simple_cert_extension revoke_extension federation_extension oauth1_extension endpoint_filter_extension endpoint_policy_extension service_v3
※ admin_token_authを削除
---
```

確認
```
$ unset OS_TOKEN OS_URL
$ openstack --os-auth-url http://192.168.0.200:35357 --os-project-name admin --os-username admin --os-auth-type password token issue
Password: 
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-06-07T05:32:00Z             |
| id         | 19213b2d320c4c209b04b2543f412564 |
| project_id | 7460bfbafaba45cdac98542810a92ac9 |
| user_id    | 859115906b1c429181dee2de5729478b |
+------------+----------------------------------+

$ openstack --os-auth-url http://192.168.0.200:35357 --os-project-domain-id default --os-user-domain-id default  --os-project-name admin --os-username admin --os-auth-type password token issue
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-06-07T05:32:19.038924Z      |
| id         | 842183196d5f496291a9ddd9fc8ee4f5 |
| project_id | 7460bfbafaba45cdac98542810a92ac9 |
| user_id    | 859115906b1c429181dee2de5729478b |
+------------+----------------------------------+

$ vi ~/admin-openrc.sh
---
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.0.200:35357/v3
---

$ vi ~/demo-openrc.sh
---
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=demo
export OS_TENANT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=password
export OS_AUTH_URL=http://192.168.0.200:5000/v3
---

$ source ~/admin-openrc.sh
$ echo 'source ~/admin-openrc.sh' | tee -a ~/.bashrc
$ openstack token issue
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-06-07T05:33:16.984394Z      |
| id         | a03550196ede496c9b8dd3ef8325ac64 |
| project_id | 7460bfbafaba45cdac98542810a92ac9 |
| user_id    | 859115906b1c429181dee2de5729478b |
+------------+----------------------------------+
```

### Glance
DB設定
```
$ sudo mysql -u root -ppassword
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> ¥q
```

keystone設定
```
$ source ~/admin-openrc.sh
$ openstack user create --password-prompt glance
---
User Password: password
Repeat User Password: password
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | 011ef396d0184fd1ac9d7a2f99294b06 |
| name     | glance                           |
| username | glance                           |
+----------+----------------------------------+
---

$ openstack role add --project service --user glance admin
---
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 167be3dc6eeb468a986f701f6c9986c6 |
| name  | admin                            |
+-------+----------------------------------+
---

$ openstack service create --name glance --description "OpenStack Image service" image
---
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image service          |
| enabled     | True                             |
| id          | 61351360283946af93d366df1a3b46ed |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
---

$ openstack endpoint create  --publicurl http://192.168.0.200:9292  --internalurl http://192.168.0.200:9292   --adminurl http://192.168.0.200:9292 --region RegionOne image
---
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| adminurl     | http://192.168.0.200:9292        |
| id           | 44019a21a38f44d3a9e24244c33aa30b |
| internalurl  | http://192.168.0.200:9292        |
| publicurl    | http://192.168.0.200:9292        |
| region       | RegionOne                        |
| service_id   | 61351360283946af93d366df1a3b46ed |
| service_name | glance                           |
| service_type | image                            |
+--------------+----------------------------------+
---
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
[DEFAULT]
...
notification_driver = noop
...

[database]
...
connection = mysql://glance:password@192.168.0.200/glance
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = glance
password = password
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
[DEFAULT]
...
notification_driver = noop
...

[database]
connection = mysql://glance:password@192.168.0.200/glance
...
[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = glance
password = password

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
$ echo "export OS_IMAGE_API_VERSION=2" | tee -a ~/admin-openrc.sh ~/demo-openrc.sh
$ source ~/admin-openrc.sh
$ sudo wget -P /usr/local/src  http://cloud-images.ubuntu.com/releases/14.04/release/ubuntu-14.04-server-cloudimg-amd64-disk1.img

$ glance image-create --name "Ubuntu14.04.2" --file /usr/local/src/ubuntu-14.04-server-cloudimg-amd64-disk1.img
 --disk-format qcow2 --container-format bare --visibility public --progress
---
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | c5229787d24e3478781f589468e376fa     |
| container_format | bare                                 |
| created_at       | 2015-05-31T08:39:19Z                 |
| disk_format      | qcow2                                |
| id               | 1b699de4-b435-4922-8df8-5adeddef0efb |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | Ubuntu15.04                          |
| owner            | f5d9444d84ff4245ae556f3c174a0e32     |
| protected        | False                                |
| size             | 284819968                            |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2015-05-31T08:39:20Z                 |
| virtual_size     | None                                 |
| visibility       | public                               |
+------------------+--------------------------------------+
---
  
$ glance image-list
---
+--------------------------------------+-------------+
| ID                                   | Name        |
+--------------------------------------+-------------+
| 1b699de4-b435-4922-8df8-5adeddef0efb | Ubuntu15.04 |
+--------------------------------------+-------------+
---
```

### nova
DB設定
```
$ mysql -uroot -ppassword
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> ¥q
```
keystone設定
```
$ source ~/admin-openrc.sh
$ openstack user create --password-prompt nova
---
User Password:
Repeat User Password:
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | 9b670915247e458aba7d961f26a75250 |
| name     | nova                             |
| username | nova                             |
+----------+----------------------------------+
---

$ openstack role add --project service --user nova admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 167be3dc6eeb468a986f701f6c9986c6 |
| name  | admin                            |
+-------+----------------------------------+

$ openstack service create --name nova --description "OpenStack Compute" compute
---
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | d7dd027e347c46939546a260ad2548d1 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
---

$ openstack endpoint create --publicurl http://192.168.0.200:8774/v2/%\(tenant_id\)s  --internalurl http://192.168.0.200:8774/v2/%\(tenant_id\)s --adminurl http://192.168.0.200:8774/v2/%\(tenant_id\)s --region RegionOne compute
---
+--------------+--------------------------------------------+
| Field        | Value                                      |
+--------------+--------------------------------------------+
| adminurl     | http://192.168.0.200:8774/v2/%(tenant_id)s |
| id           | 27c2b31a26fb42e7b197b89938c88eb4           |
| internalurl  | http://192.168.0.200:8774/v2/%(tenant_id)s |
| publicurl    | http://192.168.0.200:8774/v2/%(tenant_id)s |
| region       | RegionOne                                  |
| service_id   | d7dd027e347c46939546a260ad2548d1           |
| service_name | nova                                       |
| service_type | compute                                    |
+--------------+--------------------------------------------+
---
```

novaインストール
```
$ sudo apt-get -y install nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy nova-scheduler  python-novaclient
```

nova設定
```
$ sudo cp -raf /etc/nova $BAK
$ sudo vi /etc/nova/nova.conf
---
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone
my_ip = 192.168.0.200
vncserver_listen = 192.168.0.200
vncserver_proxyclient_address = 192.168.0.200
vnc_enabled = True
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html

[database]
connection = mysql://nova:password@192.168.0.200/nova

[oslo_messaging_rabbit]
rabbit_host = 192.168.0.200
rabbit_userid = openstack
rabbit_password = password

[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = nova
password = password

[glance]
host = 192.168.0.200

[oslo_concurrency]
lock_path = /var/lib/nova/tmp
---

$ sudo su -s /bin/sh -c "nova-manage db sync" nova
```

nova再起動
```
$ service --status-all |grep nova | awk '{print $4}' | awk '{print "sudo service "$1" restart"}'| bash

※ 以下のサービスがrestartすればOK
---
nova-api
nova-cert
nova-consoleauth
nova-scheduler
nova-conductor
nova-novncproxy
---
$ sudo rm -f /var/lib/nova/nova.sqlite
```

Hypervisor設定
```
$ sudo apt-get -y install nova-compute sysfsutils
$ sudo vi /etc/nova/nova-compute.conf
---
[libvirt]
virt_type = kvm
---
$ sudo service nova-compute restart
```

nova確認
```
5つのサービスが起動していればOK.
$ source ~/admin-openrc.sh
$ nova service-list
---
+----+------------------+-----------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host      | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+-----------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-cert        | ryunosuke | internal | enabled | up    | 2015-06-07T05:05:10.000000 | -               |
| 2  | nova-conductor   | ryunosuke | internal | enabled | up    | 2015-06-07T05:05:11.000000 | -               |
| 3  | nova-consoleauth | ryunosuke | internal | enabled | up    | 2015-06-07T05:05:11.000000 | -               |
| 4  | nova-scheduler   | ryunosuke | internal | enabled | up    | 2015-06-07T05:05:02.000000 | -               |
| 5  | nova-compute     | ryunosuke | nova     | enabled | up    | 2015-06-07T05:05:10.000000 | -               |
+----+------------------+-----------+----------+---------+-------+----------------------------+-----------------+
---

$ nova endpoints
---
WARNING: keystone has no endpoint in ! Available endpoints for this service:
+-----------+----------------------------------+
| keystone  | Value                            |
+-----------+----------------------------------+
| id        | 54c705b71eec4c0794c402a0a53a16ae |
| interface | internal                         |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://192.168.0.200:5000/v2.0   |
+-----------+----------------------------------+
+-----------+----------------------------------+
| keystone  | Value                            |
+-----------+----------------------------------+
| id        | c437777bc6524b05a68292396f5d4f72 |
| interface | admin                            |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://192.168.0.200:35357/v2.0  |
+-----------+----------------------------------+
+-----------+----------------------------------+
| keystone  | Value                            |
+-----------+----------------------------------+
| id        | d75dc09e19cf4767bba2de666a249152 |
| interface | public                           |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://192.168.0.200:5000/v2.0   |
+-----------+----------------------------------+
WARNING: glance has no endpoint in ! Available endpoints for this service:
+-----------+----------------------------------+
| glance    | Value                            |
+-----------+----------------------------------+
| id        | 488218b4b7574b43b2b1be897e68900f |
| interface | internal                         |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://192.168.0.200:9292        |
+-----------+----------------------------------+
+-----------+----------------------------------+
| glance    | Value                            |
+-----------+----------------------------------+
| id        | 52f0cb6391c8418a80b764b8d673261a |
| interface | admin                            |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://192.168.0.200:9292        |
+-----------+----------------------------------+
+-----------+----------------------------------+
| glance    | Value                            |
+-----------+----------------------------------+
| id        | 7b1c478a62d54bb59eb848fb8a477a01 |
| interface | public                           |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://192.168.0.200:9292        |
+-----------+----------------------------------+
WARNING: nova has no endpoint in ! Available endpoints for this service:
+-----------+---------------------------------------------------------------+
| nova      | Value                                                         |
+-----------+---------------------------------------------------------------+
| id        | 386c2d03bc154b07966746e6728200cc                              |
| interface | internal                                                      |
| region    | RegionOne                                                     |
| region_id | RegionOne                                                     |
| url       | http://192.168.0.200:8774/v2/7460bfbafaba45cdac98542810a92ac9 |
+-----------+---------------------------------------------------------------+
+-----------+---------------------------------------------------------------+
| nova      | Value                                                         |
+-----------+---------------------------------------------------------------+
| id        | 4a04dd033f33416bbda6d3c08a6db7f4                              |
| interface | admin                                                         |
| region    | RegionOne                                                     |
| region_id | RegionOne                                                     |
| url       | http://192.168.0.200:8774/v2/7460bfbafaba45cdac98542810a92ac9 |
+-----------+---------------------------------------------------------------+
+-----------+---------------------------------------------------------------+
| nova      | Value                                                         |
+-----------+---------------------------------------------------------------+
| id        | a4e658dcf99f4a8685b2470c87bafb4d                              |
| interface | public                                                        |
| region    | RegionOne                                                     |
| region_id | RegionOne                                                     |
| url       | http://192.168.0.200:8774/v2/7460bfbafaba45cdac98542810a92ac9 |
+-----------+---------------------------------------------------------------+
---

glanceイメージが表示されればOK.
$ nova image-list
+--------------------------------------+-------------+--------+--------+
| ID                                   | Name        | Status | Server |
+--------------------------------------+-------------+--------+--------+
| 1b699de4-b435-4922-8df8-5adeddef0efb | Ubuntu15.04 | ACTIVE |        |
+--------------------------------------+-------------+--------+--------+
```


### neutron
DB設定
```
$ mysql -uroot -ppassword
MariaDB [(none)]> CREATE DATABASE neutron;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> ¥q
```

keystone設定
```
$ source ~/admin-openrc.sh 
$ openstack user create --password-prompt neutron
User Password:
Repeat User Password:
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | e1b943661dbe4ea881509271f5bffc8a |
| name     | neutron                          |
| username | neutron                          |
+----------+----------------------------------+

$ openstack role add --project service --user neutron admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 167be3dc6eeb468a986f701f6c9986c6 |
| name  | admin                            |
+-------+----------------------------------+

$ openstack service create --name neutron --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 8067ae01260c4f849b8101acbf474f89 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

$ openstack endpoint create --publicurl http://192.168.0.200:9696 --adminurl http://192.168.0.200:9696 --internalurl http://192.168.0.200:9696 --region RegionOne network
---
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| adminurl     | http://192.168.0.200:9696        |
| id           | 200a9388f6b746e3b7fa0ab198707b43 |
| internalurl  | http://192.168.0.200:9696        |
| publicurl    | http://192.168.0.200:9696        |
| region       | RegionOne                        |
| service_id   | 8067ae01260c4f849b8101acbf474f89 |
| service_name | neutron                          |
| service_type | network                          |
+--------------+----------------------------------+
---

```

neutronパッケージ
```
$ sudo apt-get -y install neutron-server neutron-plugin-ml2 python-neutronclient
```

neutron設定
```
$ sudo cp -raf /etc/neutron $BAK
$ sudo vi /etc/neutron/neutron.conf
---
[DEFAULT]
rpc_backend = rabbit
auth_strategy = keystone
...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
...
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
nova_url = http://192.168.0.200:8774/v2

[database]
connection = mysql://neutron:password@192.168.0.200/neutron
...

[oslo_messaging_rabbit]
...
rabbit_host = 192.168.0.200
rabbit_userid = openstack
rabbit_password = password


[keystone_authtoken]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = password

[nova]
auth_url = http://192.168.0.200:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
region_name = RegionOne
project_name = service
username = nova
password = password
---
```

ml2設定
```
$ sudo vi /etc/neutron/plugins/ml2/ml2_conf.ini
---
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = gre
mechanism_drivers = openvswitch
...
[ml2_type_gre]
tunnel_id_ranges = 1:1000
...
[securitygroup]
enable_security_group = True
enable_ipset = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
---
```

nova設定追加
```
$ sudo vi /etc/nova/nova.conf
---
[DEFAULT]
...
network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
...
[neutron]
url = http://192.168.0.200:9696
auth_strategy = keystone
admin_auth_url = http://192.168.0.200:35357/v2.0
admin_tenant_name = service
admin_username = neutron
admin_password = password
---
```

設定反映
```
$ sudo su -s /bin/bash -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

neutron, nova再起動
```
$ sudo service nova-api restart
$ sudo service neutron-server restart
```

反映確認
```
$ source ~/admin-openrc.sh
$ neutron ext-list
+-----------------------+-----------------------------------------------+
| alias                 | name                                          |
+-----------------------+-----------------------------------------------+
| security-group        | security-group                                |
| l3_agent_scheduler    | L3 Agent Scheduler                            |
| net-mtu               | Network MTU                                   |
| ext-gw-mode           | Neutron L3 Configurable external gateway mode |
| binding               | Port Binding                                  |
| provider              | Provider Network                              |
| agent                 | agent                                         |
| quotas                | Quota management support                      |
| subnet_allocation     | Subnet Allocation                             |
| dhcp_agent_scheduler  | DHCP Agent Scheduler                          |
| l3-ha                 | HA Router extension                           |
| multi-provider        | Multi Provider Network                        |
| external-net          | Neutron external network                      |
| router                | Neutron L3 Router                             |
| allowed-address-pairs | Allowed Address Pairs                         |
| extraroute            | Neutron Extra Route                           |
| extra_dhcp_opt        | Neutron Extra DHCP opts                       |
| dvr                   | Distributed Virtual Router                    |
+-----------------------+-----------------------------------------------+
```


networknode追加
```
$ sudo cp -p /etc/sysctl.conf $BAK
$ sudo vi /etc/sysctl.conf
---
...
#-- For OpenStack
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
---
$ sudo sysctl -p
```

neutronパッケージ
```
$ sudo apt-get -y install neutron-plugin-ml2 neutron-plugin-openvswitch-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```

neutron設定
```
$ sudo vi /etc/neutron/plugins/ml2/ml2_conf.ini 
---
[ml2_type_flat]
flat_networks = external

[ml2_type_gre]
tunnel_id_ranges = 1:1000
...
[securitygroup]
enable_security_group = True
enable_ipset = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

[ovs]
local_ip = 192.168.0.200
bridge_mappings = external:br-ex

[agent]
tunnel_types = gre
---

$ sudo vi /etc/neutron/l3_agent.ini
---
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
external_network_bridge = br-ex
router_delete_namespaces = True
...
---

$ sudo vi /etc/neutron/dhcp_agent.ini
---
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
dhcp_delete_namespaces = True
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
dhcp_domain = local
...
---

$ sudo vi /etc/neutron/dnsmasq-neutron.conf
---
dhcp-option-force=26,1454
---
※ Enable the DHCP MTU option (26) and configure it to 1454 bytes:
※ GREを使用すると最大フレームサイズ(1500)だと、カプセル化で付与したフレームが通らない.
　そのためMTU値を低く設定する.

$ sudo pkill dnsmasq
$ sudo vi /etc/neutron/metadata_agent.ini
---
[DEFAULT]
auth_uri = http://192.168.0.200:5000
auth_url = http://192.168.0.200:35357
auth_region = RegionOne
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = password
nova_metadata_ip = 192.168.0.200
metadata_proxy_shared_secret = password
...
---

$ sudo vi /etc/nova/nova.conf
---
[neutron]
...
service_metadata_proxy = True
metadata_proxy_shared_secret = password
---

$ sudo service nova-api restart
```

ovs設定
```
$ sudo service openvswitch-switch restart
$ sudo ovs-vsctl add-br br-ex
$ sudo ovs-vsctl add-port br-ex p1p1
$ sudo ethtool -K p1p1 gro off
```
###ここまで
neutron再起動
```
$ sudo service neutron-plugin-openvswitch-agent restart
$ sudo service neutron-l3-agent restart
$ sudo service neutron-dhcp-agent restart
$ sudo service neutron-metadata-agent restart
```

確認
```
$ neutron agent-list
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| id                                   | agent_type         | host      | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| 00326010-3775-47cb-ba35-d53d74d5135d | DHCP agent         | ryunosuke | :-)   | True           | neutron-dhcp-agent        |
| 28135e73-d56e-4e82-a1ce-02ad37e4b2d8 | Metadata agent     | ryunosuke | :-)   | True           | neutron-metadata-agent    |
| 61e5dfcd-cf6f-4f8e-880f-60cdb32aff9e | L3 agent           | ryunosuke | :-)   | True           | neutron-l3-agent          |
| b63336e0-d292-433e-b8da-1688efef7a07 | Open vSwitch agent | ryunosuke | :-)   | True           | neutron-openvswitch-agent |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
```

### 仮装ネットワーク作成
#### 外部ネットワーク
マニュアルにSource the admin credentials to gain access to admin-only CLI commands:.  
と記載があるので、admin権限で作成する.  
```
$ keystone tenant-list
+----------------------------------+---------+---------+
|                id                |   name  | enabled |
+----------------------------------+---------+---------+
| 61dc76652fa14c58a4a72e0e9fad93e5 |  admin  |   True  |
~~~これを使用する.
| 3397a7d0cf9047f1b7c3a0ae149f5a3d |   demo  |   True  |
| 302f366fd65440eab9406a8f20317a93 | service |   True  |
+----------------------------------+---------+---------+

$ neutron net-create ext-net --tenant-id 61dc76652fa14c58a4a72e0e9fad93e5 --router:external True --provider:physical_network external --provider:network_type flat

Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | b46155c1-19f7-4bc5-992f-d910947d89bc |
| name                      | ext-net                              |
| provider:network_type     | flat                                 |
| provider:physical_network | external                             |
| provider:segmentation_id  |                                      |
| router:external           | True                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 61dc76652fa14c58a4a72e0e9fad93e5     |
+---------------------------+--------------------------------------+

$ neutron subnet-create ext-net --name ext-subnet --tenant-id 61dc76652fa14c58a4a72e0e9fad93e5 --allocation-pool start=192.168.0.100,end=192.168.0.150 --disable-dhcp --gateway 192.168.0.254 192.168.0.0/24

Created a new subnet:
+-------------------+----------------------------------------------------+
| Field             | Value                                              |
+-------------------+----------------------------------------------------+
| allocation_pools  | {"start": "192.168.0.100", "end": "192.168.0.150"} |
| cidr              | 192.168.0.0/24                                     |
| dns_nameservers   |                                                    |
| enable_dhcp       | False                                              |
| gateway_ip        | 192.168.0.254                                      |
| host_routes       |                                                    |
| id                | be761def-2eb4-47c8-9a93-a82e48724000               |
| ip_version        | 4                                                  |
| ipv6_address_mode |                                                    |
| ipv6_ra_mode      |                                                    |
| name              | ext-subnet                                         |
| network_id        | b46155c1-19f7-4bc5-992f-d910947d89bc               |
| tenant_id         | 61dc76652fa14c58a4a72e0e9fad93e5                   |
+-------------------+----------------------------------------------------+
```

#### 内部ネットワーク(テナント用)
Source the demo credentials to gain access to user-only CLI commands:  
とあるので、demo向けに作成する.  
```
$ keystone tenant-list
+----------------------------------+---------+---------+
|                id                |   name  | enabled |
+----------------------------------+---------+---------+
| 61dc76652fa14c58a4a72e0e9fad93e5 |  admin  |   True  |
| 3397a7d0cf9047f1b7c3a0ae149f5a3d |   demo  |   True  |
~~~これを使用する.
| 302f366fd65440eab9406a8f20317a93 | service |   True  |
+----------------------------------+---------+---------+

$ neutron net-create --tenant-id 3397a7d0cf9047f1b7c3a0ae149f5a3d  demo-net

Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 20d0e6b7-1906-4134-a3c5-4c6adfc2e186 |
| name                      | demo-net                             |
| provider:network_type     | gre                                  |
| provider:physical_network |                                      |
| provider:segmentation_id  | 1                                    |
| router:external           | False                                |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 3397a7d0cf9047f1b7c3a0ae149f5a3d     |
+---------------------------+--------------------------------------+


$ neutron subnet-create demo-net --name demo-subnet --tenant-id 3397a7d0cf9047f1b7c3a0ae149f5a3d --gateway 10.0.0.1 10.0.0.0/24  --dns-nameservers list=true 192.168.0.254

Created a new subnet:
+-------------------+--------------------------------------------+
| Field             | Value                                      |
+-------------------+--------------------------------------------+
| allocation_pools  | {"start": "10.0.0.2", "end": "10.0.0.254"} |
| cidr              | 10.0.0.0/24                                |
| dns_nameservers   | 192.168.0.254                              |
| enable_dhcp       | True                                       |
| gateway_ip        | 10.0.0.1                                   |
| host_routes       |                                            |
| id                | e6e4c91a-ac1e-4f49-963c-67059d6ddb0f       |
| ip_version        | 4                                          |
| ipv6_address_mode |                                            |
| ipv6_ra_mode      |                                            |
| name              | demo-subnet                                |
| network_id        | 20d0e6b7-1906-4134-a3c5-4c6adfc2e186       |
| tenant_id         | 3397a7d0cf9047f1b7c3a0ae149f5a3d           |
+-------------------+--------------------------------------------+
```

#### 仮想ルータ
```
$ neutron router-create --tenant-id 3397a7d0cf9047f1b7c3a0ae149f5a3d demo-router

Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| distributed           | False                                |
| external_gateway_info |                                      |
| ha                    | False                                |
| id                    | 4c8dfa17-9399-4c8e-8875-7e175737ca49 |
| name                  | demo-router                          |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | 3397a7d0cf9047f1b7c3a0ae149f5a3d     |
+-----------------------+--------------------------------------+

$ neutron router-interface-add demo-router demo-subnet
$ neutron router-gateway-set demo-router ext-net

$ sudo cp /etc/network/interfaces $BAK
$ sudo vi /etc/network/interfaces
---
auto lo
iface lo inet loopback

# for OpenStack
auto p1p1
iface p1p1 inet manual
  up ip link set dev $IFACE up
  down ip link set dev $IFACE down

# The primary network interface
auto br-ex
iface br-ex inet static
  address 192.168.0.200
  netmask 255.255.255.0
  network 192.168.0.0
  gateway 192.168.0.254
  dns-nameservers 192.168.0.254
---
$ sudo reboot
→ 再起動後、sshログインできればOK.
```

### horizon
horizon本体
```
$ sudo apt-get -y install openstack-dashboard apache2 libapache2-mod-wsgi memcached python-memcache
```

horizon設定
```
$ sudo cp -raf /etc/openstack-dashboard $BAK
$ sudo vi /etc/openstack-dashboard/local_settings.py 
---
...
OPENSTACK_HOST = "192.168.0.200"
...
ALLOWED_HOSTS = ['*']
...
TIME_ZONE = "Asia/Tokyo"
---
```

反映
```
$ sudo service apache2 restart
$ sudo service memcached restart
```

ブラウザアクセス
[horizon](http://192.168.0.200/horizon/)
