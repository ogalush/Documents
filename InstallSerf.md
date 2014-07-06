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

### serfインストール
```
cd /usr/local/src
sudo wget https://dl.bintray.com/mitchellh/serf/0.6.2_linux_amd64.zip
unzip /usr/local/src/0.6.2_linux_amd64.zip
cp -p serf /usr/local/bin/serf
sudo -u ogalush serf --help
~~~★ヘルプが表示されればOK
```

### agentの起動
```
serf agent -bind=0.0.0.0:7946 --rpc-addr=0.0.0.0:7373
~~~★agentが起動すればOK
※rpc-addrがデフォルトだとlocalhostとなる。外部と通信させたいのでrpc-addrを追加する。
```

### agentの組み込み
各々のagentをメンバーとする
```
root@serv1:~# serf members
serv1  10.0.0.47:7946  alive
root@serv1:~# serf join 10.0.0.47:7946
Successfully joined cluster by contacting 1 nodes.
root@serv1:~# serf join 10.0.0.48:7946
Successfully joined cluster by contacting 1 nodes.
root@serv1:~# serf join 10.0.0.49:7946
Successfully joined cluster by contacting 1 nodes.
root@serv1:~# serf members
serv3  10.0.0.49:7946  alive  
serv1  10.0.0.47:7946  alive  
serv2  10.0.0.48:7946  alive
~~~★対象ホストがmemberに入ればOK
root@serv1:~# 
```
