[TOC]

# IP协议

## 一、什么是IP协议

> IP协议，即Internet Protocol，是TCP/IP协议簇的**网络层协议**，主要负责路由的选择与数据的转发，提供**无连接、不可靠、尽力而为**的服务。

### 1、IPv4协议报文格式

<img src="https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208091315270.jpg" alt="img" style="zoom:150%;" /> 

- 4位版本：指定IP协议的版本，例如IPv4就是0100(4)。
- 4位首部长度：指定报文首部(即**<font color=red>20字节固定内容</font>**+选项)的长度，**以四字节为单位**，因此实际长度是**该字段 * 4**得到的。
- 8位服务类型(TOS)：用于规定本数据报的处理方式，**大多数情况下网络并未对TOS进行处理**。
- 16位总长度：**报文首部+数据**的总长度。
- 16位标识：用来**唯一地标识一个IP数据报**。如果该报文被分片，那么每一片的标识id都是相同的。
- 3位标志：最高位保留，暂无用处；**中间位为1表示该数据报禁止分片**；**最低位为1表示该报文是最后一个分片**。
- 13位片偏移：**分片相对于原始IP报文开始处的偏移量**。以八字节为单位，实际偏移的字节数是这个值 * 8 得到的。因此，除了最后一个报文之外，其他报文的长度必须是8的整数倍(否则报文就不连续了)。
- 8位生存时间(TTL,Time To Live)：表示**该数据报还可以被路由器转发几次**。每经过一个路由，TTL都会减一，如果减到0还没有到达，则数据报被丢弃。该字段主要用于**防止路由器循环**问题。
- 8位协议：表示携带的载荷用到的协议。TCP为0x06，UDP为0x11。
- 16位首部校验和：用来检验报文首部是否有损坏。
- 源IP与目的IP：发送端IP和接收端IP。

#### 关于分片

数据传输的大小受到MTU(最大传输单元，Maximum Transmission Unit)的限制。因此，对于过大的数据包，需要进行分片处理，每个片都是一个IP数据报。

但是分片发送数据会导致数据丢包的几率更大。

### 2、IP地址

IP地址的基本格式为`xxx.xxx.xxx.xxx`，由四段组成，每个字段是一个字节，总共4字节，每个字节有8位，最大值是255。

IP地址由两大部分组成：**网络号**(前三个字节)和**主机号**(最后一个字节)，二者是主从关系。

- 网络号：它标志主机（或路由器）所连接到的网络，网络地址表示其属于互联网的哪一个网络 

- 主机号：它标志该主机（或路由器）属于网络中的哪一台主机。


###  3、子网划分与CIDR

过去曾经提出一种划分网络号和主机号的方案，把所有IP地址分为五类：

<img src="https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208091315269.jpg" alt="img" style="zoom:150%;" />

<img src="https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208091315277.jpg" alt="img" style="zoom:150%;" /> 

> 一个IP的组成为：`<网络号>.<主机号>`
>

然而实际上，**大多数组织会申请B类IP**，因为A类IP的主机数过多，大多数情况下不会存在一个子网下有这么多主机，C类IP主机数又太少，所以导致大量的IP被浪费了。

针对此情况，又提出了一个新的子网划分的方式：CIDR

> CIDR，即无分类域间路由选择(Classless Inter-Domain Routing)。它完全摒弃了传统的IP分类方式，而使用子网掩码区分一个IP的网络号和主机号，使得网段划分更加灵活。
>

子网掩码是32位整数，由高位的n个连续1和低位的32-n个连续0组成。将子网掩码同IP地址按位与，得到的就是网络号，其余的就是主机号，比如说：

`IP : 141.14.72.11  子网掩码 : 255.255.255.0`

两者按位与得到141.14.72.0，这就是网络号，而11是主机号。

因此，新的IP表达方式为：`<网络号>.<主机号>/<网络号的位数>`，所谓**网络号的位数就是子网掩码前缀1的数目**。

利用这种方法，上述例子中的IP就是`141.14.72.24/24`。

### 4、特殊的IP地址 

- 将IP地址中的主机地址==全部设为0==，就成为了**网络号**，代表这个局域网；

- 将IP地址中的主机地址==全部设为1==，就成为了**广播地址**，用于给同一个链路中**相互连接的所有主机**发送数据包; 

- 127.*的IP地址用于**本机环回**(loop back)测试，通常使用`127.0.0.1`。

### 5、公网IP和私网IP

> 公网IP：由Inter NIC（因特网信息中心）负责，用来分配给向Inter NIC提出申请的组织机构。
>
> 公有IP**<font color=red>全球唯一</font>**，通过它可以直接访问因特网（**<font color=red>能直接上网</font>**)。

公网IP主要有A、B、C、D、E五类地址:

- A类:地址范围 `1.0.0.0 ~ 127.255.255.255`，主要分配给大量主机而局域网网络数量较少的**大型网络**；

- B类:地址范围`128.0.0.0 ~ 191.255.255.255`，一般用于**国际性大公司和机构**；

- C类:地址范围`192.0.0.0 ~ 223.255.255.255`，一般用于**小公司、校园网、研究机构**等；

- D类:地址范围`224.0.0.0 ~ 239.255.255.255`，第一个字节以`1110`开头，用做**广播地址**；

- E类:地址范围240.0.0.0 ~ 255.255.255.255`，**暂时保留**。


> 私网IP：属于非注册地址，专门为组织机构内部组建局域网使用，**不能直接上网**。

私网IP主要有A、B、C三类：

- A类:地址范围`10.0.0.0 ~ 10.255.255.255`。1个A类网络地址，共约1677万个IP地址。

- B类:地址范围`172.16.0.0 ~ 172.31.255.255`。16个B类网络地址，共约104万个IP地址。

- C类:地址范围`192.168.0.0 ~ 191.168.255.255`。256个C类网络地址，共约65536个IP地址


> 注：私网IP分别是公网IP中A、B、C三类地址中的一部分，只是被划分出来单独用作私网组建。
>

### 6、IP地址的分配(DHCP)

管理员可以通过手动的方式分配IP地址，也可以通过DHCP(动态主机配置协议)来完成。

通过配置，DHCP可以为想要入网的主机**分配固定的IP地址**，也可以**分配临时的IP地址**，每次网络连接时该地址可能是不同的。

除了地址分配，DHCP协议还会允许主机获得子网掩码、默认网关、DNS服务器等**网络配置信息**，使其能够顺利联网。

上述过程通过**报文交互**的方式完成。



## 二、NAT/NAPT技术

> NAT，即网络地址转换(Network Address Translation)，运行在路由设备上，是一种**把私网IP转换为公网IP**的技术，用于**解决IPv4地址不足的问题**。
>
> NAPT，即网络地址端口转换(Network Address Port Translation)，在NAT的基础上又加入了端口映射。

### 1、接口与IP的关系

**一台主机通常只有一条物理链路链接到网络**，主机与物理链路之间的边界称为==接口==。而对于路由器，**至少需要一条链路接收数据、另一条链路发送数据**，因此路由器有多个接口(至少两个)。

每台主机与路由器都能发送和接收IP数据报，因而IP要求每台主机和路由器<font color=red>**接口都拥有自己独立的IP地址**</font>。因此，从技术上讲，**一个IP地址与一个接口相关联，而不是与包含该接口的主机或路由器相关联**。

### 2、路由器的LAN口与WAN口

WAN，即*Wide Area Network*，代表广域网，是**路由器与运营商网线相连**的接口，对应WAN口IP。

LAN，即*Local Area Network*，代表局域网，是**计算机与路由器相连**的接口，对应LAN口IP。

### 3、NAPT的基本工作原理

1. 当局域网内的某台主机想要访问公网上的某台服务器时，它会将数据报发送到路由器LAN口，数据报中包含**源IP**和**TCP/UDP协议的源端口号**、**目的IP**和**TCP/UDP协议的目的端口号**。

2. NAPT路由器内部维护一张NAPT转换表。NAPT路由器收到数据报后，会为其生成没有出现在NAPT转换表中的**新端口号替换源端口号**，并用**WAN口IP替换源IP**，从而形成(新的IP,端口号)，将==(新IP,端口号):(源IP,源端口号)==的映射关系存储在表中。

3. 数据报的传输可能会经过N跳路由器，每一跳都会经历**上述IP和端口号替换**的过程。

4. 数据报到达服务器后，服务器会发送响应报文。报文每到达一个路由器，就通过查找NAPT转换表的方式**找到源IP和源端口**，直到到达目的主机。

> 注：NAT与NAPT的区别在于，NAT内部使用了不同的IP进行映射，而NAPT使用了一个WAN口IP和不同的端口号进行映射。

 

## 三、IPv4与IPv6

> IPv4采用32位地址长度(**4字节**)，约有43亿地址.
>
> IPv6地址为128位(**16字节**)，但通常写作8组，每组为四个十六进制数的形式。
>
> 例如：FE80:0000:0000:0000:AAAA:0000:00C2:0002
>

 

## 四、ICMP协议

> ICMP协议，即控制报文协议(Internet Control Message Protocol)，是网络层协议之一，起到辅助IP协议的作用。

- ICMP的作用分为两类：**错误通知和信息查询**，可以用来确认IP包是否成功到达目标地址、通知在发送过程中IP包被丢弃的原因......
- ICMP的报文是放在IP数据报里的，因此，ICMP是IP的上层协议，但是工作在网络层，而非传输层。
- ICMP只能搭配IPv4使用，而IPv6需要使用ICMPv6

### 1、关于ping命令

ping命令基于ICMP协议，用来**测试网络的连通性**，其工作的基本过程为：

1. ping命令会先发送一个ICMP Echo Request给对端; 
2. 对端接收到之后，返回一个ICMP Echo Reply。

基本使用格式：ping+主机名/IP地址，如：ping www.baidu.com

> 注：ping命令基于网络层协议，因此不存在端口一说。

### 2、关于traceroute命令

traceroute命令同样基于ICMP协议，用来检测发出数据包的主机到目标主机之间所经过的**网关数量**。

基本使用格式：traceroute+主机名/IP地址，如：traceroute www.baidu.com

 

 

 
