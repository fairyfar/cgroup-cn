本文所述方法在`Windows 11 WSL2`的`Ubuntu 22.04.5 LTS`上验证通过。

# 背景

在过去的几年里，Linux已经开始向`cgroups v2`过渡。这已经成为许多基于Linux发行版的标准，对于Mac和Linux上的`Docker Desktop`也是如此。然而，当涉及到`Windows Subsystem for Linux (WSL)`时，遇到一个小问题。默认情况下，WSL以同时支持`cgroups v1`和`cgroups v2`的混合模式运行。这种双重支持系统在运行使用`cgroups v2`的容器时引入了一些问题。

随着`cgroups v1`被弃用，目前有必要在WSL中启用`cgroups v2`。

# 配置

幸运的是，WSL附带了一个配置文件，该文件提供了设置内核参数的选项。利用这一点，我们可以禁用`cgroups v1`，从而使WSL环境与Mac OS X和现代Linux发行版保持一致。

实现这一更改非常简单。只需要在`%USERPROFILE%\.wslconfig`上创建并编辑一个文本文件。（例如，对于我的`fairyfar`用户，这将是`C:\Users\fairyfar\.wslconfig`）。

一种友好的方法是将以下内容粘贴到资源管理器栏：

```
notepad.exe %UserProfile%/.wslconfig
```

具体来说，需要添加以下几行：

```
[wsl2]
kernelCommandLine = cgroup_no_v1=all systemd.unified_cgroup_hierarchy=1
```

然后，重启WSL，使用管理员权限，在`PowerShell`中执行：

```
wsl --shutdown
```

# 问题

如果现在重新启动WSL，可能会收到以下错误：

```
远程主机强迫关闭了一个现有的连接
Press any key to continue ...
```

此时，我们需要使用`PowerShell`执行`update`，然后重启WSL：

```
wsl --update
```

