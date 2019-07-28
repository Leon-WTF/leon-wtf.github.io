---
title: "Linux IO model 和 JAVA NIO/AIO"
category: Java
tags: [java, linux]
---
### Linux I/O model ###
- blocking I/O
- non blocking I/O
- I/O multiplexing (select and poll)
- signal driven I/O (SIGIO)
- asynchronous I/O (the POSIX aio_ functions)

>[Chapter 6. I/O Multiplexing: The select and poll Functions](https://notes.shichao.io/unp/ch6/)

### JAVA实现 ###
JAVA最早实现是IO是blocking I/O,是面向流的，每次从流中读一个或多个字节，直至读取所有字节，没有被缓存在任何地方，且它不能前后移动流中的数据。之后推出的NIO是non blocking I/O，是面向缓冲区的，数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动。同时，借助于Selectors，它还可以实现I/O multiplexing，即reactor模式。之后的NIO 2.0又引入了asynchronous I/O，对应的可以实现Proactor模式。
```java
// java.nio.channels.SocketChannel
public abstract int read(ByteBuffer dst) throws IOException
/* Description copied from interface: ReadableByteChannel
Reads a sequence of bytes from this channel into the given buffer.
An attempt is made to read up to r bytes from the channel, where r is the number of bytes remaining in the buffer, that is, dst.remaining(), at the moment this method is invoked.
Suppose that a byte sequence of length n is read, where 0 <= n <= r. This byte sequence will be transferred into the buffer so that the first byte in the sequence is at index p and the last byte is at index p + n - 1, where p is the buffer's position at the moment this method is invoked. Upon return the buffer's position will be equal to p + n; its limit will not have changed.
A read operation might not fill the buffer, and in fact it might not read any bytes at all. Whether or not it does so depends upon the nature and state of the channel. A socket channel in non-blocking mode, for example, cannot read any more bytes than are immediately available from the socket's input buffer; similarly, a file channel cannot read any more bytes than remain in the file. It is guaranteed, however, that if a channel is in blocking mode and there is at least one byte remaining in the buffer then this method will block until at least one byte is read.
This method may be invoked at any time. If another thread has already initiated a read operation upon this channel, however, then an invocation of this method will block until the first operation is complete.

Specified by:
read in interface ReadableByteChannel
Parameters:
dst - The buffer into which bytes are to be transferred
Returns:
The number of bytes read, possibly zero, or -1 if the channel has reached end-of-stream
Throws:
NotYetConnectedException - If this channel is not yet connected
ClosedChannelException - If this channel is closed
AsynchronousCloseException - If another thread closes this channel while the read operation is in progress
ClosedByInterruptException - If another thread interrupts the current thread while the read operation is in progress, thereby closing the channel and setting the current thread's interrupt status
IOException - If some other I/O error occurs */
public abstract int write(ByteBuffer src) throws IOException
/* Description copied from interface: WritableByteChannel
Writes a sequence of bytes to this channel from the given buffer.
An attempt is made to write up to r bytes to the channel, where r is the number of bytes remaining in the buffer, that is, src.remaining(), at the moment this method is invoked.
Suppose that a byte sequence of length n is written, where 0 <= n <= r. This byte sequence will be transferred from the buffer starting at index p, where p is the buffer's position at the moment this method is invoked; the index of the last byte written will be p + n - 1. Upon return the buffer's position will be equal to p + n; its limit will not have changed.
Unless otherwise specified, a write operation will return only after writing all of the r requested bytes. Some types of channels, depending upon their state, may write only some of the bytes or possibly none at all. A socket channel in non-blocking mode, for example, cannot write any more bytes than are free in the socket's output buffer.
This method may be invoked at any time. If another thread has already initiated a write operation upon this channel, however, then an invocation of this method will block until the first operation is complete.

Specified by:
write in interface WritableByteChannel
Parameters:
src - The buffer from which bytes are to be retrieved
Returns:
The number of bytes written, possibly zero
Throws:
NotYetConnectedException - If this channel is not yet connected
ClosedChannelException - If this channel is closed
AsynchronousCloseException - If another thread closes this channel while the write operation is in progress
ClosedByInterruptException - If another thread interrupts the current thread while the write operation is in progress, thereby closing the channel and setting the current thread's interrupt status
IOException - If some other I/O error occurs */
// java.nio.channels.Selector
public abstract int select() throws IOException
/* Selects a set of keys whose corresponding channels are ready for I/O operations.
This method performs a blocking selection operation. It returns only after at least one channel is selected, this selector's wakeup method is invoked, or the current thread is interrupted, whichever comes first.

Returns:
The number of keys, possibly zero, whose ready-operation sets were updated
Throws:
IOException - If an I/O error occurs
ClosedSelectorException - If this selector is closed */
public abstract Selector wakeup()
/* Causes the first selection operation that has not yet returned to return immediately.
If another thread is currently blocked in an invocation of the select() or select(long) methods then that invocation will return immediately. If no selection operation is currently in progress then the next invocation of one of these methods will return immediately unless the selectNow() method is invoked in the meantime. In any case the value returned by that invocation may be non-zero. Subsequent invocations of the select() or select(long) methods will block as usual unless this method is invoked again in the meantime.
Invoking this method more than once between two successive selection operations has the same effect as invoking it just once.

Returns:
This selector */
```

### zero copy ###
Java NIO中提供的FileChannel拥有transferTo和transferFrom两个方法，可直接把FileChannel中的数据拷贝到另外一个Channel，或者直接把另外一个Channel中的数据拷贝到FileChannel。该接口常被用于高效的网络/文件的数据传输和大文件拷贝。在操作系统支持的情况下，通过该方法传输数据并不需要将源数据从内核态拷贝到用户态，再从用户态拷贝到目标通道的内核态，同时也避免了两次用户态和内核态间的上下文切换，也即使用了“零拷贝”，所以其性能一般高于Java IO中提供的方法。

> [Java进阶（五）Java I/O模型从BIO到NIO和Reactor模式](http://www.jasongj.com/java/nio_reactor/)
> [epoll 的本质是什么？](https://mp.weixin.qq.com/s/MzrhaWMwrFxKT7YZvd68jw)