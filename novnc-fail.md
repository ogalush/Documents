<!--
************************************************************
OpenStack(Grizzly)でvncクライアントが表示されない
# https://ask.openstack.org/en/question/4222/horizon-console-displays-blank-screen-with-message-novnc-ready-native-websockets-canvas-rendering/
Copyright (c) Takehiko OGASAWARA 2013 All Rights Reserved.
************************************************************
-->
<div id='title'>　</div>    

# horizonのクライアントコンソールが表示されない件の直し方

### 設定
バックアップディレクトリ作成
```
mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
cp -raf /etc/nova $BAK
cp -raf /usr/share/novnc $BAK
```

パッケージインストール  
```
apt-get install novnc nova-novncproxy
```

設定
```
vi /etc/nova/nova.conf
---
# novnc
novnc_enable = true
novncproxy_port = 6080
novncproxy_host = 192.168.0.200
novncproxy_base_url = http://192.168.0.200:6080/vnc_auto.html
vncserver_listen = 192.168.0.200
vncserver_proxyclient_address = 192.168.0.200
vnc_keymap=ja
---
```

バグ部分のコメントアウト
```
vi /usr/share/novnc/include/rfb.js
---
1784 that.connect = function(host, port, password, path) {
1785     //Util.Debug(">> connect");
1786 
//1787     nova_token     = token;
↑nova_tokenをコメントアウトする
1788     rfb_host       = host;
---
```

サービス再起動
```
/etc/init.d/nova-novncproxy restart
/etc/init.d/nova-novncproxy status
```

http://192.168.0.200/horizon/
→インスタンスのコンソール画面を表示できればOK


