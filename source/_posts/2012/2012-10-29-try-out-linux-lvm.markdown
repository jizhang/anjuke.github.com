---
layout: post
title: "LVM 简单实践"
date: 2012-10-29 21:20
comments: true
categories:
---

{%img right http://0.gravatar.com/avatar/ce1e13bbf946c92e2abf740f8909bafa %}

为什么会想到要使用lvm呢，因为家里的主力下载机是一台PC，她又是我的主要平时使用的机子（除开mac外），经常要对下载那个分区进行整理，装个游戏，压制点视频音乐有时候分区会很紧张，然而跨分区是很复杂的操作。这就让我想到了linux下面的lvm功能。

{% img center http://upload.wikimedia.org/wikipedia/commons/thumb/b/ba/LVM1.svg/500px-LVM1.svg.png %}

<!-- more -->

## 什么是lvm

[lvm][1]是Linux核心提供的逻辑卷轴管理功能，主要应用在磁盘分区上；传统的计算机使用磁盘主要通过管理分区，在常见的MBR磁盘上最多只可以使用4个主分区，或者任意个逻辑分区；虽然在GPT分区上可以实现使用任意分区数量，但是分区大小总是有限制的，即使很多文件系统工具提供了对分区大小进行热操作，仍旧不是很方便。lvm则可以通过让系统核心支持真正的逻辑卷管理，达到物理分区的虚拟化。可以随时对逻辑卷进行大小的修改。

## 使用

更加系统的讲解请参考[LVM-HOWTO][2]。

注意我们使用的lvm2版本，主要区别，可以查看[这里][3]。

我下面的演示，主要基于gentoo，安装的时候livecd里已经带了lvm工具。

解释一些lvm里使用的名词，先给一张图

```
+-- Volume Group --------------------------------+
|                                                |
|    +----------------------------------------+	 |
| PV | PE |  PE | PE | PE | PE | PE | PE | PE |	 |
|    +----------------------------------------+	 |
|      .       	  .    	     . 	      .	       	 |
|      .          .    	     .        .	         |
|    +----------------------------------------+	 |
| LV | LE |  LE | LE | LE | LE | LE | LE | LE |	 |
|    +----------------------------------------+	 |
|            .          .        .     	   .     |
|            . 	        .        .     	   .     |
|    +----------------------------------------+	 |
| PV | PE |  PE | PE | PE | PE | PE | PE | PE |	 |
|    +----------------------------------------+	 |
|                                                |
+------------------------------------------------+
```

+ volume group (VG): lvm中使用的最高层次的抽象对象，它把(物理上的)逻辑和主分区划为管理单元
+ physical volume (PV): 一个pv可以是一块磁盘，或者是看起来磁盘设备 (例如/dev/sda1)
+ logical volume (LV): 和非lvm系统中的逻辑分区一样，一个lv看起来就像是普通的块设备，而它可以包含文件系统(例如/home)
+ physical extend (PE): 每个pv都被分为几部分数据，它们就是pe，对于lvm来说，它们和le大小一样。
+ logical extend (LE): 类似PE

最后，更加直观地理解，可以看下图，

```

    hda1   hdc1      (PV:s on partitions or whole disks)
       \   /
        \ /
       diskvg        (VG)
       /  |  \
      /   |   \
  usrlv rootlv varlv (LV:s)
    |      |     |
 ext2  reiserfs  xfs (filesystems)

```

## 要求

1. 使用lvm，至少需要2.4.x的kernel
2. 安装了lvm

## 配置

### 背景

这里解释的是在gentoo安装过程中将两块磁盘中的分区配置成使用lvm的过程。

主要参考了[gentoo的lvm手册][4]。

假设，我现在有两块SATA硬盘，将其物理分区安排如下


```
/dev/sda1  -> /boot
/dev/sda2  -> (swap)
/dev/sda3  -> /
/dev/sda4  -> 将使用lvm
/dev/sdb1  -> 将使用lvm
```

**请注意操作的安全性，一个错误的操作可能会导致整个分区的数据丢失!!**

### 安装

+ 使用fdisk或者你喜欢的分区工具分区
+ 启动lvm服务，在gentoo下使用`/etc/init.d/lvm start`，livecd里默认已经启动
+ 配置lvm，打开`/etc/lvm/lvm.conf`，修改如下信息

```
# a表示add添加，r表示reject忽略
# 语法类似正则，表示把/dev/sda与/dev/sdb里的分区都加入到lvm的识别里
filter = [ "a|/dev/sd[ab]|", "r/.*/" ]

# 使之前配置的vg都生效
vgscan
vgchange -a y
```
# 把分区准备好

```
pvcreate /dev/sda4 /dev/sdb1
```

我们这个例子里，`/dev/sda1`，`/dev/sda2`，`/dev/sda3`分别是`/boot`,`swap`和`/`

**极其不推荐在/etc, /lib, /mnt, /proc, /sbin, /dev, and /root上使用lvm，可能会导致我些目录无法访问!!**

+ 创建并扩展一个vg

```
vgcreate vg /dev/sad4
vgextend vg /dev/sdb1
```

+ 创建最终要使用的lv

```
lvcreate -L10G -nusr vg # 在vg里创建一个大小为10G的lv，名字是usr
lvcreate -L5G -nhome vg # 在vg里创建一个大小为5G的lv，名字是home
lvcreate -L10G -nvar vg # 在vg里创建一个大小为10G的lv，名字是var

lvextend -L+5G /dev/vg/home # 举例，给home这个lv增加5G容量
```

+ 在lv上创建文件系统，并挂载

```
mkfs.reiserfs /dev/vg/usr
mkfs.reiserfs /dev/vg/home
mkfs.reiserfs /dev/vg/var

# 在这之前先挂载必要分区，请参考gentoo安装手册
mkdir /mnt/gentoo/usr
mount /dev/vg/usr /mnt/gentoo/usr
mkdir /mnt/gentoo/home
mount /dev/vg/usr /mnt/gentoo/home
mkdir /mnt/gentoo/var
mount /dev/vg/usr /mnt/gentoo/var
```
+ 下面继续gentoo的安装过程，chroot一直到配置kernel
+ 配置kernel，打开lvm的支持选项
+ 2.4.x的kernel

```
Multi-device support (RAID and LVM)  --->
  [*] Multiple devices driver support (RAID and LVM)
  < >  RAID support
(注意这里不选择LVM是有意的，它的意思是支持LVM1)
  < >  Logical volume manager (LVM) support
  <M>  Device-mapper support
  < >   Mirror (RAID-1) support
```

  + 2.6.x以及之后版本的kernel

```
Device Drivers  --->
 Multiple devices driver support (RAID and LVM) --->
   [*] Multiple devices driver support (RAID and LVM)
   < >   RAID support
   <M>   Device mapper support
```

编译完成后的模块名字叫`dm-mod.ko`

**请注意，务必让`/usr/src/linux`指向您正在使用的kernel核心源代码，因为lvm的ebuild会根据路径查找device-mapper的依赖性**

+ 配置chroot里的lvm.conf，和外层一致

+ 配置`/etc/fstab`，根据需要配置lvm的分区

```
/dev/sda1     /boot        ext2       defaults      1 2
/dev/sda2     none         swap       sw            0 0
/dev/sda3     /            reiserfs   noatime       0 1
/dev/cdrom    /mnt/cdrom   auto       noauto,ro     0 0
# Logical Volumes
/dev/vg/usr   /usr         reiserfs   noatime       0 2
/dev/vg/home  /home        reiserfs   noatime       0 2
/dev/vg/var   /var         reiserfs   noatime       0 2
```

+ 继续其他安装配置，一直到退出chroot前
+ 安装lvm，并将其设置为启动

```
emerge -v lvm2
rc-update add lvm boot
```

+ 退出chroot，重启前，关闭所有lvm分区

```
vgchange -a n
```

### 如果中途有重启

如果在安装过程中因为我些原因需要重启，需要按以下操作重新激活lvm分区

```
vgscan --mknodes
```

而一些比较旧的安装CD里需要按照以下顺序激活

```
# 先关闭所有vg
vgchange -a n
# 导出所有vg
vgexport -a
# 导入所有vg
vgimport -a
# 重新激活所有vg
vgchange -a y
```

### 使用

如果没有意外，那么重启后系统可以成功boot进入新安装的gentoo系统

我的gentoo box的磁盘情况如下

![lvm](/medias/20121029/lvm.jpeg)

其实也是这个我才知道前阵子上线的一台服务器上`/dev/mapper`，原来其实就是lvm啊XDDD

\_\_END\_\_

[1]: http://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)
[2]: http://tldp.org/HOWTO/LVM-HOWTO/
[3]: http://tldp.org/HOWTO/LVM-HOWTO/lvm2faq.html
[4]: http://www.gentoo.org/doc/en/lvm2.xml