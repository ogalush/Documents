<!--
************************************************************
Serfインストール
参照元: http://www.serfdom.io/intro/getting-started/install.html
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->

# Serfインストール

### 準備
バックアップディレクトリ作成
```
mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

必要パッケージの準備
```
apt-get -y install unzip
```

パッケージインストール  
```
cd /usr/local/src
sudo wget https://dl.bintray.com/mitchellh/serf/0.6.2_linux_amd64.zip
unzip /usr/local/src/0.6.2_linux_amd64.zip
```
