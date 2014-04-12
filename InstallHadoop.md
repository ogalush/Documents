<!--
************************************************************
Hadoopインストール手順
参照元: http://www.kkaneko.com/rinkou/cloudcomputing/hadoopinstall.html
        http://codesfusion.blogspot.jp/2013/10/setup-hadoop-2x-220-on-ubuntu.html?m=1
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->

# ノードをインストールする準備

## 準備
バックアップディレクトリ作成
```
$ sudo mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
$ BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

## Java DOWNLOAD
```
 ※ブラウザでダウンロードする
 http://www.oracle.com/technetwork/java/javase/downloads/server-jre7-downloads-1931105.html
 → SCPにて転送を行う。
```

## Java 展開
```
$ cd /usr/local/src
$ sudo tar xvzf server-jre-7u51-linux-x64.tar.gz 
$ sudo chown -R root:root jdk1.7.0_51
$ sudo mkdir /usr/lib/jvm
$ sudo mv jdk1.7.0_51 /usr/lib/jvm/.
$ ln -s /usr/lib/jvm/jdk1.7.0_51 /usr/lib/jvm/jdk
$ ls -al /usr/lib/jvm/jdk/bin/java
$ ogalush@name-node:/usr/local/src$ /usr/lib/jvm/jdk/bin/java -version
java version "1.7.0_51"
Java(TM) SE Runtime Environment (build 1.7.0_51-b13)
Java HotSpot(TM) 64-Bit Server VM (build 24.51-b03, mixed mode)
ogalush@name-node:/usr/local/src$ 
```

## Hadoopユーザを入れる
