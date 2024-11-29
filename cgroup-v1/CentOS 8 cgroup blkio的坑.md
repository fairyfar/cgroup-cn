[TOC]

# 一、问题

已知在CentOS 8、openEuler 20.03~23.03版本操作系统上，cgroup v1的blkio和device存在bug：blkio和device子系统下的用户层级在某些情况下会被系统删除。

已经有人给Redhat报bug了：

* [daemon-reload or daemon-reexec obliterates custom subdirectories within blkio controller](https://bugzilla.redhat.com/show_bug.cgi?id=2174599)
* [systemctl daemon-reload removes cgroups under devices and blkio](https://access.redhat.com/discussions/6999791)

截止到目前（2023.08.10）尚无答复。

与systemctl相关的命令可能会触发以上问题，例如：`systemctl daemon-reload`、`systemctl enable`等。

# 二、复现用例

在openEuler 23.03系统上复现：

```bash
# mkdir /sys/fs/cgroup/blkio/yz
# mkdir /sys/fs/cgroup/cpu/yz

# systemctl daemon-reload

# ls /sys/fs/cgroup/blkio/yz
ls: cannot access '/sys/fs/cgroup/blkio/yz': No such file or directory
# ls /sys/fs/cgroup/cpu/yz
……
```

# 2.1 trace

trace跟踪系统systemd进程删除blkio子目录的记录：

```
1     1691148727.037709 newfstatat(AT_FDCWD, "/sys/fs/cgroup/blkio", {st_mode=S_IFDIR|0555, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000007>
1     1691148727.037737 openat(AT_FDCWD, "/sys/fs/cgroup/blkio", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 22 <0.000017>
1     1691148727.037797 newfstatat(22, "", {st_mode=S_IFDIR|0555, st_size=0, ...}, AT_EMPTY_PATH) = 0 <0.000009>
1     1691148727.037842 getdents64(22, 0x563751ce3a50 /* 22 entries */, 32768) = 1000 <0.000015>
1     1691148727.037889 newfstatat(22, "blkio.bfq.io_service_bytes", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000010>
1     1691148727.037931 newfstatat(22, "cgroup.procs", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000038>
1     1691148727.037997 newfstatat(22, "blkio.bfq.io_serviced", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000006>
1     1691148727.038026 newfstatat(22, "blkio.throttle.read_iops_device", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000006>
1     1691148727.038051 newfstatat(22, "blkio.throttle.io_service_bytes", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000006>
1     1691148727.038076 newfstatat(22, "cgroup.sane_behavior", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000006>
1     1691148727.038101 newfstatat(22, "blkio.bfq.io_service_bytes_recursive", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000006>
1     1691148727.038126 newfstatat(22, "blkio.bfq.io_serviced_recursive", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000005>
1     1691148727.038154 newfstatat(22, "blkio.throttle.write_iops_device", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000009>
1     1691148727.038195 newfstatat(22, "blkio.reset_stats", {st_mode=S_IFREG|0200, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000006>
1     1691148727.038225 newfstatat(22, "blkio.throttle.read_bps_device", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000006>
1     1691148727.038249 newfstatat(22, "blkio.throttle.write_bps_device", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000006>
1     1691148727.038274 newfstatat(22, "tasks", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000006>
1     1691148727.038299 newfstatat(22, "yz", {st_mode=S_IFDIR|0755, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000006>
1     1691148727.038324 openat(22, "yz", O_RDONLY|O_NONBLOCK|O_DIRECTORY) = 23 <0.000008>
1     1691148727.038351 newfstatat(23, "", {st_mode=S_IFDIR|0755, st_size=0, ...}, AT_EMPTY_PATH) = 0 <0.000005>
1     1691148727.038376 fcntl(23, F_GETFL) = 0x18800 (flags O_RDONLY|O_NONBLOCK|O_LARGEFILE|O_DIRECTORY) <0.000004>
1     1691148727.038395 fcntl(23, F_SETFD, FD_CLOEXEC) = 0 <0.000004>
1     1691148727.038420 getdents64(23, 0x563751ceba90 /* 21 entries */, 32768) = 984 <0.000010>
1     1691148727.038450 newfstatat(23, "blkio.bfq.io_service_bytes", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000015>
1     1691148727.038485 newfstatat(23, "cgroup.procs", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000008>
1     1691148727.038513 newfstatat(23, "blkio.bfq.io_serviced", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000008>
1     1691148727.038548 newfstatat(23, "blkio.throttle.read_iops_device", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000008>
1     1691148727.038577 newfstatat(23, "blkio.throttle.io_service_bytes", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000009>
1     1691148727.038606 newfstatat(23, "blkio.bfq.io_service_bytes_recursive", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000009>
1     1691148727.038635 newfstatat(23, "blkio.bfq.io_serviced_recursive", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000010>
1     1691148727.038664 newfstatat(23, "blkio.throttle.write_iops_device", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000010>
1     1691148727.038694 newfstatat(23, "blkio.bfq.weight", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000009>
1     1691148727.038745 newfstatat(23, "blkio.reset_stats", {st_mode=S_IFREG|0200, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000016>
1     1691148727.038799 newfstatat(23, "blkio.throttle.read_bps_device", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000011>
1     1691148727.038840 newfstatat(23, "blkio.throttle.write_bps_device", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000012>
1     1691148727.038881 newfstatat(23, "tasks", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000011>
1     1691148727.038921 newfstatat(23, "notify_on_release", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000046>
1     1691148727.039006 newfstatat(23, "cgroup.clone_children", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000010>
1     1691148727.039042 newfstatat(23, "blkio.throttle.io_serviced", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000009>
1     1691148727.039071 newfstatat(23, "blkio.bfq.weight_device", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000009>
1     1691148727.039100 newfstatat(23, "blkio.throttle.io_service_bytes_recursive", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000008>
1     1691148727.039128 newfstatat(23, "blkio.throttle.io_serviced_recursive", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000009>
1     1691148727.039157 getdents64(23, 0x563751ceba90 /* 0 entries */, 32768) = 0 <0.000004>
1     1691148727.039178 close(23)       = 0 <0.000006>
1     1691148727.039199 rmdir("/sys/fs/cgroup/blkio/yz") = 0 <0.000038>
1     1691148727.039257 newfstatat(22, "notify_on_release", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000006>
1     1691148727.039284 newfstatat(22, "release_agent", {st_mode=S_IFREG|0644, st_size=0, ...}, AT_SYMLINK_NOFOLLOW) = 0 <0.000006>
```



