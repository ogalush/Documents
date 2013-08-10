<!--
************************************************************
OpenStack GrizzyをCentOS6(x86_64)へインストールする手順
参照元: http://docs.openstack.org/grizzly/basic-install/yum/content/basic-install_controller.html
Copyright (c) Takehiko OGASAWARA 2013 All Rights Reserved.
************************************************************
-->
<div id='title'>　</div>    

# OpenStack概要
## GrizzlyをCentOSへインストールする方法

Reference From
http://docs.openstack.org/grizzly/basic-install/yum/content/basic-install_controller.html>

パッケージインストール  
```
yum -y install ntp
yum -y install mysql mysql-server MySQL-python
```
mysql設定更新
```
cp -p /etc/my.cnf /root/MAINTENANCE/20130810/bak/.
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/my.cnf
service mysqld start
```
OpenStack用DB作成
```
mysql -u root -p <<EOF
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
IDENTIFIED BY 'password';
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
IDENTIFIED BY 'password';
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
IDENTIFIED BY 'password';
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'password';
CREATE DATABASE quantum;
GRANT ALL PRIVILEGES ON quantum.* TO 'quantum'@'localhost' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON quantum.* TO 'quantum'@'10.0.2.15' \
IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
EOF
→パスワードが聞かれるが、空白のままEnterしてOK.
```

Install Queing Service(s)
```
yum -y install qpid-cpp-server
# echo auth=1 >> /etc/qpidd.conf
→1を入れない。qpidd.confにauth=YESが既に入っているため。
chkconfig qpidd on
service qpidd start
```

Openstackパッケージのインストール
```
yum -y install wget
cd /etc/yum.repos.d
rpm -i http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
→CentOSパッケージ用の拡張リポジトリを指定する。OpenStack-**をインストールできる。

```
