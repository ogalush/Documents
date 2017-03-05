# はじめに
本ドキュメントは、L2TP+StrongSwanを使用したL2TP+IPSecのVPNを構築する手順である.  
参考: [strongSwan + xl2tpd でVPN(L2TP/IPsec)を構築する](http://qiita.com/namoshika/items/30c348b56474d422ef64)

# 環境
Ubuntu 14.04(x86_64)  
[Internet]--(VPN Client)-->[Router]--(192.168.0.0/24)-->[Linux(L2TP+StrongSwan)]

# インストール手順
## パッケージインストール
### OSの最新化
```
$ sudo apt-get -y update && sudo apt-get -y upgrade
$ sudo apt-get -y dust-upgrade && sudo apt-get -y autoremove
$ sudo shutdown -r now
```

### 
