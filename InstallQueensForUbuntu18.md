# Install OpenStack Queens on Ubuntu 18.04
ドキュメント: [OpenStack Docs](https://docs.openstack.org/install-guide/openstack-services.html)  
インストール先: 192.168.0.200(192.168.0.200)  
設定ファイル: [Queens-InstallConfigs](https://github.com/ogalush/Queens-InstallConfigs)
```
ogalush@ryunosuke:~$ uname -na
Linux ryunosuke 4.15.0-29-generic #31-Ubuntu SMP Tue Jul 17 15:39:52 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
ogalush@ryunosuke:~$
```

## Environment
### Network Time Protocol (NTP)
```
$ ntpq -p |grep '*'
*li1546-25.membe 10.84.87.146     2 u   44   64   17    4.045   -2.458   2.782
```
### NIC Settings
```
$ cat /etc/netplan/01-netcfg.yaml 
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
$
```
