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
* 一般モード / 特権モード
* オペレーションモード
* GlobalConfigMode
* 特定ConfigMode

と同じ考え方でモード切り替えが出来る.
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

### DHCP設定
[NEC IXのDHCP設定](https://changineer.info/network/nec_ix/nec_ix_dhcp_example.html) のDHCPサーバ設定を行う.
```
ogarouter1(config)# ip dhcp enable
ip dhcp profile web-dhcp-gigaethernet2.0
  assignable-range 192.168.3.10 192.168.3.99
  dns-server 192.168.3.254
```
