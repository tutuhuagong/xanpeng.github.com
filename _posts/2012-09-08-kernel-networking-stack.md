---
title: 从网卡往上
layout: post
category: linux
tags: kernel networking hardware socket
---

[外设](http://en.wikipedia.org/wiki/Peripheral_device)一般都有[硬件端口/接口](http://en.wikipedia.org/wiki/Computer_port_(hardware\))，通过接口和主机相连。  
设备有对应的驱动，OS通过设备驱动访问设备。  
大多数硬件会有几个[硬件寄存器](http://en.wikipedia.org/wiki/Hardware_register)，比如CPU的硬件寄存器称之为[Processor register](http://en.wikipedia.org/wiki/Processor_register)，就是C语言中的“寄存器变量”指代的寄存器。  
OS通过硬件端口连接外设，通过读写外设的硬件寄存器控制外设。*([ldd ch9 "communicating with hardware"](http://lwn.net/images/pdf/LDD3/ch09.pdf))*  

OS有两种访问硬件寄存器的方式：[memory-mapped I/O](http://en.wikipedia.org/wiki/Memory-mapped_I/O)和Port-mapped I/O。  
memory-mapped I/O指用同样的寻址总线寻址内存和I/O设备，即把内存和I/O设备放在同一地址空间。  
port-mapped I/O需要用特殊的CPU指令访问外设，如inl、inw、outl、outw等。  

外设有两种类型：[ISA](http://zh.wikipedia.org/wiki/ISA)和PCI([1](http://net.pku.edu.cn/~yhf/lyceum/linuxK/dd/pci.html),[2](http://www.csie.nctu.edu.tw/~tcwu/doc/Linux/Kernel/chapter6/chapter6.htm))。除了特殊工业用途外，ISA设备已经不再使用了，而且现在的主板都不带ISA接口([ref](http://zh.wikipedia.org/wiki/ISA#.E5.BD.93.E5.89.8D.E5.BA.94.E7.94.A8))。  

外设和CPU、内存等的关系：  
![](/images/osarch.jpg)

---

**port-mapped I/O**  
I/O ports是驱动程序和设备之间的通信方式--至少在部分时间是这样。  
使用I/O ports之前，驱动程序需要先申请，内核提供函数request_region，驱动通过该函数声明自己需要操作的端口。  
/proc/ioports记录系统所有的端口分配。  
操作端口可以使用inb、outb之类的函数，内核态用户态都可以直接操作端口。  
端口操作需要处理一个问题：CPU从总线传输数据的速度比外设快。  

**memory-mapped I/O**  
外设会有硬件寄存器，也会有自己的内存空间([PCI Address Spaces](http://tldp.org/LDP/tlk/dd/pci.html))。和设备通信的另一种方式——同时也是主要方式——是把外设的寄存器和独立内存映射到主存所在的地址空间。此时外设寄存器和独立内存都称为I/O memory，因为二者对软件来说是透明的。  
I/O内存仅仅是类似于主存RAM的一个区域，这种内存有很多用途，比如存放视频数据或以太网数据包，也可以用来实现类似于I/O端口的设备寄存器。  
访问I/O内存可以通过页表，也可以不通过页表。内核提供了request_mem_region等函数操作I/O内存。  

---

**PCI驱动**(详见ldd ch12)  

PCI设备上电时，硬件保持未激活状态，此时硬件只响应配置任务。  
每个PCI主板配备有能处理PCI的固件，如BIOS、NVRAM或PROM。固件能够读写PCI控制器中的寄存器，这提供了对配置任务的支持。  
系统引导时，Linux内核在每个PCI外设上执行配置任务，当驱动程序访问设备时，外设的内存和寄存器都已经被映射到CPU的地址空间。  

外设在内核中对应pci_dev结构，驱动在内核中对应pci_driver结构。  
驱动是通过内核模块的方式，在系统启动时加载的。举个栗子，Intel e1000网卡驱动的e1000_main.c中，模块初始化函数是e1000_init_module，它调用pci_register_driver(&e1000_driver)将驱动注册到内核某处，以便同一管理吧。e1000_driver是pci_driver实例，其中函数指针probe被定义为e1000_probe，它调用pci_enable_device()激活e1000网卡。  

---

网卡类属于PCI设备，因此可以应用前面描述的知识。以Intel Gigabit Ethernet driver(drivers/net/igb)为例，来看网卡上电到工作的过程(*参考了“[Diving into the Linux Networking Stack](http://beyond-syntax.com/blog/2011/03/diving-into-linux-networking-i/)”*)。  
1、系统启动，发现网卡，初始化pci_dev数据结构。这个数据结构包含多个收包(rx)队列，每个队列之间是独立的，因此可以被多核CPU并行处理。  
2、igb模块被加载，igb_driver结构已经设置好了针对igb网卡的probe、suspend、resume等函数，模块初始化函数将igb driver注册到系统中。  
3、但PCI bus准备好之后，igb网卡已经准备好开始工作，事先注册的igb_probe()被调用，启动igb网卡。  
3.1、igb_probe()做的工作：在PCI bus上启用网卡，设置了I/O memory，设置了设备相关的回调函数(如open、close)，调用igb_sw_init设置一些软件状态并且准备interrupt system，配置了一些硬件细节，确保设备处于一个已知的合法的状态。  
3.2、上述每一步都很重要，却并不特别，其他PCI设备也是这么初始化的。  
3.3、igb网卡真正开始工作，实在igb_open调用之后。  
4、启用igb网卡之后，调用igb_open()。在这里才开始做网卡的实事。  
4.1、分配发包、收包需要的所有资源。  
调用igb_setup_all_tx_resources分配transmit resources。  
调用igb_setup_all_rx_resources分配receive resources。  
现代网卡都支持多队列(rx_rings)，负载可被多核摊分处理，因此分配rx resources时，是对每个队列进行资源分配。  
这里的resource具体指代igb_buffer结构，它是联系igb网卡硬件相关的descriptor ring和软件相关socket buffer的纽带。  
![](/images/igbcard_skb.png)

在上图中，网卡维护一个descriptor ring，ring中每个条目指向主存的一块区域，这块区域就是socket buffer，网络数据包就是写在这块区域中。ring中的每个条目还包含包的元信息，或网卡状态。  
上图右侧，内核维护一个socket buffer pool。不难想象，使用pool的目的是加速。  
网卡和内核唯一的通信方式是中断，当网卡收到一个数据包时，它通过中断告知内核，由内核做包处理。  

igb_alloc_rx_buffers_adv做socket buffer的分配，并关联hardware descriptor。  
socket buffer是通过 **DMA** 在网卡(hardware)和内核(software)之间建立映射的，因此数据包在network stack之间穿梭时，不需要内存拷贝。  
4.2、设置interrupt handler，但收到包时，内核才知道如何处理。  
需要设置两个中断函数：硬件中断函数和软件中断函数。  
硬件中断：igb_open->igb_request_irq。现代网卡支持多接收队列，因此网卡应该支持多个硬件中断，以区分是哪个接收队列收到数据了。  
硬件中断的设计目标是快速，一旦完成便触发相关的软件中断，软件中断在一个更加安全的上下文环境中执行，并且不阻止其他中断。所以igb_request_irq会使用netif_napi_add注册一个softIRQ。  
5、网卡可以工作了。  
收包过程：  
当一个网络包到达网卡时，网卡找到下一个可用的hardware descriptor，将网络包写到descriptor指向的socket buffer。  
当然，如果所有descriptor都已经被占用了，网卡就丢弃这个包，并告警overrun。  
当这个网络包被全部接收完成时，网卡触发中断信号，内核开始执行注册好的中断程序。之后便会经历network stack。  

---

在继续收包流程之前，先来看一些相关的数据结构：sk_buff、route table。  
*下面的内容参考了[haifux](http://www.haifux.org/)的“[Linux Kernel Networking Overview](http://www.haifux.org/lectures/172/)”，此文结尾还给出了很多资料链接。*

**sk_buff**  
前面说到，接收的数据包是写到sk_buff结构中的，可知sk_buff表示了网络包的数据和各种header。  
sk_buff变量在内核代码中一般命名为skb，它有3个成员属性：transport_header(第4层transport layer)、network_header(第3层network layer)、mac_header(第2层link layer)。无需解释便可知其大致意图。  
skb的另一个属性_skb_dst，实际上是指向dst_entry结构的指针，dst_entry包含当前skb的路由信息，路由信息和routing subsystem强相关。dst有两个重要函数指针，*input和*output，指明skb代表的网络包的去向：在network stack中传递、转发给网络其他节点或丢弃。  
skb.tstamp表示接收包时的时间戳。  

**net_device**  
表示一个网卡(nic，network interface card)。网卡一般是PCI类型的，也有USB类型的，可见net_device和前面提到的pci_dev会是强相关的。  
net_device包含网卡硬件相关的重要属性：mtu(maximum transmission unit，设备能搞定的frame最大size，每个协议有其mtu，Ethernet默认的是1500)，dev_addr(mac地址)，硬件发包函数(2.6.32代码中没有hard_start_xmit指针，却有类似的函数)。

**routing subsystem**  
路由系统，顾名思义，它记录了一个网络包该发到哪里去。  
有两种记录数据结构：routing table和routing cache，实际上只有routing table，因为routing cache顾名思义是用来加速的。  
routing table的数据结构是fib_table，fib是forwarding information base的缩写。routing cache的数据结构是rtable。  
系统中可以有多个routing table，但只有一个routing cache——这也好理解，就是这么实现的。`route -C`可以看routing cache的内容。  
routing cache是通过hash实现的，不过内核还提供了另一种cache查询方式：fib_trie，看起来是利用trie数据结构加快查询，不过需要打开内核的CONFIG_IP_FIB_TRIE选项。这个选项默认是关闭的，大概是对于一般机器来说，路由信息不会太多，hash就能够搞定，还轮不到使用trie吧。  

![](/images/route_lookup.png)  
路由查询使用的算法是LPM(longest prefix match)，举个栗子，设路由系统有两个条目，192.168.20.16/28和192.168.0.0/16，处理地址192.168.20.19时，match这两个条目，但选取最合适的192.168.20.16/28，它的子网掩码最长。  

路由有两种方式：destination-address based decisions、policy routing。  
第一种方式好理解，便是你指定发到哪个地址，我就发到哪个地址。  
policy routing，便是根据一定的rule，去决议该把包发往何处。`ip rule list`可看系统中所有的rule。

---

继续收包。OSI的七层结构图。  
![](/images/osi.png)  
前面说到，网卡中断，往skb中写数据。下一步是指针会移动到IP header，准备进入ip层。——net/ethernet/eth.c  
ip层收包函数是ip_rcv。这一层会通过routing subsystem决定包的去往：local delivery、forwarding或dropped。  
local delivery会将包往network stack的上层发，forwarding会将IP header中的ttl减一。  

一份网络文档“[Linux IP Networking](http://www.cs.unh.edu/cnrg/people/gherrin/linux-net.html)”，也讨论了network stack、socket方面的内容，摘引其中两份简图：  
![](/images/r_rx.png)  
![](/images/s_tx.png)  

---

不讲细节的话，收一个网络包，从底到上大概就是这个流程。  
先不继续往上，而是来从上往下来看看，便是我们熟知的socket编程了。  
![](/images/socket_flow.png)  

socket的实现细节，和上面说的sk_buff等有什么关系呢？  
有两本书很详细地讲了socket编程各个函数的内核实现细节：《追踪Linux TCP/IP运行》、《TCP/IP Architecture Design and Implementation in Linux》。这两本书实际上在带着你看代码，其中第一本的内核代码更新。  

socket在内核对应一个文件，socket结构(一般简写为sock)，sock结构(一般简写为sk)。其中sock结构包含很多信息，可以认为它和上面描述的sk_buff那一套建立了联系。  
socket()函数的作用就是初始化这三个东西。至此，可以不负责任地说：listen()、accept()、connect()不过是做类似的事情而已，更新sock等结构，更新TCP状态，通过网卡发包和收包。  
需要特别指出四点，从中可窥一斑：  
1、系统中能创建多少TCP socket，是有上限的，由一个变量指示。  
2、服务端会创建client_sock之类的结构，表示和其建立连接的客户端。  
3、服务端socket有一些队列，如SYN队列、req队列之类。  
4、一切皆文件，socket也不例外，它和VFS是这么建立关联的：  
![](/images/socket_vfs.png)  

---

TCP在建立连接有著名的“三步握手协议”，在关闭连接时有半关闭问题。  
![](/images/tcp_flow.png)  
