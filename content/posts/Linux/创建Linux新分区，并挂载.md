---
author: "李昌"
title: "创建Linux新分区，并挂载"
date: "2023-11-26"
tags: ["mount"]
categories: ["Linux"]
ShowToc: true
TocOpen: true
---

要创建一个新的 Linux 文件系统分区并进行挂载，您可以按照以下步骤进行操作：

1. 首先，使用以下命令之一来查看系统中已连接的磁盘和分区：
```
sudo fdisk -l
```
或者
```
sudo lsblk
```
这将显示系统中的磁盘和分区列表。找到您要创建分区的磁盘。

2. 使用以下命令打开磁盘分区工具（例如 fdisk）：
```
sudo fdisk /dev/<disk_name>
```
将 <disk_name> 替换为您要创建分区的磁盘的名称，例如 /dev/sda。

3. 在磁盘分区工具中，使用以下命令创建新的分区：
```
n
```
然后按照提示选择分区类型、起始扇区和结束扇区。对于 Linux 文件系统，通常选择默认选项即可。

4. 设置分区的文件系统类型为 Linux。使用以下命令：
```
t
```
然后选择新创建的分区编号，并选择 Linux 文件系统类型（例如 83）。

5. 保存并退出磁盘分区工具。使用以下命令：
```
w
```

6. 使用以下命令创建文件系统：
```
sudo mkfs.ext4 /dev/<partition>
```
将 <partition> 替换为您刚刚创建的分区设备名称，例如 /dev/sda1。

7. 创建一个目录，用于挂载文件系统。使用以下命令：
```
sudo mkdir /mnt/<mount_point>
```
将 <mount_point> 替换为您希望挂载文件系统的目录路径。

8. 使用以下命令将文件系统挂载到目标目录：
```
sudo mount /dev/<partition> /mnt/<mount_point>
```
将 <partition> 替换为您刚刚创建的分区设备名称，<mount_point> 替换为您在第 7 步中创建的目录路径。

9. 挂载成功后，您可以使用以下命令来验证文件系统是否已挂载：
```
df -h
```
这将显示已挂载的文件系统列表，包括挂载点和可用空间。

10. 如果希望在系统启动时自动挂载文件系统，您可以将其添加到 /etc/fstab 文件中。打开 /etc/fstab 文件，并在文件末尾添加以下行：
```
/dev/<partition> /mnt/<mount_point> ext4 defaults 0 0
```
将 <partition> 替换为您刚刚创建的分区设备名称，<mount_point> 替换为您在第 7 步中创建的目录路径。


