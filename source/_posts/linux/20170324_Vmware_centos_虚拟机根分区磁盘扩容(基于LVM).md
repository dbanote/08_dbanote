---
title: VMWare CentOS 虚拟机根分区磁盘扩容(基于LVM)
date: 2017-03-24
tags:
- Linux
- lvm
---

## 关闭虚拟机并修改虚拟机配置，增大磁盘大小
![](http://oligvdnzp.bkt.clouddn.com/0324_lvm_resize_01.png)

<!-- more -->

## 启动虚拟机后查看磁盘状态
``` perl
df -hT
------------------------------------------------------------
Filesystem           Type   Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup-lv_root
                     ext4    43G  3.2G   38G   8% /
tmpfs                tmpfs  3.9G     0  3.9G   0% /dev/shm
/dev/sda1            ext4   477M   82M  366M  19% /boot
------------------------------------------------------------

fdisk -l /dev/sda
------------------------------------------------------------
Disk /dev/sda: 214.7 GB, 214748364800 bytes             # 磁盘sda已扩展到200G
255 heads, 63 sectors/track, 26108 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00070abb

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *           1          64      512000   83  Linux
Partition 1 does not end on cylinder boundary.
/dev/sda2              64        7833    62401536   8e  Linux LVM
------------------------------------------------------------
```

## 查看lvm状态
``` perl
pvs
------------------------------------------------------------
  PV         VG       Fmt  Attr PSize  PFree
  /dev/sda2  VolGroup lvm2 a--u 59.51g    0 
------------------------------------------------------------

vgs
------------------------------------------------------------
  VG       #PV #LV #SN Attr   VSize  VFree
  VolGroup   1   2   0 wz--n- 59.51g    0 
------------------------------------------------------------

lvs
------------------------------------------------------------
  LV      VG       Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_root VolGroup -wi-ao---- 43.74g                                                    
  lv_swap VolGroup -wi-ao---- 15.77g 
------------------------------------------------------------
```

## 创建新分区
``` perl
fdisk /dev/sda
------------------------------------------------------------
WARNING: DOS-compatible mode is deprecated. Its strongly recommended to
         switch off the mode (command 'c') and change display units to
         sectors (command 'u').

Command (m for help): n       # 输入n开始创建分区
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 3     # 输入3创建sda3
First cylinder (7833-26108, default 7833): 
Using default value 7833
Last cylinder, +cylinders or +size{K,M,G} (7833-26108, default 26108): 
Using default value 26108

Command (m for help): w       # 写入分区表
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
The kernel still uses the old table. The new table will be used at
the next reboot or after you run partprobe(8) or kpartx(8)
Syncing disks.    # 需要重启后才能生效
------------------------------------------------------------
```

## 重启虚拟机 格式化新分区
``` perl
fdisk -l /dev/sda
------------------------------------------------------------
Disk /dev/sda: 214.7 GB, 214748364800 bytes
255 heads, 63 sectors/track, 26108 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00070abb

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *           1          64      512000   83  Linux
Partition 1 does not end on cylinder boundary.
/dev/sda2              64        7833    62401536   8e  Linux LVM
/dev/sda3            7833       26108   146797950   83  Linux
------------------------------------------------------------

mkfs.ext4 /dev/sda3
```

## lvm扩容
``` perl
pvcreate /dev/sda3
------------------------------------------------------------
  Physical volume "/dev/sda3" successfully created
------------------------------------------------------------

pvs
------------------------------------------------------------
  PV         VG       Fmt  Attr PSize   PFree  
  /dev/sda2  VolGroup lvm2 a--u  59.51g      0 
  /dev/sda3           lvm2 ---- 140.00g 140.00g
------------------------------------------------------------

ll /dev/mapper/
------------------------------------------------------------
total 0
crw-rw----. 1 root root 10, 236 Mar 24 11:49 control
lrwxrwxrwx. 1 root root       7 Mar 24 11:49 VolGroup-lv_root -> ../dm-0
lrwxrwxrwx. 1 root root       7 Mar 24 11:49 VolGroup-lv_swap -> ../dm-1
------------------------------------------------------------

vgextend /dev/mapper/VolGroup /dev/sda3
------------------------------------------------------------
  Volume group "VolGroup" successfully extended
------------------------------------------------------------

lvextend -l +100%FREE /dev/VolGroup/lv_root /dev/sda3
------------------------------------------------------------
  Size of logical volume VolGroup/lv_root changed from 43.74 GiB (11198 extents) to 183.74 GiB (47037 extents).
  Logical volume lv_root successfully resized.
------------------------------------------------------------

resize2fs /dev/VolGroup/lv_root
------------------------------------------------------------
resize2fs 1.43-WIP (20-Jun-2013)
Filesystem at /dev/VolGroup/lv_root is mounted on /; on-line resizing required
old_desc_blocks = 3, new_desc_blocks = 12
The filesystem on /dev/VolGroup/lv_root is now 48165888 blocks long.
------------------------------------------------------------

# 扩容后查看磁盘使用情况
df -h
------------------------------------------------------------
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup-lv_root
                      181G  3.3G  170G   2% /
tmpfs                 3.9G     0  3.9G   0% /dev/shm
/dev/sda1             477M   82M  366M  19% /boot
------------------------------------------------------------
```
