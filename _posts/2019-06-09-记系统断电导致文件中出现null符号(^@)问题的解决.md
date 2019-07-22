---
title: "记系统断电导致文件中出现null符号(^@)问题的解决"
category: Linux
tag: linux
---
我们的前端数据采集系统是用的树莓派，上面跑着基于Debian的Linux操作系统，用的是EXT4的文件系统格式。之前发现采集到的日志文件末尾偶尔会出现大量的^@，经过一番调研后发现是由于系统突然断电导致的。然后再经过一番google，说是由于文件的metadata修改后变大，但是实际数据由于断电没有写入硬盘，所以会用null符号（^@）来填充。然后想了很多办法来修复这种断电后产生的异常数据（之前因为解决操作系统卡死的问题给我们的盒子加了自动重启线，每隔3天就会自动重启），一直都没有很好的解决，直到听极客时间的Linux操作讲到文件系统的journal mode，再经过一番查找，终于理解了这个问题原理，也找到了解决办法。

EXT4是一种journaling文件系统，使用了类似于[MySQL](https://segmentfault.com/a/1190000019309240)中Transaction的概念来保证一致性，把对文件的操作看做是一次事务，并且也使用了先写redo log的机制。分为以下三种模式（可以用dmesg | grep EXT4查看：mounted filesystem with ordered data mode. Opts: (null)）
- journal
将数据及元数据都先写入日志文件并落盘，commit日志文件，然后再将数据和元数据写入硬盘正确的位置(In-place write)，异步删除已经写入硬盘正确位置的日志文件数据。如果发生系统crash，可以replay日志文件中已经commit的数据进行恢复。这种模式最安全但性能较差（除了数据需要同时被写入和读取），因为数据被写入了两次。流程如下：

![ext4_journal_mode](https://raw.githubusercontent.com/Leon-WTF/leon.github.io/master/img/ext4_journal_mode.png)

- ordered（默认）
只有元数据写入日志文件，但保证先将数据写入硬盘，再commit日志文件，流程图如下（第1,2步可以同时进行，只要保证在第3步前完成就行）：

![ext4_ordered_mode](https://raw.githubusercontent.com/Leon-WTF/leon.github.io/master/img/ext4_ordered_mode.png)

如果系统crash，可以通过replay日志文件恢复元数据，保证文件系统一致性，且如果存储的数据是复用已有的文件block来存数据，可以保证数据已经落盘，但由于存在Delayed allocation机制，如果文件需要分配新的block来写入，则系统会推迟将近1分钟才真正刷入硬盘，会导致数据丢失，这也是为什么会看到文件出现^@来填补真实的数据。
> "Delayed allocation" means that the filesystem tries to delay the allocation of physical disk blocks for written data for as long as possible. This policy brings some important performance benefits. Many files are short-lived; delayed allocation can keep the system from writing fleeting temporary files to disk at all. And, for longer-lived files, delayed allocation allows the kernel to accumulate more data and to allocate the blocks for data contiguously, speeding up both the write and any subsequent reads of that data. It's an important optimization which is found in most contemporary filesystems.
> But, if blocks have not been allocated for a file, there is no need to write them quickly as a security measure. Since the blocks do not yet exist, it is not possible to read somebody else's data from them. So ext4 will not (cannot) write out unallocated blocks as part of the next journal commit cycle. Those blocks will, instead, wait until the kernel decides to flush them out; at that point, physical blocks will be allocated on disk and the data will be made persistent. The kernel doesn't like to let file data sit unwritten for too long, but it can still take a minute or so (with the default settings) for that data to be flushed - far longer than the five seconds normally seen with ext3. And that is why a crash can cause the loss of quite a bit more data when ext4 is being used.

- writeback
只有元数据写入日志文件，但不保证数据写入硬盘什么时候发生。有可能系统重启后元数据被恢复，但数据因为并没有写入硬盘而丢失。流程图如下：

![ext4_writeback_mode](https://raw.githubusercontent.com/Leon-WTF/leon.github.io/master/img/ext4_writeback_mode.png)

最后将文件系统在mount时设为journal模式，用性能来换取强一致性。修改***/etc/fstab***：
```
/dev/mapper/sd03 /pi/ ext4 defaults,data=journal 0 0
```

> [ext4 and data loss](https://lwn.net/Articles/322823/)