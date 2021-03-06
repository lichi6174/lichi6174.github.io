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
#### 1. 在阿里云控制台给需要扩容的ECS实例的某个磁盘库容到指定大小，比如/dev/vdb由原来的5G扩容到6G，通过控制台可看到磁盘已经是 6G大小，但是系统内使用命令：*fdisk -l /dev/vdb*{: style="color: red"}， 查看还是5G大小。

```bash
#fdisk -l /dev/vdb
```

#### 2. 系统中将已经挂载的分区取消挂逻辑分区载。
- 查看物理卷、磁盘空间、分区表信息

```bash
#pvdisplay
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
#
#df -lh
 Filesystem             Size  Used Avail Use% Mounted on
 /dev/vda1               40G  7.0G   31G  19% /
 devtmpfs               7.8G     0  7.8G   0% /dev
 tmpfs                  7.8G     0  7.8G   0% /dev/shm
 tmpfs                  7.8G  372K  7.8G   1% /run
 tmpfs                  7.8G     0  7.8G   0% /sys/fs/cgroup
 /dev/mapper/vg01-lv01 1008G  790G  167G  83% /home
 tmpfs                  1.6G     0  1.6G   0% /run/user/0
#
#cat /etc/fstab
#Created by anaconda on Mon Jul 10 12:22:03 2017
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=b7792c31-ad03-4f04-a650-a72e861c892d /                       ext4    defaults        1 1
/dev/vg01/lv01  /home  ext4 defaults  0 0
/home/swap swap swap defaults 0 0
```

- 取消逻辑分区的挂载

```bash
#umount /dev/vg01/lv01
```

#### 3. 取消逻辑卷的激活状态并查看逻辑卷激活状态。

```bash
#vgchange -an /dev/vg01/lv01
#lvscan
```

#### 4. 如果数据盘是和实例一起购买的且并未转换成按量付费磁盘，那么控制台操作重启实例以完成磁盘底层扩容,待系统重启完成后跳过紧随其后的两个骤继续操作,如果数据盘是单独购买的或者已经变更成按量付费磁盘，那么继续执行紧随其后的两个步骤。

#### 5. 控制台操作将磁盘卸载。

#### 6. 控制台重新挂载磁盘。

#### 7. 运行 *fdisk -l /dev/vdb*{: style="color: red"}，可以看到磁盘空间变大了。

```bash
#fdisk -l /dev/vdb
```

#### 8. 运行 *fdisk /dev/vdb*{: style="color: red"}，对磁盘进行分区操作，添加一个新分区（vdb2）并保存。

```bash
#fdisk /dev/vdb
n
p
2
wq
```

#### 9. 运行 *fdisk -l /dev/vdb*{: style="color: red"}，此时有两个分区，分别是：*/dev/vdb1*{: style="color: red"}（原有的分区名） 和 */dev/vdb2*{: style="color: red"}（新增的分区名）。

```bash
#fdisk -l /dev/vdb
```

#### 10. 让之后的vgextend扩展VG组时候能识别新建分区名 *（/dev/vdb2）*{: style="color: red"}，否则可能会报错。

```bash
#partprobe /dev/vdb
```

#### 11. 将新增的分区加入到卷组中,使用 `vgdisplay` 可以查看到 `Free PE`字段值，即为空闲的空间大小。

```bash
#vgextend vg01 /dev/vdb2
#vgdisplay
```

#### 12. 运行 *lvextend -l +100%FREE /dev/vg01/lv01*{: style="color: red"}，增加空间，`vgdisplay` 可以查看到 `Free PE`字段值为空了。

```bash
#lvextend -l +100%FREE /dev/vg01/lv01
```

#### 13. 重新挂载分区（要先挂载分区，才能执行扩展分区的命令）。

```bash
#mount /dev/vg01/lv01 /home
#df -lh
```


#### 14. 执行如下命令，确认文件系统的类型。

```bash
#fsck -N /dev/vg01/lv01
fsck from util-linux 2.23.2
[/sbin/fsck.ext4 (1) -- /home] fsck.ext4 /dev/mapper/vg01-lv01
```

#### 15. 根据上面步骤得知需扩展的LVM逻辑分区的文件系统类型，并进行真正的LVM逻辑分区扩展操作。
- ext4文件系统类型,变更分区大小命令。

```bash
#resize2fs /dev/vg01/lv01 
```

-  xfs文件系统类型,变更分区大小命令。

```bash
#xfs_growfs /dev/vg01/lv01
```

#### 16. 验证原有数据是否未丢失。

#### 17. *注意: 操作示例中 vg01 是 VG 名称，lv01 是逻辑卷名称，请根据实际情况填写。*{: style="color: red"}