# はじめに
[WireGuard](https://www.wireguard.com/)のVPN環境の構築を行う.  
IPv6(IPoE)環境で構築する場合、IPv4アドレスを他ユーザと共用する仕組みのため、IPv4の指定ポートをポートフォワード出来ない.  
このためIPv6のVPNの入り口を構築する.

# 環境
```
[VPN Client] --(IPv6 Internet)--> [IPv4/IPv6 Router] --(Open Port 51820/udp on IPv6)--> [VPN Gateway(Ubuntu22.04)] --(iptables NAT)--> [Trusted Servers]

※補足
[VPN Client]
- External Interface
 - Any
- Internal Interface
 - fd00::2/64
 - 10.0.0.2/24

[VPN Gateway(Ubuntu22.04)]
- External Interface
 - 2001:db8::1/64
 - 192.168.3.0/24
- Internal Interface
 - fd00::1/64
 - 10.0.0.1/24

※ Internal Interface -> External Interface間で、IPv4/IPv6をNATさせる.(IP MASQUERADE)
```

# 手順
## RouterのPort設定
Router管理画面から以下のポートを開く.
|Action|Src IP address| Src Port | Dest IP Address | Dest Port|
|---|---|---|---|---|
|Permit|Any|Any|2001:db8::1/128|51820/udp|

## WireGuardサーバの準備
### OS設定
IPv4, IPv6転送設定を行う.
```
@VPN Gateway
$ sudo cp -vp /etc/sysctl.conf ~
$ sudo vim /etc/sysctl.conf
----
+ net.ipv4.ip_forward=1
+ net.ipv6.conf.all.forwarding=1
----
$ sudo sysctl -p
```

### WireGuardインストール
```
@VPN Gateway
$ sudo apt -y install wireguard
```

### 公開鍵, 秘密鍵鍵の準備
WireGuardで生成する鍵は最低4つ要る.  
wg genkeyを使用して、キー生成を行い、teeコマンドでキーを保存する.
* サーバの秘密鍵
* サーバの公開鍵
* クライアントの秘密鍵
* クライアントの公開鍵
```
@VPN Gateway
$ wg genkey | sudo tee /etc/wireguard/server_private.key
$ sudo cat /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.pub
$ wg genkey | sudo tee /etc/wireguard/client_001.key
$ sudo cat /etc/wireguard/client_001.key | wg pubkey | sudo tee /etc/wireguard/client_001.pub

$ sudo chmod -v 600 /etc/wireguard/{server_private.key,server_public.pub,client_001.key,client_001.pub}
mode of '/etc/wireguard/server_private.key' changed from 0644 (rw-r--r--) to 0600 (rw-------)
mode of '/etc/wireguard/server_public.pub' changed from 0644 (rw-r--r--) to 0600 (rw-------)
mode of '/etc/wireguard/client_001.key' changed from 0644 (rw-r--r--) to 0600 (rw-------)
mode of '/etc/wireguard/client_001.pub' changed from 0644 (rw-r--r--) to 0600 (rw-------)

$ sudo ls -al /etc/wireguard/{server_private.key,server_public.pub,client_001.key,client_001.pub}
-rw------- 1 root root 45 Jul 16 01:57 /etc/wireguard/client_001.key
-rw------- 1 root root 45 Jul 16 01:57 /etc/wireguard/client_001.pub
-rw------- 1 root root 45 Jul 16 01:57 /etc/wireguard/server_private.key
-rw------- 1 root root 45 Jul 16 01:57 /etc/wireguard/server_public.pub
```

### WireGuard設定ファイル作成
WireGuardデーモンが利用する設定ファイルを準備する.
```
@VPN Gateway
$ sudo vim /etc/wireguard/wg0.conf
----
[Interface]
PrivateKey = ... (Value of server_private.key) ...
Address = 10.0.0.1/24, fd00::1/64
ListenPort = 51820
PostUp = iptables -A FORWARD -i %i -s 10.0.0.0/24 -j ACCEPT; iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o p2p1 -j MASQUERADE;  ip6tables -A FORWARD -i %i -s fd00::/64 -j ACCEPT; ip6tables -t nat -A POSTROUTING -s fd00::/64 -o p2p1 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -s 10.0.0.0/24 -j ACCEPT; iptables -t nat -D POSTROUTING -s 10.0.0.0/24 -o p2p1 -j MASQUERADE;  ip6tables -D FORWARD -i %i -s fd00::/64 -j ACCEPT; ip6tables -t nat -D POSTROUTING -s fd00::/64 -o p2p1 -j MASQUERADE

[Peer]
PublicKey = ... (Value of client_001.pub) ...
AllowedIPs = 10.0.0.2/24, fd00::2/64
----

$ sudo chmod -v 600 /etc/wireguard/wg0.conf
mode of '/etc/wireguard/wg0.conf' changed from 0644 (rw-r--r--) to 0600 (rw-------)
```

### WireGuard起動
* 起動
```
@VPN Gateway
$ sudo systemctl start wg-quick@wg0
$ sudo systemctl status wg-quick@wg0
```
→ 起動が成功するとInterface wg0が出来る.
```
$ ifconfig wg0
wg0: flags=209<UP,POINTOPOINT,RUNNING,NOARP>  mtu 1420
 inet 10.0.0.1  netmask 255.255.255.0  destination 10.0.0.1
 inet6 fd00::1  prefixlen 64  scopeid 0x0<global>
 unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 1000  (UNSPEC)
 RX packets 0  bytes 0 (0.0 B)
 RX errors 0  dropped 0  overruns 0  frame 0
 TX packets 0  bytes 0 (0.0 B)
 TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

* OS起動時に有効にする
```
@VPN Gateway
$ sudo systemctl enable wg-quick@wg0
```
* 停止
```
@VPN Gateway
$ sudo systemctl stop wg-quick@wg0 
$ sudo systemctl status wg-quick@wg0
```

## WireGuard Clinet設定
### Download WireGuard Client
公式サイトからダウンロードする.  
https://www.wireguard.com/install/

### 設定ファイルを準備する
```
@VPN Client
$ vim wg0-client.conf
----
[Interface]
PrivateKey = ... (Value of client_001.key) ...
Address = 10.0.0.2/24, fd00::2/64

[Peer]
Endpoint = [2001:db8::1]:51820
PublicKey = ... (Value of server_public.pub) ...
AllowedIPs = 10.0.0.0/24, fd00::/64, 192.168.3.0/24, 2001:db8::/64
PersistentKeepalive = 25
----
```

### Importする
WireGuard → 「ファイルからトンネルをインポート」を選ぶ

### 接続する
WireGuardから「有効化」を選び、接続さればOK.  
接続確認として、同じLANのWebページの閲覧やssh確認ができればOK.  
  
例) 確認例
* 192.168.3.x へのWebページ表示
* 2001:db8::x へのssh

# その他
## iptablesのdebug
iptablesでパケットが通ったかどうかを確認する際にはlog出力設定を行うことでどこまで通過したかを確認できる.

### ログ出力
```
$ sudo iptables -nL -t nat
---
$ sudo iptables -nL -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  10.0.0.0/24          0.0.0.0/0
---

$ sudo iptables -t nat -I POSTROUTING 1 -j LOG --log-prefix "NAT "
$ sudo iptables -nL -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
LOG        all  --  0.0.0.0/0            0.0.0.0/0            LOG flags 0 level 4 prefix "NAT " ・・・★ログ出力設定
MASQUERADE  all  --  10.0.0.0/24          0.0.0.0/0
$
```
### ログ確認
Syslogの出力結果から確認できる
```
$ sudo less /var/log/syslog
----
...
Jul 23 19:28:40 livaserver kernel: [ 4838.000669] NAT IN=wg0 OUT=p2p1 MAC= SRC=10.0.0.2 DST=192.168.3.254 LEN=64 TOS=0x00 PREC=0x00 TTL=63 ID=0 DF PROTO=TCP SPT=55161 DPT=80 WINDOW=65535 RES=0x00 CWR ECE SYN URGP=0
...
----
```
### ログ出力設定の削除
iptablesのchainを削除することで解除できる.
```
$ sudo iptables -nL -t nat --line-numbers
...

Chain POSTROUTING (policy ACCEPT)
num  target     prot opt source               destination         
1    LOG        all  --  0.0.0.0/0            0.0.0.0/0            LOG flags 0 level 4 prefix "NAT "
2    MASQUERADE  all  --  10.0.0.0/24          0.0.0.0/0

$ sudo iptables -t nat -D POSTROUTING 1
$ sudo iptables -nL -t nat --line-numbers
...
Chain POSTROUTING (policy ACCEPT)
num  target     prot opt source               destination
1    MASQUERADE  all  --  10.0.0.0/24          0.0.0.0/0
→ 削除できればOK.
```

## 参考
* [【Linux】Ubuntu 22.04 Wireguard インストール手順](https://tech.willserver.asia/2023/03/15/lix-wireguard-setup/)
* [WireGuard : サーバーの設定](https://www.server-world.info/query?os=Ubuntu_22.04&p=wireguard&f=1)
* [IPv6 + WireGuard でリモートアクセス VPN](https://qiita.com/hoto17296/items/4e6fcaa2d0a853a5fc0d)
* [iptablesのログを出力して確認する方法](https://blog.e2info.co.jp/2017/07/04/iptables_log/)
