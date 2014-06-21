<!--
************************************************************
Jenkins インストール
参照元: http://ameblo.jp/smeokano/entry-11817079759.html
Copyright (c) Takehiko OGASAWARA 2014 All Rights Reserved.
************************************************************
-->

# Jenkins インストール

### 準備
バックアップディレクトリ作成
```
mkdir -p /root/MAINTENANCE/`date "+%Y%m%d"`/{bak,new}
BAK=/root/MAINTENANCE/`date "+%Y%m%d"`/bak
```

パッケージインストール  
```
sudo apt-get -y update && apt-get -y upgrade
sudo apt-get -y dist-upgrade
~~~カーネルバージョンアップ
```

sourcelist 追加
```
wget -q -O - http://pkg.jenkins-ci.org/debian/jenkins-ci.org.key | sudo apt-key add -
cp -raf /etc/apt $BAK
echo 'deb http://pkg.jenkins-ci.org/debian binary/' >> /etc/apt/sources.list
```

### Jenkinsインストール
```
sudo apt-get -y update && sudo apt-get -y install jenkins
```




