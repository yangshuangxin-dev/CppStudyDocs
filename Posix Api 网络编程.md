# Posix Api 网络编程

- 由于linux有众多的发行版本，包括mac系统等类uninx系统，不同的操作系统的api可能是不相同的。posix api 是一组屏蔽底层差异的api，使相同的代码可以在不同的操作系统下进行编译和运行。

|  客户端   |  服务器   |  网络IO   |
| :-------: | :-------: | :-------: |
| socket(); | socket(); | select(); |
| bind(); (可选) | bind(); | poll(); |
| connetc();(可选) | linten(); | epoll_cleate(); |
| send(); | accept(); | epoll_ctl(); |
| recv(); | send(); | epoll_wait(); |
| close(); | recv(); | fcntl(); |
|  | close(); |  |



socket 含义是插座，网络的IO其实就是客户端就像插座一样，插到服务器端进行数据的通信。数据就如流（stream）一样进行传输。

- socket(); 函数调用后，内核的操作是：
	1. 分配一个int类型的句柄fd
	2. alloc()分配一块内存，建立一个传输控制块（tcb）
- listen(); 函数调用后，内核的操作是：
  1. 改变tcb的状态，tcb->status = TCP_STATUS_LISTEN
  2. 创建tcb的半连接syn队列，tcb->syn_queue
  3. 创建tcb的全连接accept队列，tcb->accept_queue



```sequence
Title: TCP三次握手
participant  client
participant server
note right of server: listen()
Note left of client: connect()
client -> server: syn,seqnum=1234
note over client: SYN_SENT
note right of server: 创建控制块tcb，加入syn_queue队列
server --> client: ack,ackNum = 1235; seqnum = 5678
note over server: SYN_RCVD
client -> server: ack, ackNum = 5679
note over client: ESTABLISTED
note right of server: tcb从syn_queue队列转移到accept_queue队列
note right of server: accept()，从accept_queue中取出tcb，分配fd
note over server: ESTABLISTED
```

- tcp的连接周期，从收到第一个syn包，server创建出控制块tcb开始。
- 第三次握手时的数据包，按照五元组：协议、源ip，源端口、目的ip、目的端口，从半连接队列中查找匹配到tcb节点。
- 为了应对syn泛滥，listen（）函数有两个参数中的第二个参数进行限制。

`lenten(fd，backlog);` 第二个参数backlog经过linux内核版本的不断迭代，有不同的含义，其中：

1. 初始版本就是syn队列的长度，应对syn泛滥攻击。
2. 迭代版本一，syn+accept队列的长度，考虑到未分配fd的tcb数量。
3. 迭代版本二，随着防火墙等应该网络攻击的手段越来越多，同时需要处理的tcp连接数量越来越多，现在新的linux内核版本为accept队列的长度。

`accept();` 函数在内核中操作有：

1.  分配一个未使用的fd
2. fd指向三次握手后的tcb

| listen fd |     client fd     |
| :-------: | :---------------: |
| accept(); | send(); / recv(); |



|            水平触发（LT）            |                边缘触发（ET）                |
| :----------------------------------: | :------------------------------------------: |
| 只要缓冲区有数据，就一直触发读写信号 | 即使缓冲区中有许多数据，也只触发一次读写信号 |

如果linten 的fd是边缘触发的，需要使用循环进行处理，代码实现如下：

```c
while(1) {
    fd = accept();
    if (fd == -1) {
        break;
    }
}
```

Posix 的网络接口都是异步的，调用后不代表数据已经发生，仅仅是把数据拷贝到内核缓冲区中，具体何时发送由内核决定。网卡MTU一般设置为1500，意味着内核会把缓冲区的数据进行打包，最多的1500字节进行发送。这会导致粘包和分包。

如下调用，内核会打包成一个1500的数据包进行发送：

```c
send(fd, buffer, 100, 0);
send(fd, buffer, 200, 0);
send(fd, buffer, 400, 0);
send(fd, buffer, 800, 0);
```

`send(fd, buffer, 2000, 0);` 内核会差分为1500的包和500的包进行发送。

```sequence
title: 网络数据发送
participant 发送端应用层
participant 发送端内核
participant 接收端内核
participant 接受端应用层

发送端应用层 -> 发送端内核: copy from user 2000
发送端内核 -> 接收端内核: 1500 byte
发送端内核 -> 接收端内核: 500 byte
接收端内核 -> 接受端应用层: 2000 byte

```



TCP 保证传输的数据是不丢失的，有序的核心方法有：

1. 慢启动，传输的数据包先指数级增长，后线性增长
2. 拥塞控制，数据包拥塞，直接减少一半传输数据量
3. 滑动窗口，反馈当前可一次性接受数据的大小
4. 延迟确认，反馈收到是数据包序号
5. 超时重传，发送端一直没有收到确认消息，重新发送数据包

```sequence
title: tcp四次挥手
participant 主动方
participant 被动方
note left of 主动方: close()
主动方 -> 被动方: fin
note over 主动方: fin_wait_1
被动方 --> 主动方: ack
note over 主动方: fin_wait_2
note over 被动方: close_wait
note right of 被动方: close();
被动方 -> 主动方: fin
note over 主动方: time_wait
note over 被动方: last_ack
主动方 --> 被动方: ack
note right of 被动方: closed

```

被动方关闭流程：

```c
ret = recv();
if (ret == 0) {
    close();
}
```

如果四次挥手时ack没有收到，先收到fin，与双方同时调用close() 时，双方的状态变化是一致的，fin_wait1 ->  closing -> time_wait



TCP做p2p时，两个客户端可以不经过服务器直接通信，使用场景物联网，万物互联数据不需要经过服务器，连个客户端直接互相通信。

```sequence
title: p2p
client1 -> client2: syn
client2 -> client1 : syn
client1 -> client2: ack, syn
client2 -> client1 : ack, syn
client1 -> client2: ack
note right of client2: ESTBALISHED
client2 -> client1 : ack
note left of client1: ESTBALISHED


```

