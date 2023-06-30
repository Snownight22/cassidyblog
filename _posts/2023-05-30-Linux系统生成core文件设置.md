---
layout: post
title:  Linux 系统生成 core 文件设置
date:   2023-05-30 21:20:00 +0800
categories: 实用小技巧
tags: Linux
---

默认 Linux 系统不生成 core 文件，要产生 core 文件则按以下设置实现：  

#### 1. 修改系统资源限制

可以通过 `ulimit -a` 查看系统各种资源限制，如 core 文件大小，数据段大小，打开文件数目等等。如：  

![ulimit_0.png]({{site.baseurl}}/styles/images/tips/ulimit_0.png)  

可以看到，core 文件默认大小为 0, 因此不会产生 core 文件，使用 `ulimit -c n` 来设置 core 文件大小，单位为 KB，如：

`ulimit -c 100` 设置 core 文件最大为 100KB。  
`ulimit -c unlimited` 不限制 core 文件大小。  

![ulimit_unlimited.png]({{site.baseurl}}/styles/images/tips/ulimit_unlimited.png)  

此设置只针对当前会话有效，如果想要长久生效，可以将配置写入 `/etc/security/limits.conf` 中。  

```
* soft core unlimited
* hard core unlimited
```

* 该文件第一列表示用户和组，第二列表示软限制还是硬限制，第三列表示限制的资源类型，第四列表示限制的最大值。  
* soft 和 hard 的区别：soft 是一个警告值，hard 是真正意义的限制值。一般设置为相同值。  
* core 是内核文件，nofile 是文件描述符，noproc 是进程数。  

另一种长久生效的方法是将配置命令 `ulimit -c unlimited` 写入 ~/.bashrc 文件中。  

#### 2. 设置 core 文件配置

修改两个 core 相关配置文件 /proc/sys/kernel/core_pattern 和 /proc/sys/kernel/core_uses_pid。  

```
echo "core-%p" > /proc/sys/kernel/core_pattern
echo "1" > /proc/sys/kernel/core_uses_pid
```

ubuntu 系统中需要使用 `sysctl -w` 来写入：  

```
sudo sysctl -w kernel.core_pattern = core
sudo sysctl -w kernel.core_uses_pid = 1
```

写入这两个文件也只是在当前会话生效，需要长久生效的方法是将配置写入 /etc/sysctl.conf，在该文件中添加以下两行：  

```
kernel.core_uses_pid = 1
kernel.core_pattern = core-%p
```

----