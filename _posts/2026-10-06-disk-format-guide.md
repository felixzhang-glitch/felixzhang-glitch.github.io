---
layout: post
title: "磁盘格式与 U 盘文件系统选择指南"
date: 2026-10-06
categories: tech
tags: disk filesystem exfat fat32 ntfs apfs
---


本文面向日常使用场景：移动硬盘、U 盘、SD 卡，以及在 Windows、macOS、Android、Linux 之间传文件。重点结论是：如果一个 120 GB U 盘要跨 Windows、macOS、Linux 和较新的 Android 设备使用，优先选 `ExFAT`。

## 1. 先区分两个概念

| 概念 | 作用 | 常见选项 |
| --- | --- | --- |
| 分区表 | 描述磁盘上有哪些分区、每个分区从哪里开始到哪里结束 | `MBR`, `GPT` |
| 文件系统 | 描述一个分区里如何存文件、目录、权限和元数据 | `FAT32`, `ExFAT`, `NTFS`, `APFS`, `ext4`, `F2FS`, `XFS`, `Btrfs` |

日常说“磁盘格式”多数是在说文件系统。macOS 磁盘工具里还会出现“方案”，那通常是在选分区表。

## 2. 常见文件系统发布时间线

| 时间 | 文件系统 | 主要平台/背景 | 现在的定位 |
| --- | --- | --- | --- |
| 1977 左右 | `FAT12` | 早期微软磁盘系统 | 软盘、老设备历史格式 |
| 1984 左右 | `FAT16` | DOS 时代硬盘、早期移动存储 | 老设备兼容 |
| 1985 | `HFS` | 经典 Mac OS | 已过时 |
| 1993 | `NTFS` | Windows NT | Windows 系统盘和大容量硬盘主力 |
| 1993 | `ext2` | Linux | 已基本被 ext3/ext4 替代 |
| 1994 | `XFS` | SGI IRIX，后进入 Linux | Linux 服务器、大文件场景常见 |
| 1996 | `FAT32` | Windows 95 OSR2 | 最强兼容，但单文件 4 GiB 限制 |
| 1998 | `HFS+` / macOS 扩展 | Mac OS 8.1 起 | 老 Mac 盘、机械盘兼容 |
| 2001 | `ext3` | Linux | ext2 加日志，已逐步让位给 ext4 |
| 2006 | `ExFAT` | 微软为闪存、大容量移动存储设计 | U 盘、SDXC、跨平台移动盘常用 |
| 2008 | `ext4` | Linux | Linux 桌面和服务器常见默认文件系统 |
| 2009 起 | `Btrfs` | Linux | 快照、校验、子卷等高级能力 |
| 2012 起 | `F2FS` | Linux/Android，面向 NAND 闪存 | Android 内部存储常见选项 |
| 2017 | `APFS` | Apple File System | macOS、iOS、iPadOS 等苹果设备主力 |

说明：上表中的时间以“推出或进入主流使用”的时间为准，不是所有系统默认启用的时间。

## 3. 主流操作系统常用格式

### Windows

| 场景 | 常用格式 | 说明 |
| --- | --- | --- |
| 系统盘、内置硬盘 | `NTFS` | Windows 默认主力，支持权限、日志、压缩、加密等 |
| 大容量移动盘 | `ExFAT` 或 `NTFS` | 跨平台优先 ExFAT，只在 Windows 内部使用可选 NTFS |
| 小容量老 U 盘、兼容老设备 | `FAT32` | 兼容强，但单文件不能超过 4 GiB |
| 服务器/高级存储 | `ReFS` | Windows Server、存储空间等场景，不适合作为通用 U 盘格式 |

### macOS

| 场景 | 常用格式 | 说明 |
| --- | --- | --- |
| 系统盘、内置 SSD | `APFS` | macOS 10.13 之后的主力格式 |
| 老 Mac 或特定兼容需求 | `HFS+` / macOS 扩展 | 老系统兼容性更好，但不是新 Mac 首选 |
| 和 Windows 互传 | `ExFAT` | Apple 磁盘工具也建议大于 32 GB 的 Windows 兼容卷使用 ExFAT |
| 和非常老的设备互传 | `MS-DOS (FAT)` / FAT32 | 适合 32 GB 及以下，受 4 GiB 单文件限制 |
| 读取 Windows NTFS 盘 | `NTFS` | macOS 通常可读，默认写入支持不完整或不建议依赖 |

### Android

| 场景 | 常用格式 | 说明 |
| --- | --- | --- |
| 手机内部存储 | `ext4` 或 `F2FS` | 由厂商和 Android 版本决定，用户通常不需要手动选择 |
| microSD 作为便携存储 | `FAT32` 或 `ExFAT` | 新设备通常更可能支持 ExFAT；老设备可能只稳定支持 FAT32 |
| microSD 作为“内部存储”扩展 | Android 自己管理 | 可能会格式化并加密，通常不能直接拿到其他设备读 |
| USB OTG U 盘 | `FAT32` 或 `ExFAT` | 设备差异很大，取决于系统、厂商和文件管理器 |

Android 的外置存储支持不是像 Windows/macOS 那样完全统一。ExFAT 历史上涉及微软专利授权，Google 官方从未强制要求设备支持它（2020 年后相关专利已并入 OIN 放开），实际支持取决于厂商。要兼容所有老 Android，FAT32 更稳；要存大文件并面向较新 Android，ExFAT 更实用。

补充：Windows XP 之后的图形格式化界面拒绝把大于 32 GB 的盘格式化为 FAT32，这是工具限制而非格式本身的硬限制，FAT32 本身最大可支持约 2 TiB 的卷。

### Linux

| 场景 | 常用格式 | 说明 |
| --- | --- | --- |
| 桌面 Linux 系统盘 | `ext4` | 最通用、最稳妥的默认选择 |
| 服务器、大文件、大容量盘 | `XFS` | RHEL/CentOS 系常见，适合大文件和高吞吐 |
| 快照、子卷、校验需求 | `Btrfs` | openSUSE、部分 Fedora 场景常见 |
| 闪存优化场景 | `F2FS` | 更常见于 Android，也可用于 Linux |
| 读写 Windows 盘 | `NTFS`, `FAT32`, `ExFAT` | 现代 Linux 对这些格式支持已经比较完整，但发行版可能需要额外工具包 |

## 4. U 盘和移动硬盘怎么选

| 需求 | 推荐格式 | 原因 |
| --- | --- | --- |
| Windows + macOS + Linux + 较新 Android 通用 | `ExFAT` | 支持大文件，跨平台较好，是大容量 U 盘首选 |
| 只在 Windows 使用 | `NTFS` | 权限、日志、稳定性更适合 Windows |
| 只在 macOS 使用 | `APFS` | 适合苹果生态，尤其 SSD |
| 要插老电视、老车机、老相机、老 Android | `FAT32` | 兼容性最强 |
| 要存超过 4 GiB 的视频、镜像、压缩包 | `ExFAT` 或 `NTFS` | FAT32 单文件最大 4 GiB，不合适 |
| Linux 系统启动盘或 Linux 专用数据盘 | `ext4` | Linux 原生、权限和稳定性好 |

## 5. FAT32 与 ExFAT 对比

| 对比项 | FAT32 | ExFAT |
| --- | --- | --- |
| 推出时间 | 约 1996 | 约 2006 |
| 新旧 | 更老 | 更新 |
| 单文件大小 | 最大约 4 GiB | 无 4 GiB 限制，理论上限 16 EiB（实际受实现限制通常到 128 TiB 量级） |
| 大容量 U 盘 | 不推荐 | 推荐 |
| 兼容性 | 极强，老设备最好 | 新系统好，老设备不一定支持 |
| 权限/日志 | 无 | 无 |
| 适合场景 | 老设备、小文件、小容量盘 | 大容量 U 盘、跨平台大文件 |
| 120 GB U 盘 | 不推荐 | 推荐 |

如果只是问“哪个更快”，不能只看文件系统。U 盘主控、闪存颗粒、USB 口版本、是否小文件写入、是否随机写入，影响通常更大。实际使用上，大文件和大容量 U 盘优先 ExFAT。

## 6. 分区表怎么选：MBR 还是 GPT

| 分区表 | 优点 | 适合场景 |
| --- | --- | --- |
| `MBR` | 老设备兼容性更好 | U 盘要插车机、电视、相机、老电脑、老 Android |
| `GPT` | 新标准，支持更大磁盘和更多分区 | 现代电脑、系统盘、大容量移动硬盘 |

对 120 GB U 盘，如果目标是“尽量多设备都能认”，可以考虑：

- 文件系统：`ExFAT`
- 分区表：`MBR`

如果只在现代 Windows/macOS/Linux 之间使用，`GPT + ExFAT` 也可以。

## 7. 具体推荐

你的 120 GB U 盘要连接 Windows、Android、macOS、Linux：

1. 首选：`ExFAT`
2. 分区表：如果磁盘工具让选“方案”，优先考虑 `MBR` 以提高老设备兼容性；现代电脑之间用 `GPT` 也没问题
3. 不建议继续用 `MS-DOS (FAT)` / FAT32，除非你的 Android、车机、电视或相机确实不认 ExFAT
4. 格式化前先备份，格式化会清空 U 盘数据

## 8. 快速结论表

| 格式 | Windows | macOS | Android | Linux | 适合 U 盘吗 |
| --- | --- | --- | --- | --- | --- |
| `FAT32` | 原生支持 | 原生支持 | 基本支持 | 原生支持 | 适合老设备，不适合大文件 |
| `ExFAT` | 原生支持 | 原生支持 | 新设备通常支持，老设备不保证 | 现代发行版通常支持 | 大容量 U 盘首选 |
| `NTFS` | 原生支持 | 通常可读，写入不建议依赖 | 取决于设备/软件 | 5.15+ 内核内置 ntfs3 驱动后读写支持已大幅改善，老内核/老发行版依赖 ntfs-3g 工具 | Windows 为主时可用 |
| `APFS` | 不原生支持 | 原生支持 | 不适合 | 不适合 | 只适合苹果生态 |
| `HFS+` | 不原生支持 | 支持 | 不适合 | 可通过工具支持 | 老 Mac 兼容 |
| `ext4` | 不原生支持 | 不原生支持 | Android 老设备内部存储常见，新机多转向 F2FS | 原生支持 | Linux 专用 |
| `F2FS` | 不原生支持 | 不原生支持 | Android 常见 | 支持 | Linux/Android 专用 |
| `XFS` | 不原生支持 | 不原生支持 | 不常见 | 原生支持 | Linux 专用 |
| `Btrfs` | 不原生支持 | 不原生支持 | 不常见 | 原生支持 | Linux 专用 |

## 参考资料

- Apple Support: [File system formats available in Disk Utility on Mac](https://support.apple.com/guide/disk-utility/file-system-formats-dsku19ed921c/mac)
- Microsoft Learn: [exFAT file system specification](https://learn.microsoft.com/en-us/windows/win32/fileio/exfat-specification)
- Microsoft Learn: [File System Functionality Comparison](https://learn.microsoft.com/en-us/windows/win32/fileio/filesystem-functionality-comparison)
- Android Open Source Project: [Traditional storage](https://source.android.com/docs/core/storage/traditional)
- Linux Kernel documentation: [ext4 Data Structures and Algorithms](https://docs.kernel.org/filesystems/ext4/index.html)
- Linux Kernel documentation: [VFAT](https://docs.kernel.org/filesystems/vfat.html)
- Linux Kernel documentation: [NTFS3](https://docs.kernel.org/filesystems/ntfs3.html)
