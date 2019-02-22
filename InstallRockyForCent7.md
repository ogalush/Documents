# Install OpenStack Rocky for CentOS7
[OpenStack Installation Guide](https://docs.openstack.org/install-guide/)  
Ubuntu18.04版がSegmentation Failtエラーが頻繁に出て不安定なため、CentOS7で構築をしてみる.
## 環境
```
[ogalush@ryunosuke ~]$ uname -n
ryunosuke.localdomain
[ogalush@ryunosuke ~]$ cat /etc/redhat-release 
CentOS Linux release 7.6.1810 (Core) 
[ogalush@ryunosuke ~]$ uname -a
Linux ryunosuke.localdomain 3.10.0-957.el7.x86_64 #1 SMP Thu Nov 8 23:39:32 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
[ogalush@ryunosuke ~]$ ip addr show |grep 192.168.0.200
    inet 192.168.0.200/24 brd 192.168.0.255 scope global noprefixroute enp3s0
[ogalush@ryunosuke ~]$ 
```
