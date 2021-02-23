# Vagrantを使ってみる.
[Vagrant](https://www.vagrantup.com/)  
VagrantはHashiCorpのツール.  
検証環境を簡単に作ることを目的にしている.  
VirtualBoxなどと組み合わせてVMを早く上げる.  
参考: [Vagrant + VirtualBoxでWindows上に開発環境をサクッと構築する](https://qiita.com/ozawan/items/160728f7c6b10c73b97e)

# 環境
* MacOS(10.13.6)

# 準備
## VirtualBoxダウンロード
VirtualBoxのインスタンスを起動させることで利用できるようにするためダウンロードしておく.
[VirtualBox](https://www.virtualbox.org/)  
→ 「Download VirtualBox6.1」からダウンロードする.  
ダウンロードしたイメージファイルからセットアップを進める.
## Vagrantダウンロード
https://www.vagrantup.com/  
→ 「Download 2.2.14」→ MacOSからダウンロードする.  
イメージを開くとセットアップが起動するので設定する.
```
$ vagrant --version
Vagrant 2.2.14
```

# 利用
## Box探し
[Discover Vagrant Boxes](https://app.vagrantup.com/boxes/search)を開く.  
開発環境として利用するOS環境（Box）を選択する.  
→ 「centos/7」を使用する事にする.

## Vagrantfile準備
検証環境の設定をするための設定ファイル(Vagrantfile)を設定する.
### Vagrantfile取得
```
$ mkdir -pv ~/Downloads/Vagrant
$ cd ~/Downloads/Vagrant

$ vagrant init centos/7
$ ls -l Vagrantfile 
-rw-r--r--  1 ogalush  staff  3014  2 23 20:17 Vagrantfile
→ ファイルがあるためOK.
```
### Vagrantfile編集
```
$ cp -v Vagrantfile{,.def}
Vagrantfile -> Vagrantfile.def
$ vim Vagrantfile
$ diff --unified=0 Vagrantfile{.def,}
--- Vagrantfile.def	2021-02-23 20:18:14.000000000 +0900
+++ Vagrantfile	2021-02-23 20:20:48.000000000 +0900
@@ -35,0 +36 @@
+  config.vm.network "private_network", ip: "192.168.33.10"
@@ -69,0 +71,3 @@
+  config.vm.provision "shell", inline: <<-SHELL
+    yum -y update
+  SHELL
$
```
→ 「config.vm.provision」にShellコマンドをいれてOS最新化する.  
Ansibleなどのコマンドラインも入れられるらしい.

### Box起動
```
$ vagrant up
```
→ VirtualBoxにOS環境が出来る.
ログ
```
h$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Box 'centos/7' could not be found. Attempting to find and install...
    default: Box Provider: virtualbox
    default: Box Version: >= 0
==> default: Loading metadata for box 'centos/7'
    default: URL: https://vagrantcloud.com/centos/7
==> default: Adding box 'centos/7' (v2004.01) for provider: virtualbox
    default: Downloading: https://vagrantcloud.com/centos/boxes/7/versions/2004.01/providers/virtualbox.box
Download redirected to host: cloud.centos.org
    default: Calculating and comparing box checksum...
==> default: Successfully added box 'centos/7' (v2004.01) for 'virtualbox'!
==> default: Importing base box 'centos/7'...
==> default: Matching MAC address for NAT networking...
==> default: Checking if box 'centos/7' version '2004.01' is up to date...
==> default: Setting the name of the VM: Vagrant_default_1614079554530_13169
...
==> default: Configuring and enabling network interfaces...
==> default: Rsyncing folder: /Users/ogalush/Downloads/Vagrant/ => /vagrant
==> default: Running provisioner: shell...
    default: Running: inline script
    default: Loaded plugins: fastestmirror
    default: Determining fastest mirrors
    default:  * base: ftp.nara.wide.ad.jp
→ Vagrantfileのyum updateが動いている.
...

$ vagrant box list
centos/7 (virtualbox, 2004.01)
→ 表示されればOK.
```
### Box利用
```
$ vagrant box list
centos/7 (virtualbox, 2004.01)
→ 表示されていること

h$ vagrant ssh
[vagrant@localhost ~]$ uname -n
localhost.localdomain
[vagrant@localhost ~]$ 
→ 接続できればOK

$ vagrant status
Current machine states:
default                   running (virtualbox)
The VM is running. ...(略)...
→ 「running」であればOK.
```
### Box停止
```
$ vagrant halt
==> default: Attempting graceful shutdown of VM...
```
### Box削除
Vagrant上のBoxは消えるが、VirtualBoxイメージ(VMDK)は残る.
```
$ vagrant box remove centos/7
Box 'centos/7' (v2004.01) with provider 'virtualbox' appears
to still be in use by at least one Vagrant environment. Removing
the box could corrupt the environment. We recommend destroying
these environments first:

default (ID: 97472edd3a5e45d1a30df4083eea721a)

Are you sure you want to remove this box? [y/N] y
Removing box 'centos/7' (v2004.01) with provider 'virtualbox'...
```
### Vagrant削除
VirtualBoxのイメージ(VMDK)も削除されるため注意する.
```
$ vagrant destroy
    default: Are you sure you want to destroy the 'default' VM? [y/N] y
==> default: Destroying VM and associated drives...
```
