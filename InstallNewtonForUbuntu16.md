## Install OpenStack Newton on Ubuntu 16.04

ドキュメント: [OpenStack Docs](http://docs.openstack.org/newton/install-guide-ubuntu/)  
インストール先: ryunosuke(192.168.0.200)
```
ogalush@livaserver:~$ ssh -c aes128-ctr -A ogalush@192.168.0.200
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-47-generic x86_64)
ogalush@ryunosuke:~$ uname -a
Linux ryunosuke 4.4.0-47-generic #68-Ubuntu SMP Wed Oct 26 19:39:52 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
ogalush@ryunosuke:~$ 
```

## Environment
### Network Time Protocol (NTP)
ntpdがdefaultでinstallされているので対応なし
```
ogalush@ryunosuke:~$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 0.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 1.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 2.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 3.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp.ubuntu.com  .POOL.          16 p    -   64    0    0.000    0.000   0.000
+v157-7-235-92.z 103.1.106.69     2 u   47  128  377    4.260   -1.575   0.776
+li400-120.membe 157.7.208.12     3 u   37  128  377    4.392   -1.853   0.942
*extendwings.com 133.243.238.163  2 u   41  128  377    4.276   -2.192   1.347
-time.platformni 150.101.114.190  2 u  102  256  377    5.153   -5.328   1.619
-alphyn.canonica 132.246.11.231   2 u   44  128  377  196.992   -1.019   0.333
ogalush@ryunosuke:~$ 
```


### OpenStack packages
### SQL database
### Message queue
### Memcached

