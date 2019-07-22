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

### zero copy ###
Java NIO中提供的FileChannel拥有transferTo和transferFrom两个方法，可直接把FileChannel中的数据拷贝到另外一个Channel，或者直接把另外一个Channel中的数据拷贝到FileChannel。该接口常被用于高效的网络/文件的数据传输和大文件拷贝。在操作系统支持的情况下，通过该方法传输数据并不需要将源数据从内核态拷贝到用户态，再从用户态拷贝到目标通道的内核态，同时也避免了两次用户态和内核态间的上下文切换，也即使用了“零拷贝”，所以其性能一般高于Java IO中提供的方法。

> [Java进阶（五）Java I/O模型从BIO到NIO和Reactor模式](http://www.jasongj.com/java/nio_reactor/)
