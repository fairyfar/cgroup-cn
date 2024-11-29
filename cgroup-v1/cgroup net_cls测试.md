[TOC]

# 一、net_cls

net\_cls 子系统使用等级识别符（classid）标记网络数据包，这让 Linux 流量管控器（tc）可以识别从特定 cgroup 中生成的数据包。可配置流量管控器，让其为不同 cgroup 中的数据包设定不同的优先级。

## 1.1 net_cls.classid

net\_cls.classid 包含表示流量控制 handle 的单一数值。从 net\_cls.classid 文件中读取的classid 值是十进制格式，但写入该文件的值则为十六进制格式。例如：0x100001 表示控制点通常写为 iproute2 所用的 10:1 格式。在 net\_cls.classid 文件中，将以数字 1048577 表示。这些控制点的格式为：0xAAAABBBB，其中 AAAA 是十六进制主设备号，BBBB 是十六进制副设备号。您可以忽略前面的零；0x10001 与 0x00010001 一样，代表 1:1。以下是在net\_cls.classid 文件中设定 10:1 控制点的示例：

```bash
cd /sys/fs/cgroup/net_cls/
echo 0x100001 > net_cls.classid
cat net_cls.classid
1048577
```

# 二、实例

## 2.1 未限流

先准备一个大文件：

```bash
[yz@bogon ~]$ ll -h big.txt
-rw-rw-r-- 1 yz yz 9.3G 11月 14 13:25 big.txt
```

在未限流的情况下执行scp，可以看到速度约70MB/s：

```bash
[yz@bogon ~]$ scp ./big.txt 192.168.4.219:/home/yz/big1.log
big.txt         16% 1529MB  69.9MB/s   01:54 ETA^
```

## 2.2 限流

准备cgroup组：

```bash
[yz@bogon ~]$ cd /sys/fs/cgroup/net_cls/
[yz@bogon net_cls]$ sudo mkdir yz    # 需要root权限
[yz@bogon net_cls]$ sudo chown yz:yz -R ./yz
[yz@bogon net_cls]$ cd yz
[yz@bogon yz]$ echo 0x100001 > net_cls.classid
[yz@bogon yz]$ cat net_cls.classid
1048577
```

## 2.3 tc配置

```bash
[yz@bogon yz]$ sudo tc qdisc add dev enp0s8 root handle 10: htb    # 需要root权限
[yz@bogon yz]$ tc -s qdisc show dev enp0s8    # 显示
qdisc htb 10: root refcnt 2 r2q 10 default 0 direct_packets_stat 55 direct_qlen 1000
 Sent 6350 bytes 55 pkt (dropped 0, overlimits 0 requeues 0)
 backlog 0b 0p requeues 0

[yz@bogon yz]$ sudo tc class add dev enp0s8 parent 10: classid 10:1 htb rate 10mbit    # 需要root权限
[yz@bogon yz]$ tc -s class show dev enp0s8    # 显示
class htb 10:1 root prio 0 rate 10Mbit ceil 10Mbit burst 1600b cburst 1600b
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0)
 backlog 0b 0p requeues 0
 lended: 0 borrowed: 0 giants: 0
 tokens: 20000 ctokens: 20000

[yz@bogon yz]$ sudo tc filter add dev enp0s8 parent 10: protocol ip prio 10 handle 1: cgroup    # 需要root权限
[yz@bogon yz]$ tc -s filter show dev enp0s8    # 显示
filter parent 10: protocol ip pref 10 cgroup
filter parent 10: protocol ip pref 10 cgroup handle 0x1
```

注意：tc的add命令需要root权限。

## 2.4 scp限流

```bash
[yz@bogon ~]$ cgexec -g net_cls:group scp ./big.txt 192.168.4.219:/home/yz/big1.log
big.txt         1% 1529MB  1.1MB/s   02:55 ETA^
```

# 四、参考

* [Cgroup子系统 net_cls文档](https://guanjunjian.github.io/2017/11/28/study-12-cgroup-net_cls-txt/)