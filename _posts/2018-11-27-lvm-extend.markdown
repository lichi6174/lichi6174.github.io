---
layout: post
title: 阿里云原地扩容LVM磁盘
date:  2018-11-27 16:11:00 +0900  
description: 阿里云原地扩容LVM磁盘配置
img: post-14.jpg # Add image post (optional)
tags: [Blog,linux,lvm,ecs,aliyun]
author: Lichi # Add name author (optional)
linux: true
---

## 资料参考：
> https://help.aliyun.com/document_detail/35097.html?spm=5176.doc35094.6.653.TUzx1g

> https://bbs.aliyun.com/read/539067.html?spm=a2c4e.11155515.0.0.593a7f85OufSGK

## 阿里云ECS实例的磁盘支持原地扩容，无需购买新磁盘来增加 LVM 单个分区的大小。

### 注意：
- 新增空间创建新分区，起始柱面不会是1。
- 本文档介绍的操作只作为标准情况下的示例。如果您有特殊的分区配置，由于使用场景千差万别，无法逐一枚举，需要您自行结合实际情况进行处理。

## 操作方法如下：
#### 在阿里云控制台给需要扩容的磁盘扩容，之后可以看到磁盘已经是 6G（原有大小 5G），但是系统内查看还是原大小（5G）。

```bash
#fdisk -l /dev/vdb
```

#### 系统中将已经挂载的分区取消挂载。

- > #pvdisplay
```bash
  --- Physical volume ---
  PV Name               /dev/vdb1
  VG Name               vg01
  PV Size               <1024.00 GiB / not usable 3.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              262143
  Free PE               0
  Allocated PE          262143
  PV UUID               CMtVvJ-ndci-BTOx-iJBL-lzN6-Xwln-0P8h4C
```

- > #df -lh

```bash
 Filesystem             Size  Used Avail Use% Mounted on
 /dev/vda1               40G  7.0G   31G  19% /
 devtmpfs               7.8G     0  7.8G   0% /dev
 tmpfs                  7.8G     0  7.8G   0% /dev/shm
 tmpfs                  7.8G  372K  7.8G   1% /run
 tmpfs                  7.8G     0  7.8G   0% /sys/fs/cgroup
 /dev/mapper/vg01-lv01 1008G  790G  167G  83% /home
 tmpfs                  1.6G     0  1.6G   0% /run/user/0
```

- > #cat /etc/fstab

```bash
#/etc/fstab
#Created by anaconda on Mon Jul 10 12:22:03 2017
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=b7792c31-ad03-4f04-a650-a72e861c892d /                       ext4    defaults        1 1
/dev/vg01/lv01  /home  ext4 defaults  0 0
/home/swap swap swap defaults 0 0
```

- > #umount /dev/vg01/lv01

#### 取消逻辑卷的激活状态并查看逻辑卷激活状态。
- > #vgchange -an /dev/vg01/lv01
- > #lvscan

#### 如果数据盘是和实例一起购买的且并未转换成按量付费磁盘，那么控制台操作重启实例以完成磁盘底层扩容,待系统重启完成后跳过紧随其后的两个骤继续操作,如果数据盘是单独购买的或者已经变更成按量付费磁盘，那么继续执行紧随其后的两个步骤。

#### 控制台操作将磁盘卸载。

#### 控制台重新挂载磁盘。

#### 运行 **==fdisk -l /dev/vdb==** 可以看到磁盘空间变大了。
- > #fdisk -l /dev/vdb

#### 运行 fdisk /dev/vdb 对磁盘进行分区操作，添加一个新分区（vdb2）并保存。

```bash
#fdisk /dev/vdb
n
p
2
wq
```

#### 运行 **==fdisk -l /dev/vdb==** 。此时有两个分区，分别是：**==/dev/vdb1==**（原有的分区名） 和 **==/dev/vdb2==**（新增的分区名）。

#### 将新增的分区加入到卷组中，vgdisplay 可以看到 Free PE 空间大小。
```bash
#vgextend vg01 /dev/vdb2
#vgdisplay
```

#### 运行 **==lvextend -l +100%FREE /dev/vg01/lv01==** 增加空间，vgdisplay 可以查看到 Free PE 为空了。
```bash
#lvextend -l +100%FREE /dev/vg01/lv01
```

#### 执行如下命令，确认文件系统的类型。
```bash
#fsck -N /dev/vg01/lv01
fsck from util-linux 2.23.2
[/sbin/fsck.ext4 (1) -- /home] fsck.ext4 /dev/mapper/vg01-lv01
```

#### 根据文件系统类型来变更分区大小。
- ext4文件系统类型,变更分区大小命令。
```bash
#resize2fs /dev/vg01/lv01 
```

-  xfs文件系统类型,变更分区大小命令。
```bash
#xfs_growfs /dev/vg01/lv01
```

#### 重新挂载分区可以查看到空间变大了，原有数据还在。
```bash
#mount /dev/vg01/lv01 /home
#df -lh
```

> 注意: 操作示例中 vg01 是 VG 名称，lv01 是逻辑卷名称，请根据实际情况填写。