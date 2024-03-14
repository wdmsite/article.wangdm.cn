---
title: "嵌入式Linux设备实现SD卡或U盘自动挂载"
date: 2024-03-14T10:16:00+08:00
categories:
- Linux
tags:
- Linux
- hotplug
draft: false
---


## 一、前言

在Linux系统中常用udev或mdev来实现可移动存储设备的节点创建和挂载。这两者功能相似，但是mdev更精简。因此在空间受限的嵌入式设备中，mdev更常用。

## 二、mdev介绍

mdev有两种用法：

1）/sbin/mdev

2）/sbin/mdev -s

区别在于1是创建或删除发生变化的设备节点，通常在发生热插拔时调用；2是创建所有设备节点，通常在启动脚本中调用。我们看到的/dev下的所有设备，都是系统在启动时通过mdev -s创建的。Linux内核支持hotplug机制，当内核检测到热插拔事件时，会调用/proc/sys/kernel/hotplug这个文件中记录的程序，来处理热插拔事件。因此，只要将/sbin/mdev写入上述的文件，就能实现热插拔时设备节点文件自动更新。

## 三、实现自动挂载

有几个前提条件必须满足：

1）内核编译时需支持hotplug功能，默认是支持的，可以通过查看是否有/proc/sys/kernel/hotplug这文件来判断是否支持了hotplug。

2）使用Busybox制作rootfs时，需支持mdev

```
Linux System Utilities  --->   
           [*] mdev      
           [*]  Support /etc/mdev.conf
           [*]    Support command execution at device addition/removal
```

实现步骤：

### 1.在启动脚本/etc/init.d/rcS中，增加如下内容：

```
mount -t tmpfs mdev /dev 
mount -t sysfs sysfs /sys
mkdir /dev/pts
mount -t devpts devpts /dev/pts
 
echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
```
需要注意是，上面的几个目录，可能有的文件系统在别处已经挂载了，这里再次挂载会报错。可以查看一下/etc/fstab这个文件，以及/etc/init.d/inittab文件。如有冲突，注释任何一方均可。

我的/etc/fstab文件：

```
# <file system>	<mount pt>	<type>	<options>	<dump>	<pass>
proc		/proc		proc	defaults	0	0
devpts		/dev/pts	devpts	defaults,gid=5,mode=620	0	0
#tmpfs		/dev/shm	tmpfs	mode=0777	0	0
tmpfs		/tmp		tmpfs	mode=1777	0	0
tmpfs		/run		tmpfs	mode=0755,nosuid,nodev	0	0
sysfs		/sys		sysfs	defaults	0	0
```

我的/etc/init.d/inittab文件：

```
# Startup the system
::sysinit:/sbin/swapoff -a
::sysinit:/bin/mount -t tmpfs tmpfs /dev
::sysinit:/bin/mkdir -p /dev/pts
::sysinit:/bin/mkdir -p /dev/shm
::sysinit:/bin/mount -a
::sysinit:/bin/hostname -F /etc/hostname
 
# now run any rc scripts
::sysinit:/etc/init.d/rcS
 
# 略...
```
### 2.编写mdev配置文件：/etc/mdev.conf

这个文件的作用是，当发生热插拔事件时，告诉mdev除了更新设备节点文件之外，还需要做什么事情。我们可以在这里填入我们自定义的命令，实现例如自动挂载和卸载等功能。

该文件的格式为：

```
<device regex> <uid>:<gid> <octal permissions> [<@$*><cmd>]
```

\@ 创建节点后执行的 

\$ 删除节点前执行的 

\* 创建后和删除前都运行的 

如果要实现SD卡和U盘插入自动挂载，拔出自动卸载，则配置文件如下：

```
sd[a-z][0-9]       0:0 0660  @/etc/hotplug/usb/udisk_insert.sh
sd[a-z]            0:0 0660  $/etc/hotplug/usb/udisk_remove.sh
mmcblk[0-9]p[0-9]  0:0 0660  @/etc/hotplug/sd/sd_insert.sh
mmcblk[0-9]        0:0 0660  $/etc/hotplug/sd/sd_remove.sh
```

### 3.编写脚本文件

sd_insert.sh脚本文件参考如下：

```
#!/bin/sh
 
mount -t vfat /dev/mmcblk0p1 /media/sd    # 挂载SD卡
kill -s SIGUSR1 `pidof ipc_ctc`           # 发信号给应用程序
```
  
<br/>
sd_remove.sh脚本文件参考如下： 

```
#!/bin/sh
 
umount /media/sd                          # 卸载SD卡目录           
kill -s SIGUSR2 `pidof ipc_ctc`           # 发信号给应用程序
```


需注意的是，上述脚本中命令是由内核调用，因此打印信息不会输出到stdout。我在调试时就被脚本中的打印给误导了，还以为脚本没被执行，后来才反应过来。



<br/>

**原文地址：**

[嵌入式Linux设备实现SD卡或U盘自动挂载](https://blog.csdn.net/fun_tion/article/details/120135756)