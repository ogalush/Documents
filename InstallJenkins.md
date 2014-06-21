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
sudo apt-get -y install git
```

### アクセス
```
http://192.168.0.103:8080/
→ Jenkinsおじさんが見えればOK
```

### github plugin
```
Jenkinsの管理 -> Pluginの管理 -> GIT Plugin を入れる
```

github登録
```
sudo -u jenkins -H git config --global user.email "take.oga@gmail.com"
sudo -u jenkins -H git config --global user.name "ogalush"
sudo -u jenkins -H ssh-keygen -t rsa -C jenkins@jenkins.local
```

gitリポジトリ登録
```
githubから New Repositryを行い、新規プロジェクトを作成する
-->  https://github.com/ogalush/python-test.git

githubへ、JenkinsのSSH鍵を取得してgithubへ登録する
--> github(home) -> Edit Your Profile -> sshKeys -> Add SSH Keyで登録する
--> 登録する鍵: cat /usr/lib/jenkins/.ssh/id_rsa.pub
```

Jenkins プロジェクト作成
```
プロジェクト作成 -> 設定 -> github url, user, sshkeyを登録する。
```

