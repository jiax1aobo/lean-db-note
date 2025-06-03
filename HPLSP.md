High Performance Linux Server Programming

# 一、TCP/IP协议族

## 1.1 TCP/IP协议族体系结构及主要协议

TCP/IP协议族是自底向上的四层协议系统。

### 1.1.1 数据链路层

**该层实现了网卡接口的网络驱动程序，用以处理数据在物理媒介（如以太网、令牌环等）上的传输。**

本层常用协议：**ARP协议**（Address Resolve Protocol，地址解析协议）和RARP协议（Reverse Address Resolve Protocol，逆地址解析协议）。它们实现了IP地址和机器物理地址（通常是MAC地址）之间的相互转换。

网络层使用IP地址寻址一台机器，数据链路层使用物理地址寻址一台机器。因此网络层需将IP地址转换为物理地址才能使用数据链路层提供的服务，这便是ARP协议的用途。

### 1.1.2 网络层

**该层实现数据包的路由和转发。**WAN（Wide Area Network，广域网）通常使用众多分级的路由器来连接分散的主机或LAN（Local Area Network，局域网），因此通信的两台主机一般是通过多个中间节点（路由器）间接连接的。**网络层的任务就是选择这些中间节点，以确定两台主机之间的通信路径。**网络层对上层协议隐藏了网络拓扑链接的细节，使得在传输层和网络应用程序看来，通信的双方是直接相连的。

网络层最核心的协议是**IP协议**（Internet Protocol，因特网协议）。IP协议根据数据包的目标IP地址决定如何投递它。若数据包不能直接发送给目标主机，IP协议就为它寻找合适的下一跳（next hop）路由器，经过多次这样的逐跳（hop by hop）的方式数据包最终被送达目标主机，或者由于发送失败而被丢弃。

网络层另一个重要协议是**ICMP协议**（Internet Control Message Protoco，因特网控制报文协议）。它是IP协议的重要补充，主要用于检测网络连接。需要指出的是该协议并非严格的网络层协议，因为它使用同一层的IP协议提供的服务（一般来说，上层协议使用下层协议提供的服务）。

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250523095026891.png" alt="image-20250523095026891" style="zoom:67%;" />

8位类型字段用于区分报文类型。它将ICMP报文分为两大类：一类是差错报文，此类报文主要用来回应网络错误，如目标不可达（类型为3）和重定向（类型为5）；另一类是查询报文，此类保温用来查询网络信息，如`ping`程序就是使用ICMP报文查看目标是否可达（类型为8）的。8位代码字段用来进一步细分不同的条件。如重定向类型的报文用代码值0表示对网络重定向，代码值1表示对主机重定向。16位校验和对整个报文（头部和内容部分）进行CRC校验。

### 1.1.3 传输层

**该层为两台主机上的应用程序提供端到端（end to end）的通信。**与网络层不同，传输层不关心数据包的中转过程，只在乎通信的起始端和目的端。传输层为应用程序封装了一条端到端的逻辑通信链路，它负责数据的收发、链路的超时重连等。

传输层的第一个主要协议是**TCP协议**（Transmission Contro Protoc，传输控制协议），它为应用层提供可靠的、面向连接的和基于流（stream）的服务。TCP协议使用超时重传、数据确认等方式确保数据包被正确的发送至目的端，因此TCP协议是可靠的。使用TCP协议通信的双方必须先建立TCP连接，并在内核中为该连接维持一些必要的数据结构，比如连接的状态、读写缓冲区，以及诸多定时器等。当通信结束时，双方必须关闭连接已释放这些内核数据。TCP服务是基于流的。基于流的数据没有边界（长度）限制，它源源不断地从通信的一端流入另一端。发送端可以逐个字节地向流写入数据，接收端也可以逐个字节地将它们读出。

与TCP协议相对应的另一个传输层主要协议是**UDP协议**（User Datagram Protocol，用户数据报协议），它为应用层提供不可靠、无连接和基于数据报的服务。“不可靠”意味着UDP协议无法保证数据从发送端正确地传送到目的端。如果数据在中途丢失，或者目的端通过数据校验发现数据错误而将其丢弃，则UDP协议只是简单地通知应用程序发送失败。无连接意味着应用程序每次发送数据都要明确指定接收端的地址。基于数据报的服务，是相对于流的服务而言，每个UDP数据包都有一个长度，接收端必须以该长度的最小单位将其所有内容一次性读出，否则数据将被截断。

### 1.1.4 应用层

**该层负责处理应用程序的逻辑。**此下的三层负责处理网络通信细节，这部分必须既稳定又高效，因此它们都在内核空间中实现。而应用层则在用户空间实现。

ping是应用程序，而不是协议，它利用ICMP报文检测网络连接，是调试网络环境的必备工具。**telnet协议**是一种远程登录协议，它使我们能在本地完成远程任务。**OSPF**（Open Shortest Path First，开放最短路径优先）协议是一种动态路由更新协议，用于路由器之间的通信，以告知对方各自的路由信息。**DNS**（Domain Name Service，域名服务）协议提供机器域名到IP地址的转换。

## 1.2 封装

上层协议通过封装（encapsulation）使用下层协议提供的服务。即，应用程序数据发送到物理网络上之前，将沿着协议栈从上往下依次传递。每层协议都将在上层数据的基础上加上自己的头部信息（有时也包括尾部信息），以实现该层的功能。

经TCP封装后的数据称为TCP报文段（TCP message segment），简称TCP段。TCP协议为通信双方维持的连接信息存储在内核中的TCP头部信息和TCP内核发送缓冲区中，正是这两部分一起构成了TCP报文段。

经UDP封装后的数据称为UDP数据报（UDP datagram）。与TCP报文段不同的是，UDP无需为应用层数据保存副本，当一个UDP数据报超过发出后，UDP内核缓冲区的该数据报就被丢弃（因为它提供的是不可靠的服务，无需维护这些信息）。如需重新发送该数据报，应用程序需要重新从用户空间将数据拷贝到UDP内核发送缓冲区中。

经IP封装后的数据成为IP数据包（IP datagram）。它包含IP头部信息和数据部分，数据部分就是一个TCP报文段、UPD数据报或者ICMP报文。

经数据链路层封装的数据称为帧（frame）。传输媒介不同，帧的类型也不同。比如，以太网上传输的是以太网帧（Ethernet frame），而令牌环网络上传输的是令牌环帧（token ring frame）。

帧才是最终在物理网络上传输的字节序列。至此，封装过程完成。

## 1.3 分用

当帧到达目的主机是，将沿着协议栈自底向上依次传递。各层协议一次处理帧中本层负责的头部数据，一获取所需信息，最终将处理后的帧交给目标应用程序。这个过程称为分用（demultiplexing）。**分用是依靠头部信息中的类型字段实现的。**

这里算是解释了各层协议的数据报文中的报文类型是做什么的了，它的作用是表明本层的服务是为谁提供的，因为上层可能有多个协议都是用该协议提供的服务，多个协议通过类型来判断这个报文应该发送给上层的哪一个协议模块。

## 1.4 测试网络

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250522145814059.png" alt="image-20250522145814059" style="zoom:67%;" />

## 1.5 ARP协议工作原理

**ARP协议能够实现任意网络层地址（IP地址）到任意物理地址（MAC地址）的转换。**其工作原理是：主机向自己所在的网络广播一个ARP请求，该请求包含目标机器的网络地址。此网络中的其他机器都将收到这个请求，但只有被请求的目标机器会回应一个ARP应答，其中包含了被请求机器的物理地址。

### 1.5.1 以太网ARP请求/应答报文详解

格式如下（括号内为字节长度）：
```
| 硬件类型(2) | 协议类型(2) | 硬件地址长度(1) | 协议地址长度(1) | 操作(2) | 发送(2) |~
~| 发送端以太网地址(6) | 发送端IP地址(4) | 目的端以太网地址(6) | 目的端IP地址(4) |
```
### 1.5.2 ARP高速缓存的查看和修改

通常，ARP维护一个（动态变化的）高速缓存，其中包含进程访问（比如网关地址）或最近访问的机器的IP地址到物理地址的映射。如此便避免了重复的ARP请求，提高了发送数据包的速度。

### 1.5.3 使用tcpdump观察ARP通信过程

## 1.6 DNS工作原理

### 1.6.1 DNS查询和应答报文详解

DNS是一套分布式的域名服务系统。每个DNS服务器上都存放着大量的机器名和IP地址的映射，并且是动态更新的。众多网络客户端程序都是用DNS协议来向DNS服务器查询目标主机的IP地址。

```
| 16位标识 | 16位标志 | 16位问题个数 | 16位应答资源记录个数 |~
~| 16位授权资源记录数目 | 16位额外的资源记录数目 |~
~| 查询问题(变长) | 应答(资源记录数目可变，变长) |~
~| 授权(资源记录数目可变，变长) |~
~| 额外信息(资源记录数目可变，变长) |
```

- 标识用于标识一对DNS查询和应答，以此区分一个DNS应答是哪个DNS查询的回应。
- 标志字段用于协商具体的通信方式和反馈通信状态。
- 接下来的四个字段分别指出DNS报文的最后4个字段的资源记录数目。
    - 对查询报文而言，它一般包含1个查询问题，而应答资源记录数、授权资源记录数和额外资源记录数则为0。
    - 应答报文的应答资源记录数则至少为1，而授权资源记录数和额外资源记录数可为0或非0。
- 查询问题字段包含三部分：查询名（封装好的要查询的主机名）、查询类型（如何执行查询，如获取主机IP地址、获取主机别名、反向查询）和查询类（通常为1，标识获取IP地址）。
- 应答字段、授权字段和额外信息字段都使用资源记录（Resource Record，RR）格式（包括32位域名、16位类型、16位类、32位生存时间、16位资源数据长度和变长的资源数据）。

**递归查询**是指如果目标DNS服务器无法解析请求的主机名则会向其他DNS服务器继续查询，如此递归，直到获得结果并把该结果返回给客户端。
**迭代查询**是指如果目标DNS服务器无法解析请求的主机名则会将自己知道的其他DNS服务器的IP地址返回给客户端，以供客户端参考。

### 1.6.2 Linux下访问DNS服务

Linux下的DNS服务器地址存放文件：`/etc/resolv.conf`
Linux下常用的访问DNS服务器的客户端程序是`host`

### 1.6.3 使用tcpdump观察DNS通信过程

## 1.7 socket和TCP/IP协议族的关系

由于数据链路层、网络层和传输层协议是在内核中实现的，因此操作系统提供了一组系统调用来让应用程序能够访问这些协议提供的服务。实现这组系统调用的API主要有两套：socket（常用）和XTI（基本不使用）。

由socket实现的这一组API提供两点功能：
1. 将应用程序数据从用户缓冲区中复制到TCP/UDP内核发送缓冲区，以交付内核来发送数据，或者是从内核TCP/UDP内核接收缓冲区中复制数据到用户缓冲区，以读取数据。
2. 应用程序可以通过这组API来修改内核中各层协议的某些头部信息或者其他数据结构，从而精细地控制底层通信的行为。

socket是一套通用的网络编程接口，它不但可以访问内核中TCP/IP协议栈，而且可以访问其他网络协议栈（如X.25协议栈、UNIX本地域协议栈等）。

# 二、IP协议详解

- IP头部信息：IP头部出现在每个IP数据报中，用于指定IP通信的源端IP地址、目的端IP地址，指导IP分片和重组，以及指定部分通信行为。
- IP数据报的路由和转发：IP数据报的路由和转发发生在除目标机器之外的所有主机和路由器上。它们决定数据包是否应该转发以及如何转发。

## 2.1 IP服务的特点

IP协议是TCP/IP协议族的动力，它为上层协议提供无状态、无连接、不可靠的服务。
- **无状态**：IP通信的双方不同步传输数据的状态信息，因此所有IP数据报的发送、传输和接收都是相互独立、没有上下文关系的。这种服务的最大缺点就是无法处理乱序和重复的IP数据报。先发出的数据报可能比后发出的数据报先到达，同一个数据报也可能经由不同路径多次到达接收端。**接收端的IP模块只要收到了完整的IP数据报（如果是IP分片的话，IP模块将先执行重组），就将其数据部分（TCP报文段、UDP数据报或ICMP报文）上交给上层协议。**虽然IP数据报头部提供了一个标识字段用以唯一标识一个IP数据报，但它是被用来处理IP分片和重组的，而不是用来指示接收顺序的。**无状态服务的有点也很明显，那就是简单、高效。**
- **无连接**：IP通信双方都不长久地维持对方的任何信息。这样，上层协议每次发送数据的时候，都必须明确指定对方的IP地址。
- **不可靠**：IP协议不能保证IP数据报准确地到达接收端。例如某个中专路由器发现IP数据报在网络中存活时间太长，就将丢弃它并返回一个ICMP（超时）错误信息给发送端，又或者接收端发现收到的IP数据报不正确，也将丢弃之，也将发送一个ICMP（IP头部参数）错误信息给发送端。无论哪种情况，发送端的IP模块一旦检测到IP数据报发送失败，就通知上层协议发送失败，但并不会试图重传。所以，使用IP服务的上层协议（如TCP协议）需自己实现数据确认、超时重传等机制以达到可靠传输的目的。

## 2.2 IPv4头部结构

### 2.2.1 IPv4头部结构

IPv4的头部通常为20字节，除非含有可变长的选项部分。

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250526180515571.png" alt="image-20250526180515571" style="zoom:67%;" />

- **4位版本号（version）**：对IPv4来说，值为4。其他IPv4协议的拓展版本（如SIP协议，PIP协议）则有不同版本号，且它们也有不同的头部结构。
- **4位头部长度（header length）**：标识该IP头部有多少个32bit字，所以4位最大能表示60字节的头部。
- **8位服务类型（Type Of Service，TOS）**：包含3位优先权字段（现已忽略），4位TOS字段和1位保留字段（必须置0）。4位TOS字段分别表示：最小延时，最大吞吐量，最高可靠性和最小费用。其中最多只有一位能置为1，应用应根据实际需要来设置它。
- **16位总长度（total length）**：指整个IP数据报的长度，以字节为单位（理论最大长度为$2^{16}-1$字节），但由于MTU的限制，导致超过MTU的数据报将被分片传输，所以实际传输的IP数据报长度远没有达到做大值。
- **16位标识（identification）**：**用于实现分片。**唯一地标识主机发送的每一个数据报。其初始值由系统随机生成：每发送一个数据报，其值就加1。该标识在同一个IP数据报的多个分片中是相同的。
- **3位标志字段**：**用于实现分片。**第一位保留，第二位表示“禁止分片”（Don't Fragment）。若设置了DF位，IP模块就不会对数据报进行分片，如果IP数据报长度超过MTU则会放弃该数据报并返回ICMP错误报文。第三位表示“更多分片”（More Fragment）。除了数据报的最后一个分片，其他分片都要把它置1。
- **13位分片偏移（fragmentation offset）**：**用于实现分片。**是分片相对于原始IP数据报开始处（仅指数据部分）的偏移。实际的偏移是该值左移3位后得到的，也就是说除了最后一个IP分片外，每个IP分片的数据部分的长度必须是8的整数倍。
- **8位生存时间（Time To Live，TTL）**：是数据报到达目的地之前允许经过的路由器跳数，由发送端设置。数据报在转发过程中每经过一个路由就减1。当TTL减为0时，路由器就丢弃该数据报并向发送端发送ICMP差错报文。**TTL可以防止数据报陷入路由循环。**
- **8位协议（protocol）**：用来区分上层协议（即该IP数据报被哪个上层协议模块使用，如ICMP为1，TCP为6，UDP为17）。
- **16位头部校验和（header checksum）**：由发送端填充，接收端使用CRC算法检验IP数据报**头部**在传输过程中是否损坏。
- **32位源端IP地址和目的端IP地址**：用来标识数据报的发送端和接收端。一般情况下，这两个地址在整个传递过程中保持不变。
- **选项字段（option）**：可变长的可选信息（最多包含40字节）。IP选项包括：
  - 记录路由（record route），记录所有传递过程中经过的路由器IP地址。
  - 时间戳（timestamp），记录每个路由器转发的时间。
  - 松散源路由选择（loose source routing），指定一个路由器IP地址列表，数据报发送过程中必须经过其中所有的路由器。
  - 严格源路由选择（strict source routing），和松散源路由选择类似，不过数据报只能经过指定的路由器。


### 2.2.2 使用tcpdump观察IPv4头部结构

## 2.3 IP分片

当IP数据报长度超过MTU时，它将被分片传输。分片可能发生在发送端，也可能发生在中专路由器上，且可能在传输过程中被多次分片，但只有在最终的目标机器上，这些分片才会被内核IP模块重新组装。

IP层传递给数据链路层的数据可能是一个完整的IP数据报，也可能是一个IP分片，它们统称为IP分组（packet）。后续无特殊声明，不区分IP数据报和IP分组。

> MTU（Maximum Transmit Unit，最大传输单元）是由物理链路层决定的，如以太网帧的MTU是1500字节，即以太网帧的数据部分最多携带1500字节的上层协议数据（如IP数据报）。

## 2.4 IP路由

IP协议的一个核心任务是数据包的路由，即决定发送数据包到目标机器的路径。

### 2.4.1 IP模块工作流程

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250522142812627.png" alt="image-20250522142812627" style="zoom:67%;" />

当IP模块接收到来自数据链路层的IP数据报时，它先对数据报头部做CRC校验，确认无误后就分析其头部的具体信息。

若该IP数据报的头部设置了源站选路选项（松散源路由或严格源路由选择），则IP模块调用数据包转发子模块来处理该数据报。若该数据报是发送给本机的，则IP模块就根据数据包头部的协议字段来决定将它派发给哪个上层应用（分用）。若该数据报不是发送给本机的，则也调用数据报转发子模块处理。

数据包转发子模块首先检测系统是否允许转发，若不允许，IP模块将数据报丢弃。若允许转发，数据包转发子模块将对该数据包执行一些操作，然后将它交给IP数据包输出子模块。

IP数据报该发送至哪个下一跳路由（或目标机器），以及经过哪个网卡来发送，就是IP路由过程，即图2-3中“计算下一跳路由”子模块。IP模块实现数据报路由的核心数据结构是路由表。这个表按照数据报的目标IP地址分类，同一类的IP数据报将被发往相同的下一跳路由器（或目标机器）。

IP数据队列中存放的是所有待发送IP数据报，包含需转发的IP数据报以及封装了本机上层数据（ICMP报文、TCP报文段和UDP数据报）的IP数据报。

### 2.4.2 路由机制

路由表可用`route`命令或`netstat`命令查看，路由表有8个字段。

| 字段        | 含义                                                         |
| ----------- | ------------------------------------------------------------ |
| Destination | 目标网络或主机                                               |
| Gateway     | 网关地址，*表示目标和本机在同一个网络，不需要路由            |
| Genmask     | 网络掩码                                                     |
| Flags       | 路由项标志，常见的有5种：<br>U，该路由项是活动的；<br>H，该路由项的目标是一台主机；<br>G，该路由项的目标是网关；<br>D，该路由项是由重定向生成的；<br>M，该路由项被重定向修改过 |
| Metric      | 路由距离，即到达指定网络所需的中转数                         |
| Ref         | 路由项被引用的次数（Linux未使用）                            |
| Use         | 该路由项被使用的次数                                         |
| Iface       | 该路由项对应的输出网卡接口                                   |

给定数据报的目标IP地址，它将如何匹配路由表种的各项呢？这便是IP的路由机制，分为3各步骤：

1.  查找路由表中和数据报的目标IP地址完全匹配的主机IP地址。若找到则使用该路由项，否则转步骤2。
2. 查找路由表中和数据报的目标IP地址具有相同网路ID和网络IP地址。若找到则使用该路由项，否则转步骤3。
3. 选择默认路由项，这通常意味着数据报的下一跳路由是网关。

### 2.4.3 路由表更新

## 2.5 IP转发

不是发送给本机的IP数据报将由数据报转发子模块来处理。路由器都能执行数据报的转发操作，而主机一般指发送和接收数据报，但可以通过设置`/proc/sys/net/ipv4/ip_forward/`内核参数为`1`来允许主机的转发功能。

对允许IP数据报转发的系统（主机或路由器），数据包转发子模块将对期望转发的数据报执行如下操作：

1) 检查数据包头部的TTL值。如果TTL值已经是0，则丢弃该数据报。
2) 查看数据报头部的严格源路由选择选项。如果该选项被设置，则检测数据包的目标IP地址是否是本机的某个IP地址。如果不是，则发送一个ICMP源站选路失败报文给发送端。
3) 如有必要，则给源端发送一个ICMP重定向报文，以告诉他一个更合理的下一跳路由器。
4) 将TTL值减一。
5) 处理IP头部选项。
6) 如有必要，执行IP分片操作。

## 2.6 重定向

ICMP重定向报文也能用于更新路由表。

### 2.6.1 ICMP重定向报文

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250523094733046.png" alt="image-20250523094733046" style="zoom:67%;" />

ICMP重定向报文段类型值为5，代码字段有4各可选值，用来区分不同的重定向类型。主机重定向代码值为1。

ICMP重定向报文的数据部分含义很明确，它给接收方提供了如下两个信息：

- 引起重定向的IP数据报（图2-4中的原始IP数据报）的源端IP地址。
- 应该使用的路由器的IP地址。

接受主机根据这两个信息就可断定引起重定向的IP数据报应该使用哪个路由器来转发，并以此来更新路由表（通常是更新路由表缓冲，而非直接更改路由表）。

`/proc/sys/net/ipv4/conf/all/send_redirects`内核参数指定是否允许发送ICMP重定向报文，而`/proc/sys/net/ipv4/conf/all/accept_redirects`内核参数指定是否允许接收ICMP重定向报文。

### 2.6.2 主机重定向实例

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250523102050148.png" alt="image-20250523102050148" style="zoom:67%;" />

## 2.7 IPv6头部结构

IPv6不仅解决了IPv4地址不够用的问题，还做了很大的改进。如，增加了多播和流的功能，为网络上多媒体内容的质量提供精细的控制；引入自动配置功能，使得局域网管理更方便；增加了专门的网络安全功能等。

### 2.7.1 IPv6固定头部结构

IPv6头部由40字节的固定头部和可变长的扩展头部组成。

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250523110029378.png" alt="image-20250523110029378" style="zoom:67%;" />

### 2.7.2 IPv6扩展头部

IPv6协议不是IPv4协议的简单扩展，而是完全独立的协议。用以太网帧封装的IPv6数据报和IPv4数据报具有不同的类型值（0x86dd和0x800）。

# 三、TCP协议详解

## 3.1 TCP服务的特点

- **面向连接**：使用TCP协议通信的双方必须先建立连接，然后才能开始数据的读写。
- **字节流**：发送端执行的写操作次数和接收端执行的读操作次数之间没有任何数量关系：应用程序对数据的发送和接收是没有边界限制的。
- **可靠传输**：首先，**TCP协议采用发送应答机制**，即发送端发送的每个TCP报文段都必须得到接收方的应答才可认为这个TCP报文段传输成功。其次，**TCP协议采用超时重传机制**，发送的TCP报文段超过定时时间没收到应答则重传。最后，TCP协议会对IP数据报进行重排和整理再交付给应用层。

## 3.2 TCP头部结构

### 3.2.1 TCP固定头部结构

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250523115425721.png" alt="image-20250523115425721" style="zoom:67%;" />

- **16位端口号（port number）**：告知主机该报文段来自哪里（源端口）以及传给哪个上层协议或应用程序（目的端口）的。
- **32位序号（sequence number）**：一次TCP通信（从TCP连接建立到断开）过程中某一个传输方向上的字节流的每个字节的编号。
- **32位确认号（acknowledgement number）**：用作对另一方发送来的TCP报文段的响应。其值是收到的TCP报文段的序号值加1。
- **4位头部长度（header length）**：表示该TCP头部有多少个32bit（4字节）。因为4位最大能表示15，所以TCP头部最长是60字节。
- **6位标志位**：
  - URG标志：紧急指针（urgent pointer）是否有效。
  - ACK标志：确认号字段是否有效。
  - PSH标志：提示接收端应用程序应立即从TCP接收缓冲区种读走数据，为接收后续数据腾出空间。
  - RST标志：表示要求对方重新建立连接。
  - SYN标志：表示请求建立一个连接。
  - FIN标志：表示通知对方本端要关闭连接了。
- **16位窗口大小**：TCP流量控制的一个手段。这里的窗口，指的是接收通告窗口（Received Window，RWND）。它告诉对方本段的TCP接收缓冲区还能容纳多少字节的数据，这样对方就可以控制发送数据的速度。
- **16位校验和（TCP checksum）**：由发送端填充，接收端对TCP报文段执行CRC算法以检验TCP报文段（既包含TCP头部，也包括数据部分）是否在传输过程中损坏。
- **16位紧急指针（urgent point）**：是一个正的偏移量。它和序号字段的值相加表示最后一个紧急数据的下一字节的序号。TCP的紧急指针是发送端项接收端发送紧急数据的方法。

### 3.2.2 TCP头部选项

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250523143038807.png" alt="image-20250523143038807" style="zoom:67%;" />

length字段指定选项的长度（包含1字节的kind和1字节的length以及信息字段长度）。

kind=2是最大报文段长度（Maximum Segment Size，MSS）选项。TCP模块通常将MSS设置为（MTU-40）字节（减去的是20字节的TCP头部和20字节的IP头部）。这样设置MSS可以避免本机发生IP分片。

kind=3是窗口扩大因子选项。TCP连接初始化时，通信双方用该选项协商RWND的扩大因子。由于TCP头部中RWND大小的位数限制了可以表达的实际RWND大小，窗口扩大因子解决了这个问题。假设TCP头部中RWND的大小设置为N，窗口扩大因子为M，则实际的RWND大小为$N\times 2^{M}$。

### 3.2.3 使用tcpdump观察TCP头部信息

## 3.3 TCP连接的建立和关闭

### 3.3.1 使用tcpdump观察TCP连接的建立和关闭

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250523165726345.png" alt="image-20250523165726345" style="zoom:67%;" />

### 3.3.2 半关闭状态

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250523165901048.png" alt="image-20250523165901048" style="zoom:67%;" />

### 3.3.3 连接超时

超时重连次数由`/proc/sys/net/ipv4/tcp_syn_retries`内核变量决定。

## 3.4 TCP状态转移

TCP连接的任意一端再任一时刻都处于某种状态，当前状态可以通过`netstat`命令查看。

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250523161640412.png" alt="image-20250523161640412" style="zoom:67%;" />

上图是完整的状态转移图，它描绘了所有的TCP状态以及可能的状态转移。**粗虚线表示典型的服务器端连接的状态转移；粗实线表示典型的客户端连接的状态。CLOSED是一个假想的起始点，并不是一个实际的状态。**（我思考这里的“典型”是指服务器端提前进入监听，然后客户端主动连接服务器端，最后客户端执行完任务主动断开与服务器端的连接的场景。）

### 3.4.1 TCP状态转移总图

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250523172249665.png" alt="image-20250523172249665" style="zoom:67%;" />

### 3.4.2 TIME_WAIT状态

客户端在首都奥服务器的结束报文段后，并未直接进入CLOSED状态，而是转移到TIME_WAIT状态。在此状态下，客户端要等待一段长为2MSL（Maximum Segment Life，报文最大生存时间）的时间，才能完全关闭。MSL再标准文档RFC 1122的建议值位2 min。

TIME_WAIT状态存在的原因有两点：

- **可靠地终止TCP连接。**假设图3-9的报文段7丢失，即服务器端发送的报文段6未收到到应答，那么它将重发报文段6，如果客户端处在CLOSED状态就无法进行应答，因此它需要一个“过渡”状态来等待可能的服务器端重发并进行应答。
- **保证让迟来的TCP报文段有足够的时间被识别并丢弃。**

## 3.5 复位报文段

### 3.5.1 访问不存在的端口

### 3.5.2 异常终止连接

### 3.5.3 处理半打开连接

## 3.6 TCP交互数据流

TCP报文段所携带的应用程序数据按照长度分为两种：交互数据和成块数据。交互数据仅包含很少的字节。使用交互数据的的应用程序（或协议）对实时性要求高。成块数据的长度则通常为TCP报文段允许的最大数据长度。使用成块数据的应用程序（或协议）对传输效率要求高。

## 3.7 TCP成块数据流

## 3.8 带外数据

有些传输层协议具有带外（Out Of Band，OOB）数据的概念，用于迅速通告对方本端发生的重要事件。因此，带外数据比普通数据（也称带内数据）有更高的优先级，它应该总是立即被发送，而不论发送缓冲区中是否有排队等待发送的普通数据。带外数据的传输可以使用条独立的传输层连接，也可以映射到传输普通数据的连接中。实际应用中，很少使用带外数据。

UDP未实现带外数据，TCP也没有正真的带外数据。不过TCP利用其头部中的紧急指针标志和紧急指针两个字段，给应用程序提供了一种紧急方式。

## 3.9 TCP超时重传

TCP模块未每个TCP报文段都维护一个重传定时器，该定时器在TCP报文段第一次被发送时启动。若超时时间内为首都奥接收方的应答，TCP模块将重传TCP报文段并重置定时器。至于下次重传的超时时间如何选择，以及最多执行多少次重传，就是TCP的重传策略。

## 3.10 拥塞控制

### 3.10.1 拥塞控制概述

TCP模块还有一个重要的任务，就是提高网络利用率，降低丢包率，并保证网络资源对每条数据流的公平性。这就是所谓的拥塞控制。

TCP拥塞控制的标准文档（RFC 5681）包括四个部分：**慢启动**（slow start）、**拥塞避免**（congestion avoidance）、**快速重传**（fast retransmit）和**快速恢复**（fast recovery）。拥塞控制在Linux下有多种实现，如Reno算法、Vegas算法和cubic算法等，它们都或部分或全部实现了上述四个部分。

拥塞控制的最终受控变量是发送端向网络一次连续写入（收到其第一个数据的确认之前）的数据量，称为SWND（Send Window，发送窗口）。不过，发送端最终以TCP报文段来发送数据，所以SWND限定了发送端能连续发送的TCP报文段数量。这些TCP报文段的最大长度（仅指数据部分）称为SMSS（Sender Maximum Segment Size，发送端最大段大小），其值一般等于MSS。

如果SWND太小，会引起明显的网络延迟；反之，如果SWND太大，则容易导致网络拥塞。接收方可以通过其接受通告窗口（RWND）来控制发送端的SWND。这还不够，所以发送端引入了拥塞窗口（Congestion Window，CWND）的状态变量。实际上CWND值是RWND和SWND中的较小者。

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250526172755276.png" alt="image-20250526172755276" style="zoom:67%;" />

### 3.10.2 慢启动和拥塞避免

TCP建立好之后，CWND被设置为初始值IW（Initial Window），大小为SMSS的倍数。此时发送端最多能发送IW字节数据。此后发送端每收到接收端的一个确认，其CWND就按式（3-1）增加：
$$
\text{CWND}\text{+=} min(N,\text{SMSS})\tag{3-1}
$$
其中*N*是此次确认中包含的之前未被确认的字节数。如此CWND将按指数形式扩大，这就是慢启动。慢启动算法的理由是，TCP模块刚开始发送数据时并不知道网络的实际情况，需要用一种试探的方式平滑地增加CWND的大小。

为避免CWND快速膨胀导致网络拥塞。TCP拥塞控制定义了另一个重要的状态变量：慢启动门限（slow start threshold size，ssthresh）。当CWND超过该值时，TCP拥塞控制将进入拥塞避免阶段。拥塞避免算法是的CWND按照线性方式增加，从而减缓其扩大。标志文档中提到了两种实现方式：

- 每个RTT时间内按照式（3-1）计算新的CWND，而不论该RTT时间内发送端收到多少个确认。
- 每收到一个对新数据的确认报文段，就按照式（3-2）来更新CWND。

$$
\text{CWND+=SMSS*SMSS/CWND}\tag{3-2}
$$

以上是发送端在未检测到拥塞时所采用的积极避免拥塞的方法。接下来是拥塞发生时（可能发生在慢启动或拥塞避免阶段）拥塞控制的行为。

发送端判断拥塞发生的依据有两个：

- 传输超时，或着说TCP重传定时器溢出。
- 接收到重复的确认报文段。

拥塞控制对这两种情况有不同的处理方式。对第一种情况任然使用慢启动和拥塞避免。对第二种情况使用快速重传和快速恢复。注意第二种情况如果发生在重传定时器溢出之后，则也被拥塞控制当成第一种情况来对待。

若发送端检测到拥塞原因式传输超时，即第一种情况，则它将执行重传，并做如下调整：
$$
\text{ssthresh=max(FlightSize/2, 2*SMSS)}\\ \tag{3-3}
\text{CWND<=SMSS}
$$
其中FlightSize是已经发送但未收到确认的字节数。这样调整后，CWND江小鱼SMSS，那么也必然小于新的ssthresh，故而拥塞控制再次进入慢启动阶段。

### 3.10.3 快速重传和快速恢复

> 发送端收到重复ACK报文段的原因：
>
> 1. 接收端检测到数据包失序
>    - 接收端期望按顺序接受数据包。
>    - 若接收端收到了一个非预期的序列号的数据包（如跳过了中间某个包），它会立即发送一个**对已接受的最大连续序列号的确认**（即重复ACK）。
>    - 接收端每收到一个失序的数据包，就会发送一次相同的ACK。
> 2. 网络丢包（丢包造成接收端接收失序的数据包）
> 3. ACK报文在网络中重复传输

发送端若连续收到3个重复的确认报文段，就认为是拥塞发生了。然后它启用开苏重传和快速恢复算法来处理拥塞，过程如下：

1. 当收到第3个重复的确认报文段时，按照式（3-3）计算ssthresh，然后立即重传丢失的报文段，并按照式（3-4）设置CWND。

$$
\text{CWND=ssthresh+3*SMSS}\tag{3-4}
$$

2. 每次收到一个重复的确认时，设置$\text{CWND=CWND+SMSS}$。此时发送端可以发送新的TCP报文段（若新CWND允许的话）。
3. 当收到新数据的确认时，设置CWND=ssthresh（ssthresh是新的慢启动门限，由第一步得到）。

快速重传和快速回复完成后，拥塞控制将恢复到拥塞避免阶段，这一点由第3部操作可得知。

# 四、TCP/IP通信案例：访问Internet上的Web服务器

## 4.1 实例总图

<img src="D:\X1AOBO\Documents\github\lean-db-note\assets\image-20250527141940498.png" alt="image-20250527141940498" style="zoom:67%;" />

## 4.2 部署代理服务器

### 4.2.1 HTTP代理服务器的工作原理

在HTTP通信链上，客户端和服务器之间通常存在某些中转代理服务器，它们提供对目标资源的中转访问。代理服务器按照其使用方式和左右，分为正向代理服务器、反向代理服务器和透明代理服务器。

正向代理需要客户端自己设置代理服务器的地址。客户端的每次请求都将直接发送到该代理服务器，并由代理服务器来请求目标资源。

反向代理则被设置在服务器端，客户端无需任何设置。反向代理是指用代理服务器来接收Internet上的连接请求，然后将请求转发给内部网络上的服务器，并将从内部服务器上得到的结果返回给客户端（反向代理服务器对外表现得就像一个真实的服务器）。

透明代理只能设置在网关上。用户访问Internet的数据报必然都经过网关，设置在网关上对用户来说是透明的。透明代理可看作正向代理的一种特殊情况。

### 4.2.2 部署squid代理服务器

## 4.3 使用tcpdump抓取传输数据包

# 五、Linux网络编程基础API

## 5.1 socket地址API

> 考虑32位机。

### 5.1.1 主机字节序和网络字节序

字节序分为大端字节序（big endian）和小端字节序（little endian）。大端字节序是指一个整数的高位字节（23~31 bit）存储在内存的低地址处，低位字节（0~7 bit）存储在内存的高地址处。小端字节序则是指证书的高位字节存储在内存的高地址处，而低位字节则存储在内存的低地址处。

现代PC大多采用小端字节序，因此小端字节序又被称为主机字节序。不同的字节序的主机在发送数据时必然把数据转化成大端字节序，因此大端字节序也称为网络字节序。需要指出的是，即使是同一台机器上的两个进程（如一个由C语言编写，另一个由JAVA编写）通信，也要考虑字节序的问题（JAVA虚拟机采用大端字节序）。

Linux提供了如下4个函数来完成主机字节序与网络字节序之间的转换

```c
#include <netinet/in.h>
unsigned long int htonl(unsigned long int hostlong);
unsigned short int htons(unsigned short int hostshort);
unsigned long int ntohl(unsigned long int netlong);
unsigned short int ntohs(unsigned short int netshort);
```

### 5.1.2 通用socket地址

socket网络编程接口中表示socket地址的是结构体`sockaddr`，其定义如下：

```c
#include <bits/socket.h>
struct sockaddr
{
    sa_family_t sa_family; 	// 地址族/协议族类型
    char sa_data[14];		// 地址值
};
```

14字节的`sa_data`无法完全容纳多数协议族的地址值。因此Linux定义了新的通用socket地址结构体：

```c
#include <bits/socket.h>
struct sockaddr_storage
{
    sa_family_t sa_family;	// 地址族/协议族类型
    unsigned long int __ss_align;	// 内存对齐值
    char __ss_padding[128-sizeof(__ss_align)];	// 地址值
};
```

### 5.1.3 专用socket地址

上述的socket地址结构体在设置与获取IP地址和端口号时需执行位操作，很麻烦。因此Linux为各个协议族提供了专门的socket地址结构体。

UNIX本地域协议族：

```c
#include <sys/un.h>
struct sockaddr_un
{
    sa_family_t sin_family; // AF_UNIX/PF_UNIX
    char sun_path[108];
};
```

TCP/IP协议族分别用于IPv4和IPv6的地址结构体：

```c
struct in_addr
{
    u_int32_t s_addr; // IPv4地址
}
struct sockaddr_in
{
    sa_family_t sin_family; // AF_INET/PF_INET
    u_int16_t sin_port;
    struct in_addr sin_addr;
};
struct in6_addr
{
    unsigned char sa_addr[16]; // IPv6地址
};
struct sockaddr_in6
{
    sa_family_t sin6_family; // AF_INET6/PF_INET6
    u_int16_t sin6_port;
    u_int16_t sin6_flowinfo; // 流信息，应设为0
    struct in6_addr sin6_addr;
    u_int32_t sin6_scope_id; // scope ID
};
```

所有socket地址结构体（`sockaddr_storage`以及`sockaddr_in`和`sockaddr_in6`）在实际使用时都需要转化为通用socket地址`sockaddr`（强制转换），因为socket编程接口使用的地址参数类型都是`sockaddr`。

### 5.1.4 IP地址转换函数

下面的3个函数可用于用点分十进制字符串表示的IPv4地址和用网络字节序整数表示的IPv4地址之间的转换：

```c
#include <arpa/inet.h>
in_addr_t inet_addr(const char *strptr);
int inet_aton(const char *cp, struct in_addr *inp);
char *inet_ntoa(struct in_addr in); // 不可重入
```

下面的函数和上面的函数作用相同，且同时适用于IPv4地址和IPv6地址：

```c
#include <arpa/inet.h>
int inet_pton(int af, const char *src, void *dst);
const char *inet_ntop(int af, const void *src, char *dst, socklen_t cnt);
```

第二个函数作用与第一个相反，将网络字节序整数表示的地址转换为字符串表示的IP地址，最后一个参数表示目标存储单元的大小：

```c
#include <netinet/in.h>
#define INET_ADDRSTRLEN 16
#define INET6_ADDRSTRLEN 46
```

## 5.2 创建socket

在Linux下，socket也是文件，即可读、可写、可控制、可关闭的文件描述符。下面的系统调用可以创建socket：

```c
#include <sys/types.h>
#include <sys/socket.h>
int socket(int domain, // 协议族
           int type, // 服务类型：流/数据报
           int protocol); // 一般置为0,表示使用默认协议
```

从Linux内核2.6.17开始，type参数可以接受服务类型与下面两个重要的标志相与的值：`SOCK_NONBLOCK`和`SOCK_CLOEXEC`。

## 5.3 命名socket

创建socket时，制定了协议族，但并未指定使用哪个具体的socket地址。将一个socket与socket地址绑定称为给socket命名。在服务器程序中，通常要命名socket，而客户端程序则通常不需要命名socket，而是使用匿名方式（使用操作系统自动分配的socket地址）。命名socket的系统调用是bind，如下：

```c
#include <sys/types.h>
#include <sys/socket.h>
int bind(int sockfd, 
         const struct sockaddr *my_addr, 
         socklen_t addrlen);
```

## 5.4 监听socket

socket被命名以后，还需要用一个系统调用来创建一个监听队列以存放待处理的客户端连接（换句话说，为客户端socket创建一个指定大小的监听队列）：

```c
#include <sys/socket.h>
int listen(int sockfd, int backlog);
```

backlog参数提示内核监听队列的最大长度。监听队列的长度如果超过backlog，服务器将不受理新的客户端连接，客户端将收到`ECONNREFUSED`错误信息。

执行过`listen`调用、**处于LISTEN状态的socket称为监听socket**，所有**处于ESTABLISHED状态的socket则称为连接socket**。

## 5.5 接受连接

下面的系统调用从listen监听队列中接受一个连接：

```c
#include <sys/types.h>
#include <sys/socket.h>
int accept(int sockfd, // 执行过listen的监听socket
           struct sockaddr *addr, // 被接受连接的远端地址
           socklen_t *addrlen);
```

accept成功时返回一个新的连接socket，该socket唯一的标识了被接受的这个连接，服务器可通过读写该socket来与被接受连接对应的客户端通信。

accept只是从监听队列中取出连接，而不论连接出于何种状态，更不关心任何网络状态的变化。

## 5.6 发起连接

服务器使用listen调用来被动接受连接，客户端需要通过如下系统调用来主动与服务器简历连接：

```c
#include <sys/types.h>
#include <sys/socket.h>
int connect(int sockfd, // 客户端socket
	const struct sockaddr *serv_addr, // 服务器监听地址
    socklen_t *addrlen);
```

## 5.7 关闭连接

关闭一个连接实际上就是关闭该连接对应的socket，这可以通过如下关闭普通文件描述符的系统调用来完成：

```c
#include <unistd.h>
int close(int fd);
```

close系统调用并非总是立即关闭一个连接，而是将fd的引用计数减1。只有当fd的引用计数为0时，才真正关闭连接。多进程程序中（不做特殊设置），父子进程都对socket执行close调用才能将连接关闭。

想要立即终止连接（而不是减少引用计数），可以用shutdown系统调用（专为网络编程设计）：

```c
#include <sys/socket.h>
int shutdown(int sockfd, int howto);
```

howto参数决定了shutdown的行为：

| 可选值    | 含义                                                         |
| --------- | ------------------------------------------------------------ |
| SHUT_RD   | 关闭sockfd上读的这一半。程序不能再针对sockfd执行读操作，且该socket的接收缓冲区中的数据都被丢弃。 |
| SHUT_WR   | 关闭sockfd上写的这一半。sockfd的发送缓冲区中的数据会在正在关闭连接前全部发送出去，程序不能再对sockfd执行写操作。在这种情况下，连接处于半关闭状态。 |
| SHUT_RDWR | 同时关闭sockfd上的读和写。                                   |

## 5.8 数据读写

### 5.8.1 TCP数据读写

对文件的读写操作read和write同样适用于socket。同时socket编程接口提供了几个专用于socket数据读写的系统调用：

```c
#include <sys/types.h>
#include <sys/socket.h>
ssize_t recv(int sockfd, void *buf, size_t len, 
             int flags);
ssize_t send(int sockfd, const void *buf, size_t len, 
             int flags);
```

flags参数为数据收发提供了额外的控制，一般取0，它可以取下表中的一个或几个的逻辑或。

| 选项名        | 含义                                                         | send | recv |
| ------------- | ------------------------------------------------------------ | ---- | ---- |
| MSG_CONFIRM   | 指示数据链路层协议持续监听对方的回应，知道得到答复。仅能用于SOCK_DGRAM/SOCK_RAW类型的socket | Y    | N    |
| MSG_DONTROUTE | 不查看路由表，直接将数据发送给本地局域网络内的主机。这表示发送者确切地直到目标主机就在本地网络上 | Y    | N    |
| MSG_DONTWAIT  | 对socket的此次操作将是非阻塞的                               | Y    | Y    |
| MSG_MORE      | 告诉内核应用程序还有更多数据要发送，内核将超时等待新数据写入TCP发送缓冲区后一并发送。这可以防止TCP发送过多小的报文段，从而提高传输效率 | Y    | N    |
| MSG_WAITALL   | 读操作仅在读取到指定数量的字节后才返回                       | N    | Y    |
| MSG_PEEK      | 窥探读缓存中的数据，此次读操作不会导致这些数据被清除         | N    | Y    |
| MSG_OOB       | 发送或接收紧急数据                                           | Y    | Y    |
| MSG_NOSIGNAL  | 往读端关闭的管道或者socket连接中写数据时不引发SIGPIPE信号    | Y    | N    |

flags参数只对send和recv的当前调用生效，而通过setsockopt系统调用可以永久性地修改socket地某些属性。

### 5.8.2 UDP数据读写

socket编程接口中用于UDP数据报读写的系统调用时：

```c
#include <sys/types.h>
#include <sys/socket.h>
ssize_t recvfrom(int sockfd, void *buf, size_t len, 
             	 int flags, struct sockaddr *src_addr,
                 socklen_t *addrlen);
ssize_t sendto(int sockfd, const void *buf, size_t len, 
               int flags, const struct sockaddr *dest_addr,
               socklen_t addrlen);
```

因为UDP通信没有连接的概念，所以每次读写数据都需要发送端和目标端的socket地址。flags参数的含义与前面介绍的相同。其实这两个系统调用也能用于面向连接（STREAM）的socket的数据读写，只需要把最后两个参数都设置为NULL。

### 5.8.3 通用数据读写函数

socket编程接口还提供了一堆通用的数据读写系统调用。它们既可以用于TCP流数据，也能用于UDP数据报：

```c
#include <sys/socket.h>
ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
ssize_t sendmsg(int sockfd, struct msghdr *msg, int flags);
```

msg参数是msghdr结构体类型的指针：

```c
struct msghdr
{
    void *msg_name; // socket地址
    socklen_t msg_namelen;// socket地址长度
    struct iovec *msg_iov; // 分散的内存块
    int msg_iovlen; // 分散内存块的数量
    void *msg_control; // 指向辅助数据的起始位置
    socklen_t msg_controllen; // 辅助数据的大小
    int msg_flags; // 复制函数中的flags参数并在调用过程中更新
};
struct iovec
{
    void *iov_base; // 内存的起始地址
    size_t iov_len; // 这块内存的长度
};
```

msg_name成员指向一个socket地址结构变量。它指定通信对方的socket地址。对于面向连接的TCP协议，该成员没有意义，必须被设置为NULL。msg_flags成员无需设定，它会复制recvmsg/sendmsg的flags参数的内容以影响数据读写过程。recvmsg/sendmsg的flags参数以及返回值的含义与send/recv的flags参数及返回值相同。

## 5.9 带外数据

实际应用中无法预期带外数据何时到来，但是内核由两种通知应用程序带外数据到来的方式：I/O复用产生的异常事件和SIGURG信号。除此之外，程序还需要直到带外数据在数据流中的具体位置，才能准确接收。这可以通过sockatmark系统调用实现：

```c
#include <sys/socket.h>
int sockatmark(int sockfd);
```

sockatmark判断sockfd是否处于带外数据，即下个被读取的数据是否是带外数据。若是，sockatmark返回1，此时可以用带MSG_OOB标志的recv调用接收带外数据。否则，sockatmark返回0。

## 5.10 地址信息函数

获取一个连接socket的本端socket地址和远端socket地址有以下两个接口：

```c
#include <sys/socket.h>
int getsockname(int sockfd, struct sockaddr *address, 
                socklen_t *address_len); // 本端
int getpeername(int sockfd, struct sockaddr *address, 
                socklen_t *address_len); // 远端
```

## 5.11 socket选项

专门用来读取和设置socket文件描述符的方法：

```c
#include <sys/socket.h>
int getsockopt(int sockfd, int level, 
               int option_name, void *option_value, 
               socklen_t * restrict option_len);
int setsockopt(int sockfd, int level,
               int option_name, const void *option_value,
               socklen_t option_len);
```

level参数指定要操作哪个协议的选项（即属性），如IPv4、IPv6、TCP等。option_name参数指定选项名称，option_value和option_len参数分别是被操作选项的值和长度。

<table>
<tr><th>level</th><th>option name</th><th>数据类型</th><th>说明</th></tr>
<tr><th rowspan="14">SOL_SOCKET（socket通用选项）</th>
    <td>SO_DEBUG</td><td>int</td><td>打开调试信息</td></tr>
<tr><td>SO_REUSEADDR</td><td>int</td><td>重用本地地址</td></tr>
<tr><td>SO_TYPE</td><td>int</td><td>获取socket类型</td></tr>
<tr><td>SO_ERROR</td><td>int</td><td>获取并清除socket错误状态</td></tr>
<tr><td>SO_DONTROUTE</td><td>int</td><td>不查看路由表，直接将数据发送给本地局域网内的主机，含义和send系统调用的MSG_DONTROUTE标志类似</td></tr>
<tr><td>SO_RCVBUF</td><td>int</td><td>TCP接收缓冲区大小</td></tr>
<tr><td>SO_SNDBUF</td><td>int</td><td>TCP发送缓冲区大小</td></tr>
<tr><td>SO_KEEPALIVE</td><td>int</td><td>发送周期性保活报文以维持连接</td></tr>
<tr><td>SO_OOBINLINE</td><td>int</td><td>带外数据将留在普通数据的输入队列中，此时不能使用MSG_OOB标志读取带外数据</td></tr>
<tr><td>SO_LINGER</td><td>linger</td><td>若有数据待发送，则延迟关闭</td></tr>
<tr><td>SO_RCVLOWAT</td><td>int</td><td>TCP接收缓冲区低水位标记</td></tr>
<tr><td>SO_SNDLOWAT</td><td>int</td><td>TCP发送缓冲区低水位标记</td></tr>
<tr><td>SO_RCVTIMEO</td><td>timeval</td><td>接收数据超时</td></tr>
<tr><td>SO_SNDTIMEO</td><td>timeval</td><td>发送数据超时</td></tr>
<tr><th rowspan="2">IPPROTO_IP（IPv4选项）</th>
    <td>IP_TOS</td><td>int</td><td>服务类型</td></tr>
<tr><td>IP_TTL</td><td>int</td><td>存活时间</td></tr>
<tr><th rowspan="4">IPPROTO_IPV6（IPv6选项）</th>
    <td>IPV6_NEXTHOP</td><td>sockaddr_in6</td><td>下一跳IP地址</td></tr>
<tr><td>IPV6_RECVPKTINFO</td><td>int</td><td>接收分组信息</td></tr>
<tr><td>IPV6_DONTFRAG</td><td>int</td><td>禁止分片</td></tr>
<tr><td>IPV6_RECVTCLASS</td><td>int</td><td>接收通信类型</td></tr>
<tr><th rowspan="2">IPPROTO_TCP（TCP选项）</th>
    <td>TCP_MAXSEG</td><td>int</td><td>TCP最大报文段大小</td></tr>
<tr><td>TCP_NODELAY</td><td>int</td><td>禁止Nagle算法</td></tr>
</table>

需要注意的是，对服务器而言，部分socket选项只有在调用listen系统调用前针对该socket设置才有效。这是因为连接socket（ESTABLISHED状态）只能由`accept`调用返回，而accept从listen监听队列中接受的连接至少已经完成了TCP三次握手的前两个步骤（因为listen监听队列中的连接至少已经进入了SYN_RCVD状态），这说明服务器已经往被接受链接上发送出了TCP同步报文段。担忧的socket选项却应该在TCP同步报文段中设置，如TCP最大报文段选项。对这种情况，Linux给出的解决方案是，对监听socket设置这些socket选项，那么accept返回的连接socket将自动基础这些选项。这些socket选项包括：SO_DEBUG、SO_DONTROUTE、SO_KEEPALIVE、SO_LINGER、SO_OOBINLINE、SO_RCVBUF、SO_RCVLOWAT、SO_SNDBUF、SO_SNDLOWAT、TCP_MAXSEG、TCP_NODELAY。基于同样的理由，客户端在设置这些选项时也应该在调用`connect()`函数之前，因为`connect()`调用成功返回后，TCP三次握手已完成（同步报文已经发送了）。
