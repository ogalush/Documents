# Install OpenStack Rocky on Ubuntu 16.04
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/)  
インストール先: 192.168.0.200(192.168.0.200)  
設定ファイル: [URL](URL)
```
ogalush@ryunosuke:~$ uname -a
Linux ryunosuke 4.4.0-135-generic #161-Ubuntu SMP Mon Aug 27 10:45:01 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
ogalush@ryunosuke:~$
```
## Networking
### Configure network interfaces
127.0.1.1 がホスト名に紐づいているので、実際のIPアドレスへ置換する.
```
$ sudo cp -pv /etc/hosts /tmp/hosts
'/etc/hosts' -> '/tmp/hosts'
$ sudo vim /etc/hosts
$ diff -u /tmp/hosts /etc/hosts |egrep '^(\+|\-)'
--- /tmp/hosts  2018-09-13 00:21:30.118757705 +0900
+++ /etc/hosts  2018-09-16 14:20:41.397700858 +0900
-127.0.1.1      ryunosuke
+192.168.0.200  ryunosuke
```
## Network Time Protocol (NTP)
時刻同期できているのでOK.
```
$ dpkg -l |grep ntp
ii  ntp 1:4.2.8p4+dfsg-3ubuntu5.9  amd64 Network Time Protocol daemon and utility programs
$ ntpq -p |grep '*'
*ntp-b2.nict.go. .NICT.           1 u   36   64  377    4.494   -0.679   1.021
```
## OpenStack packages for Ubuntu
```
$ sudo apt -y install software-properties-common
$ sudo add-apt-repository cloud-archive:rocky
'rocky': not a valid cloud-archive name.
Must be one of ['folsom', 'folsom-proposed', 'grizzly', 'grizzly-proposed', 'havana', 'havana-proposed', 'icehouse', 'icehouse-proposed', 'juno', 'juno-proposed', 'kilo', 'kilo-proposed', 'liberty', 'liberty-proposed', 'mitaka', 'mitaka-proposed', 'newton', 'newton-proposed', 'ocata', 'ocata-proposed', 'pike', 'pike-proposed', 'queens', 'queens-proposed', 'tools', 'tools-proposed']
→ Ubuntu16.04ではRockyをサポートしていないのでUbuntu18でのセットアップが必要.
```
