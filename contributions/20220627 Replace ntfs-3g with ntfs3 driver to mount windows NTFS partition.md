[#]: subject: "如何使用新的原生 NTFS 驱动替代旧版的 FUSE NTFS 驱动"
[#]: via: "https://www.insidentally.com/articles/000029/"
[#]: author: "insidentally https://www.insidentally.com"
[#]: keywords: " 文件系统 ntfs3 ntfs-3g 分区"
[#]: url: "发布后链接，由发布人填写"

如何使用新的原生 NTFS 驱动替代旧版的 FUSE NTFS 驱动
======

> <ruby>NTFS<rt>New Technology File System</rt></ruby> 是 Windows NT 内核的系列操作系统支持的、一个特别为网络和磁盘配额、文件加密等管理安全特性设计的磁盘文件系统格式。而 NTFS3 是功能齐全的 NTFS 读写驱动程序。该驱动程序适用于最高 3.1 的 NTFS 版本。

![使用 ntfs3 驱动替换 ntfs-3g 挂载 windows NTFS 分区][1]

### 简介

最初 Linux 内核没有对 NTFS 做原生支持，后来的原生支持也仅有只读功能，来自 Tuxera 的 NTFS-3G 是目前主流的解决方案，但在实际使用中也有不少小问题。NTFS-3G 是借助 Linux 的用户空间文件系统 FUSE 模块在用户层实现的一个模仿对 NTFS 支持的文件系统，对 NTFS 的访问逻辑代码都是在用户层代码实现的。

在 NTFS3 出现之前 Linux 上使用 NTFS 主要问题还是缺乏稳定且功能齐全的读/写支持。

2020年，Paragon Software 做出了一个惊人的决定：尝试将之前只用于商业的 NTFS3 驱动程序 Mainline 化。最终 Linux Kernel 5.15 合并了 Paragon 提供的 NTFS3 内核驱动，它拥有更高的性能和更多的特性。

* 该驱动程序实现了对 NTFS 文件系统中的正常、稀疏和压缩文件的读/写支持。

* 支持本地日志回放。

* 支持挂载的 NTFS 卷的 NFS 导出。

* 支持文件和文件夹的权限管理。

### 挂载

挂载时使用的文件系统类型是 ntfs3。

#### 手动挂载

以前使用 NTFS-3g 驱动的挂载方式是：

``` bash
# mount -t ntfs-3g /dev/sdxY /mnt
```

现在只需要将 ntfs-3g 替换为 ntfs3 即可：

``` bash
# mount -t ntfs3 /dev/sdxY /mnt
```

`-t` 指出文件系统类型，`/dev/sdxY` 是你分区的路径，可以使用 `lsblk` 命令查看。`/mnt` 是挂载到哪个文件夹。

如果需要挂载参数，就使用 `-o` 后面接参数就行了，如：

``` bash
# mount -t ntfs3 -o iocharset=utf8,umask=22,prealloc /dev/sdxY /mnt
```

这里 iocharset=utf8,umask=22,prealloc 都是挂载参数，详见[后文](#挂载参数)

#### 开机自动挂载

编辑 `/etc/fstab` 文件,添加行：

```
UUID=**** /data ntfs3 iocharset=utf8,umask=0,prealloc 0 0
```

其中 `UUID=****` 是指定分区的 UUID。使用 `UUID` 的好处在于它们与磁盘顺序无关。如果你在 BIOS 中改变了你的存储设备顺序，或是重新拔插了存储设备，或是因为一些 BIOS 可能会随机地改变存储设备的顺序，那么用 `UUID` 来表示将更有效。可以使用 `blkid` 命令查看 `UUID` 。 

`/data`  是挂载位置。本示例的位置是 `/data` 你需要提前创建这个文件夹。

后面的选项都是挂载参数，详细可看[后文](#挂载参数)介绍。

最后两个 `0 0` ，表示是否备份和是否检查。`0 0` 表示不备份，不检查。

### 挂载参数

<table>
    <tr>
        <th>参数</th>
        <th>解释</th>
    </tr>
    <tr>
        <td>iocharset=name</td>
        <td>此选项告知驱动程序如何解释路径字符串，并将其转换为 Unicode 或返回。如果未设置此选项，将使用默认代码页（CONFIG\u NLS\u default）。示例：iocharset=utf8</td>
    </tr>
    <tr>
        <td>uid=</td>
        <td>挂载用户 id</td>
    </tr>
    <tr>
        <td>gid=</td>
        <td>挂载组 id</td>
    </tr>
    <tr>
        <td>umask=</td>
        <td>控制装载 NTFS 卷后创建的文件/目录的默认权限。</td>
    </tr>
    <tr>
        <td>dmask=</td>
        <td rowspan='2'> fmask 只适用于文件，dmask 只适用于目录，而不是指定同时适用于文件和目录的 umask。</td>
    </tr>
    <tr>
        <td>fmask=</td>
    </tr>
    <tr>
        <td rowspan='3'>noacsrules</td>
        <td>“无访问规则”装载选项将文件/文件夹的访问权限设置为 777，所有者/组设置为 root。此装载选项吸收所有其他权限。</td>
    </tr>
    <tr>
        <td>文件/文件夹的权限更改将报告为成功，但仍将保持 777。</td>
    </tr>
    <tr>
        <td>所有者/组更改将报告为成功，但他们将保留为 root 用户。</td>
    </tr>
    <tr>
        <td>nohidden</td>
        <td>Linux 下不会显示具有 Windows 特定隐藏（FILE_ATTRIBUTE_HIDDEN）属性的文件。</td>
    </tr>
    <tr>
        <td>sys_immutable</td>
        <td>具有 Windows 特定系统（FILE_ATTRIBUTE_SYSTEM）属性的文件将标记为系统不可变文件。</td>
    </tr>
    <tr>
        <td>discard</td>
        <td>支持 TRIM 命令以提高删除操作的性能，建议将其用于固态驱动器（SSD）。</td>
    </tr>
    <tr>
        <td>force</td>
        <td>即使卷被标记为脏，也强制驱动程序装载分区。不建议使用。</td>
    </tr>
    <tr>
        <td>sparse</td>
        <td>创建稀疏的新文件。</td>
    </tr>
    <tr>
        <td>showmeta</td>
        <td>使用此参数可显示已装入 NTFS 分区上的所有元文件（系统文件）。默认情况下，所有元文件都是隐藏的。</td>
    </tr>
    <tr>
        <td>prealloc</td>
        <td>当写入时文件大小增加时，为文件过度预分配空间。减少对不同文件执行并行写入操作时的碎片。</td>
    </tr>
    <tr>
        <td>acl</td>
        <td>支持 POSIX ACL（访问控制列表）。如果内核支持，则有效。不要与 NTFS ACL 混淆。指定为 acl 的选项支持 POSIX acl。</td>
    </tr>
</table>

### NTFS3 的优点

NTFS3 是内核态的驱动，ntfs3 比 nfts-3g 无论是速度还是负载都要好上不少。

已经有诸多网友做过测试：

* [ntfs-3g 与 Linux 5.15+ ntfs3 驱动的简单性能测试](https://biluohc.github.io/posts/ntfs3gvsntfs3/)

* [Linux 5.15内核NTFS3性能评测](https://bbs.deepin.org/post/236260)

除了性能更好以外，NTFS3 还支持挂载用户和文件权限管理等功能。具体使用方法可以自行学习 gid、uid 以及 umask 的用法。

另外 NTFS3 还支持 NTFS 的 prealloc，可以大幅减少文件碎片的产生。

### 关于 NTFS3 驱动无人维护的问题

Paragon 2020年在 GNU 通用许可证下发布了 NTFS3 驱动程序，在开源后的一年里，NTFS3 的驱动经过了多轮审查和修改，用来提高代码质量。直到 2021 年合并进入内核主线。

但是自从该驱动 2021 年在 Linux 5.15 中最终被主线化以来，至今为止，在接近一年的时间里，还没有任何重大的错误修复被送入驱动。

有人推测是该驱动的维护者 Konstantin Komarov 身处俄罗斯，受到俄乌战争影响的原因。

随后包括 Linus Torvalds 在内的诸多程序员都对此事表达了关切，并且愿意参与到贡献中来。

现在，我们看到 Paragon 软件公司的 Konstantin Komarov 在因休息和其他事务而离开后，又重新活跃在内核邮件列表中。Komarov 在 2022 年 6 月 3 日为Linux 5.19 的合并窗口提交了一批 NTFS3 的修正。

我相信 ntfs3 未来会越来越好。并且目前，nfts3 已经是 Linux 中最好用 NTFS 驱动了，我觉得您不妨尝试一下。

---

作者简介：一个喜欢瞎鼓捣的医学生

------

via: https://www.insidentally.com/articles/000029/

作者：[insidentally](https://www.insidentally.com)
编辑：[wxy](https://github.com/wxy)

本文由贡献者投稿至 [Linux 中国公开投稿计划](https://github.com/LCTT/Articles/)，采用 [CC-BY-SA 协议](https://creativecommons.org/licenses/by-sa/4.0/deed.zh) 发布，[Linux中国](https://linux.cn/) 荣誉推出

[1]: https://s1.ax1x.com/2022/06/27/jVemcj.jpg
