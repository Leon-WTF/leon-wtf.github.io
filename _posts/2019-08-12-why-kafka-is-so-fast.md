---
title: "Why Kafka is so fast？"
category: Kafka
tag: kafka
---
- Batching of multiple published messages to yield less frequent network round trips to the broker
- Support for multiple in-flight messages
- End-to-end compression
- Partitioning of topics across multiple brokers in a cluster
- Immediately write data to a persistent log on the filesystem without necessarily flushing to disk(transferred into the kernel's pagecache) rather than accumulate them in JVM heap
- Maximized use of sequential disk reads and writes
>Kafka does not always access disk sequentially but it does some things that make it much more likely that disk access is often sequential. All Kafka messages are stored in larger segment files (1GB each by default) and since Kafka messages are not deleted when consumed (like in other message brokers) Kafka will not end up creating a fragmented filesystem over time by continuously creating and deleting many variable length files. Instead it creates segment files and then appends to that file until it reaches 1GB (a configurable limit). Only when all messages in the segment expire will it delete the entire 1GB segment. This means that often these 1GB sections of disk are actually laid out as contiguous blocks. It is a recommended best practice to keep these Kafka commit log files on a dedicated filesystem so it does not get fragmented by other apps reading and writing variable length files into the same filesystem. More importantly most reading an writing to these segment files is sequential and goes through OS page cache so as to reduce disk I/O even further by caching the most often accessed pages in memory. This is why it is a recommendation to tune the kernel to set swappiness to 1 to reduce the likelihood that these cached pages would get swapped out of memory.
- Use queue instead of tree as the data structure to avoid random I/O
- Zero-copy processing of messages
- Limit minimum bytes return to consumer to yield less frequent network round trips

[Introduction to apache kafka](https://www.slideshare.net/ShiaoAnYuan/introduction-to-apache-kafka-84009739)
[SocketServer与NIO-1+N+M 模型](https://blog.csdn.net/chunlongyu/article/details/53036414)
[Documentation](http://kafka.apache.org/documentation/)

In Kafka, the index and timeindex file is read/written using memory mapped file, but the segment log file is read/written using regular file I/O, the reason can be found below:
> read() calls usually don't have the overhead of page faulting. Page faulting is seriously expensive, so if you're page faulting frequently you're going to slow things around you. And as seen above the main logic behind mmap() is to page fault on the address if the contents of the file are not in memory. If you're doing a sequential read/write then it is better to perform I/O via your usual read() and write() calls as the copy overhead is negligible when compared to the overhead incurred because of frequent page faults. And usually all our file accesses fall into this category.

[How is a mmaped file I/O different from a regular file I/O with regard to the kernel?](https://www.quora.com/How-is-a-mmaped-file-I-O-different-from-a-regular-file-I-O-with-regard-to-the-kernel)
