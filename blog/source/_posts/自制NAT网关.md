---
title: 自制NAT网关
date: 2020-07-07 11:03:20
tags:
- 网络
- NAT
---

前面的Blog也提到了，DHCP服务只负责分配IP，路由器里面DHCP服务并不是入网服务，那么真正的入网是哪一部份负责呢？

这里提一个大家都很熟悉的名字：网关。

![网关](/img/NAT_IMG/GATEWAY.png)

网关的在Wiki定义是：转发其他服务器通信数据的服务器，接受客户端发送来的请求时，它就像自己拥有资源的服务器一样对请求进行处理。

简单来说，网关连接了内部局域网与外部互联网的通信，路由器里面也同样存在一个节点拥有该功能，这个节点可以是软件也可以是硬件。

需要注意的是，路由与网关概念并不相同，只是目前网络环境中，路由器往往也同时作为网关出现在网络拓扑链路中，慢慢的这俩东西好像就合二为一了。

其实当前网路环境中，路由的概念逐渐模糊，主要是因为现在的网络为了简化网络拓补图，一台终端往往只有一个出口，路由就显得不那么重要了。

路由其实更多的功能是选径功能，让报文能更快更准确的到达要通信的目标，路由功能其实不只是路由器具有，每台终端基本都有这个功能。

说回网关，我们先类比一下PPPoE服务器，PPPoE服务器通信的时候，模型可以简化成这样：
![PPPoE服务器功能](/img/NAT_IMG/PPPOE_SERVER.png)

不难看出，PPPoE服务器本身也是一台网关设备。

路由器中网关与此类似，但是没有PPPoE这么复杂，报文内容从IPv4→IPv4，不涉及协议转换的内容，大概是这么个场景：
![Router](/img/NAT_IMG/ROUTER.png)


从内网转发到外网，协议不需要做转换，只是将通信设备信息做变换，完成内外网信息交换，这个技术就是我们今天的主角：NAT。

### 初识NAT

NAT(Network Address Translation)是一种局域网设备想要与外部通信时，不能或者不想使用外部IP作为通信节点进行通信所需要的使用的一种方法，内部设备要访问外部网络资源时，数据传送给网关，网关会对外分派一个出口，然后记录对应关系，外部服务器通过该出口与网关通信，网关将该出口收到的数据转发给对应的内部设备。

旧版本的NAT中，给内网设备分配的出口是外网IP地址，一个IP地址对应一台设备，这种NAT又叫做静态NAT，也叫基本网络地址转换（Basic NAT）。目前市面上使用这种NAT技术的服务已经不多了，主要是因为公网IP资源紧张，ISP往往只给每个客户一个公网IP地址，这种NAT没有用武之地。

目前我们所接触到的NAT更多的是网络地址端口转换（NAPT），也是路由器网关采用的技术。这种方法的策略采用的是端口映射技术，多台内网设备公用一个外网IP，网关分配给每个设备/服务不同的端口，维护一份外网端口到内网IP终结点的映射。

举个例子：
![路由器网络](/img/NAT_IMG/ROUTER_TRACE.png)

在上面的网络中，如果内网设备发起了一个请求，它的流程是这样的：
```js
DEVICE           ACTION                                              TARGET
Inside device    Request   192.168.0.2:36210   → 10.91.56.4:80       Router gateway
Router gateway   Request   10.81.137.54:36212  → 10.91.56.4:80       Next gateway/Server
                 Record    192.168.0.2:36210   ↔ 36212               -
Server response  Response  10.91.56.4:80       → 10.81.137.54:36212  Router gateway
Router gateway   Search    36212               ↔ 192.168.0.2:36210   -
                 Response  10.91.56.4:80       → 192.168.0.2:36210   Inside device
```

从例子中我们可以看出，网关对于发往外部的请求，修改请求来源为自己，对于转发到内部的请求，修改请求的目标到客户端，其他内容不做修改。

看起来我们要实现NAT，我们只需要修改刚刚提到的东西就好了。但是真的是这样吗？

### 校验和

网络通信中一个很重要的内容就是数据校验，如果数据传输出错了，那么通信就是无效的。

OSI模型中，除了链路层，网络层和传输层都有校验和，协议中只要有一个字节被修改都有可能引起校验和错误。

所以只要你修改了报文中任何一个字节，影响到的校验和就必须重新计算，这就是一个NAT网关所必须具备的基本能力。

下面分别介绍一下各部分校验和算法。

#### 网络层校验和（IPv4头部）

先了解IPv4报文头结构（括号内单位**bit**）：
![IPv4报文头](/img/NAT_IMG/IPHeader.png)

各部分介绍：
**version**: IPv4，所以是4。
**IHL**: IPv4报文头长度，长度值为该值 ×4 得到的数值，所以头部长度永远是4的倍数。
**TOS**: 服务类型，不做关注。
**totalLength**: 报文总长，包含头部和下层报文的全长，也其实就是报文去掉链路层的长度。
**id**: 数据包标示，唯一。
**flag**: 三位flag表示R，DF，MF，与报文分片有关，这里不做过多说明。
**fragmentOffset**: 与flag结合使用，分片数据重定位使用，使用时该值 ×8。
**TTL**: 每经过一个路由减一，当TTL变成0时，报文会被丢弃。
**protocol**: 报文类型：ICMP - 1、IGMP - 2、TCP - 6、UDP - 17。
**checkSum**: **校验和**。
**srcIP**: 请求来源IP。
**dstIP**: 请求目的IP。
**options**: 选项字段。

一般来说，头部内容我们需要修改的就是srcIP、dstIP、TTL。IHL和totalLength如果你对报文长度进行了更改那就也需要修改。
修改完这些之后checkSum要发生变化，算法如下：
>1. 将0x0000填写到IPv4报文头的checkSum字段。
>2. 初始化计算结果为```uint32```类型，初始值0。
>3. 把IPv4报文头数据，按照16bit一节分节，将其视为```uint16```把各部分累加到结果中。
>4. 由于会产生进位，结果可能大于```uint16```的长度，如果大于```uint16```最大长度，将超出16位的部分提出并且```>> 16```，得到数据与低16位相加，重复这个过程，直到结果在```uint16```范围内。
>5. 对结果取反，取低16位，得到校验和。

将计算好的校验和重新填入报文中，IPv4头部的报文就修改完毕了。

#### TCP校验和

TCP报文组成结构（括号内单位**bit**）：
![TCP报文](/img/NAT_IMG/TCPHeader.png)

各部分介绍：
**srcPort**: 来源端口号。
**dstPort**: 目标端口号。
**seqNumber**: 本地发送序列的确认号。
**ackNumber**: 远端数据序列的确认号。
**Hlen**: 头部长度，跟IP层一样 x4 得到的数值才是头部长度。
**flags**: TCP流控制标示，不做赘述。
**windowSize**: 自己的窗口大小。
**checkSum**: **校验和**。
**uPointer**: 当flags中URG设置位1时用作数据重定位。
**options**: TCP头选项。
**data**: TCP数据。

一般来说，这部分我们可能修改的有srcPort、dstPort。Hlen看情况，一般是不变的。
修改之后，checkSum计算方法是：
>1. 在TCP头部补充伪报文头，最终数据结构如下：
>![伪报文头](/img/NAT_IMG/TCP_ADDON.png)
>其中protocol与IP报文头数值一样，前方补充0x00拓展成16位，length为TCP数据总长，包括TCP头部。
>2. 把0x0000填写到TCP报文的checkSum字段。
>3. 将加上伪报文头的数据按照IPv4报文头的checkSum计算方法进行计算，过程不赘述。

最后，得到的校验和要重新填入TCP校验和对应的位置。

#### UDP校验和

UDP报文组成结构：
![UDP报文](/img/NAT_IMG/UDPPacket.png)

各部分内容在TCP那边大部分都有介绍，length表示的是UDP数据总长，包括UDP头部。

checkSum的计算方法与TCP类似，填入伪报文头然后计算，不再说明。

### 发送和接收

路由器是一个网络层设备，所以网关也是主要转发网络层报文。

所以完成一个NAT网关，我们也需要监听网络层的所有报文，这种监听方式操作系统中有对应的Socket实例，那就是Raw Socket，Raw Socket对应各个操作系统如何实现请自行查阅资料。

在处理内部发过来的报文时，我们要过滤掉不需要转发到外部的报文，把真正需要转发的报文转发出去以节省带宽资源。

发送的时候，有个知识点，如果你抓包查看客户端发给网关的报文，你会发现dstIP虽然是外部资源，但是dstMac填写的确是网关的Mac。你可能会很好奇，IP和Mac竟然还可以不对应？

这其实就是OSI模型中链路分时复用的精髓，链路层就是Mac到Mac，不管你上面承载的数据，IP才需要处理路径选择，查看报文到底要转发到何方。我们发送的时候采用的也是这种方法，找到对应路径的Mac，送过去。

所以我们发送的时候，事实上是提交了一个链路层报文，而非网络层报文，你可以尝试一下使用Raw Socket发送数据，不会达到我们想要的效果。我们需要从链路层的Mac地址开始，全部进行修改，然后组织好数据通过链路层发送这个报文。

所以内网转发到外部时，我们实际做出的修改有：srcMac改为对外出口的Mac，dstMac修改为下一跳的目标，srcIP改为对外出口的IP，srcPort改为分配好的port。
外网数据返回转发到内部时，我们需要修改的有：srcMac改为对内出口的Mac，dstMac改为对应客户端的Mac，dstIP改为对应客户端的IP，dstPort改为对应客户端的请求port。

链路层编程在 [PPPoE服务器](/2018/02/12/PPPoE服务器编写) 里面已经有提及，这里不再赘述。

### NAT类型

当前网络中的NAT有如下四种类型，选择实现任何一种类型都可以完成NAT的工作：
![NAT类型](/img/NAT_IMG/NAT_TYPE.png)

**完全锥形**：内网设备一接入之后，发起任意一个外部请求，NAT便打通内部IPEP到外部端口的映射，任意外部设备都可以通过该端口与内网设备通信。
**受限锥形**：内网设备接入之后，发起一个对外部设备访问的请求，随后NAT开启一条转换路径，被指明访问的设备可以通过这个端口与内网设备通信（发起端口，类型都无限制，可以复用），其他的外部设备无法通过这个端口通信。
**端口受限锥形**：内网设备接入之后，发起一个对外部设备某个端口的访问请求，随后NAT开启一条转换路径，被指明的外部设备只能通过被点名的端口与内网设备通信（通信类型无限制，可以复用），其他的外部设备无法通过这个端口通信。
**对称型**：内网设备接入之后，每一个请求，只要源地址，目标地址，源端口，请求端口，请求协议类型有一个被修改，NAT设备就建立一个映射关系，每个请求都走的不同的出口，无法复用，这也是P2P网络最头疼的NAT类型。

### 其他技术细节

#### 网卡

网卡是我们通往成功路上的绊脚石之一。

由于checkSum计算很浪费系统资源，所以目前市面上大部分网卡都提供checkSum offload功能，简单来说，就是将checkSum的计算工作交给网卡代理，不管是发出还是接收报文。

这项技术本身是一项好技术，但是，经过实验发现，打开了checkSum offload会让我们本身计算正确的checkSum变成错的，导致checkSum校验不通过报文被丢弃。

所以我们在实现NAT程序之后，wan和lan口的网卡checkSum offload功能一定要关闭掉发送时的offload功能。
> *checkSum offload功能分为两部分，Rx表示接收 Tx表示发送，关闭Tx即可，不可关闭Rx功能，会导致无法收到报文*

*网上很多资料说，如果有checkSum offload功能，TCP、UDP的checkSum字段填0x0000即可，我试了，从来没成功过，有哪位道友研究出来了欢迎分享结果。*

#### 防火墙

由于我们转发的报文中，肯定包含TCP报文，然而我们的程序又没有正常的走TCPSocket到系统内核，这样会导致系统认为这些TCP报文来源非法，内核触发自我保护机制，对TCP链路直接进行RST重置，导致我们报文虽然转发成功，但是依旧无法通信。

这个问题很好解决，将程序添加到系统防火墙白名单即可。

*这里也算是当时踩过的坑之一，所以这里提示一下后来者。*

### 成果展示
![NAT类型](/img/NAT_IMG/NAT_PROGRAM.png)
该实现中的NAT类型是全锥形。

不要问为什么只有一个Lan？我哪儿来那么多网口啊kora(╯‵□′)╯︵┻━┻
> *其实也用不到那么多网口，lan只需要一个，然后通过集线器拓展多台设备即可，NAT可以很好的记录他们的对应关系，路由器上面的lan口大多也是这种配置类型。*

实测结果是：看视频，玩游戏，浏览网页完全没有问题，注意结合 [DHCP服务器](/2018/02/11/DHCP服务器编写) / [PPPoE服务器](/2018/02/12/PPPoE服务器编写) 一起使用，体验更佳。

### 后记

如果你的电脑存在多个网卡，又想省下一个路由器的钱，自己制作一个NAT网关是很好的选择。

当然，优秀的网络管理软件有很多，但是大部分都不在Windows上，本文主要是通过NAT原理，指明一条自己实现NAT网关的道路，当然Windows自带的也有网络共享这种东西，但是秉承着对知识探求的精神，自己研究出这套解决方案的实现方法还是很有成就感的。

在实现整个NAT技术栈过程中，个人感觉对网络链路层和网络层的分时复用有了更深入的理解，整体下来对自己网络知识技术的提升有一定帮助。

不过我没有实现的功能是网络层其他报文的转发，比如ICMP/IGMP报文，感兴趣的可以自己尝试实现一下。

### 结语

时隔……我看下……两年之后，我又回来啦。
人是有惰性的，摸鱼一时爽，一直摸一直爽。
但是，摸多了总是隐隐感到一丝不安，你不进步就有可能落后。
所以在隔了这么久之后，重新捡起来以前的博客。
学习才能使人快乐，获得幸福。
与大家共勉。
