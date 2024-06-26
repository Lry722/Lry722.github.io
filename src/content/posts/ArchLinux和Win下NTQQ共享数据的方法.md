---
title: ArchLinux和Win下NTQQ共享数据的方法
published: 2024-01-05
category: 文章
tags:
  - ArchLinux
---

昨天下定决心将 Arch Linux 作为主系统，晚上把系统搞定了，今天来装软件。

然而 QQ 的历史记录和缓存是个大问题，我原先 Win 上 QQ 占了 47G 的空间，清理完也还剩 20+，要是在 Linux 上把这么多东西再存一遍就太浪费了。

考虑到新版 NTQQ 已经是全平台统一架构了，或许可以让两个系统的 QQ 共享数据文件夹。

不过在此之前，建议新建一个分区用来存两个系统共享的数据，这步可以用 Win 的磁盘管理工具来做。然后把 Win 下 QQ 的设置里的储存位置设到这个分区里，再在 Linux 下挂载这个分区。分区大小看个人需求，我是分了 32G。

## 挂载共享分区

Linux 能直接挂载 ntfs 分区，但是写入还是有风险，单独划一个共享分区是为了防止把 Win 的数据写炸，操作流程如下。

- 安装 ntfs-3g

```bash
sudo pacman -S ntfs-3g
```

- 查看共享分区的 UUID

```bash
sudo blkid
```

- 选个位置创建好挂载用的目录，然后把下面这行加到/etc/fstab 的末尾

```
UUID=共享分区的UUID 刚刚创建的目录 ntfs-3g defaults 0 0
```

- 验证配置是否正确

```bash
# 无输出则成功
sudo mount -a
```

之后重启时就会自动挂载了。

## 找目录

既然要共享，就要找到结构相同的文件夹。

### Windows

Win 下 QQ 的文件储存路径在设置里能看到，无论存在哪（此时应该已经被存到共享分区中了），最后都是在 Tencent Files 这个文件夹下。找到其中和 qq 号同名的一个文件夹，里面的 `nt_qq` 就是目标。

### ArchLinux

Linux 下 QQ 的文件存在 `~/.config/QQ/` 目录下一个形如 `nt_qq_xxxxxx` 的文件夹。如果有多个这种文件夹，可以先登录要共享文件的 QQ，再找到修改时间最新的文件夹就行。

## 绑定

接下来把这俩文件夹绑定在一起就行。

```bash
sudo mount --bind Linux下的目录 Win下的目录
```

为了让开机能自动绑定，还要把这行命令加到开机自动执行中。这步有很多实现方式，建议自行搜索。

## 一个小问题

以上就是全部操作了，但是执行完后可能会发现 QQ 统计出来的文件大小在两个系统上不一致。

经过观察，这是因为 `nt_qq/nt_data/Emoji/personal_emoji` 这个文件夹（用于存放收藏的表情）不会被 Linux 下的 QQ 统计进去。但是内容还是一样的，在聊天界面也能正常发送收藏的表情。

除了 `personal_emoji` ，可能也存在其他文件夹统计时被区别对待，但总之这只是统计上的差异，不影响实际使用。
