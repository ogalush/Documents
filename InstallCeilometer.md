<!--
************************************************************
OpenStack IceHouse(CeiloMeter)をUbuntu14.04(x86_64)へインストールする手順
参照元: http://docs.openstack.org/icehouse/install-guide/install/apt/content/metering-service.html
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->

# CeiloMeterをインストールする方法(OpenStack IceHouse)

### 準備
バックアップディレクトリ作成
```
# mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
# BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

### CeiloMeterインストール
#### インストール
```
Telemetry Service
# apt-get -y install ceilometer-api ceilometer-collector ceilometer-agent-central ceilometer-agent-notification ceilometer-alarm-evaluator ceilometer-alarm-notifier python-ceilometerclient

Database Store
#  apt-get -y install mongodb-server

DBサイズがデフォルトで1GBのため、節約する
# service mongodb stop
# rm /var/lib/mongodb/journal/prealloc.*

bindするInterfaceを変更する
# cp -p /etc/mongodb.conf $BAK
# sed -i 's/bind_ip = 127.0.0.1/bind_ip = 192.168.0.200/' /etc/mongodb.conf
# diff -u $BAK/mongodb.conf /etc/mongodb.conf
---
-bind_ip = 127.0.0.1
+bind_ip = 192.168.0.200
---
# service mongodb start

```

### 設定
```
DB作成
# mongo --host 192.168.0.200 --eval '
db = db.getSiblingDB("ceilometer");
db.addUser({user: "ceilometer",
            pwd: "password",
            roles: [ "readWrite", "dbAdmin" ]})'

----
MongoDB shell version: 2.4.9
connecting to: 192.168.0.200:27017/test
{
        "user" : "ceilometer",
        "pwd" : "c3f1480e5fb171fed77480ec8a9c1a7f",
        "roles" : [
                "readWrite",
                "dbAdmin"
        ],
        "_id" : ObjectId("53dc8c0af94ef3b5ab1e801f")
}
----

# cp -raf /etc/ceilometer $BAK
# vi /etc/ceilometer/ceilometer.conf
---
[DEFAULT]
connection = mongodb://ceilometer:password@192.168.0.200:27017/ceilometer
...
rabbit_host=192.168.0.200
rabbig_password=admin!
log_dir = /var/log/ceilometer
auth_strategy = keystone
...
[publisher]
metering_secret = password
...
[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http
auth_uri = http://192.168.0.200:5000
admin_tenant_name = service
admin_user = ceilometer
admin_password = password 
...
[service_credentials]
os_auth_url = http://192.168.0.200:5000/v2.0
os_username = ceilometer
os_tenant_name = service
os_password = password
---

keystone登録
$ keystone service-create --name=ceilometer --type=metering --description="Telemetry"
$ keystone endpoint-create  --service-id=$(keystone service-list | awk '/ metering / {print $2}')  --publicurl=http://192.168.0.200:8777 --internalurl=http://192.168.0.200:8777 --adminurl=http://192.168.0.200:8777

再起動
# service ceilometer-agent-central restart
# service ceilometer-agent-notification restart
# service ceilometer-api restart
# service ceilometer-collector restart
# service ceilometer-alarm-evaluator restart
# service ceilometer-alarm-notifier restart
```

### agent(compute node)
#### インストール
```
# apt-get -y install ceilometer-agent-compute
```

#### 設定
```
# cp -raf /etc/nova $BAK
# vi /etc/nova/nova.conf
---
instance_usage_audit = True
instance_usage_audit_period = hour
notify_on_state_change = vm_and_task_state
notification_driver = nova.openstack.common.notifier.rpc_notifier
notification_driver = ceilometer.compute.nova_notifier
---

# service nova-compute restart

# vi /etc/ceilometer/ceilometer.conf
---
metering_secret = password
---

# vi /etc/ceilometer/ceilometer.conf
---
rabbit_host=192.168.0.200
rabbig_password=admin!
...
[keystone_authtoken]
auth_host = 192.168.0.200
auth_port = 35357
auth_protocol = http 
auth_uri = http://192.168.0.200:5000
admin_tenant_name = service
admin_user = ceilometer
admin_password = password
...

[service_credentials]
os_auth_url = http://192.168.0.200:5000/v2.0
os_username = ceilometer
os_tenant_name = service
os_password = password

log_dir = /var/log/ceilometer
---

# service ceilometer-agent-compute restart
```

