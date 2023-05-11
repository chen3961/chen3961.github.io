---
layout:     post
title:      扩展LVM ThinPool Metadata Size
date:       2019-03-25
summary:    当LVM中ThinPool的metadata资源耗尽，会造成新的卷无法创建，同时已创建的卷无法写入。需要扩展ThinPool的metadata的空间，让thinpool重新正常工作
categories: LVM
---

通过Openstack创建卷的过程，遇到一个奇怪的问题，开始卷可以创建成功，卷写入也没有问题。但是创建多个卷后发现无法创建，已创建的卷也无法写入。在已创建的卷上创建文件系统总是显示:：
```
/dev/vdb: write failed after 0 of 4096 at 0: Input/output error
```

开始以为是磁盘错误，所以通过chkblk等命令检查磁盘，并没有发现问题。


然后测试创建新的卷，检查cinder-volume的日志，发现：
```
oslo_messaging.rpc.server Stderr: u'File descriptor 20 (/dev/urandom) leaked on lvcreate invocation. Parent PID 769: /usr/bin/python2\n  Thin pool cinder--volumes-cinder--volumes--pool-tpool (253:2) transaction_id is 91, while expected 92.\n  Failed to suspend cinder-volumes/cinder-volumes-pool with queued messages.\n'
```
看样子是后台lvm出现的问题。

检查后台lvm，发现空间还有空余，但是metadata的空间已经满了：
```
  LV                                          VG             Attr       LSize    Pool                Origin Data%  Meta%  Move Log Cpy%Sync Convert
  cinder-volumes-pool                         cinder-volumes twi-aotz-- <509.54g                            17.81  100
  volume-235d0896-9f64-4199-bafb-259fc3d862d5 cinder-volumes Vwi-aotz--  100.00g cinder-volumes-pool        10.08
  volume-32d210e6-bff0-4b72-b74d-aaf130576f59 cinder-volumes Vwi-aotz--  100.00g cinder-volumes-pool        0.56
  volume-46da9715-ffda-49b9-9426-3fd315dae873 cinder-volumes Vwi-aotz--   40.00g cinder-volumes-pool        38.18
  volume-4d056bf3-c8f6-4fb2-b870-695b8416b178 cinder-volumes Vwi-aotz--  100.00g cinder-volumes-pool        0.55
  volume-4d42f4f2-c795-48f4-875d-e910d63a5490 cinder-volumes Vwi-aotz--  100.00g cinder-volumes-pool        10.08
  volume-59af0aa2-e946-4c62-8bf2-00cc7f9f8161 cinder-volumes Vwi-aotz--  100.00g cinder-volumes-pool        0.55
```
不清楚为啥数据还有这么多空余，而metadata居然满了。查了一下文档发现thinpool的空间默认只有12M，果断增大thinpool的metadata的空间：
```
lvextend cinder-volumes/cinder-volumes-pool--poolmetadatasize -L+512M
```
之后发现创建新的卷还是失败。通过搜索相关日志，发现cinder-volumes-pool的transcation id已经出现损坏的数据，发现网上还有相关的修复的方法，尝试了一下：
```
vgcfgbackup cinder-volumes -f ~/backup
```
导出volume group的配置，找到cinder-volumes-pool的描述，然后修改transaction_id，重新导入
```
vgcfgrestore cinder-volumes -f ~/backup
```
这样就修复了lvm系统，可以创建新卷，已创建的卷也能够写入了。
