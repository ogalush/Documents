## Install OpenStack Mitaka on Ubuntu 16.04

ドキュメント: [OpenStack Docs](http://docs.openstack.org/mitaka/install-guide-ubuntu/environment.html)  
インストール先: ryunosuke(192.168.0.200)  
```
ogalush@ryunosuke:~$ uname -a
Linux ryunosuke 4.4.0-21-generic #37-Ubuntu SMP Mon Apr 18 18:33:37 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
```

## Environment
### ntp
```
$ ntpq -p
→ 時刻同期していればOK.
ogalush@ryunosuke:~$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 0.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 1.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 2.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 3.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp.ubuntu.com  .POOL.          16 p    -   64    0    0.000    0.000   0.000
+golem.canonical 192.150.70.56    2 u    6   64  177  235.278    1.376  58.327
+juniperberry.ca 140.203.204.77   2 u    8   64  177  250.272    9.587  58.079
+64.71.128.26 (1 216.218.254.202  2 u    4   64   77  110.667   -0.890  51.408
+y.ns.gin.ntt.ne 249.224.99.213   2 u    2   64   77    7.290   -3.315  49.621
+extendwings.com 10.84.87.146     2 u   72   64   76    8.171   -8.905  51.148
+x.ns.gin.ntt.ne 249.224.99.213   2 u   67   64   37    5.258  -12.825  50.265
*sv01.azsx.net   103.1.106.69     2 u    3   64   77    9.597   -2.385  49.716
+balthasar.gimas 181.170.187.161  3 u    4   64   77   68.825   30.177  77.336
+sv1.localdomain 133.243.238.163  2 u   67   64   37    5.650   -8.568  49.828
ogalush@ryunosuke:~$ 
```

### OpenStack packages
```
$ sudo apt-get -y install software-properties-common
$ sudo add-apt-repository cloud-archive:mitaka
 Ubuntu Cloud Archive for OpenStack Mitaka
 More info: https://wiki.ubuntu.com/ServerTeam/CloudArchive
Press [ENTER] to continue or ctrl-c to cancel adding it

cloud-archive for Mitaka only supported on trusty
~~~ このコマンドはubuntu16.04では使用できないらしい。mitaka使えるん？

$ sudo apt-get -y update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade
```

### OpenStack Client
```
$ sudo apt-get -y install python-openstackclient
```
