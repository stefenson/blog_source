---
title: PPPoE服务器编写
date: 2018-02-12 09:52:25
tags:
- 网络
- PPPoE
---

按照约定，今天简单说一下编写PPPoE服务器。

PPPoE（Point-to-Point Protocol Over Ethernet）协议是一个以太网上的点对点协议，是将点对点协议（PPP）封装在以太网（Ethernet）框架中的一种网络隧道协议。
它是一个链路层协议，如果要想自己实现一个PPPoE服务器需要了解链路层编程的相关知识。
网络五层架构中：应用——传输——网络——链路——物理。链路层属于比较靠底层的一层，所以难度还是有的。编程的时候相当于是直接与网卡打交道，编写难度比较高。

这里在Windows中，我比较推荐 [WinPcap](https://www.winpcap.org) 进行编程，虽然Windows中也有面向网卡的编程，但是一般只有C的接口，并且相关资料匮乏，往往还需要与各个型号的网卡打交道，所以推荐使用WinPcap，接口统一并且易于掌握，只是有些时候速度不太理想。
使用WinPcap，你甚至可以发送一个一字节的报文（虽然他毫无意义）！在WinPcap里，TCP/IP报文头将不再是限制！你甚至不会被链路层默认封装头限制！报文里的每一个字节全部由你自由定制！WinPcap！你值得拥有！

咳咳，扯远了（感觉跟打了个广告一样……）下面说回PPPoE编程。

### PPPoE工作流程
[![PPPoE流程](/img/PPPoE_PROC.png)](/img/PPPoE_PROC.png)
上面的图是一个PPPoE完整的工作流程（点击可查看大图）。
双向箭头的地方表示顺序可以互换，任何一方都可以发起，没有严格要求谁先发起。

从图中可以看出，PPPoE流程分为五个阶段。
其实不细分，PPPoE只有两个阶段，一个是PPPoED阶段，后面的LCP/Auth/IPCP/PPP全部都可以纳入PPP Session阶段。

PPPoE服务器在后续数据交互的过程还要参与，它不仅仅是提供IP配置给客户端，后续通讯中全部建立在PPP Session之上，服务器需要处理这些Session包，一是与客户端通讯确认是否在线管理session，二是对数据进行重新封包，去掉PPP报文头，转换为一般的报文与外部目标设备通讯，并且吧通讯结果使用PPP封包发送给客户端。所以说PPPoE服务器对服务器性能要求非常高。
从上面的描述中也可以看出PPP Session是一个很重要的概念，PPPoE通讯中，链接保持、数据传输、链路通讯、链路状态汇报和修改全部都是在PPP Session通讯上进行的，Session是一个PPPoE通讯中的关键通道。

下面就每一个阶段讲解其流程和报文结构。

### 链路报文头
首先介绍一下链路报文头：

| Destination | Source  | Type    |
|-------------|---------|---------|
| 6 Bytes     | 6 Bytes | 2 Bytes |

由于我们做的是链路层编程，报文里面每一个字节都需要我们填进去，所以开头这一部分我们也需要了解。
Destination：目标的Mac地址，PADI阶段客户端会使用FF:FF:FF:FF:FF:FF广播地址。
Source：来源MAC，该值在组织报文时理论上可以随意设置，可以不与当前发送设备的MAC一致，链路层编程时Source不会被自动填入当前设备的MAC地址，所以这里可以随便写（要来搞事吗www）。
Type：报文类型，目前我所知道的取值有这些
>0x8863.................PPPoE Discovery
>0x8864.................PPPoE Session
>0x0800.................IPv4

PPPoE Discovery对应的就是PPPoED阶段报文类型，在PPPoED阶段Type取该值。
PPPoE Session对应的是PPPoED阶段之后链路通讯的报文类型，在LCP/Auth/IPCP/PPP阶段Type都使用该值。
IPv4为目前网络中使用的普通报文类型，现阶段v6没有普及，如果不使用PPPoE方式进行网络接入，报文类型一般都是IPv4。

### PPPoE报文通用部分
在链路报文头之后，PPPoE报文有一段通用的部分：

| Version&Type | Code    | Session ID | Payload Length |
|--------------|---------|------------|----------------|
| 1 Bytes      | 1 Bytes | 2 Bytes    | 2 Bytes        |

Version&Type：现阶段只有一个取值：0x11，表示PPPoE版本1，类型1。
Code：表示PPPoE报文细分类型，目前有如下取值

| Code值 | 含义         | 描述                |
|--------|--------------|---------------------|
| 0x00   | Session Data | PPP流程通讯时的类型 |
| 0x09   | PADI         | PADI                |
| 0x07   | PADO         | PADO                |
| 0x19   | PADR         | PADR                |
| 0x65   | PADS         | PADS                |
| 0xA7   | PADT         | PADT                |

PADI/PADO/PADR/PADS/PADT是PPPoED阶段使用的Code，Session Data是PPPoE Session阶段使用的Code。
其中Session ID在PADI/PADO/PADR阶段为0x0000，在PADS中服务器会分配一个Session ID给客户，在PADT中需要携带Session ID告知接收方要断开的Session。
Session Data阶段，Session ID表示客户端与服务器之间建立的Session编号。
Payload Length为后面跟的数据总长，不包括PPPoE头和链路头。

### PPPoED（PPPoE Discovery）
PPPoED阶段主要工作是客户端发现并确定要注册的PPPoE服务器，在相关的PPPoE服务器注册Session。
它的流程简单描述如下：
>1.客户端通过PADI通知PPPoE服务器自己的接入请求。
>2.PPPoE服务器接到PADI请求之后判断是否要答复，需要答复回复PADO。
>3.客户端判断PADO内容是否是自己期望的服务器，如果是，回复PADR申请Session。
>4.PPPoE服务器收到了PADR，根据当前服务器状况判断是否要开启Session，如果可以接受回复PADS。

去掉开头的链路报文头和PPPoE通用部分，PPPoED阶段报文数据形式：

| Tag Type | Tag Length | Tag Data |
|----------|------------|----------|
| 2 Bytes  | 2 Bytes    | x Bytes  |

PPPoED中Payload就是PPPoE Tag的列表，上面这一段会重复多次，他的各个部分含义：
Tag Type表示类型，可取值

| Type值   | 含义               | 备注                       |
|----------|--------------------|----------------------------|
| 0x0000   | End of list        | Tags列表结束标志           |
| 0x0101   | Service-Name       | 服务器名称，客户端携带，说明选择特定的服务器 <br> 长度可以为0表示接受所有PPPoE服务器 <br> 一般情况下长度是0 |
| 0x0102   | AC-Name            | 服务器返回服务器自己的名字 |
| 0x0103   | Host-Uniq          | Discovery阶段特征值，作用类似于SessionID <br> 每个客户端在一个流程中会保持该值唯一 <br> 服务器答复的时候最好对该值进行判断并维持该值返回 |
| 0x0104   | AC-Cookie          | -                          |
| 0x0105   | Vendor-Specific    | -                          |
| 0x0110   | Reply-Session-ID   | -                          |
| 0x0201   | Service-Name-Error | -                          |
| 0x0202   | AC-System-Error    | -                          |

以上有备注说明的是比较重要的字段，也是一个PPPoE服务器的基本能力所应具备的字段，其他字段解释请查阅RFC文档。
AC-Name当然可以自己发挥了，比如我写的就是stefenson-pppoe-server，当然也可以迎合客户端要求，根据客户端 0x0101 所需进行返回，机不机智？

Tag Length表示Tag的数据长，不包含Type和Length占用的长度。

Tag Data表示Tag携带的数据。

### PPPoE Session阶段共用部分
PPPoED成功完成之后进入PPPoE Session阶段。
Session阶段会先携带一个通用的两字节数据 Point to Point Protocol
该值表示接下来的Session数据是哪种类型，包括以下取值
>0x8021..............IPCP
>0xC021..............LCP
>0xC023..............PAP
>0xC223..............CHAP
>0x0021..............IPv4

<u>当取值 0x0021 时，该报文携带的是要通讯的数据，后面的Data与链路报文类型IPv4内容一致，去掉PPPoE部分，修改链路头Type为IPv4这就是一个正常的IPv4报文。</u>
** 注意这一点在后面数据链路通讯时十分重要，是PPPoE真正实现通讯的基本要点。 **

紧接着的内容，如果不是IPv4，数据组织形式一样：

| Code    | Identifier | Length  | Data    |
|---------|------------|---------|---------|
| 1 Bytes | 1 Bytes    | 2 Bytes | x Bytes |

Code根据不同类型的有不同的意义，后面分开进行说明。
Identifier作为PPP阶段报文应答标识，用于区别报文应答关系。
Length为全长，包括Code、Identifier、Length占用的长度，不仅仅是Data的长度，也就是说 Length = Data.Length + 4。

### LCP Discussion
LCP全称 Link Control Protocol，链路控制协议，控制链路吞吐量，链路通断等链路基本信息。
它的Code有如下取值

| Code值 | 含义                  | 描述         |
|--------|-----------------------|--------------|
| 0x01   | Configuration Request | 链路协商     |
| 0x02   | Configuration ACK     | 协商通过     |
| 0x03   | Configuration NAK     | 协商拒绝     |
| 0x04   | Configuration Reject  | 协商回绝     |
| 0x05   | Terminate Request     | 链路终止请求 |
| 0x06   | Terminate ACK         | 终止请求确认 |
| 0x09   | Echo Requeest         | 心跳请求     |
| 0x0A   | Echo Reply            | 心跳确认     |

NAK和Reject的区别是，NAK表示对方的所列出的链路配置恕无法接受，而Reject则表示发过来的消息意义不明或者信息缺失，这边解析不了很迷茫（看不懂你到底要表达啥的意思）。

Data的组织形式为下面结构的多项叠加

| Type    | Length  | Data    |
|---------|---------|---------|
| 1 Bytes | 1 Bytes | x Bytes |

这里的Length表示全长，包括Type和Length占用的字节，所以Length = Data.Length + 2。

Type取值和含义
>0x01..............MRU（Maximum Receive Unit）
>0x02..............Async-Control-Character-Map
>0x03..............Auth Type
>0x05..............Magic Number
>0x06..............Link Quality Compression/Monitoring
>0x07..............Procotol Field Compression
>0x08..............Address and Control Field Compression

当Type为MRU时，长度为 2 Bytes，表示报文发送方最大可接收的报文大小，一般不超过1484。
当Type为Auth Type时，Data可取值有这些：0xC023表示采用PAP认证、0xC223 + 0x05表示使用MD5挑战方式的CHAP认证。当然也存在其他取值，具体可参阅RFC文档。
当Type为Magic Number时，长度为 4 Bytes，值随机，答复的时候要带上用于辨识答复方数据可靠性。
其他的Type含义参阅RFC文档，这里所说的是几个比较重要的Type值，这几个是实现PPPoE服务器中必须处理和了解的几个值。

LCP Discussion阶段中双方互相发送Configuration Request，相互确认对方的信息，同意回复Configuration ACK并携带对方的Request信息。
如果不同意对方的Request，这时候回复NAK，并且在NAK中携带自己期待的配置信息，收到NAK之后如果接受列出的配置，带上NAK中的配置重新Configuration Request。
如果发现对方报文中存在错误，这时候需要回复Reject，并且携带对方报文中意义不明或者错误的信息内容等待进一步答复。
>比如 Configuration Request: Auth Type 0x0000，这时候接收方会回复Configuration Reject: Auth Type 0x0000，相当于该信息我不知道它对不对，反正我这解析不出来，我不管你得换一个。

LCP Discussion过程可能会来回多次，反复确认双方配置（Request--NAK--Request--Reject--Request--...)，如果在多次Configuration依旧无法协商完成只能LCP Terminate Request结束这个短暂的邂逅了。虽然这种情况一般很少见，但也不排除，相当于两个人谈一场交易，结果怎么也谈不拢，那交易肯定也没法进行，再死皮赖脸的协商怕不是要打一架了。

_ 其实这个过程看起来就跟两个人在进行一场协商一样的不是吗www？_
_ 用自然语言表述这个过程可以编这么一个对话 _
>A：请求：我可接收最大重量为1484的包裹，暗号7A908B06，请确认。
>B：OK我方收到，已知阁下可接收最大重量为1484的包裹，答复暗号7A908B06。
>B：另外我方最大可接受重量为1400的包裹，认证方式编码C023，暗号7A908B07，请确认。
>A：哈？你现在还在用C023认证？你不怕泄密吗？回绝（Reject）认证方式编码C023，你得换一个，答复暗号7A908B07。
>B：真是事多……重新请求：我方最大可接受重量为1400的包裹，认证方式编码C223+05，暗号7A908B08，请确认。
>A：等等，1400最大重量太小了吧，我这边接受不了，拒绝（NAK），你得把配置换成这样：最大可接受重量为1450的包裹，认证方式编码C223+05，答复暗号7A908B08。
>B：(MMP……)行行行，再次请求：我方最大可接受重量为1450的包裹，认证方式编码C223+05，暗号7A908B09，这回可以了吧？
>A：……
>B：……喂？
>A：……啊，不好意思，我这边刚刚有点事没听清，你再说一遍？
>B：你大爷……时间到了，老子不伺候了！再见！
>------用户 B 断开连接------
>A：？？？？？？

### Auth（认证）
LCP协商之后接下来就是认证了。
由于认证的方式是在LCP阶段确认的，所以这里根据不同的认证方式有两种数据组织方式

#### PAP
PAP阶段Code取值有
>0x01..........Auth Request
>0x02..........Auth ACK
>0x03..........Auth NAK

当Code为Auth Request时Data组织形式如下

| Peer-ID-Length | Peer-ID | Password-Length | Password |
|----------------|---------|-----------------|----------|
| 1 Bytes        | x Bytes | 1 Bytes         | x Bytes  |

其中Peer-ID就是用户名，Password为密码。两个Length都是指后面数据长，不包括Length所占用的字节。
从这里不难看出，PAP直接携带用户名密码，并且所支持的最大用户名长和密码长都是255字节。
使用PAP方式，客户端直接发送PAP-Auth Request，服务器收到报文后解析用户名密码判断该用户是否合法，认证成功返回Auth ACK，否则返回Auth NAK。

当Code为Auth Ack或者Auth NAK的时候，后面Data先跟一字节长度提示，然后填入长度控制的字符串作为认证提示，认证提示可以随便填写，就是一个字符串。

_ 比如你可以在Auth ACK之后跟上信息：Authentication Failed! Rua!，没错就是可以这么傲娇。 _

#### CHAP
CHAP阶段Code取值有
>0x01..........Challenge
>0x02..........Response
>0x03..........Success
>0x04..........Failure

当Code为Challenge或者Response的时候Data组织形式如下：

| Value Size | Value   | Name    |
|------------|---------|---------|
| 1 Bytes    | x Bytes | y Bytes |

其中Value Size表示Value长度，一般是16。Name为用户名，以0x00结尾。
Value一般是16字节的数据，各个阶段意义略微有区别。
当时用CHAP认证的时候，服务器首先发送Challenge消息，里面的用户名可以为空。
客户端收到Challenge之后，通过之前商讨的挑战方式，重新计算Value，并携带用户名，发送Response给服务器。
服务器通过客户端答复的Value判断认证是否成功，成功返回Success，否则返回Failure。

同样的当Code为Failure或者Success的时候，后面Data是一个返回给客户端的字符串，表示认证信息，以0x00结尾。

MD5挑战模式下，Challenge的期望值为：MD5(id + password + value)。
其中id为PPPoE Session阶段携带的Identifier，value为服务器Challenge发过来的值。计算完之后重新放入Value返回给服务器。
服务器收到之后也会用同样的方式处理，对比两者结果是否一致判断用户是否合法。

认证失败一般紧接着就会发送LCP Terminate Request断开链接，释放当前Session。

### IPCP Discussion
认证完成之后就是IP协商了。
IPCP全称 IP Control Protocol，IP控制协议，用于控制接入设备的IP配置信息。
IPCP阶段Code可以取值：
>0x01..........Configuration Request
>0x02..........Configuration ACK
>0x03..........Configuration NAK

Data组织形式为：

| Type    | Length  | IP Address |
|---------|---------|------------|
| 1 Bytes | 1 Bytes | 4 Bytes    |

Type可取值
>0x03..........IP
>0x81..........DNS1
>0x83..........DNS2

Length为总长，Length = 6。
从这里不难发现，PPP接入下没有子网掩码，因为在PPP模式下不存在子网概念，所有设备相互独立，所以客户端一般在配置PPP网络的时候习惯性将子网掩码设置为255.255.255.255。

IPCP流程
服务器首先发出一个IPCP Request报文，只带0x03字段，这个报文告知客户端服务器的IP地址信息。
客户端一般在收到服务器发来的IPCP Request报文之后才发送IPCP Request请求自己的IP地址。
一般情况下，客户端发过来的请求中携带的IP信息为0.0.0.0，也有可能是上次认证时分配到的IP。
服务器收到客户端的IPCP Request之后判断配置信息是否正确，正确回复ACK，否则回复NAK并附上正确的IP配置信息。

这个过程其实跟LCP Discussion过程类似，只是起手动作需要服务器发起，客户端需要先确认服务器IP才会进行接下来的确认。
后续流程除了没有Reject消息，跟LCP Discussion流程是一样的，对了回复ACK，不对回复NAK并携带上所需的信息。

IPCP协商完成之后，PPP链接建立过程结束。

### PPP链接建立之后
PPPoE链路建立之后，上文也提到后续的数据依旧是以PPPoE Session方式发送，不过P2P Protocol的值变为0x0021（IPv4）
这时候，一般的网卡是无法解析该消息的，需要把PPP消息中PPPoE部分剔除封装成普通的IPv4报文，再发送给相关设备，这样网卡才能处理该消息。
这个策略可以考虑使用假MAC的方式，PPPoE服务器通过假MAC发送消息给相关设备，相关设备的信息发送回来之后PPPoE服务器截获该消息（其实WinPcap直接就能收到）然后重新用PPP封装在发送给客户端。
换句话说，PPPoE链路建立之后，通讯过程依旧跟普通的IPv4通信相差很大。

### PPPoE报文总览
一张图总结一下PPPoE报文
[![PPPoE报文](/img/PPPoEPacket.png)](/img/PPPoEPacket.png)


### 其他一些注意事项和技巧
- WinPcap监听会监听到Destination非自己的报文，包括自己发的，如果不过滤Source会造成报文没有被过滤，自己的报文自己答复接着进入死循环浪费资源。这点需要注意
- LCP控制报文中如果收到Echo Request需要及时回复Echo Reply，如果没有答复Echo Request，超过一定次数之后会使对方认为链路断开，链接丢失。所写的PPPoE也应该具备这个机制，毕竟Session资源紧张。
- Session报文的Identifier变化一般有两种思路，当然这个策略也可以自己设计
>1.当没有回复的时候使ID + 1重新发送，直到得到答复之后ID重置为0发送下一个消息。
>2.当Package的类型改变的时候重制ID为0，否则ID递增。

- LCP的Echo Request心跳报文中，ID应该只随心跳次数递增，双方个自己记录心跳情况。
- LCP/IPCP都需要双方各一个Request和ACK之后才可配置好链接相关信息。
- Session建立过程中，如果在某阶段耗时过长即可认为流程超时，断开并清理当前Session。
- 为了保证报文质量，PPPoE建立过程中，所有报文长度最短应为60字节，不够应当补齐。

### 写在最后
看完PPPoE协议的内容，是不是有种晕头转向的感觉？
确实DHCP协议与PPPoE协议比起来简直是小巫见大巫，毕竟链路层协议，复杂程度肯定是要上升的。
不过也不需要太敬畏它，PPPoE依旧只是一个协议，一个准则而已，有死板的地方，也有可以搞事的地方（喂）。
总的来看其实PPPoE协议就是一层一层展开的，理解了PPPoE协议其实你也就理解了网络分这么多层到底有什么意义了。
网络报文其实就是约定好的一层一层数据组装到一起，按照一定模式一定能够将其解开，这样的设计还方便了各部分只关注自己所需要关注的部分，省去了很多解析上的麻烦，只能感叹当年这些网络工程师的智慧。
这里面的很多策略都是可以自己发挥的地方，可以尝试一下设计一套自己的高效的策略。
设计PPPoE服务器最关键的地方就是学会链路层编程，理解网络报文这种层级关系，理解PPPoE协议中各个部分所需要遵循的规则（比如CHAP认证时的Challenge规则）。
由于PPPoE服务器在后期通信中依旧担当一个举足轻重的角色，所以PPPoE服务器对服务器要求很高，普通电脑性能不好的话是无法承担这个任务的，这就是为什么很少在PC平台下有PPPoE服务器软件的原因。
目前商用的PPPoE服务器都是各厂商自己设计好的一套解决方案，而不简单是一个软件，里面设计好的各个单元不同分工，这样才有能力承接PPPoE的所有工作。
所以我们设计PPPoE服务器并不是说要在普通PC上实现能用的PPPoE服务器，而是通过写PPPoE服务器的过程，理解链路层编程，理解网络分层结构，这才是最终目的。

可以拓展的地方：不适用WinPcap，使用更底层的C完成PPPoE服务器编程，这样效率更高，说不定性能可以达到和一些PPPoE服务器持平。

希望本文对你有所启发，Thanks for Reading！

### 相关文献
[RFC2561(PPPoE)](https://tools.ietf.org/html/rfc2516)
[RFC1661(PPP)](https://tools.ietf.org/html/rfc1661)
[RFC1570(LCP Etensions)](https://tools.ietf.org/html/rfc1570)
[RFC2484(LCP Option)](https://tools.ietf.org/html/rfc2484)
[RFC1334(PAP)](https://tools.ietf.org/html/rfc1334)
[RFC1994(CHAP)](https://tools.ietf.org/html/rfc1994)
[RFC2865(RADIUS)](https://tools.ietf.org/html/rfc2865)
[RFC1332(IPCP)](https://tools.ietf.org/html/rfc1332)
[RFC1877(IPCP DNS)](https://tools.ietf.org/html/rfc1877)
