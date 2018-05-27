# Ubuntu 18.04でIPアドレスを設定する方法
## 概要
Ubuntu18.04では、IPアドレス設定をcloud-init.ymlへ記述するようになったのでメモする.

## 設定方法
### interface名確認
```
ogalush@ryunosuke:~$ ifconfig
enp3s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.200  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::82ee:73ff:fe63:f007  prefixlen 64  scopeid 0x20<link>
        inet6 ...(略)...  prefixlen 64  scopeid 0x0<global>
        ether 80:ee:73:63:f0:07  txqueuelen 1000  (Ethernet)
        RX packets 60242  bytes 89488196 (89.4 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 21741  bytes 2188354 (2.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
→ 設定する先のI/Fはこれ.

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 178  bytes 14526 (14.5 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 178  bytes 14526 (14.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ogalush@ryunosuke:~$
```

### cloud-init.yml編集
```
$ sudo vim /etc/netplan/50-cloud-init.yaml
----
ogalush@ryunosuke:~$ cat /etc/netplan/50-cloud-init.yaml
# This file is generated from information provided by
# the datasource.  Changes to it will not persist across an instance.
# To disable cloud-init's network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    ethernets:
        enp3s0:
            dhcp4: no
            dhcp6: true
            addresses: [192.168.0.200/24]
            gateway4: 192.168.0.254
            nameservers:
              addresses: [192.168.0.254, 8.8.8.8, 8.8.4.4]
    version: 2
ogalush@ryunosuke:~$
----
```

### 設定の適用
```
$ sudo netplan apply
```

### IPアドレスの確認
```
ogalush@ryunosuke:~$ ip addr show enp3s0
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 80:ee:73:63:f0:07 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.200/24 brd 192.168.0.255 scope global enp3s0
→ 意図したIPアドレスに変更できていればOK.
       valid_lft forever preferred_lft forever
    inet6 ...(略)...
       valid_lft 86264sec preferred_lft 14264sec
    inet6 fe80::82ee:73ff:fe63:f007/64 scope link 
       valid_lft forever preferred_lft forever
ogalush@ryunosuke:~$ 
```

## 参考
* [Ubuntu 18.04 LTSで固定IPアドレスの設定](https://qiita.com/zen3/items/757f96cbe522a9ad397d)
