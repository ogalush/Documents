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


■rabbitMQの初期化
rabbitMQを初期化（設定＋データ）するには、以下を実行する

rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl start_app

※guestユーザのパスワードも初期化されるので、実行時は注意。
http://stackoverflow.com/questions/6742938/deleting-queues-in-rabbitmq

初期化後は、以下でアクセス可能。
http://192.168.0.200:55672/
ID: guest
PW: guest

OpenStackで利用できるように、guestのパスワードを変更する。
rabbitmqctl change_password guest password
→http://192.168.0.200:55672/
 ここへ新パスワードでログインできること。
 
nova-manage service list
nova list
→いずれの項目でもHTTP(500)エラーが返ってこないこと。
（項目が表示されるor空白表示であればOK）

以上
