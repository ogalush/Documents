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
## 
