# Ubuntu14.04のCPUスピードを調整する

## スピード制限解除
CPUの省電力機能によってCPUをフルパワーで利用できていない状況を改善する.  
(RedHat系だとcpuspeed, Ubuntu系だとcpufrequtils)  

CPU GOVERNERを変更することで改善.  
参考: [CPU GOVERNER](http://kledgeb.blogspot.jp/2013/06/ubuntu-cpufreq-1-cpufreq-cpufreq-cpufreq.html)

## 手順
```
$ sudo apt-get -y install cpufrequtils
$ cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
→ powersaveの時は、省エネモード. パフォーマンスを上げてやる必要がある.

$ BAK=/root/MAINTENANCE/20150607/bak
$ sudo mkdir -p $BAK
$ sudo cp /etc/init.d/cpufrequtils $BAK
$ sudo sed -ie 's/GOVERNOR="ondemand"/GOVERNOR="performance"/' /etc/init.d/cpufrequtils
$ grep GOVERNOR /etc/init.d/cpufrequtils
$ sudo service cpufrequtils restart
$ cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
→ performanceとなっていればOK.
$ sudo shutdown -r now
$ cat /proc/cpuinfo
---
...
model name      : Intel(R) Core(TM) i5-3475S CPU @ 2.90GHz
stepping        : 9
microcode       : 0x15
cpu MHz         : 3496.425
~~~負荷の高い時は、Intel ターボ・ブースト利用時の最大周波数となる.
cache size      : 6144 KB
...
---
```
