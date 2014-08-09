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
$ sudo apt-get -y install erlang-nox
$ cd /usr/local/src
$ sudo wget http://www.rabbitmq.com/rabbitmq-signing-key-public.asc
$ sudo apt-key add rabbitmq-signing-key-public.asc
$ sudo apt-get -y update
$ sudo apt-get -y install rabbitmq-server
```

## Redis
```
$ sudo apt-get -y install redis-server
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

## 設定
```
$ sudo mkdir -p /etc/sensu/ssl
$ sudo cp -p  /usr/local/src/ssl_certs/client/{cert.pem,key.pem} /etc/sensu/ssl
$ ls -al /etc/sensu/ssl
```

monitor設定
```
$ sudo vi /etc/sensu/conf.d/rabbitmq.json
---
{
  "rabbitmq": {
    "ssl": {
      "cert_chain_file": "/etc/sensu/ssl/cert.pem",
      "private_key_file": "/etc/sensu/ssl/key.pem"
    },
    "host": "sensu.localdomain",
    "port": 5672,
    "vhost": "/sensu",
    "user": "sensu",
    "password": "password"
  }
}
---

$ sudo vi /etc/sensu/conf.d/redis.json
---
{
  "redis": {
    "host": "localhost",
    "port": 6379
  }
}
---

$ sudo vi /etc/sensu/conf.d/api.json
---
{
  "api": {
    "host": "sensu.localdomain",
    "port": 4567,
    "user": "admin",
    "password": "password"
  }
}
---

$ sudo vi /etc/sensu/conf.d/client.json
---
{
  "client": {
    "name": "sensu",
    "address": "sensu.localdomain",
    "subscriptions": [ "all" ]
  }
}
---
```

サービス有効化
```
$ sudo update-rc.d sensu-server defaults
$ sudo update-rc.d sensu-client defaults
$ sudo update-rc.d sensu-api defaults
$ sudo /etc/init.d/sensu-server start
$ sudo /etc/init.d/sensu-client start
$ sudo /etc/init.d/sensu-api start
```

## まめ知識
```
(1) デバッグ開始
$ set -x
(2) デバッグ終了
$ set +x
```
