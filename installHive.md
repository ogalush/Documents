<!--
************************************************************
Hiveインストール手順
概要:
 Hiveとは、hadoop上で、SQLライクな構文を使って、MapReduce処理を行ってくれるフレームワーク
参照元: http://kakakikikeke.blogspot.jp/2012/08/hadoophive.html
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->

# Hive インストール手順

## 構成
```
namenode:  name-node(10.5.5.11)
```

## 準備
バックアップディレクトリ作成
```
$ sudo mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
$ BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

