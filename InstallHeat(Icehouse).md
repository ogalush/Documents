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
---
##connection=sqlite:////var/lib/heat/$sqlite_db
connection = mysql://heat:password@192.168.0.200/heat
---
```
