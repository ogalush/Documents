<!--
************************************************************
OpenStack (Compute node) をUbuntu12(x86_64)へインストールする手順
参照元: http://docs.openstack.org/havana/install-guide/install/apt/content/nova-compute.html
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->
<div id='title'>　</div>    

# Compute nodeを追加する方法

### 準備
バックアップディレクトリ作成
```
# mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
# BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

準備
```
# ntp
# apt-get -y install ntp

# mysql
# apt-get -y install python-mysqldb mysql-client
```

