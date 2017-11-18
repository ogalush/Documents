# Ubuntu 自動インストールDiskの作成方法
## 概要
OSバージョン: ubuntu 16.04.3 + macOS High Sierra(10.13.1)  
参考: http://sig9.hatenablog.com/entry/2016/09/10/120000

## 準備
### パッケージインストール
```
$ sudo apt-get -y install syslinux mtools mbr genisoimage dvd+rw-tools
```

### 編集用ディレクトリ作成
```
$ mkdir -pv {dvd,dvdr}
```

## ISOイメージの読込みと編集
### ISOを編集用ディレクトリへコピー
```
$ sudo mount -t iso9660 ubuntu-16.04.3-server-amd64.iso dvd
$ cd dvd
$ find . ! -type l | cpio -pdum ../dvdr/
$ cd -
$ sudo umount dvd
```

### isolinux.cfg の書換え
```
$ sudo rm -v dvdr/isolinux/isolinux.cfg
$ sudo vim dvdr/isolinux/isolinux.cfg
----
default install
label install
  menu label ^Install Ubuntu Server
  kernel /install/vmlinuz
  append DEBCONF_DEBUG=5 auto=true locale=en_US.UTF-8 console-setup/charmap=UTF-8 console-setup/layoutcode=us console-setup/ask_detect=false pkgsel/language-pack-patterns=pkgsel/install-language-support=false vga=normal file=/cdrom/preseed/preseed.cfg initrd=/install/initrd.gz quiet --
label hd
   menu label ^Boot from first hard disk
   localboot 0x80
----
```

### preeseed.cfg の作成
Ubuntuの自動インストール用Configファイルを作成する.
```
$ sudo vim dvdr/preseed/preseed.cfg
----
# Localization
d-i debian-installer/locale string en_US

# Keyboard selection.
d-i console-keymaps-at/keymap select jp
d-i console-setup/ask_detect boolean false
d-i keyboard-configuration/layoutcode string jp
d-i keyboard-configuration/modelcode jp106

# Network configuration
d-i netcfg/choose_interface select auto
d-i netcfg/disable_autoconfig boolean false
d-i netcfg/get_domain string localdomain
d-i netcfg/get_hostname string localhost

# Mirror settings
d-i mirror/country string manual
d-i mirror/http/hostname string archive.ubuntu.com
d-i mirror/http/directory string /ubuntu
d-i mirror/http/proxy string

# Clock and time zone setup
d-i clock-setup/utc boolean true
d-i time/zone string Asia/Tokyo
d-i clock-setup/ntp boolean true
d-i clock-setup/ntp-server string 0.pool.ntp.org

# Partitioning
d-i partman-auto/disk string /dev/sda
d-i partman-auto/method string regular
d-i partman-auto/choose_recipe select atomic
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true

# Base system installation
d-i base-installer/kernel/image string linux-server

# Account setup
d-i passwd/root-login boolean true
d-i passwd/make-user boolean true
d-i passwd/root-password password password
d-i passwd/root-password-again password password

# To create a normal user account.
d-i passwd/user-fullname string ogalush
d-i passwd/username string ogalush
d-i passwd/user-password password password
d-i passwd/user-password-again password password
d-i user-setup/allow-password-weak boolean true
d-i passwd/user-default-groups string sudo
d-i user-setup/encrypt-home boolean false

# Package selection
tasksel tasksel/first multiselect none
d-i pkgsel/include string openssh-server
d-i pkgsel/upgrade select none
d-i pkgsel/update-policy select none
popularity-contest popularity-contest/participate boolean false
d-i pkgsel/updatedb boolean true

# Boot loader installation
d-i grub-installer/only_debian boolean true

# Finishing up the installation
d-i finish-install/reboot_in_progress note

# This first command is run as early as possible, just after
# preseeding is read.
d-i preseed/late_command string apt-install axel bash-completion chrony curl dnsutils fping git httpie httping jq libxml2-utils lsof man nmap open-vm-tools psmisc sshpass tcpdump screen tree unzip vim-nox wget zip zsh; in-target apt-get -y update; in-target apt-get -y upgrade; in-target apt-get -y clean;
----
```

### .iso の保存
```
$ sudo genisoimage -N -J -R -D -V "PRESEED" -o ubuntu-16.04.3-server-amd64.proceed.iso -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table dvdr
```

## 編集したISOをUSBメモリへ出力する.
macOSでiso→usbメモリへ出力する.
参考: http://marmotte.pyrites.jp/blog/2015/01/04/create-bootable-usb-from-iso-file/

### Disk確認
```
MacBook1:Downloads ogalush$ diskutil list
----
...(略)...
/dev/disk2 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *1.0 GB     disk2
   1:                 DOS_FAT_32 USB                     1.0 GB     disk2s1
```

### USBメモリを初期化
```
$ diskutil eraseDisk MS-DOS UNTITLED /dev/disk2
---
Started erase on disk2
Unmounting disk
Creating the partition map
Waiting for partitions to activate
Formatting disk2s1 as MS-DOS (FAT) with name UNTITLED
512 bytes per physical sector
/dev/rdisk2s1: 2023528 sectors in 252941 FAT32 clusters (4096 bytes/cluster)
bps=512 spc=8 res=32 nft=2 mid=0xf8 spt=32 hds=128 hid=2048 drv=0x80 bsec=2027520 bspf=1977 rdcl=2 infs=1 bkbs=6
Mounting disk
Finished erase on disk2
---

$ diskutil unmountDisk /dev/disk2
Unmount of all volumes on disk2 was successful
```

### USBディスクへ書込み
```
$ sudo dd if=ubuntu-16.04.3-server-amd64.proceed.iso of=/dev/disk2 bs=4028
```

### USBディスクの取外し
```
$ diskutil eject /dev/disk2
```

あとはUbuntuインストール機でUSBディスクで起動させてインストールさせる.
