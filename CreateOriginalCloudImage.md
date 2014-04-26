<!--
************************************************************
OpenStackのOSイメージを独自に作成する方法
参照元: http://docs.openstack.org/image-guide/content/ubuntu-image.html#d6e1013
        http://docs.openstack.org/image-guide/content/ch_openstack_images.html#support-resizing
        https://help.ubuntu.com/community/Installation/MinimalCD#A64-bit_PC_.28amd64.2C_x86_64.29
        http://docs.openstack.org/image-guide/content/ch_openstack_images.html
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->

# OpenStack用のOriginal OS Imageを作成する方法
## 環境
 OpenStack(Havana) + KVM

### インストールするOSイメージの準備
Download
```
$ wget http://archive.ubuntu.com/ubuntu/dists/precise/main/installer-amd64/current/images/netboot/mini.iso
 → 上記はUbuntu12.04(x86_64)
```

### パッケージインストール
```
$ sudo apt-get -y install virtinst
```

### 仮想ゲスト起動（ゴールデンイメージ用）
```
$ sudo su -
# qemu-img create -f qcow2 /home/ogalush/ubuntu_mini.qcow2 10G
# # virt-install --connect qemu:///system \
             --name test \
             --ram=1024 \
             --hvm \
             --virt-type=kvm \
             --location /home/ogalush/mini.iso \
             --os-type=Linux \
             --os-variant=virtio26 \
             --disk=/home/ogalush/ubuntu_mini.qcow2,format=qcow2 \
             --nographics \
             --network network:default \
             --accelerate \
             --keymap ja \
             --extra-args='console=tty0 console=ttyS0,115200n8'

→ Ubuntuのインストーラが起動する。
```

### 仮想ゲストのセットアップ
インストールのポイントは以下。
```
 Language: English
 地域: Asia Japan
 hostname: ubuntu
   → cloud-initで書き換えるのでこの値を使用する。
 Mirror site: デフォルト
   →  そのまま出てきたものを使用
 作成する一般ユーザ: ubuntu
   → cloud-initで書き換えるのでこの値を使用する。
 作成する一般ユーザのパスワード: ubuntu
   → パスワードは何でもOK
 HOMEディレクトリの暗号化: No
 パーティションの作成方法:   Manual
  → ■重要■
    設定内容は後述。
 Configuring discove: No automatic updates
  → OpenStackのインストールガイドではAutoUpdate推奨だが、ひとまずOff。(本質とは関係ないため)
 Software selection: OpenSSH Serverのみ有効
 Install the GRUB boot loader on a hard disk: YES
 Finish the installation  (Time UTC): YES
   →サーバの設定に合わせる。
 Finish the installation (再起動): Continue
   →インストールが終了する。
```

### パーティションの作成方法
仮想ゲストでOSをインストールする時に、パーティションを1パーティションにする必要がある。
(cloud-init-ramfs...が書換えることが出来るのがsingleパーティションのみのため)
```
Partition disks
 → Manual -> 「Virtual disk 1 (vda) - 10.7 GB Virtio Block Device」を開く
 ※cloud-initでresizefsできるよう、1パーティションとする。
 ※ /bootも含めた1パーティションとする。
 ※ SWAP領域は作成しない。
 Create new empty partition table on this device?: YES
 ---
 Virtual disk 1 (vda) - 10.7 GB Virtio Block Device
  │  >   pri/log  10.7 GB FREE SPACE
  ~~~開く
 ---
 Create a new partition -> New Partition sizeは、10.7GB(100%)とする。
 パーティションの種類: Primary(Pertition)
 ---
  Use as:  Ext4 journaling file system
  Mount point: /
  Mount options: defaults
  Label: none
  Reserved blocks:  5%
  Typical usage: standard
  Bootable flag: on
 ---

 「 Done setting up the partition」で設定を終了する。
 「 Finish partitioning and write changes to disk」で書込む。
  → その後、swapを作るか？(Do you want to return to the partitioning menu?)
    と表示されるので、Noとする。
  → Write the changes to disks? : YES

パーティションが作成される。
```

### 仮想ゲストのセットアップ
仮想ゲストへログインできるようになるので、必要なパッケージを入れる。

#### OSアップデート
```
$ sudo apt-get -y update
$ sudo apt-get -y upgrade
```

#### Cloud イメージ用のインストール
ディスクのresizeやホスト名変更に使用するモジュールを入れる。
```
$ sudo apt-get -y install cloud-init
$ sudo dpkg-reconfigure cloud-init
 → EC2にチェックを入れる。OpenStackはEC2互換形式で設定する。
 → cloud-initを入れることで、インスタンス作成時にhostname変更をしてくれる。

 $ sudo apt-get -y install cloud-initramfs-growroot
  → 仮想ゲストのDiskサイズをresizeしてくれる。

 $ sudo apt-get -y install cloud-initramfs-rescuevol
  → いざというときに役立ちそう。 " boot off a rescue volume rather than root filesystem"
```

#### 仮想ゲスト設定
仮想ゲスト間で共通する設定を入れる
```
$ sudo adduser ogalush
$ sudo adduser tendo
※その他sshキー(authorized_keys)等を入れる
```

#### 仮想ゲスト終了
仮想ゲストのOSイメージをexportするため、shutdownする。
```
$ sudo sync;sync;sync; shutdown -h now
```

### 仮想ゲストのOSイメージ編集
#### CD-ROM確認
```
# virsh dumpxml test
----
 <disk type='block' device='cdrom'>
  <driver name='qemu' type='raw'/>
  <target dev='hdc' bus='ide'/>
  <readonly/>
  <address type='drive' controller='0' bus='1' target='0' unit='0'/>
   ~~~★この状態であればOK
</disk>
----

※ cdromにISOがマウントされている場合は、以下でunmountする。
 # virsh start test --paused
 # virsh attach-disk --type cdrom --mode readonly test "" hdc
 # virsh resume test
 # virsh console test
```

#### 仮想ゲストのMACアドレス消去(logなども)
```
# apt-get -y install libguestfs-tools guestmount
# virt-sysprep -d test
---
 W: kvm binary is deprecated, please use qemu-system-x86_64 instead
 W: kvm binary is deprecated, please use qemu-system-x86_64 instead
---
上記の表示で終わればOK

→ virt-sysprepを実行することで、MACアドレスの固有設定が消える。
  消去する情報(参考): http://www.slideshare.net/moriwaka/virtsysprep
```

### 仮想イメージの圧縮
OSイメージは、そのままだと大容量のため、整理して圧縮する。
```
# qemu-img convert -c /home/ogalush/ubuntu_mini.qcow2 -O qcow2 /home/ogalush/ubuntu_compress.qcow2
→ 1.8GB--> 633MBへ圧縮された。
```

### OpenStackへ登録
```
 # glance image-create --is-public true --disk-format qcow2 --container-format bare --name "Ubuntu12.04_oga" < /home/ogalush/ubuntu_compress.qcow2
```

### OpenStackからインスタンス起動
Horizonからインスタンスを起動する。
```
イメージ選択欄から、作成したオリジナルイメージを選択する。
★k起動したインスタンスで以下を確認できればOK
 (1) ホスト名が変わっていること。
 (2) df -h の/の容量が、インスタンス作成時の容量(20GBなど)となること。
 (3) 外部へ通信できること。apt-get -y updateなど。
 (4) sshログインできること。 ip netns ...か、floatingIPを付けた後、外部からインスタンスへsshする。
```
 
### 仮想ゲストの削除 ※実施しない(ゴールデンイメージが不要な場合に実施)
qemu上から仮想ゲストを削除する。
```
# virsh undefine test
→ /home/ogalush/ubuntu_mini.qcow2は残るが、virshからOSイメージが消える。
★ゴールデンイメージで残すなら、実行しなくて良い？
```
