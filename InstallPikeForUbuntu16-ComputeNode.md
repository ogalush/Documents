## Install OpenStack Pike on Ubuntu 16.04(ComputeNode)

ドキュメント: [OpenStack Docs](https://docs.openstack.org/nova/pike/install/)  
インストール先: hayao(192.168.0.210)  
Repo: https://github.com/ogalush/Pike-InstallConfigs
```
ogalush@hayao:~$ uname -a
Linux hayao 4.4.0-112-generic #135-Ubuntu SMP Fri Jan 19 11:48:36 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
ogalush@hayao:~$
```

## Environment
### Network Time Protocol (NTP)
時刻同期しているので新たなインストールは不要。
```
ogalush@hayao:~$ ntpq -p |grep '*'
*li461-162.membe 103.1.106.69     2 u   78  128  377    4.188   -0.835   1.923
ogalush@hayao:~$ 
```
### Register Pike Repositry
```
$ sudo add-apt-repository cloud-archive:pike
```

### OS Update
```
$ sudo apt-get -y update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade
```
