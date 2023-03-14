## 概要
自宅ネットワークのレスポンス改善のために, NEC UNIVERGE IX2215を中古で購入した.  
IPoE IPv6 + IPv4接続を行っているためセットアップを行う.

## 参考
* [NECルータIX2215の初期設定（その１）スーパーリセット](https://souiunogaii.hatenablog.com/entry/NEC-IX2215-SuperReset)
* [NECルータIX2215の初期設定（その２）telnet接続、WEBコンソール](https://souiunogaii.hatenablog.com/entry/NEC-IX2215-telnet)
* [NECルータIX2215の初期設定（その３）PPPoEの設定](https://souiunogaii.hatenablog.com/entry/NEC-IX2215-PPPoE)

## 環境
### 回線環境
```
DTI光 IPoE(v6Plus / MAP-E) -- ONU -- NEC UNIVERGE IX2215 -- MacBook / 無線LAN
```
### ルータ構成
Device/Interfaceは以下の構成である.
* Tunnel0.0
WAN IPoE ipv4アドレス
* GigaEthernet0.0
WAN IPv6 アドレス RA受け取り
* GigaEthernet2.0
LAN 192.168.3.0/24

## 設定手順
### シリアルコンソール接続
(1) 接続
```
ogalush@MacBook-Pro1 ~ % ls -l /dev/tty.usbserial-FTFNUBTA 
crw-rw-rw-  1 root  wheel  0x9000000  3 15 00:38 /dev/tty.usbserial-FTFNUBTA
ogalush@MacBook-Pro1 ~ % 
ogalush@MacBook-Pro1 ~ % screen /dev/tty.usbserial-FTFNUBTA

login:
→ ルータのプロンプトが表示されればOK.
```
