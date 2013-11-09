rabbitMQ管理画面を表示する方法


$ sudo apt-get rabbitmq-server

次に、RabbitMQ Managementをインストールします。

$ cd /usr/lib/rabbitmq/lib/rabbitmq_server-2.7.1/sbin
$ sudo ./rabbitmq-plugins enable rabbitmq_management

rabbitMQ管理画面を開く
http://192.168.0.200:55672

ID: guest
PW: password

Queueの状態を参照することができればOK。
