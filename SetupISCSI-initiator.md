# iSCSI initatorを設定する.
## 構成
### 経路
10.0.0.0/24 (LinuxBridge) → Neutron L3 agent(Router) → FloatingIP(192.168.3.134) → qnapnas(192.168.3.248)
### iSCSI Target (Server)
qnapnas(QNAP TS-231p)  
192.168.3.248
### iSCSI initiator (Client)
OpenStack nova  
10.0.0.0/24
```
[ogalush@hayao-test ~]$ cat /etc/redhat-release
CentOS Stream release 8
[ogalush@hayao-test ~]$ ip addr show eth0 |grep 10\.0\.0
    inet 10.0.0.52/24 brd 10.0.0.255 scope global dynamic noprefixroute eth0
[ogalush@hayao-test ~]$
```
### iSCSI Target準備
#### 保存領域を準備する
ストレージ＆スナップショットで、ボリュームサイズ変更をして保存領域を空ける.
#### iSCSI Target作成
QNAP管理画面 → iSCSI&ファイバーチャネル → 新しいiSCSIターゲットの作成にて作成する.  
iSCSIターゲットリストに接続ポイントが表示される.
|項目|値|
|---|---|
|alias|qnapnas|
|iqn|iqn.2004-04.com.qnap:ts231p:iscsi.qnapnas.22dcb3|
