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
## 手順
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
node.session.auth.username(password)を設定する.  
node.session.auth.username_in(password_in)はTarget→Initiator方向の認証パラメータのため省略OK.  
```
$ sudo cp -v /etc/iscsi/iscsid.conf{,.def}
'/etc/iscsi/iscsid.conf' -> '/etc/iscsi/iscsid.conf.def'
$ sudo vim /etc/iscsi/iscsid.conf
$ sudo diff --unified=0 /etc/iscsi/iscsid.conf{.def,}
--- /etc/iscsi/iscsid.conf.def  2021-02-23 17:55:34.050254657 +0900
+++ /etc/iscsi/iscsid.conf      2021-02-23 18:26:21.507143301 +0900
@@ -58 +58 @@
-#node.session.auth.authmethod = CHAP
+node.session.auth.authmethod = CHAP
@@ -65 +65 @@
-#node.session.auth.chap_algs = SHA3-256,SHA256,SHA1,MD5
+node.session.auth.chap_algs = SHA3-256,SHA256,SHA1,MD5
@@ -69,2 +69,2 @@
-#node.session.auth.username = username
-#node.session.auth.password = password
+node.session.auth.username = admin
+node.session.auth.password = (パスワード文字列)

適用
$ sudo systemctl restart iscsid.service
$ sudo systemctl status iscsid.service |grep Active
   Active: active (running) since Tue 2021-02-23 18:00:03 JST; 23s ago
```
### iSCSI接続
#### initiator → target接続確認
```
$ sudo iscsiadm -m node -T iqn.2004-04.com.qnap:ts-231p:iscsi.qnapnas.22dcb3 -p 192.168.3.248 --login
Logging in to [iface: default, target: iqn.2004-04.com.qnap:ts-231p:iscsi.qnapnas.22dcb3, portal: 192.168.3.248,3260]
Login to [iface: default, target: iqn.2004-04.com.qnap:ts-231p:iscsi.qnapnas.22dcb3, portal: 192.168.3.248,3260] successful.
→ successfulと表示されているためOK.
```
#### ブロックデバイスとして認識したか確認
```
$ sudo lsblk --scsi
NAME HCTL       TYPE VENDOR   MODEL             REV TRAN
sda  2:0:0:0    disk QNAP     iSCSI Storage    4.0  iscsi
→ QNAP + iscsi表示できているためOK.
```
### フォーマット
#### Partition作成
```
[ogalush@hayao-test ~]$ sudo fdisk /dev/sda -l
Disk /dev/sda: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 2048 bytes / 2048 bytes
[ogalush@hayao-test ~]$
→ 20GBと認識している.

作成
$ sudo fdisk /dev/sda
Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x2205b7ed.

Command (m for help): p
Disk /dev/sda: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 2048 bytes / 2048 bytes
Disklabel type: dos
Disk identifier: 0x2205b7ed
→ 同じブロックデバイスを指定しているためOK.

Command (m for help): n
→ 新しいパーティション作成

Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
→ プライマリパーティション, 1番目を作成.

First sector (2048-41943039, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-41943039, default 41943039):
→ 指定無しで最大容量作成

Created a new partition 1 of type 'Linux' and of size 20 GiB.

Command (m for help): w
→ 反映

The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

[ogalush@hayao-test ~]$ ls -l /dev/sda*
brw-rw---- 1 root disk 8, 0 Feb 23 18:34 /dev/sda
brw-rw---- 1 root disk 8, 1 Feb 23 18:34 /dev/sda1
[ogalush@hayao-test ~]$
→ /dev/sda1が作成できたのでOK.
```
#### Format Partition
xfsでフォーマットしておく.
```
[ogalush@hayao-test ~]$ sudo mkfs -t xfs /dev/sda1
mkfs.xfs: Volume reports stripe unit of 2048 bytes and stripe width of 0, ignoring.
meta-data=/dev/sda1              isize=512    agcount=4, agsize=1310656 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=5242624, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[ogalush@hayao-test ~]$
```
#### Mount Partition
```
$ sudo mkdir -pv /mnt/iscsidrive
[ogalush@hayao-test ~]$ sudo mount -t xfs /dev/sda1 /mnt
$ sudo mount -t xfs /dev/sda1 /mnt/iscsidrive
$ df -h /mnt/iscsidrive
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        20G  175M   20G   1% /mnt/iscsidrive
```
→ マウントできればOK.

#### fstab設定 (適宜)
OS再起動後もマウントさせる場合は/etc/fstabへも設定する.

#### 接続情報の表示(参考)
セッション情報を表示するコマンド.
```
[ogalush@hayao-test ~]$ sudo iscsiadm -m session -P 3
iSCSI Transport Class version 2.0-870
version 6.2.1.2-0
Target: iqn.2004-04.com.qnap:ts-231p:iscsi.qnapnas.22dcb3 (non-flash)
        Current Portal: 192.168.3.248:3260,1
        Persistent Portal: 192.168.3.248:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.1994-05.com.redhat:408bbd707ab
                Iface IPaddress: 10.0.0.52
                Iface HWaddress: default
                Iface Netdev: default
                SID: 27
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 120
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: admin
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 2  State: running
                scsi2 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sda          State: running
$ 
```
## 性能測定
### Read性能
689.76Mbps  
(= 86.22MB * 8(Byte → bit))  
* NIC: 1Gbps  
* NAS: 6Gbps (SATA 6Gbps * 2本 RAID1)  

NICに依存しいるように見える.  
Neutron L3 agent(Router)を介しているためNIC速度よりも若干遅れる部分も妥当な印象.
```
iSCSI性能
$ sudo hdparm -Tt /dev/sda1

/dev/sda1:
 Timing cached reads:   19918 MB in  1.99 seconds = 10007.56 MB/sec・・・キャッシュ有
 Timing buffered disk reads: 260 MB in  3.02 seconds =  86.22 MB/sec・・・キャッシュ無し


SATA SSD性能（比較用）
$ sudo hdparm -Tt /dev/vda1

/dev/vda1:
 Timing cached reads:   19570 MB in  1.99 seconds = 9832.63 MB/sec・・・キャッシュ有
 Timing buffered disk reads: 2058 MB in  3.02 seconds = 682.34 MB/sec・・・キャッシュ無し
```
### Write性能
