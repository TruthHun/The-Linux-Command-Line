# 存储媒介

在前面章节中，我们已经在文件级别上见识了数据的操作。在这章里，我们将从设备级别来考虑数据。Linux 有着令人惊奇的能力来处理存储设备，不管是物理设备，比如说硬盘，还是网络设备，或者是虚拟存储设备，像 RAID（独立磁盘冗余阵列)和 LVM（逻辑卷管理器）。

然而，这不是一本关于系统管理的书籍，我们不会试图深入地覆盖整个主题。我们将努力做的就是介绍一些概念和用来管理存储设备的重要命令。

为了做这一章的练习，我们将会使用 USB 闪存，CD-RW 光盘（如果系统配备了 CD-ROM 烧录器）和一张软盘（如果系统有这样配备的话）。

我们将看看以下命令：

* mount – 挂载一个文件系统

* umount – 卸载一个文件系统

* fsck – 检查和修复一个文件系统

* fdisk – 分区表控制器

* mkfs – 创建文件系统

* fdformat – 格式化一张软盘

* dd — 把面向块的数据直接写入设备

* genisoimage (mkisofs) – 创建一个 ISO 9660的映像文件

* wodim (cdrecord) – 把数据写入光存储媒介

* md5sum – 计算 MD5检验码

###16.1 挂载和卸载存储设备

Linux 桌面系统的最新进展已经使存储设备管理对于桌面用户来说极其容易。大多数情况下，我们只要把设备连接到系统中，它就能工作。在过去（比如说，2004年），这个工作必须手动完成。在非桌面系统中（例如，服务器中），这仍然是一个主要地手动过程，因为服务器经常有极端的存储需求和复杂的配置要求。

管理存储设备的第一步是把设备连接到文件系统树中。这个叫做"挂载"的过程允许设备连接到操作系统中。回想一下第三章，类 Unix 的操作系统，比如Linux在单一文件系统树中维护连接在各个节点的各种设备。这与其它操作系统形成对照，比如说 MS-DOS 和 Windows 系统中，每个设备（例如 C:\，D:\，等）保持着单独的文件系统树。

有一个叫做/etc/fstab 的文件可以列出系统启动时要挂载的设备（典型地，硬盘分区）。下面是来自于 Fedora 7系统的/etc/fstab 文件实例：

```
LABEL=/12               /               ext3        defaults        1   1
LABEL=/home             /home           ext3        defaults        1   2
LABEL=/boot             /boot           ext3        defaults        1   2
tmpfs                   /dev/shm        tmpfs       defaults        0   0
devpts                  /dev/pts        devpts      gid=5,mode=620  0   0
sysfs                   /sys            sysfs       defaults        0   0
proc                    /proc           proc        defaults        0   0
LABEL=SWAP-sda3         /swap           swap        defaults        0   0
```

在这个实例中所列出的大多数文件系统是虚拟的，并不适用于我们的讨论。就我们的目的而言，前三个是我们感兴趣的：

```
LABEL=/12               /               ext3        defaults        1   1
LABEL=/home             /home           ext3        defaults        1   2
LABEL=/boot             /boot           ext3        defaults        1   2
```

这些是硬盘分区。每行由六个字段组成，如下所示：

|字段|内容|描述(说明)|
|-----|-----|
1|设备名|传统上，这个字段包含与物理设备相关联的设备文件的实际名字，比如说/dev/hda1（第一个 IDE 通道上第一个主设备分区）。然而今天的计算机，有很多热插拔设备（像 USB 驱动设备），许多现代的 Linux 发行版用一个文本标签和设备相关联。当这个设备连接到系统中时，这个标签（当储存媒介格式化时，这个标签会被添加到存储媒介中）会被操作系统读取。那样的话，不管赋给实际物理设备哪个设备文件，这个设备仍然能被系统正确地识别。
|2|挂载点|>设备所连接到的文件系统树的目录。|
|3|文件系统类型|Linux 允许挂载许多文件系统类型。大多数本地的 Linux 文件系统是 ext3，但是也支持很多其它的，比方说 FAT16 (msdos), FAT32 (vfat)，NTFS (ntfs)，CD-ROM (iso9660)，等等。|
|4|选项|文件系统可以通过各种各样的选项来挂载。有可能，例如，挂载只读的文件系统，或者挂载阻止执行任何程序的文件系统（一个有用的安全特性，避免删除媒介。）|
|5|频率|一位数字，指定是否和在什么时间用 dump 命令来备份一个文件系统。|
|6|次序|一位数字，指定 fsck 命令按照什么次序来检查文件系统。|

####16.1.1 查看已挂载的文件系统列表

这个 mount 命令被用来挂载文件系统。执行这个不带参数的命令，将会显示一系列当前挂载的文件系统：

```
[me@linuxbox ~]$ mount
/dev/sda2 on / type ext3 (rw)
proc on /proc type proc (rw)
sysfs on /sys type sysfs (rw)
devpts on /dev/pts type devpts (rw,gid=5,mode=620)
/dev/sda5 on /home type ext3 (rw)
/dev/sda1 on /boot type ext3 (rw)
tmpfs on /dev/shm type tmpfs (rw)
none on /proc/sys/fs/binfmt_misc type binfmt_misc (rw)
sunrpc on /var/lib/nfs/rpc_pipefs type rpc_pipefs (rw)
fusectl on /sys/fs/fuse/connections type fusectl (rw)
/dev/sdd1 on /media/disk type vfat (rw,nosuid,nodev,noatime,
uhelper=hal,uid=500,utf8,shortname=lower)
twin4:/musicbox on /misc/musicbox type nfs4 (rw,addr=192.168.1.4)
```

这个列表的格式是：设备 on 挂载点 type 文件系统类型（选项）。例如，第一行所示设备/dev/sda2 作为根文件系统被挂载，文件系统类型是 ext3，并且可读可写（这个“rw”选项）。在这个列表的底部有两个有趣的条目。倒数第二行显示了在读卡器中的一张2G 的 SD 内存卡，挂载到了/media/disk 上。最后一行是一个网络设备，挂载到了/misc/musicbox 上。

第一次实验，我们将使用一张 CD-ROM。首先，在插入 CD-ROM 之前，我们将看一下系统：

```
[me@linuxbox ~]$ mount
/dev/mapper/VolGroup00-LogVol00 on / type ext3 (rw)
proc on /proc type proc (rw)
sysfs on /sys type sysfs (rw)
devpts on /dev/pts type devpts (rw,gid=5,mode=620)
/dev/hda1 on /boot type ext3 (rw)
tmpfs on /dev/shm type tmpfs (rw)
none on /proc/sys/fs/binfmt_misc type binfmt_misc (rw)
sunrpc on /var/lib/nfs/rpc_pipefs type rpc_pipefs (rw)
```

这个列表来自于 CentOS 5系统，使用 LVM（逻辑卷管理器）来创建它的根文件系统。正如许多现在的 Linux 发行版一样，这个系统试图自动挂载插入的 CD-ROM。当我们插入光盘后，我们看看下面的输出：

```
[me@linuxbox ~]$ mount
/dev/mapper/VolGroup00-LogVol00 on / type ext3 (rw)
proc on /proc type proc (rw)
sysfs on /sys type sysfs (rw)
devpts on /dev/pts type devpts (rw,gid=5,mode=620)
/dev/hda1 on /boot type ext3 (rw)
tmpfs on /dev/shm type tmpfs (rw)
none on /proc/sys/fs/binfmt_misc type binfmt_misc (rw)
sunrpc on /var/lib/nfs/rpc_pipefs type rpc_pipefs (rw)
/dev/hdc on /media/live-1.0.10-8 type iso9660 (ro,noexec,nosuid,nodev,uid=500)
```

当我们插入光盘后，除了额外的一行之外，我们看到和原来一样的列表。在列表的末尾，我们看到 CD-ROM 已经挂载到了/media/live-1.0.10-8上，它的文件类型是 iso9660（CD-ROM）。就我们的实验目的而言，我们对这个设备的名字感兴趣。当你自己进行这个实验时，这个设备名字是最有可能不同的。

------

**警告：**

在随后的实例中，至关重要的是你要密切注意用在你系统中的实际设备名，并且不要使用此文本中使用的名字！

------

还要注意音频 CD 和 CD-ROM 不一样。音频 CD 不包含文件系统，这样在通常意义上，它就不能被挂载了。

现在我们拥有 CD-ROM 光盘的设备名字，让我们卸载这张光盘，并把它重新挂载到文件系统树的另一个位置。我们需要超级用户身份（使用系统相应的命令）来进行操作，并且用 umount（注意这个命令的拼写）来卸载光盘：

```
[me@linuxbox ~]$ su -
Password:
[root@linuxbox ~]# umount /dev/hdc
```

下一步是创建一个新的光盘挂载点。简单地说，一个挂载点就是文件系统树中的一个目录。它没有什么特殊的。它甚至不必是一个空目录，即使你把设备挂载到了一个非空目录上，你也不能看到这个目录中原来的内容，直到你卸载这个设备。就我们的目的而言，我们将创建一个新目录：

```
[root@linuxbox ~]# mkdir /mnt/cdrom
```

最后，我们把这个 CD-ROW 挂载到一个新的挂载点上。这个-t 选项用来指定文件系统类型：

```
[root@linuxbox ~]# mount -t iso9660 /dev/hdc /mnt/cdrom
```

之后，我们可以通过这个新挂载点来查看 CD-ROW 的内容：

```
[root@linuxbox ~]# cd /mnt/cdrom
[root@linuxbox cdrom]# ls
```

注意当我们试图卸载这个 CD-ROW 时，发生了什么事情。

```
[root@linuxbox cdrom]# umount /dev/hdc
umount: /mnt/cdrom: device is busy
```

这是怎么回事呢？原因是我们不能卸载一个设备，如果某个用户或进程正在使用这个设备的话。在这种情况下，我们把工作目录更改到了 CD-ROW 的挂载点，这个挂载点导致设备忙碌。我们可以很容易地修复这个问题通过把工作目录改到其它目录而不是这个挂载点。

```
[root@linuxbox cdrom]# cd
[root@linuxbox ~]# umount /dev/hdc
```

现在这个设备成功卸载了。


> ####为什么卸载重要
>
> 如果你看一下 free 命令的输出结果，这个命令用来显示关于内存使用情况的统计信息，你会看到一个统计值叫做”buffers“。计算机系统旨在尽可能快地运行。系统运行速度的一个阻碍是缓慢的设备。打印机是一个很好的例子。即使最快速的打印机相比于计算机标准也极其地缓慢。一台计算机确实会运行地非常慢，如果它要停下来等待一台打印机打印完一页。在早期的个人电脑时代（多任务之前），这真是个问题。如果你正在编辑电子表格或者是文本文档，每次你要打印文件时，计算机都会停下来而且变得不能使用。计算机能以打印机可接受的最快速度把数据发送给打印机，但由于打印机不能快速地打印，这个发送速度会非常慢。这个问题被解决了，由于打印机缓存的出现，一个包含一些 RAM 内存的设备，位于计算机和打印机之间。通过打印机缓存，计算机把要打印的结果发送到这个缓存区，数据会迅速地存储到这个 RAM 中，这样计算机就能回去工作，而不用等待。与此同时，打印机缓存将会以打印机可接受的速度把缓存中的数据缓慢地输出给打印机。
>
> 缓存被广泛地应用于计算机中，使其运行地更快。别让偶尔地读取或写入慢设备的需求阻碍了系统的运行速度。在真正与比较慢的设备交互之前，操作系统会尽可能多的读取或写入数据到内存中的存储设备里。以 Linux 操作系统为例，你会注意到系统看似填充了多于它所需要的内存。这不意味着 Linux 正在使用所有的内存，它意味着 Linux 正在利用所有可用的内存，来作为缓存区。
>
> 这个缓存区允许非常快速地写入存储设备，因为写入物理设备的操作被延迟到后面进行。同时，这些注定要传送到设备中的数据正在内存中堆积起来。时不时地，操作系统会把这些数据写入物理设备。
>
> 卸载一个设备需要把所有剩余的数据写入这个设备，所以设备可以被安全地移除。如果没有卸载设备，就移除了它，就有可能没有把注定要发送到设备中的数据输送完毕。在某些情况下，这些数据可能包含重要的目录更新信息，这将导致文件系统损坏，这是发生在计算机中的最坏的事情之一。


####16.1.2 确定设备名称


有时很难来确定设备名称。在以前，这并不是很难。一台设备总是在某个固定的位置，也不会挪动它。类 Unix 的系统喜欢设备那样安排。退回到 Unix 系统的时代，“更改一个磁盘驱动器”更像是要用一辆叉车从机房中移除一台如洗衣机大小的设备那样困难。最近几年，典型的桌面硬件配置已经变得相当动态，并且 Linux 已经发展地比其祖先更加灵活。在以上事例中，我们利用现代 Linux 桌面系统的功能来“自动地”挂载设备，然后再确定设备名称。但是如果我们正在管理一台服务器或者是其它一些（这种自动挂载功能）不会发生的环境，我们又如何能查清设备名呢？

首先，让我们看一下系统怎样来命名设备。如果我们列出目录/dev（所有设备的住所）的内容，我们会看到许许多多的设备：

```
[me@linuxbox ~]$ ls /dev
```

这个列表的内容揭示了一些设备命名的模式。这里有几个：

|模式|设备|
|-----|-----|
|/dev/fd*|软盘驱动器|
|/dev/hd*|老系统中的 IDE(PATA)磁盘。典型的主板包含两个 IDE 连接器或者是通道，每个连接器带有一根缆线，每根缆线上有两个硬盘驱动器连接点。缆线上的第一个驱动器叫做主设备，第二个叫做从设备。设备名称这样安排，/dev/hdb 是指第一通道上的主设备名；/dev/hdb 是第一通道上的从设备名；/dev/hdc 是第二通道上的主设备名，等等。末尾的数字表示硬盘驱动器上的分区。例如，/dev/hda1是指系统中第一硬盘驱动器上的第一个分区，而 /dev/hda 则是指整个硬盘驱动器。|
|/dev/lp*|打印机|
|/dev/sd*|SCSI 磁盘。在最近的 Linux 系统中，内核把所有类似于磁盘的设备（包括 PATA/SATA 硬盘，闪存，和 USB 存储设备，比如说可移动的音乐播放器和数码相机）看作 SCSI 磁盘。剩下的命名系统类似于上述所描述的旧的/dev/hd*命名方案。|
|/dev/sr*|光盘（CD/DVD 读取器和烧写器）|

另外，我们经常看到符号链接比如说/dev/cdrom，/dev/dvd 和/dev/floppy，它们指向实际的设备文件，提供这些链接是为了方便使用。如果你工作的系统不能自动挂载可移动的设备，你可以使用下面的技巧来决定当可移动设备连接后，它是怎样被命名的。首先，启动一个实时查看文件/var/log/messages（你可能需要超级用户权限）：

```
[me@linuxbox ~]$ sudo tail -f /var/log/messages
```

这个文件的最后几行会被显示，然后停止。下一步，插入这个可移动的设备。在这个例子里，我们将使用一个16MB 闪存。瞬间，内核就会发现这个设备，并且探测它：

```
Jul 23 10:07:53 linuxbox kernel: usb 3-2: new full speed USB device
using uhci_hcd and address 2
Jul 23 10:07:53 linuxbox kernel: usb 3-2: configuration #1 chosen from 1 choice
Jul 23 10:07:53 linuxbox kernel: scsi3 : SCSI emulation for USB Mass Storage devices
Jul 23 10:07:58 linuxbox kernel: scsi scan: INQUIRY result too short (5), using 36
Jul 23 10:07:58 linuxbox kernel: scsi 3:0:0:0: Direct-Access Easy
Disk 1.00 PQ: 0 ANSI: 2
Jul 23 10:07:59 linuxbox kernel: sd 3:0:0:0: [sdb] 31263 512-byte hardware sectors (16 MB)
Jul 23 10:07:59 linuxbox kernel: sd 3:0:0:0: [sdb] Write Protect is off
Jul 23 10:07:59 linuxbox kernel: sd 3:0:0:0: [sdb] Assuming drive
cache: write through
Jul 23 10:07:59 linuxbox kernel: sd 3:0:0:0: [sdb] 31263 512-byte hardware sectors (16 MB)
Jul 23 10:07:59 linuxbox kernel: sd 3:0:0:0: [sdb] Write Protect is off
Jul 23 10:07:59 linuxbox kernel: sd 3:0:0:0: [sdb] Assuming drive cache: write through
Jul 23 10:07:59 linuxbox kernel: sdb: sdb1
Jul 23 10:07:59 linuxbox kernel: sd 3:0:0:0: [sdb] Attached SCSI removable disk
Jul 23 10:07:59 linuxbox kernel: sd 3:0:0:0: Attached scsi generic sg3 type 0

After the display pauses again, type Ctrl-c to get the prompt back. The interesting parts
of the output are the repeated references to “[sdb]” which matches our expectation of a
SCSI disk device name. Knowing this, two lines become particularly illuminating:
```

显示再次停止之后，输入 Ctrl-c，重新得到提示符。输出结果的有趣部分是一再提及“[sdb]”，这正好符和我们期望的 SCSI 磁盘设备名称。知道这一点后，有两行输出变得颇具启发性：

```
Jul 23 10:07:59 linuxbox kernel: sdb: sdb1
Jul 23 10:07:59 linuxbox kernel: sd 3:0:0:0: [sdb] Attached SCSI removable disk
```

这告诉我们这个设备名称是/dev/sdb 指整个设备，/dev/sdb1是这个设备的第一分区。正如我们所看到的，使用 Linux 系统充满了有趣的监测工作。

```
Tip: Using the tail -f /var/log/messages technique is a great way to
watch what the system is doing in near real-time.
```

------

**小贴士：**

使用这个 tail -f /var/log/messages 技巧是一个很不错的方法，可以实时观察系统的一举一动。

------

```
既然知道了设备名称，我们就可以挂载这个闪存驱动器了：

[me@linuxbox ~]$ sudo mkdir /mnt/flash
[me@linuxbox ~]$ sudo mount /dev/sdb1 /mnt/flash
[me@linuxbox ~]$ df
Filesystem      1K-blocks   Used        Available   Use%    Mounted on
/dev/sda2       15115452    5186944     9775164     35%     /
/dev/sda5       59631908    31777376    24776480    57%     /home
/dev/sda1       147764      17277       122858      13%     /boot
tmpfs           776808      0           776808      0%      /dev/shm
/dev/sdb1       15560       0           15560       0%      /mnt/flash
```

这个设备名称会保持不变只要设备与计算机保持连接并且计算机不会重新启动。

###16.2 创建新的文件系统

假若我们想要用 Linux 本地文件系统来重新格式化这个闪存驱动器，而不是它现用的 FAT32系统。这涉及到两个步骤：1.（可选的）创建一个新的分区布局若已存在的分区不是我们喜欢的。2.在这个闪存上创建一个新的空的文件系统。

------

**注意：**

在下面的练习中，我们将要格式化一个闪存驱动器。拿一个不包含有用数据的驱动器作为实验品，因为它将会被擦除！再次，请确定你指定了正确的系统设备名称。未能注意此警告可能导致你格式化（即擦除）错误的驱动器！

------

####16.2.1 用 fdisk 命令操作分区

这个 fdisk 程序允许我们直接在底层与类似磁盘的设备（比如说硬盘驱动器和闪存驱动器）进行交互。使用这个工具可以在设备上编辑，删除，和创建分区。以我们的闪存驱动器为例，首先我们必须卸载它（如果需要的话），然后调用 fdisk 程序，如下所示：

```
[me@linuxbox ~]$ sudo umount /dev/sdb1
[me@linuxbox ~]$ sudo fdisk /dev/sdb
```

注意我们必须指定设备名称，就整个设备而言，而不是通过分区号。这个程序启动后，我们将看到以下提示：

```
Command (m for help):
```

输入"m"会显示程序菜单：

```
Command action
a       toggle a bootable flag
....
```

我们想要做的第一件事情是检查已存在的分区布局。输入"p"会打印出这个设备的分区表：

```
Command (m for help): p

Disk /dev/sdb: 16 MB, 16006656 bytes
1 heads, 31 sectors/track, 1008 cylinders
Units = cylinders of 31 * 512 = 15872 bytes

Device Boot     Start        End     Blocks   Id        System
/dev/sdb1           2       1008      15608+   b       w95 FAT32
```

在此例中，我们看到一个16MB 的设备只有一个分区(1)，此分区占用了可用的1008个柱面中的1006个,并被标识为 Windows 95 FAT32分区。有些程序会使用这个标志符来限制一些可以对磁盘所做的操作，但大多数情况下更改这个标志符没有危害。然而，为了叙述方便，我们将会更改它，以此来表明是个 Linux 分区。在更改之前，首先我们必须找到被用来识别一个 Linux 分区的 ID 号码。在上面列表中，我们看到 ID 号码“b”被用来指定这个已存在的分区。要查看可用的分区类型列表，参考之前的程序菜单。我们会看到以下选项：

```
l   list known partition types
```

如果我们在提示符下输入“l”，就会显示一个很长的可能类型列表。在它们之中会看到“b”为已存在分区类型的 ID 号，而“83”是针对 Linux 系统的 ID 号。

回到之前的菜单，看到这个选项来更改分区 ID 号：

```
t   change a partition's system id
```

我们先输入“t”，再输入新的 ID 号：

```
Command (m for help): t
Selected partition 1
Hex code (type L to list codes): 83
Changed system type of partition 1 to 83 (Linux)
```

这就完成了我们需要做得所有修改。到目前为止，还没有接触这个设备（所有修改都存储在内存中，而不是在此物理设备中），所以我们将会把修改过的分区表写入此设备，再退出。为此，我们输入在提示符下输入"w":

```
Command (m for help): w
The partition table has been altered!
Calling ioctl() to re-read partition table.
WARNING: If you have created or modified any DOS 6.x
partitions, please see the fdisk manual page for additional
information.
Syncing disks.
[me@linuxbox ~]$
```

如果我们已经决定保持设备不变，可在提示符下输入"q"，这将退出程序而没有写更改。我们可以安全地忽略这些不祥的警告信息。

####16.2.2 用 mkfs 命令创建一个新的文件系统

完成了分区编辑工作（它或许是轻量级的），是时候在我们的闪存驱动器上创建一个新的文件系统了。为此，我们会使用 mkfs（"make file system"的简写），它能创建各种格式的文件系统。在此设备上创建一个 ext3文件系统，我们使用"-t"选项来指定这个"ext3"系统类型，随后是我们要格式化的设备分区名称：

```
[me@linuxbox ~]$ sudo mkfs -t ext3 /dev/sdb1
mke2fs 1.40.2 (12-Jul-2007)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
3904 inodes, 15608 blocks
780 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=15990784
2 block groups
8192 blocks per group, 8192 fragments per group
1952 inodes per group
Superblock backups stored on blocks:
  8193
Writing inode tables: done
Creating journal (1024 blocks): done
Writing superblocks and filesystem accounting information: done
This filesystem will be automatically checked every 34 mounts or
180 days, whichever comes first. Use tune2fs -c or -i to override.
[me@linuxbox ~]$
```

当 ext3被选为文件系统类型时，这个程序会显示许多信息。若把这个设备重新格式化为它最初的 FAT32文件系统，指定"vfat"作为文件系统类型：

```
[me@linuxbox ~]$ sudo mkfs -t vfat /dev/sdb1
```

任何时候添加额外的存储设备到系统中时，都可以使用这个分区和格式化的过程。虽然我们只以一个小小的闪存驱动器为例，同样的操作可以被应用到内部硬盘和其它可移动的存储设备上像 USB 硬盘驱动器。

###16.3 测试和修复文件系统

在之前讨论文件/etc/fstab 时，我们会在每行的末尾看到一些神秘的数字。每次系统启动时，在挂载系统之前，都会按照惯例检查文件系统的完整性。这个任务由 fsck 程序（是"file system check"的简写）完成。每个 fstab 项中的最后一个数字指定了设备的检查顺序。在上面的实例中，我们看到首先检查根文件系统，然后是 home 和 boot 文件系统。若最后一个数字是零则相应设备不会被检查。

除了检查文件系统的完整性之外，fsck 还能修复受损的文件系统，其成功度依赖于损坏的数量。在类 Unix 的文件系统中，文件恢复的部分被放置于 lost+found 目录里面，位于每个文件系统的根目录下面。

检查我们的闪存驱动器（首先应该卸载），我们能执行下面的操作：

```
[me@linuxbox ~]$ sudo fsck /dev/sdb1
fsck 1.40.8 (13-Mar-2008)
e2fsck 1.40.8 (13-Mar-2008)
/dev/sdb1: clean, 11/3904 files, 1661/15608 blocks
```

以我的经验，文件系统损坏情况相当罕见，除非硬件存在问题，如磁盘驱动器故障。在大多数系统中，系统启动阶段若探测到文件系统已经损坏了，则会导致系统停止下来，在系统继续执行之前，会指导你运行 fsck 程序。


> ####什么是 fsck?
>
> 在 Unix 文化中，"fsck"这个单词往往会被用来指代另一个和它仅有一个字母差别的常用词。因为如果你遇到了迫不得已需要运行 fsck 命令的糟糕境遇时，这个词经常会脱口而出。

###16.4 格式化软盘

对于那些还在使用配备了软盘驱动器的计算机的用户，我们也能管理这些设备。准备一张可用的空白软盘要分两个步骤。首先，对这张软盘执行低级格式化，然后创建一个文件系统。为了完成格式化，我们使用 fdformat 程序，同时指定软盘设备名称（通常为/dev/fd0）：

```
[me@linuxbox ~]$ sudo fdformat /dev/fd0
Double-sided, 80 tracks, 18 sec/track. Total capacity 1440 kB.
Formatting ... done
Verifying ... done
```

接下来，通过 mkfs 命令，给这个软盘创建一个 FAT 文件系统：

```
[me@linuxbox ~]$ sudo mkfs -t msdos /dev/fd0
```

注意我们使用这个“msdos”文件系统类型来得到旧（小的）风格的文件分配表。当一个软磁盘被准备好之后，则可能像其它设备一样挂载它。

###16.5 直接把数据移入/出设备

虽然我们通常认为计算机中的数据以文件形式来组织数据，也可以“原始的”形式来考虑数据。如果我们看一下磁盘驱动器，例如，我们看到它由大量的数据“块”组成，而操作系统却把这些数据块看作目录和文件。然而，如果把磁盘驱动器简单地看成一个数据块大集合，我们就能执行有用的任务，如克隆设备。

这个 dd 程序能执行此任务。它可以把数据块从一个地方复制到另一个地方。它使用独特的语法（由于历史原因），经常它被这样使用：

```
dd if=input_file of=output_file [bs=block_size [count=blocks]]
```

比方说我们有两个相同容量的 USB 闪存驱动器，并且要精确地把第一个驱动器（中的内容）复制给第二个。如果连接两个设备到计算机上，它们各自被分配到设备/dev/sdb 和 /dev/sdc 上，这样我们就能通过下面的命令把第一个驱动器中的所有数据复制到第二个驱动器中。

```
dd if=/dev/sdb of=/dev/sdc
```

或者，如果只有第一个驱动器被连接到计算机上，我们可以把它的内容复制到一个普通文件中供以后恢复或复制数据：

```
dd if=/dev/sdb of=flash_drive.img
```

------

**警告！**

这个 dd 命令非常强大。虽然它的名字来自于“数据定义”，有时候也把它叫做“清除磁盘”因为用户经常会误输入 if 或 of 的规范。在按下回车键之前，要再三检查输入与输出规范！

------

###16.6 创建 CD-ROM 映像

写入一个可记录的 CD-ROM（一个 CD-R 或者是 CD-RW）由两步组成；首先，构建一个 iso 映像文件，这就是一个 CD-ROM 的文件系统映像，第二步，把这个映像文件写入到 CD-ROM 媒介中。

####16.6.1 创建一个 CD-ROM 的映像拷贝

如果想要制作一张现有 CD-ROM 的 iso 映像，我们可以使用 dd 命令来读取 CD-ROW 中的所有数据块，并把它们复制到本地文件中。比如说我们有一张 Ubuntu CD，用它来制作一个 iso 文件，以后我们可以用它来制作更多的拷贝。插入这张 CD 之后，确定它的设备名称（假定是/dev/cdrom），然后像这样来制作 iso 文件：

```
dd if=/dev/cdrom of=ubuntu.iso
```

这项技术也适用于 DVD 光盘，但是不能用于音频 CD，因为它们不使用文件系统来存储数据。对于音频 CD，看一下 cdrdao 命令。

####16.6.2 从文件集合中创建一个映像

创建一个包含目录内容的 iso 映像文件，我们使用 genisoimage 程序。为此，我们首先创建一个目录，这个目录中包含了要包括到此映像中的所有文件，然后执行这个 genisoimage 命令来创建映像文件。例如，如果我们已经创建一个叫做~/cd-rom-files 的目录，然后用文件填充此目录，再通过下面的命令来创建一个叫做 cd-rom.iso 映像文件：

```
genisoimage -o cd-rom.iso -R -J ~/cd-rom-files
```

"-R"选项添加元数据为 Rock Ridge 扩展，这允许使用长文件名和 POSIX 风格的文件权限。同样地，这个"-J"选项使 Joliet 扩展生效，这样 Windows 中就支持长文件名了。


> 一个有着其它名字的程序。。。
>
> 如果你看一下关于创建和烧写光介质如 CD-ROMs 和 DVD 的在线文档，你会经常碰到两个程序叫做 mkisofs 和 cdrecord。这些程序是流行软件包"cdrtools"的一部分，"cdrtools"由 Jorg Schilling 编写成。在2006年春天，Schilling 先生更改了部分 cdrtools 软件包的协议，Linux 社区许多人的看法是，这创建了一个与 GNU GPL 不相兼容的协议。结果，就 fork 了这个 cdrtools 项目，目前新项目里面包含了 cdrecord 和 mkisofs 的替代程序，分别是 wodim 和 genisoimage。


###16.7 写入 CD-ROM 镜像

有了一个映像文件之后，我们可以把它烧写到光盘中。下面讨论的大多数命令对可记录的 CD-ROW 和 DVD 媒介都适用。

####16.7.1 直接挂载一个 ISO 镜像

有一个诀窍，我们可以用它来挂载 iso 映像文件，虽然此文件仍然在我们的硬盘中，但我们当作它已经在光盘中了。添加 "-o loop" 选项来挂载（同时带有必需的 "-t iso9660" 文件系统类型），挂载这个映像文件就好像它是一台设备，把它连接到文件系统树上：

```
mkdir /mnt/iso_image
mount -t iso9660 -o loop image.iso /mnt/iso_image
```

上面的示例中，我们创建了一个挂载点叫做/mnt/iso_image，然后把此映像文件 image.iso 挂载到挂载点上。映像文件被挂载之后，可以把它当作，就好像它是一张真正的 CD-ROM 或者 DVD。当不再需要此映像文件后，记得卸载它。

####16.7.2 擦除可度写的 CD-ROM

可重写入的 CD-RW 媒介在被重使用之前需要擦除或清空。为此，我们可以用 wodim 命令，指定设备名称和清空的类型。此 wodim 程序提供了几种清空类型。最小（且最快）的是 "fast" 类型：

```
wodim dev=/dev/cdrw blank=fast
```

####16.7.3 写入镜像文件

写入一个映像文件，我们再次使用 wodim 命令，指定光盘设备名称和映像文件名：

```
wodim dev=/dev/cdrw image.iso
```

除了设备名称和映像文件之外，wodim 命令还支持非常多的选项。常见的两个选项是，"-v" 可详细输出，和 "－dao" 以 disk-at-once 模式写入光盘。如果你正在准备一张光盘为的是商业复制，那么应该使用这种模式。wodim 命令的默认模式是 track-at-once，这对于录制音乐很有用。

###16.8 附加认证

通常验证一下我们已经下载的 iso 映像文件的完整性很有用处。在大多数情况下，iso 映像文件的贡献者也会提供一个 checksum 文件。一个 checksum 是一个神奇的数学运算的计算结果，这个数学计算会产生一个能表示目标文件内容的数字。如果目标文件的内容即使更改一个二进制位，checksum 的结果将会非常不一样。生成 checksum 数字的最常见方法是使用 md5sum 程序。当你使用 md5sum 程序的时候，它会产生一个独一无二的十六进制数字：

```
md5sum image.iso
34e354760f9bb7fbf85c96f6a3f94ece    image.iso
```

当你下载完映像文件之后，你应该对映像文件执行 md5sum 命令，然后把运行结果与发行商提供的 md5sum 数值作比较。

除了检查下载文件的完整性之外，我们也可以使用 md5sum 程序验证新写入的光学存储介质。为此，首先我们计算映像文件的 checksum 数值，然后计算此光学存储介质的 checksum 数值。这种验证光学介质的技巧是限定只对 光学存储介质中包含映像文件的部分计算 checksum 数值。通过确定映像文件所包含的 2048 个字节块的数目（光学存储介质总是以 2048 个字节块的方式写入）并从存储介质中读取那么多的字节块，我们就可以完成操作。某些类型的存储介质，并不需要这样做。一个以 disk-at-once 模式写入的 CD-R，可以用下面的方式检验：

```
md5sum /dev/cdrom
34e354760f9bb7fbf85c96f6a3f94ece    /dev/cdrom
```

许多存储介质类型，如 DVD 需要精确地计算字节块的数目。在下面的例子中，我们检验了映像文件 dvd-image.iso以及 DVD 光驱中磁盘 /dev/dvd 文件的完整性。你能弄明白这是怎么回事吗？

```
md5sum dvd-image.iso; dd if=/dev/dvd bs=2048 count=$(( $(stat -c "%s" dvd-image.iso) / 2048 )) | md5sum
```

### 拓展阅读

我们刚才谈到了很多方法，可以使用命令行管理存储介质。看看我们所讲过命令的手册页。一些命令支持大量的选项和操作。此外，寻找一些如何添加硬盘驱动器到 Linux 系统（有许多）的在线教程，这些教程也要适用于光介质存储设备。

<br />
<br />
