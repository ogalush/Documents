<!--
************************************************************
OpenStack LoadBalancer
# https://wiki.openstack.org/wiki/Neutron/LBaaS/HowToRun
Copyright (c) Takehiko OGASAWARA 2013 All Rights Reserved.
************************************************************
-->
<div id='title'>　</div>    

# OpenStackでロードバランサ機能を使用する方法

### 設定
バックアップディレクトリ作成
```
mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
cp -raf /etc/nova $BAK
cp -raf /usr/share/novnc $BAK
```

パッケージインストール  
```
# apt-get -y install quantum-lbaas-agent haproxy
```

設定
```
# quantum net-create lb-segment
# quantum subnet-create lb-segment 172.17.0.0/16
# quantum subnet-list
※ subnet-idを取得する
# quantum lb-pool-create \
  --lb-method ROUND_ROBIN \
  --name lbnet --protocol HTTP  --subnet-id 6cc32c74-4fc6-4595-8fa2-da8a346d8dac
※subnet-idは、対象サブネットを指定する。
```

horizon設定
```
# cp -p /etc/openstack-dashboard/local_settings.py $BAK
# vi /etc/openstack-dashboard/local_settings.py
---
OPENSTACK_QUANTUM_NETWORK = {
    'enable_lb': True
}
---

```
→ horizonでロードバランサ画面が出ていればOK  

