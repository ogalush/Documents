# はじめに
本ドキュメントは、L2TP+StrongSwanを使用したL2TP+IPSecのVPNを構築する手順である.  
参考: [strongSwan + xl2tpd でVPN(L2TP/IPsec)を構築する](http://qiita.com/namoshika/items/30c348b56474d422ef64)

# 環境
Ubuntu 14.04(x86_64)  
[Internet]--(VPN Client)-->[Router]--(192.168.0.0/24)-->[Linux(L2TP+StrongSwan, 192.168.0.220)]

# インストール手順
## 準備
### OS最新化
```
$ sudo apt-get -y update && sudo apt-get -y upgrade
$ sudo apt-get -y dust-upgrade && sudo apt-get -y autoremove
$ sudo shutdown -r now
```

## パッケージインストール
```
$ sudo apt-get install -y strongswan xl2tpd

$ dpkg -l |grep strongswan
ii  libstrongswan                         5.1.2-0ubuntu2.5                    amd64        strongSwan utility and crypto library
ii  strongswan                            5.1.2-0ubuntu2.5                    all          IPsec VPN solution metapackage
ii  strongswan-ike                        5.1.2-0ubuntu2.5                    amd64        strongSwan Internet Key Exchange (v2) daemon
ii  strongswan-plugin-openssl             5.1.2-0ubuntu2.5                    amd64        strongSwan plugin for OpenSSL
ii  strongswan-starter                    5.1.2-0ubuntu2.5                    amd64        strongSwan daemon starter and configuration file parser
~~~ パッケージが入っていればOK.

$ dpkg -l |grep xl2tpd
ii  xl2tpd                                1.3.6+dfsg-1                        amd64        layer 2 tunneling protocol implementation
~~~ パッケージが入っていればOK. 
```

## 設定
### StrongSwan (IPSec)
IPSecの設定とPSKの事前共有キーを設定する.
```
$ sudo cp -pv /etc/ipsec.conf /etc/ipsec.conf.def
‘/etc/ipsec.conf’ -> ‘/etc/ipsec.conf.def’
$ sudo vim /etc/ipsec.conf
----
# ipsec.conf - strongSwan IPsec configuration file

# basic configuration
config setup
  # 必須。VPNクライアントがNATの内側にある場合に必要。
  nat_traversal=yes

# Add connections here.
# デフォルト設定(全接続の共通設定が記述されます)
conn %default
    # 必須。strongSwan起動時に接続プロファイルを受信待機状態にする。
    auto=add
# L2TP/IPSec用接続設定
conn L2TP-NAT
    # 必須。IPSec/L2TPではIPSecはHost to Hostで繋ぐ。
    type=transport
    # 省略可。指定するならばVPNサーバーのIPアドレス。
    # (ex: 192.168.x.x)
    #left=%any 
    # 必須(default: pubkey)。認証方式。事前共有鍵を使用。
    leftauth=psk
    # 省略可。VPNクライアントのIPアドレス
    # (ex: 192.168.y.y)
    #right=%any
    # 必須。認証方式。事前共有鍵を使用。
    rightauth=psk
----

$ sudo vim /etc/ipsec.secrets
----
: PSK "***(ナイショ)***"
----
```

### xl2tpd (L2TP)
```
$ sudo cp -pv /etc/xl2tpd/xl2tpd.conf /etc/xl2tpd/xl2tpd.conf.def
‘/etc/xl2tpd/xl2tpd.conf’ -> ‘/etc/xl2tpd/xl2tpd.conf.def’
$ sudo vim /etc/xl2tpd/xl2tpd.conf
----
[global]
[lns default]                               ; Our fallthrough LNS definition
  ; VPNクライアントへ振るIPアドレス範囲
  ip range = 192.168.0.20-192.168.0.30
  ; VPNサーバーのIPアドレス
  local ip = 192.168.0.220
  length bit = yes                          ; * Use length bit in payload?
  refuse pap = yes                          ; * Refuse PAP authentication
  refuse chap = yes                         ; * Refuse CHAP authentication
  require authentication = yes              ; * Require peer to authenticate
  name = l2tp                               ; * Report this as our hostname
  pppoptfile = /etc/ppp/options.l2tpd.lns   ; * ppp options file
----

$ sudo vim /etc/ppp/options.l2tpd.lns
----
name l2tp
refuse-pap
refuse-chap
refuse-mschap
require-mschap-v2
nodefaultroute
lock
nobsdcomp
mtu 1280
mru 1280
logfile /var/log/xl2tpd.log
----

$ sudo vim /etc/ppp/chap-secrets
----
# Secrets for authentication using CHAP
# client        server  secret                  IP addresses
"ogalush" * "(ナイショ)" *
----
```

### sysctl
```
$ sudo vim /etc/sysctl.conf
----
...
net.ipv4.ip_forward=1
----
```

### 反映
設定を反映する.
```
$ sudo sysctl -p
...
net.ipv4.ip_forward = 1 ← ip_forward=1 が表示されればOK.

$ sudo service strongswan restart
strongswan stop/waiting
strongswan start/running

$ sudo service xl2tpd restart
Restarting xl2tpd: xl2tpd.
```

### ルータのポート開放
[atermサポートページ](http://www.aterm.jp/function/guide8/list-data/j-all/bl/k/m01_m34.html)を参考にする.  
* InBound UDP500, 4500をL2TP+IPSecサーバ宛にポートフォワードする.  
* プロトコル番号: 50, 51を開放する.
