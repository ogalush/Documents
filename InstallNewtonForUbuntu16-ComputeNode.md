## Install OpenStack Newton on Ubuntu 16.04(ComputeNode)

ドキュメント: [OpenStack Docs](http://docs.openstack.org/newton/install-guide-ubuntu/)  
インストール先: hayao(192.168.0.210)
```
ogalush@livaserver:~$ ssh -c aes128-ctr -A ogalush@192.168.0.210
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-47-generic x86_64)
ogalush@hayao:~$ uname -a
Linux hayao 4.4.0-47-generic #68-Ubuntu SMP Wed Oct 26 19:39:52 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
ogalush@hayao:~$ 
```

## Environment
### Network Time Protocol (NTP)
時刻同期しているので新たなインストールは不要。
```
ogalush@hayao:~$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 0.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 1.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 2.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 3.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp.ubuntu.com  .POOL.          16 p    -   64    0    0.000    0.000   0.000
+chilipepper.can 17.253.34.125    2 u  327 1024  357  241.069    7.374   4.651
-golem.canonical 17.253.34.253    2 u  788 1024  377  249.159    2.955   1.296
+alphyn.canonica 17.253.34.125    2 u  754 1024  377  204.071   -4.922   6.715
-juniperberry.ca 17.253.34.253    2 u  829 1024  377  237.363    9.041  10.556
*laika.paina.jp  131.113.192.40   2 u  917 1024  377    4.104   -1.678   4.743
```

### OpenStack packages
```
$ sudo apt -y install software-properties-common
$ sudo add-apt-repository cloud-archive:newton
$ sudo apt -y update && sudo apt -y dist-upgrade
$ sudo apt -y install python-openstackclient
```

## Compute service
### Install and configure a compute node
#### Install and configure components
#### Finalize installation
### Verify operation
