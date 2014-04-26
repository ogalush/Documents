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

## ダウンロード, 展開
```
$ cd /usr/local/src
$ sudo wget http://ftp.tsukuba.wide.ad.jp/software/apache/hive/stable/apache-hive-0.13.0-bin.tar.gz
$ sudo tar xvzf apache-hive-0.13.0-bin.tar.gz
$ sudo chown -R hduser:hduser apache-hive-0.13.0-bin
$ sudo mv apache-hive-0.13.0-bin /usr/local/
$ sudo ln -s /usr/local/apache-hive-0.13.0-bin /usr/local/hive
```

## 環境変数設定
```
$ sudo vi /home/hduser/.bashrc
---
#-- for hive
HIVE_HOME=/usr/local/hive
PATH=$HIVE_HOME/bin:$PATH
---

$ sudo vi /root/.bashrc
---
#-- for hive
HIVE_HOME=/usr/local/hive
PATH=$HIVE_HOME/bin:$PATH
---

source /home/hduser/.bashrc
```

## ログディレクトリ作成
```
$ sudo mkdir /var/log/hive
$ sudo chmod 777 /var/log/hive
$ touch /var/log/hive/hive.log
$ chown hduser:hduser /var/log/hive/hive.log
$ ls -al /var/log/hive
```

## workディレクトリの権限変更
```
$ chmod 777 /usr/local/hive/bin/metastore_db
```

## 設定
```
$ cd /usr/local/hive/conf/
$ ls -1 | sed 's/.template//g'| awk '{print "cp -p "$1".template " $1}' |bash
 → 末尾の.templateを除いてconfを作成
$ vi hive-log4j.properties
---
##hive.log.dir=${java.io.tmpdir}/${user.name}
hive.log.dir=/var/log/hive
---
```

## 実行
```
$ /usr/local/hive/bin/hive
 → hive> というコンソールが表示されればOK
$ hive> show databases;
 → 一覧が何かしら表示されればOK
```

## memo
hiveでエラーになったときは、lockファイルを削除する。
```
# find /usr/local/hive/bin/metastore_db -name "*.lck"
# find /usr/local/hive/bin/metastore_db -name "*.lck" -exec rm -f {} \;
```
