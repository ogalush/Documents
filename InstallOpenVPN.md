# OpenVPNインストール
## はじめに
ブロードバンドルータのポート開放でUDPポートだけを通せば使用できるOpenVPNをインストールする.  

## 環境
|マシン|OS|IPアドレス|
|---|---|---|
|VPNServer|Ubuntu16.04(x86_64)|192.168.0.220|
|クライアント|MaxOS Sierra(10.12.4)|192.168.0.X|

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
#### 環境設定作成
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
```

# 参考
- [OpenVPN.Net](https://openvpn.net/)
- [Ubuntu 16.04 の OpenVPN サーバの設定](http://transitive.info/2016/05/21/openvpn-on-ubuntu-1604/)
