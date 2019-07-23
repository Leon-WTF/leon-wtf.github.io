---
title: "TCP与SOCKET"
category: Network
tag: network
---
### 三次握手 ###

![tcp_connect_workflow](https://raw.githubusercontent.com/Leon-WTF/leon.github.io/master/img/tcp_connect_workflow.png)
### 四次挥手 ###

![tcp_disconnect_workflow](https://raw.githubusercontent.com/Leon-WTF/leon.github.io/master/img/tcp_disconnect_workflow.png)
### SOCKET ###

![socket_file_structure](https://raw.githubusercontent.com/Leon-WTF/leon.github.io/master/img/socket_file_structure.png)

在内核中，会维护两个队列：
- 半连接队列（syn队列），存放处于SYN-RCVD状态的socket
- 全连接队列（accept队列），存放处于ESTABLISHED状态的socket
accept函数会从全连接队列里取出一个连接，如果没有则阻塞。这里取出的是一个新的连接，叫已连接socket，不同于一开始创建的监听socket。Socket在Linux中以文件的形式存在，有自己的文件描述符，读写就如同一个文件流。socket结构里有一个发送队列和一个接收队列，里面保存着缓存sk_buff，里面可以看到完整的包结构。
### 数据接收流程 ###
网卡接收到数据后写入内存，并向CPU发起中断，操作系统执行中断程序唤醒对应socket等待队列里的进程。
如何同时监听多个socket：
- select
将进程放入所有需要监听的socket的等待队列里，如果有一个socket接收到了数据就唤醒进程，然后遍历socket列表查看是哪个socket接收到了数据。
- epoll
```c++
int epfd = epoll_create(...)
epoll_ctl(epfd, ...)
while(1){
    int n = epoll_wait(...)
    for (接收到数据的socket){
        //处理
    }
}
```
利用epoll_ctl将需要监视的socket添加到epoll_create创建的对象（eventpoll）中，然后利用epoll_wait进行阻塞，将进程放入eventpoll的等待队列中。当有socket接收到数据，中断程序将其被eventpoll的就绪列表所（rdllist）引用，然后唤醒进程，进程只需要遍历就绪列表就可以了。socket在eventpoll中利用红黑树存储，而就绪列表利用双向链表存储。