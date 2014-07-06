<!--
************************************************************
Consulインストール
参照元: http://www.consul.io/intro/getting-started/install.html
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->

# Consulインストール

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

### Consulインストール
```
cd /usr/local/src
sudo wget https://dl.bintray.com/mitchellh/consul/0.3.0_linux_amd64.zip
sudo unzip 0.3.0_linux_amd64.zip
sudo cp -p consul /usr/local/bin/
sudo -u ogalush consul --help
~~~★ヘルプが表示されればOK
```

