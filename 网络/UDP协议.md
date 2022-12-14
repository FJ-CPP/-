[TOC]



# UDP协议

## 一、协议需要解决的根本问题

- 如何将报头和有效载荷分离
- 将有效载荷交付给上层的哪一个协议



##  二、UDP协议

### 1、什么是UDP协议

> UDP协议，即用户数据报协议(User Datagram Protocol)，它是一种<font color=red>无连接</font>的**传输层协议**，提供面向事务的<font color=red>简单不可靠</font>信息传送服务。

UDP的特点包括：

1. ==无连接==：只要知道对端的IP及端口号即可通信，没有TCP三次握手建立连接和四次挥手断开连接的消耗，因此比较**高效**。
2. ==不可靠==：UDP只提供尽最大努力的交付，没有确认机制，并且在数据发生丢失时，也不会进行重传。
3. ==面向数据报==：对于应用层交付的报文，UDP会直接将其存放到UDP报文的数据部分，在添加简单的报头后直接交付给下层的IP协议，而不会对应用层的报文进行拆分与合并的工作。
4. 报头很短，只有8字节。
5. UDP的socket能同时进行读和写，是全双工的。
6. ==没有拥塞控制==，因此网络出现的拥塞不会使源主机的发送速率降低，这对某些实时应用是很重要的。很多的实时应用（如：IP电话、实时视频会议等）要求源主机以**恒定的速率发送数据**，并且允许在网络出现拥塞时丢失一部分数据，但却**不允许数据有太大的时延**。UDP 协议正好适合这种要求。

<img src="https://mypicture-1307604235.cos.ap-nanjing.myqcloud.com/mytyporaimage-20220803212925328.png" alt="image-20220803212925328" style="zoom: 67%;" />

### 2、UDP数据报的格式

![img](https://mypicture-1307604235.cos.ap-nanjing.myqcloud.com/mytyporawps2.jpg) 

- 源端口号：数据报发送方的端口号
- 目的端口号：数据报接收方的端口号
- UDP长度：UDP数据报的长度，包括报头的长度和数据的长度，由于报头占据8字节，因此UDP长度**<font color=red>最小为8字节</font>**；由于UDP长度最大是16位，因此UDP能传输的**最大数据量是65535字节**(`2^16-1`Byte)
- UDP校验和：用来校验数据在传输过程中是否损坏，如果损坏则丢弃

> 注：数据报的报头可以通过位段来定义

### 3、UDP如何解决协议的根本问题？

- 对于“将报头和有效载荷分离”：报头部分是前8个字节，因此可以通过`UDP长度-报头长度(8字节)`来分离出有效载荷部分。
- 对于“将有效载荷交付给上层的哪一个协议”：由于应用层对应的端口号是确定的，因此可以通过目的端口号来确定交付给哪个上层应用。

### 4、UDP缓冲区

#### I.接收缓冲区

UDP的接收缓冲区**不能保证**收到的UDP报的顺序和发送UDP报的顺序一致。

如果缓冲区满了，再到达的UDP数据就会被**丢弃**。

#### II.发送缓冲区

UDP**不存在发送缓冲区**，只要有数据就会直接交给内核，然后再由内核转交网络层协议进行传输。



## 三、UDP如何实现可靠传输和速率控制

参考TCP协议，在应用层实现：==确认应答、添加发送缓冲区用于超时重传、滑动窗口和拥塞控制==。

注：**RUDP**、**RTP**和**UDT**都是基于UDP实现的可靠数据传输协议。



## 四、基于UDP的应用层协议

> DNS: 域名解析协议 53端口
>
> DHCP: 动态主机配置协议 68端口
>
> TFTP: 简单文件传输协议 69端口
>
> ...

