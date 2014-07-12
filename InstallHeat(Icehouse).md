<!--
************************************************************
OpenStack IceHouse(Heat)をUbuntu14.04(x86_64)へインストールする手順
参照元: http://docs.openstack.org/icehouse/install-guide/install/apt/content/heat-install.html
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->

# Heatをインストールする方法(OpenStack IceHouse)
Heatは、Controller nodeへインストールする

### 準備
バックアップディレクトリ作成
```
mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

### Heatインストール
```
# apt-get -y install heat-api heat-api-cfn heat-engine
```

### 設定
```
# cp -raf /etc/heat $BAK
# vi /etc/heat/heat.conf
---
[DEFAULT]
verbose = True
log_dir=/var/log/heat
rabbit_host = 192.168.0.200
rabbit_password = admin!
heat_metadata_server_url = http://192.168.0.200:8000
heat_waitcondition_server_url = http://192.168.0.200:8000/v1/waitcondition

[database]
##connection=sqlite:////var/lib/heat/$sqlite_db
connection = mysql://heat:password@192.168.0.200/heat

[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
auth_uri = http://192.168.0.200:5000/v2.0
admin_tenant_name = service
admin_user = heat
admin_password = password
[ec2authtoken]
auth_uri = http://192.168.0.200:5000/v2.0
---
# rm /var/lib/heat/heat.sqlite
```

DB作成
```
# mysql -u root -p
mysql> CREATE DATABASE heat;
mysql> GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%' IDENTIFIED BY 'password';
mysql> FLUSH PRIVILEGES;
```

KeyStone作成
```
# keystone user-create --name=heat --pass=password --email=heat@example.com
# keystone user-role-add --user=heat --tenant=service --role=admin
# keystone service-create --name=heat --type=orchestration --description="Orchestration"
# keystone endpoint-create --service-id=$(keystone service-list | awk '/ orchestration / {print $2}') \
  --publicurl=http://192.168.0.200:8004/v1/%\(tenant_id\)s \
  --internalurl=http://192.168.0.200:8004/v1/%\(tenant_id\)s \
  --adminurl=http://192.168.0.200:8004/v1/%\(tenant_id\)s
# keystone service-create --name=heat-cfn --type=cloudformation --description="Orchestration CloudFormation"
# keystone endpoint-create --service-id=$(keystone service-list | awk '/ cloudformation / {print $2}') \
  --publicurl=http://192.168.0.200:8000/v1 --internalurl=http://192.168.0.200:8000/v1 \
  --adminurl=http://192.168.0.200:8000/v1
# keystone role-create --name heat_stack_user
```

Heat再起動
```
# ls -1 /etc/init.d/heat* | xargs -I{} bash {} restart
または、以下
# service heat-api restart
# service heat-api-cfn restart
# service heat-engine restart
```
