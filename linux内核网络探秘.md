一.内核是如何接收网络包的（[linux - 图解Linux网络包接收过程_个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000038374889)）

​		收包前的准备：

- 创建ksoftirqd线程，并设置好该线程函数，后面通过它来处理软中断

- 协议栈注册，linux要实现许多协议，比如ARP、ICMP、IP、UDP和TCP，每一个协议都会都会将自己的处理函数注册一下，

  方便包来了迅速找到对应的处理函数

- 网卡驱动初始化，每个驱动都有一个初始化函数，内核会让驱动也初始化一下。在初始化过程中，把自己的DMA准备好，

  把NAPI的pol函数地址告诉内核。

- 启用网卡，分配RX、TX队列，注册中断对应的处理函数。

  当以上准备好后，就可以打开硬中断，等待数据包的到来。

​		流程图：

​					

​		问题1：防火墙封禁了某个IP的访问，还能用tcpdump抓到包吗？

​					服务端封禁某客户端IP https://cloud.tencent.com/developer/article/1722230

​					客户端去Ping该服务端，不能ping通

​					服务端用tcpdump -nn icmp去抓包。



​		问题2：什么是NAPI?

​					随着网络带宽的发展，网速越来越快，之前的中断收包模式已经无法适应目前千兆，万兆的带宽了。如果每个数据包大小等于MTU大小1460字节。当驱动以千兆网速收包时，CPU将每秒被中断91829次。在以MTU收包的情况下都会出现每秒被中断10万次的情况。过多的中断会引起一个问题，CPU一直陷入硬中断而没有时间来处理别的事情了。为了解决这个问题，内核在2.6中引入了NAPI机制。

NAPI就是混合中断和轮询的方式来收包，当有中断来了，驱动关闭中断，通知内核收包，内核软中断轮询当前网卡，在规定时间尽可能多的收包。时间用尽或者没有数据可收，内核再次开启中断，准备下一次收包。

[Linux网络协议栈：NAPI机制与处理流程分析（图解）_rtoax的博客-CSDN博客_netif_napi_add](https://blog.csdn.net/Rong_Toa/article/details/109401935)

​	问题3：网卡丢包如何解决

​				1.加大RingBuffer数据长度

​							查看RingBuffer的大小 ： ethtool -g eth0

​							查看是否由于RingBuffer溢出而丢包： ethtool -S eth0   （rx_fifo_errors如果不为0（在ifconfig中体现为overruns

​									指标增长），就表示有包因为RingBuffer装不下而被丢弃了）

​							加大RingBuffer：  ethtool -G  eth0 rx 4096 tx 4096									

​				2.配置网卡多队列

​							查看网卡多队列 ：  ethtool -l eth0  或者  ls /sys/class/net/eth0/queues(查看生效的队列数）

​							加大队列数：ethtool -L eth0 combined 15

​				

​				说明：每 一个队列都有自己的中断号（cat /proc/interrupts）。队列的中断号和CPU的亲和性一般不需要手工维护，由

​							irqbalance服务来自动管理,查看该进程(ps -ef | grep irqb）， irqbalance会根据系统中断负载的情况，自动维护和

​							迁移各个中断的CPU亲和性，以保持各个CPU之间的中断开销均衡。

​				

​							

一.内核是如何发送网络包的 （[25 张图，一万字，拆解 Linux 网络包发送过程-面包板社区 (eet-china.com)](https://www.eet-china.com/mp/a88353.html)）

如图所示：用户数据被拷贝到内核态，然后经过协议栈处理后进入RringBuffer。随后网卡驱动真正将数据发送了出去。当发送完成的时候，是通过硬中断来通知CPU，然后清理RingBuffer。

​		问题1：多队列网卡

​		现代服务器上的网卡一般都支持多队列的。每一个队列都是由一个RingBuffer表示的，开启了多队列以后的网卡就会对应有多个

​		RingBuffer。

[多队列网卡简介_turkeyzhou的博客-CSDN博客_多队列网卡](https://blog.csdn.net/turkeyzhou/article/details/7528182)



​		问题2：发送网络数据（待发送数据）涉及哪些内存拷贝？

​				第一次拷贝：内核申请完skb之后，将用户传递进来的buffer里的数据内容拷贝到skb。

​				第二次拷贝：从传输层进入网络层时，每一个skb都会被克隆出来一个新的副本。目的是保存原始的skb，当没有收到对方网络

​										发送的ACK时候，还可以重新发送，以实现TCP中要求的可靠传输。这只是浅拷贝，只拷贝skb描述符本身，所指									向数据还是复用的。

​				第三次拷贝：当IP层发现skb大于MTU时，会执行分片，将原来的一个skb拷贝为多个小的skb。



​		问题3： 127.0.0.1本机网络IO需要经过网卡吗？(抓包试验)

​					  不需要经过网卡。([深入操作系统，彻底搞懂127.0.0.1本机网络通信|服务器|linux|key_网易订阅 (163.com)](https://www.163.com/dy/article/GDJEB7R50511X1MK.html))

​	