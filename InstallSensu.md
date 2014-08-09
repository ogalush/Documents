<!--
************************************************************
Sensuインストール
参照元: http://sensuapp.org/docs/latest/guide
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->

# Sensuインストール
## 環境
 Ubuntu14.04.1


## 準備
```
$ sudo apt-get -y update && sudo apt-get -y upgrade
```

## 証明書作成
```
$ openssl version
$ cd /usr/local/src
$ sudo wget http://sensuapp.org/docs/0.13/tools/ssl_certs.tar
$ sudo tar -xvf ssl_certs.tar
$ cd /usr/local/src/ssl_certs
$ sudo ./ssl_certs.sh generate
```

## rabbitMQ
```
```


## Sensuインストール
```
$ cd /usr/local/src
$ sudo wget -q http://repos.sensuapp.org/apt/pubkey.gpg -O- | sudo apt-key add -
$ echo "deb     http://repos.sensuapp.org/apt sensu main" | sudo tee /etc/apt/sources.list.d/sensu.list
$ echo "deb     http://repos.sensuapp.org/apt sensu unstable" | sudo tee /etc/apt/sources.list.d/sensu.list
$ sudo apt-get -y update
$ apt-get -y install sensu
```

## まめ知識
```
(1) デバッグ開始
$ set -x
(2) デバッグ終了
$ set +x
```
