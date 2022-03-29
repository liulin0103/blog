---
title: Kudu 磁盘缩容
date: 2022/03/21 14:42:00
updated: 2022/03/21 14:42:00
categories:
- Kudu
tags:
- Kudu
---

## Kudu 磁盘缩容

### 停止服务

1. 停止要缩容磁盘的 ts 服务(我这个是已经停掉了，所以按钮是灰的)

<img src="https://image-1257603108.cos.ap-guangzhou.myqcloud.com/image-20220321163933969.png" alt="image-20220321163933969" style="zoom: 67%;" />

<!--more-->

### 格式化磁盘

1. 在终端中，使用 `lsblk` 命令列出挂接到实例的磁盘，并找到要格式化和挂接的磁盘。

   ```shell
   [cdh@kudu1 ~]$ sudo lsblk
   NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
   sda      8:0    0  100G  0 disk
   ├─sda1   8:1    0  200M  0 part /boot/efi
   └─sda2   8:2    0 99.8G  0 part /
   sdb      8:16   0    2T  0 disk /data1
   sdc      8:32   0   50G  0 disk /data2
   sdd      8:48   0  500G  0 disk /data-new
   ```

2. 使用 [`mkfs`](http://manpages.ubuntu.com/manpages/xenial/man8/mkfs.8.html) 工具格式化磁盘。此命令会**删除**指定磁盘中的所有数据，因此确保正确指定磁盘设备。

   可以使用任何想要的文件格式，建议使用单个 `ext4` 文件系统，而不要使用分区表。这样可以在未来方便的[增加磁盘大小](https://cloud.google.com/compute/docs/disks/resize-persistent-disk)，而无需修改磁盘分区。

   为了最大限度地提高磁盘性能，在 `-E` 标志中[使用推荐的格式化选项](https://cloud.google.com/compute/docs/disks/optimizing-pd-performance#formatting_parameters)。无需在此辅助磁盘上为根卷保留空间，因此请指定 `-m 0` 以使用所有可用的磁盘空间。

   ```bash
   [cdh@kudu1 ~]$ sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdd
   ```

### 装载磁盘

1. 创建用新磁盘装载点的目录。

   ```bash
   [cdh@kudu1 ~]$ sudo mkdir -p /data-new
   [cdh@kudu1 ~]$ sudo mkdir -p /data-old
   ```

2. 使用 [mount 工具](http://manpages.ubuntu.com/manpages/xenial/man8/mount.8.html) 将磁盘装载到实例，并启用 `discard` 选项：

   ```bash
   [cdh@kudu1 ~]$ sudo mount -o discard,defaults /dev/sdd /data-new
   ```

3. 配置对磁盘的读写权限。

   ```bash
   [cdh@kudu1 ~]$ sudo chmod a+w /data-new
   ```
   
   108402672

### 移动数据

1. ~~使用 `rsync`命令，同步文件及其元数据~~

   ```bash
   [cdh@kudu1 ~]$ sudo rsync -av /data1 /data-new
   ```

2. 不知道为什么 `rsync` 命令会一直写入到磁盘写满，这里用 `cp` 命令

   ```bash
   [cdh@kudu1 ~]$ cp -r /data1/* /data-new/
   ```

3. 使用 cp 命令的话，无法同步权限信息，需要手动修改权限

   ```bash
   [cdh@kudu1 ~]$ chown -R kudu. /data-new/kudu/tserver/
   ```

### 修改挂载点

1. 创建当前 `/etc/fstab` 文件的备份。

   ```bash
   [cdh@kudu1 ~]$ sudo cp /etc/fstab /etc/fstab.backup
   ```

2. 使用 `blkid` 命令列出磁盘的 UUID。

   ```bash
   [cdh@kudu1 ~]$ sudo blkid /dev/sdd
   /dev/sdd: UUID="49b89232-e7f5-480d-a2fb-999f7fac1218" TYPE="ext4"
   ```

3. 在文本编辑器中打开 `/etc/fstab` 文件，使用新磁盘的 UUID 作为原数据目录的 uuid，并把旧磁盘的 uuid 映射到备份目录。例如：

   ![](https://image-1257603108.cos.ap-guangzhou.myqcloud.com/image-20220321171128550.png)

4. 保存并使用 `cat` 命令验证 `/etc/fstab` 条目内容正确

5. 完成后重启节点



