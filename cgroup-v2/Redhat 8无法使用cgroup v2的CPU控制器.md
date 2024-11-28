翻译文档。

原文：[The cpu controller in cgroup v2 can not be used in Red Hat Enterprise Linux 8](https://access.redhat.com/solutions/6582021)

标题：Red Hat Enterprise Linux 8无法使用cgroup v2的CPU控制器。

# 一、环境

* Red Hat Enterprise Linux 8

# 二、问题

*  `Red Hat Enterprise Linux 8`系统使用`systemd.unified_cgroup_hierarchy=1`启动参数启用了 `cgroup` v2，并希望将一些控制器委托给非特权用户。

* 除了`cpu`控制器，其它都能正常工作。

* 无法手动添加`cpu`控制器。

* 当启用CPU控制器时，报以下错误：

  ```bash
  # echo +cpu > /sys/fs/cgroup/cgroup.subtree_control
  -bash: echo: write error: Invalid argument
  ```

# 三、解决方法

* 如果系统中有实时（`Real Time`）非内核线程进程，`cgroups V2`的CPU控制器将无法挂载。
* 想要使用CPU控制器，可以停止使用`Real Time`调度策略的应用程序，也可以手动将这些进程迁移到根组。

# 四、原因

* 有些应用程序以实时优先级运行，可能会触发此问题。如果不需要，可以禁用这些应用程序。下面是一个非详尽的此类应用程序列表：
  * `rtkit-daemon` ：这篇文章说明了如何禁用之： [How to disable rtkit-daemon ?](https://access.redhat.com/solutions/737243)。
  * `Oracle RAC`
  * `Red Hat Cluster`

# 五、诊断步骤

* 检查是否有实时调度进程正在运行：

```bash
r8 # chrt -r 40 sleep 5000 &  # example sleep process running in real time
r8 # ps axo pid,cls,cmd | awk '$2 ~ /(FF|RR)/' | grep -v '[[]'
 1831  RR sleep 5000

r8 # echo +cpu > cgroup.subtree_control 
-bash: echo: write error: Invalid argument
```

* 或者，按照下面的命令查找调度类为`RR`(`SCHED_RR`)的进程。
* 此示例中，`timekeeper`使用实时调度策略(`RR`)运行。为了使用cpu控制器，可以停止该进程，或者将其迁移至根组。

```bash
$ ps -T axo pid,ppid,user,group,lwp,nlwp,start_time,comm,cgroup,cls|grep RR
1895    1870 root     root    1936   15 Jan09 tk_status_file  0::/system.slice/timekeeper  RR

$ ps -fe | grep 1895
root        1895    1870  0 Jan09 ?        00:02:42 /opt/timekeeper/release64/timekeeperapp

# 杀掉进程
$ kill 1895
# 或者迁移进程至根组：
$ echo 1895 > /sys/fs/cgroup/cgroup.procs
```