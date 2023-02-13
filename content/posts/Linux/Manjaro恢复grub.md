---
author: "李昌"
title: "Manjaro恢复grub"
date: "2023-02-13"
tags: ["Linux", "grub"]
categories: ["Linux"]
ShowToc: true
TocOpen: true
---

https://wiki.manjaro.org/index.php/GRUB/Restore_the_GRUB_Bootloader#EFI_System

启动笔记本时，突然发现gurb引导项故障了，记录一下修复过程：

首先用U盘制备一个manjaro引导盘。然后有两种方式
1. timeshift

进入系统BIOS设置U盘启动，然后直接从U盘启动一个manjaro。

如果你有timeshift备份，可以先尝试使用timeshift进行恢复，不行的话，就尝试方法二

2. grub恢复

进入系统BIOS设置U盘启动，在manjaro提示你选择如何启动系统时，选择`Select a bootloader`，然后选择grub，不出意外，你的原系统就启动了，但这时如果重启，还是找不到grub引导项。可做如下操作：


首先更新grub
```bash
sudo pacman -Syu grub
```

然后：

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=manjaro --recheck
```

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

如果第二个命令提示：`/usr/bin/grub-probe：警告： 未知的设备类型 nvme0n1.`

那么直接删除`/etc/grub.d/60_memtest86+`这个文件即可（无风险）

然后重试：
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

提示完成后，拔掉U盘重启，应该就可以了。