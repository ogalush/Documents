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


```
