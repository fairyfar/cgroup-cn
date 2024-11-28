[TOC]

# 一、Linux cgroup v2

## 1.1 默认启用cgroup v2的Linux发行版

* Fedora (since 31)
* Arch Linux (since April 2021)
* openSUSE Tumbleweed (since c. 2021)
* Debian GNU/Linux (since 11)
* Ubuntu (since 21.10)
* RHEL and RHEL-like distributions (since 9)

## 1.2 cgroup v2内核要求

启用cgroup v2需要以下两个基础条件：

* Linux最小内核版本为4.15，推荐5.2或更新。
* Linux最小systemd版本是239。

# 二、CentOS 8.5启用cgroup v2

如果操作系统为CentOS 8.5。

## 2.1 系统版本信息

```bash
[root@bogon ~]# cat /proc/version
Linux version 4.18.0-348.el8.x86_64 (mockbuild@kbuilder.bsys.centos.org) (gcc version 8.5.0 20210514 (Red Hat 8.5.0-3) (GCC)) #1 SMP Tue Oct 19 15:14:17 UTC 2021

[root@bogon ~]# cat /etc/os-release
NAME="CentOS Linux"
VERSION="8"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="8"
PLATFORM_ID="platform:el8"
PRETTY_NAME="CentOS Linux 8"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:8"
HOME_URL="https://centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"
CENTOS_MANTISBT_PROJECT="CentOS-8"
CENTOS_MANTISBT_PROJECT_VERSION="8"
```

## 2.2 cgroup第一版

`CentOS 8`默认是cgroup第一版。

先看一下当前cgroup配置情况（有省略）：

```bash
[root@bogon ~]# mount -l | grep cgroup
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_prio,net_cls)
……

[root@bogon ~]# ls -l /sys/fs/cgroup
total 0
drwxr-xr-x 6 root root  0 Jan  9 15:11 blkio
lrwxrwxrwx 1 root root 11 Jan  9 15:11 cpu -> cpu,cpuacct
lrwxrwxrwx 1 root root 11 Jan  9 15:11 cpuacct -> cpu,cpuacct
drwxr-xr-x 8 root root  0 Jan  9 15:11 cpu,cpuacct
drwxr-xr-x 5 root root  0 Jan  9 15:11 cpuset
drwxr-xr-x 5 root root  0 Jan  9 15:11 devices
drwxr-xr-x 2 root root  0 Jan  9 15:11 freezer
……
```

## 2.3 切换到cgroup v2

修改启动参数，并重启系统：

```bash
[root@bogon ~]# vim /etc/default/grub
GRUB_CMDLINE_LINUX添加"cgroup_no_v1=all systemd.unified_cgroup_hierarchy=1"

[root@bogon ~]# grub2-mkconfig -o /boot/grub2/grub.cfg
[root@bogon ~]# reboot
```

验证是否切换：

```bash
[root@bogon ~]# mount -l | grep cgroup2
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,seclabel,nsdelegate)

#或者
[root@bogon ~]# cat /proc/self/mounts | grep cgroup2
cgroup2 /sys/fs/cgroup cgroup2 rw,seclabel,nosuid,nodev,noexec,relatime,nsdelegate 0 0
```

查看挂载目录：

```bash
[root@bogon ~]# ls -l /sys/fs/cgroup
total 0
-r—-r—r--.  1 root root 0 Apr 29 12:03 cgroup.controllers
-rw-r—r--.  1 root root 0 Apr 29 12:03 cgroup.max.depth
-rw-r—r--.  1 root root 0 Apr 29 12:03 cgroup.max.descendants
-rw-r—r--.  1 root root 0 Apr 29 12:03 cgroup.procs
-r—-r—r--.  1 root root 0 Apr 29 12:03 cgroup.stat
-rw-r—r--.  1 root root 0 Apr 29 12:18 cgroup.subtree_control
-rw-r—r--.  1 root root 0 Apr 29 12:03 cgroup.threads
-rw-r—r--.  1 root root 0 Apr 29 12:03 cpu.pressure
……
```

## 2.4 配置

安装内核文档cgroup v2。

先查看支持的子系统：

```bash
[root@localhost cgroup]# pwd
/sys/fs/cgroup
[root@localhost cgroup]# cat cgroup.controllers
cpuset cpu io memory hugetlb pids rdma
```

默认情况下子级启用memory和pids子系统：

```bash
[root@localhost cgroup]# cat cgroup.subtree_control
memory pids
```

现在我们想启用cpu子系统：

```bash
[root@localhost cgroup]# echo "+cpu" > cgroup.subtree_control
-bash: echo: write error: Invalid argument
```

报错了。经查，是因为系统中存在实时模式的进程，需要先停止之。具体原因与解决方法请参考：

[《Redhat 8无法使用cgroup v2的CPU控制器》](Redhat%208无法使用cgroup%20v2的CPU控制器.md)

# 三、Ubuntu启用cgroup v2

配置：

```bash
# sudo cat /etc/default/grub
GRUB_CMDLINE_LINUX="cgroup_no_v1=all systemd.unified_cgroup_hierarchy=1 cgroup_cpu=1 cgroup_enable=cpu"

# sudo update-grub
```

重启，

```bash
# sudo reboot
```

重启后，

```bash
# sudo stat -fc %T /sys/fs/cgroup/
cgroup2fs
# cat /sys/fs/cgroup/cgroup.controllers
# cpuset io memory hugetlb pids rdma
```

缺少了cpu项，解决方法：见参考[5]。

# 四、参考

1. [Control Group v2](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
2. [The cpu controller in cgroup v2 can not be used in Red Hat Enterprise Linux 8](https://access.redhat.com/solutions/6582021)
3. [How to disable rtkit-daemon ?](https://access.redhat.com/solutions/737243)
4. [《Redhat 8无法使用cgroup v2的CPU控制器》](Redhat%208无法使用cgroup%20v2的CPU控制器.md)
5. [Enabling cpu cpuset and io delegation](https://rootlesscontaine.rs/getting-started/common/cgroup2/#enabling-cpu-cpuset-and-io-delegation)

