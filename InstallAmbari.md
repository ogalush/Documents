## ambari-setup 2016.6.4
https://cwiki.apache.org/confluence/display/AMBARI/Installation+Guide+for+Ambari+2.2.2

### ホスト名設定
```
Ambari-server起動時にJDBC接続エラーになるため、追記する。
$ sudo vi /etc/hosts
----
10.0.0.43 hadoop-1.localdomain
10.0.0.45 hadoop-2.localdomain
10.0.0.44 hadoop-3.localdomain
10.0.0.46 hadoop-4.localdomain
----

$ sudo vi /etc/hostname
---
hadoop-1.localdomain
---
$ sudo shutdown -r now
```

### リポジトリ設定
```
$ cd /etc/apt/sources.list.d
$ sudo wget http://public-repo-1.hortonworks.com/ambari/ubuntu14/2.x/updates/2.2.2.0/ambari.list
```

### Ambari-Serverインストール
```
・最新化
$ sudo apt-key adv --recv-keys --keyserver keyserver.ubuntu.com B9733A7A07513CAD
$ sudo apt-get -y update
$ sudo apt-get install -y ambari-server
```

### Ambari-Server 設定
```
$ sudo ambari-server setup
Customize user account for ambari-server daemon [y/n] (n)? n
Adjusting ambari-server permissions and ownership...
Checking firewall status...
Checking JDK...
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Oracle JDK 1.7 + Java Cryptography Extension (JCE) Policy Files 7
[3] Custom JDK
==============================================================================
Enter choice (1): 1

Downloading JCE Policy archive from http://public-repo-1.hortonworks.com/ARTIFACTS/jce_policy-8.zip to /var/lib/ambari-server/resources/jce_policy-8.zip

Successfully downloaded JCE Policy archive to /var/lib/ambari-server/resources/jce_policy-8.zip
Installing JCE policy...
Completing setup...
Configuring database...
Enter advanced database configuration [y/n] (n)? n

$ sudo ambari-server start

http://192.168.0.232:8080/
admin/admin
```
