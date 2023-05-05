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
### ルータ情報
[ハードウェア仕様　UNIVERGE IX2000シリーズ](https://jpn.nec.com/univerge/ix/Spec/hw-spec.html?#ix2215)
### ルータ構成
Device/Interfaceは以下の構成である.
* Tunnel0.0
WAN IPoE ipv4アドレス
* GigaEthernet0.0
WAN IPv6 アドレス RA受け取り
* GigaEthernet2.0
LAN 192.168.3.0/24

## 設定手順1
[NECルータIX2215の初期設定（その１）スーパーリセット](https://souiunogaii.hatenablog.com/entry/NEC-IX2215-SuperReset)  
の必要な箇所を流していく.
### シリアルコンソール接続
```
ogalush@MacBook-Pro1 ~ % ls -l /dev/tty.usbserial-FTFNUBTA 
crw-rw-rw-  1 root  wheel  0x9000000  3 15 00:38 /dev/tty.usbserial-FTFNUBTA
ogalush@MacBook-Pro1 ~ % 
ogalush@MacBook-Pro1 ~ % screen /dev/tty.usbserial-FTFNUBTA

login:
→ ルータのプロンプトが表示されればOK.
```
### SuperReset
[IX2215のスーパーリセット](https://souiunogaii.hatenablog.com/entry/NEC-IX2215-SuperReset#%E6%89%8B%E9%A0%86%E3%81%9D%E3%81%AEIX2215%E3%81%AE%E3%82%B9%E3%83%BC%E3%83%91%E3%83%BC%E3%83%AA%E3%82%BB%E3%83%83%E3%83%88)  
ルータ設定を消去し、初期状態にする.
```
Router# config
Enter configuration commands, one per line. End with CNTL/Z.
router(config)# reload
Notice: The router will be RELOADED. This is to ensure that
        the peripheral devices are properly initialized.
Are you sure you want to reload the router? (Yes or [No]): yes
→ Routerの再起動が始まる.

NEC Bootstrap Software
Copyright (c) NEC Corporation 2001-2022. All rights reserved.

%BOOT-INFO: Trying flash load, exec-image [ix2215-ms-10.7.17.ldc].
Loading: ####################
→「Ctrl + C」する.

NEC Bootstrap Software, Version 22.1
Copyright (c) NEC Corporation 2001-2022. All rights reserved.
boot[1]> cc
Enter "Y" to clear startup configuration: y
% Startup configuration is cleared.
→ 「cc」+「y」 startup-configを消去することができる.

boot[1]> b
→ 「b」でbootさせる.

NEC Portable Internetwork Core Operating System Software
Copyright Notices:
Copyright (c) NEC Corporation 2001-2022. All rights reserved.
Copyright (c) 1985-1998 OpenROUTE Networks, Inc.
Copyright (c) 1984-1987, 1989 J. Noel Chiappa.
Router# 
→ 「Router#」のプロンプトが表示されればOK.
```

### オペレーションモードからグローバルコンフィグモードに移行
Router/Switch全般によくある話.  
|項目|役割|接続方法|プロンプト表示|
|---|---|---|---|
|オペレーションモード|設定を閲覧するモード|Console or telnet(有効化後)| `Router#` |
|Global ConfigMode|Configモード.<br>ルータ全般の設定.| `Router# enable-config` | `Router(config)#` |
|Device ConfigMode|各ポートの物理的な設定(Speed, Duplex)| `Router(config)# device GigaEthernet2` | `Router(config-GigaEthernet2)#` |
|Interface ConfigMode|各ポートの論理的な設定(IP Address, VLAN etc...)| `Router(config)# interface GigaEthernet2.0` | `Router(config-GigaEthernet2.0)#` |

```
Router# config
Enter configuration commands, one per line. End with CNTL/Z.
Router(config)# 
```
抜ける時はexitで終了出来る.


### バージョンの確認
Flets'光Next IPoE IPv6は10.0以降のイメージが必要なため、クリアできている状態.  
* [フレッツ 光ネクスト向け 設定ガイド](https://jpn.nec.com/univerge/ix/Support/ipv6/native/ipv6-internet_ra.html)
```
Router(config)# show version
NEC Portable Internetwork Core Operating System Software
IX Series IX2215 (magellan-sec) Software, Version 10.7.17, RELEASE SOFTWARE ・・・「10.7.17」のためOK.
Compiled Sep 22-Thu-2022 15:22:13 JST #2 by sw-build, coregen-10.7(17)

ROM: System Bootstrap, Version 22.1
System Diagnostic, Version 22.1
Initialization Program, Version 2.1

System uptime is 6 minutes
System woke up by reload, caused by command execution
System started at Mar 15-Wed-2023 01:42:10 JST
System image file is "ix2215-ms-10.7.17.ldc"

Processor board ID <0>
IX2215 (P1010E) processor with 262144K bytes of memory.
3 GigaEthernet/IEEE 802.3 interfaces
1 ISDN Basic Rate interface
1 USB interface
1024K bytes of non-volatile configuration memory.
32768K bytes of processor board System flash (Read/Write)
Router(config)# 
```

### ホスト名の設定
任意のRouter名を設定する.
```
Router(config)# hostname ogarouter1
ogarouter1(config)# 
```

### ログイン名とパスワードの設定
Terminal/WebUIで使用する管理パスワードを設定する.
```
ogarouter1(config)# username admin password plain 1 ***(naisho)*** administrator
% User 'admin' has been added.
```

### NTPの設定と時刻の確認
信頼できるNTPサーバのIPアドレスを指定する.  
[ＮＴＰサーバーの構築（タイムサーバー）](http://miyakoshi.mydns.jp/freebsd/ntp.html)

|項目|FQDN|IP Address|
|---|---|---|
|東京大学|ntp.nc.u-tokyo.ac.jp|130.69.251.23|
```
ogarouter1(config)# ntp interval 3600
ogarouter1(config)# ntp retry 10
ogarouter1(config)# ntp server 130.69.251.23
```
#### NTP確認
```
ogarouter1(config)# show ntp
NTP status:
  Clock is not synchronized, reference is nothing
    Rcvd: 0 requests, 0 responses
    Sent: 0 requests, 0 responses
  NTP server      VRF name                            St  Ver   Timeout   Last Receive
  130.69.251.23                                        0    0        64       0:00:00
ogarouter1(config)# 
```
WAN未設定のため、表示されない.  
WAN設定後に改めて確認する.
#### Clock確認
```
ogarouter1(config)# show clock
Wednesday, 15 March 2023 02:01:17 +09 00
```

### LANインターフェースの設定
GigaEthernet2 8Ports  
考え方として、Device, Interfaceの単位での設定になる.  
#### 事前確認
```
ogarouter1(config)# show devices
Device GigaEthernet0 is down
...
Device GigaEthernet1 is down
...
Device GigaEthernet2 is down ・・・ これ.
```
#### Device設定
```
ogarouter1(config)# device GigaEthernet2 
ogarouter1(config-GigaEthernet2)# speed auto
ogarouter1(config-GigaEthernet2)# duplex auto
ogarouter1(config-GigaEthernet2)# no shutdown
```
#### Interface設定
```
ogarouter1(config-GigaEthernet2)# interface GigaEthernet2.0
ogarouter1(config-GigaEthernet2.0)# ip address 192.168.3.254/24
ogarouter1(config-GigaEthernet2.0)# no shutdown
ogarouter1(config-GigaEthernet2.0)# exit 
```

#### logging設定
[実機演習資料(初級編)実機演習資料(初級編) UNIVERGE IX2215 ](https://www.express.nec.co.jp/idaten/network/ix/ix2k3k-learning-ver8.10_10.0.pdf)  
→ 「ログの収集(2)」  
に触れられている基本的なログ設定を行っておく.
```
ogarouter1(config)# logging subsystem all warn
→ 全ての機能で警告レベルのみにする.
ogarouter1(config)# logging timestamp datetime
→ 人が見やすいようにdatetime形式にする
ogarouter1(config)# logging buffered
→ ログを内部バッファに保存する
```
ログ参照時は `show logging` を利用する.
```
ogarouter1(config)# show logging
Buffer logging enabled, 131072 bytes, type cyclic
  3 messages (1-3), 209 bytes logged, 0 messages dropped

Log Buffer (1-3):
2023/05/06 03:44:37 CNFG.003: Startup config write done  (TTY:1).
ogarouter1(config)#
```
メモリ上に周期的に記録する(ringbuffer的な)動きの模様.  

logging出来るメモリ量は `show memory` で余っている量を割り当てられるとのこと.  
今回はとりあえずログ記録しておく位の優先度のためデフォルトを利用する.  
[UNIVERGE IXシリーズ　FAQ - ロギング、SYSLOGに関するFAQ](https://jpn.nec.com/univerge/ix/faq/els.html)  
```
Q.1-5 メモリに蓄積できるログの量は設定できますか？
Q.1-4のメモリ蓄積コマンドの[バッファサイズ]（単位：バイト）で、蓄積するログの量を設定することができます。
ロギングメッセージ1行あたり、80バイト程度となりますので、show memoryコマンドで残りのメモリ量を見て、
何行程度ログを保存するかを確認の上、バッファサイズを決定してください。
```

#### 設定確認
```
ogarouter1(config)# show running-config
→ 今の設定内容
ogarouter1(config)# check config
→ 文法などエラーがあれば表示される.
```
#### 設定保存
一般的な手順
```
ogarouter1(config)# copy running-config startup-config
Copying from "running-config" to "startup-config"
Building configuration...
% Warning: do NOT enter CNTL/Z while saving to avoid config corruption.
ogarouter1(config)#
```

「write memory」でも保存可能.
```
ogarouter1(config)# write memory
Building configuration...
% Warning: do NOT enter CNTL/Z while saving to avoid config corruption.
```

## 設定手順2
[NECルータIX2215の初期設定（その２）telnet接続、WEBコンソール](https://souiunogaii.hatenablog.com/entry/NEC-IX2215-telnet)  
の必要な箇所を流していく.

### DNS設定
[DNS Proxy](https://changineer.info/network/nec_ix/nec_ix_dns.html) を設定する.  
DNSキャッシュの設定もあるが、細かい内容は一旦割愛する.
```
ogarouter1(config)# proxy-dns ip enable
ogarouter1(config)# proxy-dns ip request both
```

### IPアドレス設定
MacBookへStatic IPアドレス設定をする.  
|title|value|
|---|---|
|IP Address|192.168.3.xx|
|Subnet Mask|255.255.255.0|
|Default Gateway|192.168.3.254|
```
ogalush@MacBook-Pro1 ~ % ifconfig en9 |grep 192
	inet 192.168.3.10 netmask 0xffffff00 broadcast 192.168.3.255
```

### ping疎通テスト
```
ogalush@MacBook-Pro1 ~ % ping -c 3 192.168.3.254
PING 192.168.3.254 (192.168.3.254): 56 data bytes
64 bytes from 192.168.3.254: icmp_seq=0 ttl=64 time=0.534 ms
64 bytes from 192.168.3.254: icmp_seq=1 ttl=64 time=0.618 ms
64 bytes from 192.168.3.254: icmp_seq=2 ttl=64 time=0.556 ms

--- 192.168.3.254 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.534/0.569/0.618/0.036 ms
ogalush@MacBook-Pro1 ~ %

→ Echo Replyが返ってこえばOK.
```

### Telnetサーバの有効化
```
ogarouter1(config)# telnet-server ip enable
ogarouter1(config)# 
```
### WEBコンソール画面の有効化
```
ogarouter1(config)# http-server ip enable
ogarouter1(config)# http-server username admin
```

### 設定保存
```
ogarouter1(config)# copy running-config startup-config
Copying from "running-config" to "startup-config"
Building configuration...
% Warning: do NOT enter CNTL/Z while saving to avoid config corruption.
ogarouter1(config)# exit
ogarouter1# exit
```

### アクセス確認
#### Telnet
```
ogalush@MacBook-Pro1 ~ % telnet 192.168.3.254
Trying 192.168.3.254...
Connected to 192.168.3.254.
Escape character is '^]'.
login: xxxx
Password: xxxxx
NEC Portable Internetwork Core Operating System Software
Copyright Notices:
Copyright (c) NEC Corporation 2001-2022. All rights reserved.
Copyright (c) 1985-1998 OpenROUTE Networks, Inc.
Copyright (c) 1984-1987, 1989 J. Noel Chiappa.
ogarouter1#
→ Promptが表示されればOK.
```
#### Web Console
https://192.168.3.254/  
→ WebUIの表示とアクセスが出来ればOK.


## 設定3
DHCP, IPoE設定を行う.
### IPoE設定
#### WebUIログイン
https://192.168.3.254/
```
「かんたん設定」  
↓
「接続種別の選択」
IPv6 IPoE接続
↓
パスワードの設定変更せず
次へ
↓
プロバイダの設定: JPNE(v6プラス）
グローバルIPアドレスの割り当て方式: 動的
↓
インターネット接続の設定
インターネット接続: IPv4接続＋IPv6接続
セキュリティ強度 レベル1
----
レベル1
- 外部からの不要なパケットを
NAPT(IPv4)、ダイナミックフィルタ(IPv6)により廃棄します。
----
LAN1: LANの設定(GigaEthernet2.0)
LAN側IPアドレス 192.168.3.254/24
----
↓
設定の確認と反映
内容が良ければ「反映」ボタン.
↓
「設定の保存」をする.
```

#### DHCP設定
「詳細設定」→「DHCPサーバの設定」  
https://192.168.3.254/admin/detail/dhcp_server.html  
|項目|値|
|---|---|
|DHCPサーバ|有効|
|割り当て範囲|固定設定|
|IPアドレス|192.168.3.10 - 192.168.3.99|

→ 保存、設定の反映を進める.

##設定3
[NECルータIX2215の初期設定（その３）PPPoEの設定](https://souiunogaii.hatenablog.com/entry/NEC-IX2215-PPPoE) から必要な設定を行う.
#### NAPT設定
そのままだとNAT設定が入らないため、IPv4 NATさせる.
```
ogarouter1(config)# interface Tunnel0.0
ogarouter1(config-Tunnel0.0)# ip napt enable
ogarouter1(config-Tunnel0.0)# ip tcp adjust-mss auto
ogarouter1(config-Tunnel0.0)# no shutdown
ogarouter1(config-Tunnel0.0)# exit
ogarouter1(config)# copy run start
Copying from "running-config" to "startup-config"
Building configuration...
% Warning: do NOT enter CNTL/Z while saving to avoid config corruption.
ogarouter1(config)#
```

#### ルーティング設定
ルータの検索キャッシュの有効化と、デフォルトルートを設定する.
```
ogarouter1(config)# ip ufs-cache enable
ogarouter1(config)# ip route default Tunnel0.0
ogarouter1(config)# copy running-config startup-config
Copying from "running-config" to "startup-config"
Building configuration...
% Warning: do NOT enter CNTL/Z while saving to avoid config corruption.
```

#### ping確認
Ping確認を行い、L3まで接続出来る事を確認する.
##### ping4
```
ogalush@MacBook-Pro1 ~ % ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=120 time=4.210 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=120 time=3.658 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=120 time=3.712 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 3.658/3.860/4.210/0.248 ms
ogalush@MacBook-Pro1 ~ % 
```
##### ping6
```
ogalush@MacBook-Pro1 ~ % ping6 -c 3 www.google.com
PING6(56=40+8+8 bytes) 2001:db8:xxx:1 --> 2404:6800:4004:820::2004
16 bytes from 2404:6800:4004:820::2004, icmp_seq=0 hlim=116 time=14.323 ms
16 bytes from 2404:6800:4004:820::2004, icmp_seq=1 hlim=116 time=4.126 ms
16 bytes from 2404:6800:4004:820::2004, icmp_seq=2 hlim=116 time=3.724 ms

--- www.google.com ping6 statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/std-dev = 3.724/7.391/14.323/4.904 ms
ogalush@MacBook-Pro1 ~ % 
```
