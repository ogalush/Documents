<!--
************************************************************
Hiveインストール手順
概要:
 Hiveとは、hadoop上で、SQLライクな構文を使って、MapReduce処理を行ってくれるフレームワーク
参照元: http://kakakikikeke.blogspot.jp/2012/08/hadoophive.html
        http://www.ayutaya.com/dev/hadoop/hive
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

## ダウンロード
```
$ cd /usr/local/src
$ sudo wget http://ftp.tsukuba.wide.ad.jp/software/apache/hive/stable/apache-hive-0.13.0-bin.tar.gz
$ sudo tar xvzf apache-hive-0.13.0-bin.tar.gz
$ sudo chown -R hduser:hduser apache-hive-0.13.0-bin
$ sudo mv apache-hive-0.13.0-bin /usr/local/
$ sudo ln -s /usr/local/apache-hive-0.13.0-bin /usr/local/hive
```
