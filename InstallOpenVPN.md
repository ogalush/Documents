# OpenVPNインストール
## はじめに
ブロードバンドルータのポート開放でUDPポートだけを通せば使用できるOpenVPNをインストールする.  

## 環境
|マシン|OS|IPアドレス|
|---|---|---|
|VPNServer(IPv4)|Ubuntu16.04(x86_64)|192.168.0.220/24|
|VPNServer(IPv6)|Ubuntu22.04(x86_64)|2400:2410:xxxx:xxxx::b55f/64|
|クライアント|macOS Sierra(10.12.4)|任意(外部からの接続)|

## 構成
LAN内のLinuxサーバにOpenVPNを構築する.
```
[外部]-(グローバルIP)->[ルータ(udp:1194)]-(192.168.0.0/24)->[VPNServer(Real:192.168.0.220/24, VPN:10.8.0.0/24)]
```

## VPNServer インストール
### パッケージインストール
動作に必要なパッケージをインストールする。
```
$ ssh -A 192.168.0.220
$ sudo apt-get -y update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade && sudo apt-get -y autoremove
$ sudo apt-get -y install openvpn easy-rsa
```

### 証明書の作成
OpenVPNで使用する証明書を作成する.  
#### 環境設定
```
$ make-cadir ~/openvpn-ca
$ vi ~/openvpn-ca/vars
----
##export KEY_COUNTRY="US"
##export KEY_PROVINCE="CA"
##export KEY_CITY="SanFrancisco"
##export KEY_ORG="Fort-Funston"
##export KEY_EMAIL="me@myhost.mydomain"
##export KEY_OU="MyOrganizationalUnit"
export KEY_COUNTRY="JP"
export KEY_PROVINCE="Tokyo"
export KEY_CITY="ナイショ"
export KEY_ORG="ナイショ"
export KEY_EMAIL="ナイショ"
export KEY_OU="OGALUSH"
##export KEY_NAME="EasyRSA"
export KEY_NAME="ナイショ"
----
```

#### 環境設定読込み
```
$ cd ~/openvpn-ca/
$ source vars 
NOTE: If you run ./clean-all, I will be doing a rm -rf on /home/ogalush/openvpn-ca/keys
```

#### サーバ証明書作成
```
$ ./clean-all
$ ./build-ca 
Generating a 2048 bit RSA private key
.........................................+++
.....................................+++
writing new private key to 'ca.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [JP]:
State or Province Name (full name) [Tokyo]:
Locality Name (eg, city) [****]:
Organization Name (eg, company) [****]:
Organizational Unit Name (eg, section) [OGALUSH]:
Common Name (eg, your name or your server's hostname) [teamlush CA]:
Name [****]:
Email Address [****@****]:
-----

$ ./build-key-server server
Generating a 2048 bit RSA private key
................+++
............................+++
writing new private key to 'server.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [JP]:
State or Province Name (full name) [Tokyo]:
Locality Name (eg, city) [***]:
Organization Name (eg, company) [teamlush]:
Organizational Unit Name (eg, section) [OGALUSH]:
Common Name (eg, your name or your server's hostname) [server]:
Name [***]:
Email Address [***]:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
Using configuration from /home/ogalush/openvpn-ca/openssl-1.0.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
countryName           :PRINTABLE:'JP'
stateOrProvinceName   :PRINTABLE:'Tokyo'
localityName          :PRINTABLE:'***'
organizationName      :PRINTABLE:'teamlush'
organizationalUnitName:PRINTABLE:'OGALUSH'
commonName            :PRINTABLE:'server'
name                  :T61STRING:'****'
emailAddress          :IA5STRING:'***'
Certificate is to be certified until Apr 12 16:03:20 2027 GMT (3650 days)
Sign the certificate? [y/n]: y
1 out of 1 certificate requests certified, commit? [y/n] y
Write out database with 1 new entries
Data Base Updated
----

$ ./build-dh
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
....+.............................(略)...

$ openvpn --genkey --secret keys/ta.key
→ サーバ証明書と秘密鍵が出来上がる.
$ ls -al keys/server.*
-rw-rw-r-- 1 ogalush ogalush 5665 Apr 15 01:04 keys/server.crt
-rw-rw-r-- 1 ogalush ogalush 1090 Apr 15 01:03 keys/server.csr
-rw------- 1 ogalush ogalush 1708 Apr 15 01:03 keys/server.key
```

### クライアント証明書作成
```
$ ./build-key client1
ogalush@livaserver:~/openvpn-ca$ ./build-key client1
Generating a 2048 bit RSA private key
.............+++
...............................................................................................+++
writing new private key to 'client1.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [JP]:
State or Province Name (full name) [Tokyo]:
Locality Name (eg, city) [***:
Organization Name (eg, company) [teamlush]:
Organizational Unit Name (eg, section) [OGALUSH]:
Common Name (eg, your name or your server's hostname) [client1]:
Name [***]:
Email Address [***@***]:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
Using configuration from /home/ogalush/openvpn-ca/openssl-1.0.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
countryName           :PRINTABLE:'JP'
stateOrProvinceName   :PRINTABLE:'Tokyo'
localityName          :PRINTABLE:'***'
organizationName      :PRINTABLE:'teamlush'
organizationalUnitName:PRINTABLE:'OGALUSH'
commonName            :PRINTABLE:'client1'
name                  :T61STRING:'***'
emailAddress          :IA5STRING:'***'
Certificate is to be certified until Apr 12 16:08:03 2027 GMT (3650 days)
Sign the certificate? [y/n]:y  

1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
→ クライアント証明書が出来上がる.
$ ls -al keys/client1.*
-rw-rw-r-- 1 ogalush ogalush 5550 Apr 15 01:08 keys/client1.crt
-rw-rw-r-- 1 ogalush ogalush 1094 Apr 15 01:08 keys/client1.csr
-rw------- 1 ogalush ogalush 1704 Apr 15 01:08 keys/client1.key
```

### OpenVPNの設定

#### 証明書配置
CA証明書、Server証明書をOpenVPNディレクトリへコピーする.
```
$ sudo cp -v ~/openvpn-ca/keys/{ca.crt,ca.key,server.crt,server.key,ta.key,dh2048.pem} /etc/openvpn/
[sudo] password for ogalush: 
‘/home/ogalush/openvpn-ca/keys/ca.crt’ -> ‘/etc/openvpn/ca.crt’
‘/home/ogalush/openvpn-ca/keys/ca.key’ -> ‘/etc/openvpn/ca.key’
‘/home/ogalush/openvpn-ca/keys/server.crt’ -> ‘/etc/openvpn/server.crt’
‘/home/ogalush/openvpn-ca/keys/server.key’ -> ‘/etc/openvpn/server.key’
‘/home/ogalush/openvpn-ca/keys/ta.key’ -> ‘/etc/openvpn/ta.key’
‘/home/ogalush/openvpn-ca/keys/dh2048.pem’ -> ‘/etc/openvpn/dh2048.pem’

$ sudo chmod -v 400 /etc/openvpn/{ca.crt,ca.key,server.crt,server.key,ta.key,dh2048.pem}
mode of ‘/etc/openvpn/ca.crt’ retained as 0400 (r--------)
mode of ‘/etc/openvpn/ca.key’ retained as 0400 (r--------)
mode of ‘/etc/openvpn/server.crt’ retained as 0400 (r--------)
mode of ‘/etc/openvpn/server.key’ retained as 0400 (r--------)
mode of ‘/etc/openvpn/ta.key’ retained as 0400 (r--------)
mode of ‘/etc/openvpn/dh2048.pem’ retained as 0400 (r--------)

$ sudo chown -v nobody:nogroup /etc/openvpn/{ca.crt,ca.key,server.crt,server.key,ta.key,dh2048.pem}
changed ownership of ‘/etc/openvpn/ca.crt’ from root:root to nobody:nogroup
changed ownership of ‘/etc/openvpn/ca.key’ from root:root to nobody:nogroup
changed ownership of ‘/etc/openvpn/server.crt’ from root:root to nobody:nogroup
changed ownership of ‘/etc/openvpn/server.key’ from root:root to nobody:nogroup
changed ownership of ‘/etc/openvpn/ta.key’ from root:root to nobody:nogroup
changed ownership of ‘/etc/openvpn/dh2048.pem’ from root:root to nobody:nogroup
```

#### 設定
サンプルの設定ファイルを配置する.
```
$ gunzip -vc /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz | sudo tee /etc/openvpn/server.conf
$ sudo vim /etc/openvpn/server.conf
----
; tls-auth ta.key 0 # This file is secret
tls-auth ta.key 0 # This file is secret
→コメントを外す.

; user nobody
; group nogroup
user nobody
group nogroup
→コメントを外す.

;dh dh1024.pem
dh dh2048.pem
→ ファイル名を修正する.

# ルーティング設定.
push "route 192.168.0.0 255.255.255.0"
----
```

#### Server起動
OpenVPNサーバを起動させる.
```
$ sudo service openvpn status
 * VPN 'server' is not running
$ sudo service openvpn start
 * Starting virtual private network daemon(s)...
 *   Autostarting VPN 'server'
$ sudo service openvpn status
 * VPN 'server' is running
→ サーバが起動すればOK.
```

#### NAT設定
OpenVPNサーバから外部サーバへ出ていけるようにNAT設定する.
```
$ sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o p2p1 -j MASQUERADE
$ sudo iptables -nL -t nat
ogalush@livaserver:~$ sudo iptables -nL -t nat
Chain POSTROUTING (policy ACCEPT)・・・POSTROUTING(NAT)が入っていればOK.
target     prot opt source               destination         
MASQUERADE  all  --  10.8.0.0/24          0.0.0.0/0           

$ sudo apt-get -y install iptables-persistent
→ Ubuntuの場合iptables saveを使用できないので↑を入れる.
ogalush@livaserver:~$ sudo iptables-save 
# Generated by iptables-save v1.4.21 on Sat Apr 15 03:48:15 2017
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.8.0.0/24 -o p2p1 -j MASQUERADE
COMMIT
# Completed on Sat Apr 15 03:48:15 2017
# Generated by iptables-save v1.4.21 on Sat Apr 15 03:48:15 2017
*filter
:INPUT ACCEPT [95:6260]
:FORWARD ACCEPT [4:460]
:OUTPUT ACCEPT [61:6612]
COMMIT
# Completed on Sat Apr 15 03:48:15 2017

$ sudo vim /etc/sysctl.conf
----
#-- for OpenVPN 2017.4.15
net.ipv4.ip_forward=1
----

$ sudo sysctl -p
→ 反映できればOK.
```



## Macクライアントの設定
### クライアント設定ファイルの準備
OpenVPNサーバでクライアント用設定ファイルを作成する.
```
$ mkdir -pv ~/client-configs/files
$ cd ~/client-configs
$ cp -v /usr/share/doc/openvpn/examples/sample-config-files/client.conf base.conf

$ vim base.conf
----
remote 192.168.0.220 1194
tls-auth ta.key 1
#ca ca.crt
#cert client.crt
#key client.key
----
```
  
以下のコマンドでファイルを作成する.
```
$ vim ~/client-configs/make_config.sh
----
#!/bin/bash

# First argument: Client identifier

KEY_DIR=~/openvpn-ca/keys
OUTPUT_DIR=~/client-configs/files
BASE_CONFIG=~/client-configs/base.conf

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn
----
→ シェルが作成されている.
$ chmod -v 755 ~/client-configs/make_config.sh
$ ./make_config.sh client1
$ ls -al ~/client-configs/files/client1.ovpn
→ client1.ovpnファイルがあればOK.
```

### 設定ファイルのダウンロード
OpenVPNサーバで作成したclient設定をダウンロードする.
```
$ scp 192.168.0.220:~/client-configs/files/client1.ovpn ~/Downloads/.
scp 192.168.0.220:~/openvpn-ca/keys/ta.key  ~/Downloads/.
```
### Tunnelblick の取得
OpenVPN対応の無料Clientがあるのでインストールする.  
[Tunnelblick 	free software for OpenVPN on OS X and macOS](https://tunnelblick.net/downloads.html)のダウンロードを開く.  
ダウンロード先: [Tunnelblick 3.7.0](https://tunnelblick.net/release/Tunnelblick_3.7.0_build_4790.dmg)  

### Tunnelblick のインストール
ダウンロードパッケージをダブルクリックしてインストールを進める.  
- 変更のチェック: 「変更のチェック」を押す.
- 設定ファイルがある.
- VPNを追加
- client1.ovpnをドラッグして追加する.
- VPN接続先が登録されればOK.
- 接続先登録後、ta.keyは削除する.(初回のみファイルが存在しないとチェックされるため)
- その後、VPN接続でサーバへ接続できれば完了.

## ルータのポート開放
ルータの管理画面でUDP 1194をOpenVPNサーバへ転送する設定にする.

# 参考
- [OpenVPN.Net](https://openvpn.net/)
- [Ubuntu 16.04 の OpenVPN サーバの設定](http://transitive.info/2016/05/21/openvpn-on-ubuntu-1604/)
- [OpenVPN and Routing and IPtables](https://serverfault.com/questions/570164/openvpn-and-routing-and-iptables)


## 2022.10.1追記 IPv6設定
IPv6アドレスでVPN接続出来るようにす.  
ISPを変更するとIPv4 Over IPv6(MAP-E, transix)の影響で外部からIPv4接続でPort番号の制限を受けるため.  
  
参考: [IPv6 in OpenVPN](https://community.openvpn.net/openvpn/wiki/IPv6)  
### OpenVPN設定
```
## For IPv6 Settings (2022.10.1)
## https://community.openvpn.net/openvpn/wiki/IPv6
proto udp6
##server-ipv6 2400:2410:a741:fc00::/64
##tun-ipv6
##push tun-ipv6
##ifconfig-ipv6 2400:2410:a741:fc00::1 2400:2410:a741:fc00::2
##push "route-ipv6 2400:2410:a741:fc00::/64"
```
### IPv6 Port開放
SoftBank光(BBユニット)でIPv6をポート開放する.  
  
「ルーター機能の設定」 → パケットフィルタ設定 → フィルタエントリー設定（IPv6）
```
有効/無効  ルールNo.  種類  送信元IPv6アドレス  プロトコル  送信元ポート番号 	通信方向
宛先IPv6アドレス  宛先ポート番号

(1) 往路
有効  1  許可  ANY  UDP  ANY  WAN→LAN
2400:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx:b55f/128  1194-1194

(2) 復路
有効  2  許可  2400:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx:b55f/128  UDP  1194-1194  LAN→WAN
ANY  ANY
```
