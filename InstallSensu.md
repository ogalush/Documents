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
$ sudo rabbitmqctl add_vhost /sensu
$ sudo rabbitmqctl add_user sensu password
$ sudo rabbitmqctl set_permissions -p /sensu sensu ".*" ".*" ".*"
$ sudo rabbitmqctl list_permissions -p /sensu
$ sudo rabbitmq-plugins enable rabbitmq_management
$ sudo service rabbitmq-server restart
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
$ sudo apt-get -y update
$ sudo apt-get -y install sensu=0.12.6-5
~~~ Ubuntu14.04.1ではsensu0.13のインストールが失敗するので、1つ前のバージョンをインストールする
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
$ sudo update-rc.d sensu-dashboard defaults
$ sudo /etc/init.d/sensu-server start
$ sudo /etc/init.d/sensu-client start
$ sudo /etc/init.d/sensu-api start
$ sudo /etc/init.d/sensu-service dashboard start
```

簡単な監視ジョブ
```
$ sudo apt-get -y install ruby1.9.1-dev make
$ sudo gem install sensu-plugin --no-rdoc --no-ri
$ cd /usr/local/src
$ sudo wget -O /etc/sensu/plugins/check-procs.rb https://raw.github.com/sensu/sensu-community-plugins/master/plugins/processes/check-procs.rb
$ sudo chmod 755 /etc/sensu/plugins/check-procs.rb

$ sudo vi /etc/sensu/conf.d/check_cron.json
---
{
  "checks": {
    "cron_check": {
      "handlers": ["default"],
      "command": "/etc/sensu/plugins/check-procs.rb -p crond -C 1 ",
      "interval": 60,
      "subscribers": [ "webservers" ]
    }
  }
}
---

反映
$ sudo /etc/init.d/sensu-server restart
$ sudo /etc/init.d/sensu-client restart
```

## dashboard
```
$ sudo apt-get -y install uchiwa
$ sudo mkdir -p /root/MAINTENANCE/20140817/bak
$ sudo cp -p uchiwa.js*  /root/MAINTENANCE/20140817/bak
$ sudo vi /etc/sensu/uchiwa.json
---
{
    "sensu": [
        {
            "name": "Sensu",
            "host": "127.0.0.1",
            "ssl": false,
            "port": 4567,
            "user": "admin",
            "pass": "password",
            "path": "",
            "timeout": 5000
        }
    ],
    "uchiwa": {
        "user": "",
        "pass": "",
        "port": 3000,
        "stats": 10,
        "refresh": 10000
    }
}
---
$ sudo update-rc.d uchiwa defaults
$ sudo /etc/init.d/uchiwa start

■ sensu dashboard
http://192.168.0.109:8080/

■ uchiwa
http://192.168.0.109:3000/

```

## sensu-client
監視対象サーバ  
```
$ echo 'deb     http://repos.sensuapp.org/apt sensu main' | sudo tee /etc/apt/sources.list.d/sensu.list
$ sudo apt-get -y update
$ sudo apt-get -y install sensu
$ sudo update-rc.d sensu-client defaults
$ sudo /etc/init.d/sensu-client start
$ sudo /etc/init.d/sensu-client status
$ sudo vi /etc/sensu/conf.d/client.json
---
{
  "client": {
    "name": "ryunosuke",
    "address": "192.168.0.200",
    "subscriptions": [ "all" ]
  }
}
---
```

## まめ知識
```
(1) デバッグ開始
$ set -x
(2) デバッグ終了
$ set +x
```
