# はじめに
Openwrtを使ってIPv6 IPoE (MAP-E)を接続させるために初期設定を行ったので記録を残す.

# 環境
## 回線種別
|項目|値|
|---|---|
|回線|NTT東日本 フレッツ 光ネクスト|
|VNE|JPNE v6plus(MAP-E)|

## 接続構成
```
[Internet]
↓
[ONU]
↓
[OpenWrt Router]
 - WAN(106.72.0.0/15 + 240b:10:xx::/64)
 - LAN (192.168.3.0/24)
↓
PC (MacOS)
```

参考資料:  
* [徹底解説v6プラス](https://www.jpne.co.jp/ebooks/v6plus-ebook.pdf)

# 手順
## OpenWrtダウンロード
### ISOイメージのDownload
[Index of (root)/releases/22.03.5/targets/x86/64/](https://downloads.openwrt.org/releases/22.03.5/targets/x86/64/)  
→「generic-ext4-combined-efi.img.gz」をダウンロードする.  
  
* x86_64 = 通常のPC向けISOイメージ  
* 参考: [OpenWrt on x86 hardware (PC / VM / server)](https://openwrt.org/docs/guide-user/installation/openwrt_x86)
  
### gzファイルの展開 
```
@Mac
% gunzip openwrt-22.03.5-x86-64-generic-ext4-combined-efi.img.gz 
gunzip: openwrt-22.03.5-x86-64-generic-ext4-combined-efi.img.gz: trailing garbage ignored
```

## ISOイメージの準備
USBメモリへ書き込んで、PCからUSBBootさせるためISOイメージを準備する.

### イメージの書き込み.
```
@Mac
% sudo dd if=/Users/${USER:?}/Downloads/openwrt-22.03.5-x86-64-generic-ext4-combined-efi.img of=/dev/disk4
% diskutil eject /dev/disk4
Disk /dev/disk4 ejected
```

## USBメモリからBootさせる.
* PCのBIOS設定でUSBメモリをBootさせるように順序変更する（1番目)
* PCの電源を入れる.
* OpenWrtが起動するのが確認できればOK.

## OpenWrt - PCの接続　
* LANケーブルを繋ぐ
OpenWrtへLANケーブルを接続する.
* 設定画面が表示されればOK.  
http://192.168.1.1/

## 基本設定
### ホスト名設定
 `[System] -> [System]` を開き、下記を設定する
* Hostname: OpenWrt

### 日時設定
 `[System] -> [System]` を開き、下記を設定する
* TimeZone: `Asia/Tokyo` 

### パスワード設定
 `[System]-> [Administration] -> [Router Password]` を開きパスワードを設定する

### IPアドレス設定
 `[Network] -> [Interface] -> [Interface] -> [Edit]` を開く.  
 LANの設定を以下のようにする.  
```
General Settings
  Protocol Static Address
Device br-lan
  Bring up on boot Checked
    IPv4 address 192.168.3.240
    IPv4 netmask 255.255.255.0
    IPv4 gateway none
    IPv4 broadcast 192.168.3.255

  DHCP Server
    Advanced Settings
    Dynamic DHCP: チェックを外す

  IPv6 Settings
    RA-Service relay mode
    DHCPv6-Service relay mode
    NDP-Proxy relay mode

  Global network options
    IPv6 ULA-Prefix 空白にする
    → 初期設定でユニークローカルアドレス(unique local addresses)が設定されているため削除しておく.
```

### SSH設定
無し.  
デフォルトで接続できる.  
参考: [SSH access for newcomers](https://openwrt.org/docs/guide-quick-start/sshadministration)

### IPv6パススルー設定
WANから届いたRAをLANへ流すためパススルー設定を行う.  
参考: https://nabe.adiary.jp/0633  

#### WAN6
IPv6のDHCP ServerをRelayModeへ変更する
 `Interface -> WAN6 -> DHCP Server` 
```
IPv6 Settings
 - Designated master: チェック有り
 - RA-Service relay mode
 - DHCPv6-Service relay mode
 - NDP-Proxy relay mode
 - NDP-Proxy チェック
 - NDP-Proxy slave: チェック無し
```

#### LAN
IPv6のDHCP ServerをRelayModeへ変更する
 `Interface -> lan -> DHCP Server` 
```
IPv6 Settings
 - Designated master: チェック無し
 - RA-Service relay mode
 - DHCPv6-Service relay mode
 - NDP-Proxy relay mode
 - NDP-Proxy: チェック有り
 - NDP-Proxy slave: チェック無し
```

 `Interface -> WAN6 -> Advanced Settings` 
```
 - Delegate IPv6 prefixes: チェック有り
```

## PPPoE設定
IPoE(MAP)パッケージを入れるため、一時的にPPPoE設定にして通信出来る状態にする.

### PPPoE設定
開放されているPPPoEアカウントを利用して一時的に繋ぐ.  
* [新型コロナ対策のためソフトイーサ社のフレッツ用 PPPoE 実験用アクセスポイントをテレワーク用に無償開放](https://www.softether.jp/7-news/2020.03.06)  
 `Network -> Interface -> WAN -> Edit`
```
- Protocol PPPoE
- PAP/CHAP username open@open.ad.jp
- PAP/CHAP password xxxx
→ Saveして反映する.
```

## IPoE設定
### MAPパッケージの準備
IPoE(MAP-E)に必要なMAPパッケージをインストールする.  
参考: https://note.com/sol_stl/n/nd230789b24b1
```
% ssh root@192.168.3.240
root@OpenWrt:~# opkg update
root@OpenWrt:~# opkg install map
root@OpenWrt:~# opkg info map
Package: map
Version: 7
Depends: libc, kmod-ip6-tunnel, libubox20220515, libubus20220601, iptables-mod-conntrack-extra, kmod-nat46
Provides: map-t
Status: install user installed
Section: net
Architecture: x86_64
Size: 7926
Filename: map_7_x86_64.ipk
Description: Provides support for MAP-E (RFC7597), MAP-T (RFC7599) and
 Lightweight 4over6 (RFC7596) in /etc/config/network.
 MAP combines address and port translation with the tunneling
 of IPv4 packets over an IPv6 network
Installed-Time: 1693747341
```

### Reboot実施
MAPパッケージを認識させるため.
```
root@OpenWrt:~# reboot
```

### IPv6アドレス計算
IPoEは [RFC 7597](https://datatracker.ietf.org/doc/html/rfc7597)にあるMapRuleに則ってIPv4アドレス、IPv6アドレスを計算する.  
OpenWrt設定の際に必要になってくるため事前に算出する.

#### 計算の実施
http://ipv4.web.fc2.com/map-e.html  
へアクセスし、以下の情報を入力する.  
|項目|値|
|---|---|
|IPv6 プレフィックスかアドレスを入力|240b:10:...::/64|
  
計算すると以下のような値が出力されるためメモする.  
|項目|値|
|CE|240b:10:....abcd|
|IPv4 アドレス|106.72.xx.xx|
|PSID|xxx|
```
option peeraddr 2404:...  ・・・ JPNEのBR (Border Relay)の模様.
option ipaddr 106.72.xxx.xxx
option ip4prefixlen 15
option ip6prefix 240b:10::
option ip6prefixlen a
option ealen b
option psidlen c
option offset d

export LEGACY=1
```

### IPoE設定 at OpenWrt
MAP-E接続用のInterfaceを作成してIPoE接続をする.

#### MAP-E Interface作成
 `Network -> Interface -> Add Interface [wan6_MAPE]` で作成を行う.
設定値は以下を入れる.
```
- Protocol MAP / LW4over6
- Type MAP-E
- BR / DMR / AFTR 2404:...
- IPv4 prefix 106.72.xxx.xxx
- IPv4 prefix length 15
- IPv6 prefix 240b:10::
- IPv6 prefix length a
- EA-bits length b
- PSID-bits length c
- PSID offset d

Advanced Settings
 - Use legacy MAP: チェック有り
 - Delegate IPv6 prefixed: チェック有り
```
→ 「Error: MAP rule is invalid」と表示される.  
次手順で直すためそのまま進める.

#### WAN6編集
調べた結果、以下に該当することが判明したため手修正する.
https://github.com/fakemanhk/openwrt-jp-ipoe
```
WAN6 interface also needs to add Customized Prefix Delegation, using SSH to login router, edit /etc/config/network, add the line marked in Italics under WAN6 interface section, note the 2400:aaaa:bbbb:cccc is your WAN IP prefix (first 64 bit), this will give the WAN6 interface proper IPv6-PD:

config interface 'wan6'
...
 option ip6prefix '2400:aaaa:bbbb:cccc::/64'

Note: Previously I had failed my setup because of missing this step, it wasn't mentioned in most resources I found on web, and I eventually got a MAP rule invalid error.
```

以下で修正
```
# ssh root@192.168.3.240
# cp -pv /etc/config/network /etc/config/network.def
$ vim /etc/config/network
----
...
config interface 'wan6'
	option device 'eth1'
	option proto 'dhcpv6'
	option reqaddress 'try'
	option reqprefix 'auto'
+ 	option ip6prefix '240b:10:...::/64'
....
```
保存後、 `[Interface] -> [wan6]` 右の `[Restart]` を押して設定反映する.

#### MAP-Eインターフェイス確認
IPv4アドレスを取得できていればOK.

* WAN6
```
wan6
Type: Ethernet Adapter
Device: eth1
Connected: yes
MAC: 80:EE:73:...

eth1
Protocol: DHCPv6 client
Uptime: 0h 8m 50s
MAC: 80:EE:73:...
...
IPv6: 240b:10:.../64
IPv6-PD: 240b:10:...::/64 ・・・ ★表示されればOK.
```

* wan6_MAPE
```
wan6_MAPE
Type: Tunnel Interface
Device: map-wan6_MAPE
Connected: yes

map-wan6_MAPE
Protocol: MAP / LW4over6
Uptime: 0h 9m 40s
...
IPv4: 106.72.xxx.xx/32 ・・・ ★表示されればOK.

wan6_MAPE_
Type: Ethernet Adapter
Device: eth1
Connected: yes
MAC: 80:EE:73:...

eth1
Protocol: Virtual dynamic interface (Static address)
Uptime: 0h 9m 25s
IPv6: 240b:10:.../128 ・・・ ★表示されればOK.
```

### 動作確認
PCからOpenWrtを通して外部接続できればOK.
* PC設定
```
IP: 192.168.3.xx
NetMask: 255.255.255.0
DefaultGateway: 192.168.3.240
DNS: 8.8.8.8, 2001:4860:4860::8888 etc...
```
* IPv4アクセス例  
https://www.yahoo.co.jp/
* IPv6アクセス例  
https://www.youtube.com/

ブラウザアクセスできればOK.

# 備忘メモ
設定ファイル
```
% ssh root@192.168.3.240
root@192.168.3.240's password: 


BusyBox v1.35.0 (2023-04-27 20:28:15 UTC) built-in shell (ash)

  _______                     ________        __
 |       |.-----.-----.-----.|  |  |  |.----.|  |_
 |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
 |_______||   __|_____|__|__||________||__|  |____|
          |__| W I R E L E S S   F R E E D O M
 -----------------------------------------------------
 OpenWrt 22.03.5, r20134-5f15225c1e
 -----------------------------------------------------
root@OpenWrt:~# 
root@OpenWrt:~# cat /etc/config/network

config interface 'loopback'
	option device 'lo'
	option proto 'static'
	option ipaddr '127.0.0.1'
	option netmask '255.0.0.0'

config globals 'globals'

config device
	option name 'br-lan'
	option type 'bridge'
	list ports 'eth0'
	option ip6segmentrouting '1'
	option drop_unsolicited_na '1'

config interface 'lan'
	option device 'br-lan'
	option proto 'static'
	option netmask '255.255.255.0'
	option ip6assign '60'
	option ipaddr '192.168.3.240'

config interface 'wan6'
	option device 'eth1'
	option proto 'dhcpv6'
	option reqaddress 'try'
	option reqprefix 'auto'
	option ip6prefix '240b:10:...::/64'

config interface 'wan6_MAPE'
	option proto 'map'
	option maptype 'map-e'
	option peeraddr '2404:...'
	option ipaddr '106.72.xxx.xxx'
	option ip4prefixlen '15'
	option ip6prefix '240b:10::'
	option ip6prefixlen 'xx'
	option ealen 'xx'
	option psidlen 'x'
	option offset 'x'
	option tunlink 'wan6'
	option legacymap '1'

root@OpenWrt:~# 
```
以上
