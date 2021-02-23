# iSCSI initatorを設定する.
QNAP NASで利用可能なiSCSI機能を試してみる.  
参考ページ
* [LinuxからQNAPをiSCSIでマウントする手順](https://agora.sanei-hy.co.jp/technology/2019/05/00076/)
* (Open-iSCSI)(https://wiki.archlinux.jp/index.php/Open-iSCSI)
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
→ CHAP認証を入れる.
|項目|値|
|---|---|
|alias|qnapnas|
|iqn|iqn.2004-04.com.qnap:ts-231p:iscsi.qnapnas.22dcb3|
|CHAP認証|Enabled|
|ユーザ名|admin|
|パスワード|...(12~16Chars)...|

### iSCSI initiator準備
#### install Packages
```
$ sudo dnf -q -y install iscsi-initiator-utils
[sudo] password for ogalush: 
Installed:
  iscsi-initiator-utils-6.2.1.2-0.gita8fcb37.el8.x86_64                                                         
  iscsi-initiator-utils-iscsiuio-6.2.1.2-0.gita8fcb37.el8.x86_64                                                
  isns-utils-libs-0.99-1.el8.x86_64                                                                             
$
```
#### Enable & Active iscsid
systemctlで有効化、起動する.
```
$ sudo systemctl status iscsid.service
● iscsid.service - Open-iSCSI
   Loaded: loaded (/usr/lib/systemd/system/iscsid.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
     Docs: man:iscsid(8)
           man:iscsiuio(8)
           man:iscsiadm(8)

$ sudo systemctl start iscsid.service
$ sudo systemctl is-active iscsid.service
active
→ OK.

$ sudo systemctl enable iscsid.service
Created symlink /etc/systemd/system/multi-user.target.wants/iscsid.service → /usr/lib/systemd/system/iscsid.service.
$ sudo systemctl is-enabled iscsid.service
enabled
→ OK
```
#### Find iSCSI Target(s)
iscsiadmでTargetを探す.
```
$ sudo iscsiadm -m discovery -t st -p 192.168.3.248
192.168.3.248:3260,1 iqn.2004-04.com.qnap:ts-231p:iscsi.qnapnas.22dcb3
→ 見つかるのでOK.
```
#### Set Authentication File
iscsi.confを編集して認証情報を設定する.  
CHAP認証をするためauthmethodを設定.  
node.session.auth.username_in(password_in)を設定する.  
node.session.auth.username(password)はTarget側のパラメータのため変更しなくてOK.  
```
$ sudo cp -v /etc/iscsi/iscsid.conf{,.def}
'/etc/iscsi/iscsid.conf' -> '/etc/iscsi/iscsid.conf.def'
$ sudo vim /etc/iscsi/iscsid.conf

$ sudo diff --unified=0 /etc/iscsi/iscsid.conf{.def,}
--- /etc/iscsi/iscsid.conf.def  2021-02-23 17:55:34.050254657 +0900
+++ /etc/iscsi/iscsid.conf      2021-02-23 18:05:00.515715318 +0900
@@ -58 +58 @@
-#node.session.auth.authmethod = CHAP
+node.session.auth.authmethod = CHAP
@@ -74,2 +74,2 @@
-#node.session.auth.username_in = username_in
-#node.session.auth.password_in = password_in
+node.session.auth.username_in = admin
+node.session.auth.password_in = passwordpassword
[ogalush@hayao-test ~]$

適用
$ sudo systemctl restart iscsid.service
$ sudo systemctl status iscsid.service |grep Active
   Active: active (running) since Tue 2021-02-23 18:00:03 JST; 23s ago
```
#### Connect iSCSI Target
```
```
