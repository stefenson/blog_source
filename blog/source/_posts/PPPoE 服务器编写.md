---
title: PPPoE 服务器编写
date: 2018-02-12 09:52:25
tags:
- 网络
- PPPoE
---

PPPoE（Point-to-Point Protocol Over Ethernet）协议是一个以太网上的点对点协议，是将点对点协议（PPP）封装在以太网（Ethernet）框架中的一种网络隧道协议。
它是一个链路层协议，如果要想自己实现一个 PPPoE 服务器需要了解链路层编程的相关知识。
网络五层架构中：应用——传输——网络——链路——物理。链路层属于比较靠底层的一层，所以难度还是有的。编程的时候相当于是直接与网卡打交道，编写难度比较高。

<!-- more -->

这里在 Windows 中，我比较推荐 [WinPcap](https://www.winpcap.org) 进行编程，虽然 Windows 中也有面向网卡的编程，但是一般只有 C 的接口，并且相关资料匮乏，往往还需要与各个型号的网卡打交道，所以推荐使用 WinPcap，接口统一并且易于掌握，只是有些时候速度不太理想。
使用 WinPcap，你甚至可以发送一个一字节的报文（虽然他毫无意义）！在 WinPcap 里，TCP/IP 报文头将不再是限制！你甚至不会被链路层默认封装头限制！报文里的每一个字节全部由你自由定制！WinPcap！你值得拥有！

咳咳，扯远了（感觉跟打了个广告一样……）下面说回 PPPoE 编程。

### PPPoE 工作流程
![PPPoE 流程](/img/PPPoE_PROC.png)
上面的图是一个 PPPoE 完整的工作流程（点击可查看大图）。
双向箭头的地方表示顺序可以互换，任何一方都可以发起，没有严格要求谁先发起。

从图中可以看出，PPPoE 流程分为五个阶段。
其实不细分，PPPoE 只有两个阶段，一个是 PPPoED 阶段，后面的 LCP/Auth/IPCP/PPP 全部都可以纳入 PPP Session 阶段。

PPPoE 服务器在后续数据交互的过程还要参与，它不仅仅是提供 IP 配置给客户端，后续通讯中全部建立在 PPP Session 之上，服务器需要处理这些 Session 包，一是与客户端通讯确认是否在线管理 Session，二是对数据进行重新封包，去掉 PPP 报文头，转换为一般的报文与外部目标设备通讯，并且把通讯结果使用 PPP 封包发送给客户端。所以说 PPPoE 服务器对服务器性能要求非常高。
从上面的描述中也可以看出 PPP Session 是一个很重要的概念，PPPoE 通讯中，链接保持、数据传输、链路通讯、链路状态汇报和修改全部都是在 PPP Session 通讯上进行的，Session 是一个 PPPoE 通讯中的关键通道。

下面就每一个阶段讲解其流程和报文结构。

### 链路报文头
首先介绍一下链路报文头：

| Destination | Source  | Type    |
|-------------|---------|---------|
| 6 Bytes     | 6 Bytes | 2 Bytes |

由于我们做的是链路层编程，报文里面每一个字节都需要我们填进去，所以开头这一部分我们也需要了解。
Destination：目标的 MAC 地址，PADI 阶段客户端会使用 FF:FF:FF:FF:FF:FF 广播地址。
Source：来源 MAC，该值在组织报文时理论上可以随意设置，可以不与当前发送设备的 MAC 一致，链路层编程时 Source 不会被自动填入当前设备的 MAC 地址，所以这里可以随便写（是的，你写成其他设备 MAC 也可以）。
Type：报文类型，目前我所知道的取值有这些
>0x8863.................PPPoE Discovery
>0x8864.................PPPoE Session
>0x0800.................IPv4

PPPoE Discovery 对应的就是 PPPoED 阶段报文类型，在 PPPoED 阶段 Type 取该值。
PPPoE Session 对应的是 PPPoED 阶段之后链路通讯的报文类型，在 LCP/Auth/IPCP/PPP 阶段 Type 都使用该值。
IPv4 为目前网络中使用的普通报文类型，现阶段 v6 没有普及，如果不使用 PPPoE 方式进行网络接入，报文类型一般都是 IPv4。

### PPPoE 报文通用部分
在链路报文头之后，PPPoE 报文有一段通用的部分：

| Version&Type | Code    | Session ID | Payload Length |
|--------------|---------|------------|----------------|
| 1 Bytes      | 1 Bytes | 2 Bytes    | 2 Bytes        |

Version&Type：现阶段只有一个取值：0x11，表示PPPoE版本1，类型1。
Code：表示 PPPoE 报文细分类型，目前有如下取值

| Code 值 | 含义         | 描述                |
|--------|--------------|---------------------|
| 0x00   | Session Data | PPP 流程通讯时的类型 |
| 0x09   | PADI         | PADI                |
| 0x07   | PADO         | PADO                |
| 0x19   | PADR         | PADR                |
| 0x65   | PADS         | PADS                |
| 0xA7   | PADT         | PADT                |

PADI/PADO/PADR/PADS/PADT 是 PPPoED 阶段使用的 Code，Session Data 是 PPPoE Session 阶段使用的 Code。
其中 Session ID 在 PADI/PADO/PADR 阶段为 0x0000，在 PADS 中服务器会分配一个 Session ID 给客户，在 PADT 中需要携带 Session ID 告知接收方要断开的 Session。
Session Data 阶段，Session ID 表示客户端与服务器之间建立的 Session 编号。
Payload Length 为后面跟的数据总长，不包括 PPPoE 头和链路头。

### PPPoED（PPPoE Discovery）
PPPoED 阶段主要工作是客户端发现并确定要注册的 PPPoE 服务器，在相关的 PPPoE 服务器注册 Session。
它的流程简单描述如下：
>1.客户端通过 PADI 通知 PPPoE 服务器自己的接入请求。
>2.PPPoE 服务器接到 PADI 请求之后判断是否要答复，需要答复回复 PADO。
>3.客户端判断 PADO 内容是否是自己期望的服务器，如果是，回复 PADR 申请 Session。
>4.PPPoE 服务器收到了 PADR，根据当前服务器状况判断是否要开启 Session，如果可以接受回复 PADS。

去掉开头的链路报文头和 PPPoE 通用部分，PPPoED 阶段报文数据形式：

| Tag Type | Tag Length | Tag Data |
|----------|------------|----------|
| 2 Bytes  | 2 Bytes    | x Bytes  |

PPPoED 中 Payload 就是 PPPoE Tag 的列表，上面这一段会重复多次，他的各个部分含义：
Tag Type 表示类型，可取值

| Type 值   | 含义               | 备注                       |
|----------|--------------------|----------------------------|
| 0x0000   | End of list        | Tags 列表结束标志           |
| 0x0101   | Service-Name       | 服务名称，客户端携带，说明期望服务器具备的功能 <br> 长度可以为 0 表示接受所有类型 PPPoE 服务器 <br> 一般情况下长度是 0 |
| 0x0102   | AC-Name            | 服务器返回服务器自己的名字 |
| 0x0103   | Host-Uniq          | Discovery 阶段特征值，作用类似于 Session ID <br> 每个客户端在一个流程中会保持该值唯一 <br> 服务器答复的时候最好对该值进行判断并维持该值返回 |
| 0x0104   | AC-Cookie          | -                          |
| 0x0105   | Vendor-Specific    | -                          |
| 0x0110   | Reply-Session-ID   | -                          |
| 0x0201   | Service-Name-Error | -                          |
| 0x0202   | AC-System-Error    | -                          |

以上有备注说明的是比较重要的字段，也是一个 PPPoE 服务器的基本能力所应具备的字段，其他字段解释请查阅 RFC 文档。
AC-Name 当然可以自己发挥了，比如我写的就是 stefenson-pppoe-server，起名字嘛，创意最重要。

Tag Length 表示 Tag 的数据长，不包含 Type 和 Length 占用的长度。

Tag Data 表示 Tag 携带的数据。

### PPPoE Session 阶段共用部分
PPPoED 成功完成之后进入 PPPoE Session 阶段。
Session 阶段会先携带一个通用的两字节数据 Point to Point Protocol
该值表示接下来的 Session 数据是哪种类型，包括以下取值
>0x8021..............IPCP
>0xC021..............LCP
>0xC023..............PAP
>0xC223..............CHAP
>0x0021..............IPv4

<u>当取值 0x0021 时，该报文携带的是要通讯的数据，后面的 Data 与链路报文类型 IPv4 内容一致，去掉 PPPoE 部分，修改链路头 Type 为 IPv4 这就是一个正常的 IPv4 报文。</u>
** 注意这一点在后面数据链路通讯时十分重要，是 PPPoE 真正实现通讯的基本要点。 **

紧接着的内容，如果不是 IPv4，数据组织形式一样：

| Code    | Identifier | Length  | Data    |
|---------|------------|---------|---------|
| 1 Bytes | 1 Bytes    | 2 Bytes | x Bytes |

Code 根据不同类型的有不同的意义，后面分开进行说明。
Identifier 作为 PPP 阶段报文应答标识，用于区别报文应答关系。
Length 为全长，包括 Code、Identifier、Length 占用的长度，不仅仅是 Data 的长度，也就是说 Length = Data.Length + 4。

### LCP Discussion
LCP 全称 Link Control Protocol，链路控制协议，控制链路吞吐量，链路通断等链路基本信息。
它的 Code 有如下取值

| Code 值 | 含义                  | 描述         |
|--------|-----------------------|--------------|
| 0x01   | Configuration Request | 链路协商     |
| 0x02   | Configuration ACK     | 协商通过     |
| 0x03   | Configuration NAK     | 协商拒绝     |
| 0x04   | Configuration Reject  | 协商回绝     |
| 0x05   | Terminate Request     | 链路终止请求 |
| 0x06   | Terminate ACK         | 终止请求确认 |
| 0x09   | Echo Requeest         | 心跳请求     |
| 0x0A   | Echo Reply            | 心跳确认     |

NAK 和 Reject 的区别是，NAK 表示对方的所列出的链路配置项没有问题，但是值不符合预期，返回 NAK 同时会携带自己期望的配置值，而 Reject 则表示发过来的某项配置服务器不支持，无法接受，返回 Reject 时会列出不接受的配置项。

Data 的组织形式为下面结构的多项叠加

| Type    | Length  | Data    |
|---------|---------|---------|
| 1 Bytes | 1 Bytes | x Bytes |

这里的 Length 表示全长，包括 Type 和 Length 占用的字节，所以 Length = Data.Length + 2。

Type 取值和含义
>0x01..............MRU（Maximum Receive Unit）
>0x02..............Async-Control-Character-Map
>0x03..............Auth Type
>0x05..............Magic Number
>0x06..............Link Quality Compression/Monitoring
>0x07..............Procotol Field Compression
>0x08..............Address and Control Field Compression

当 Type 为 MRU 时，长度为 2 Bytes，表示报文发送方最大可接收的报文大小，一般不超过 1484。
当 Type 为 Auth Type 时，Data 可取值有这些：0xC023 表示采用 PAP 认证、0xC223 + 0x05 表示使用 MD5 挑战方式的 CHAP 认证。当然也存在其他取值，具体可参阅 RFC 文档。
当 Type 为 Magic Number 时，长度为 4 Bytes，值随机，答复的时候要带上用于辨识该报文是否存在问题，使用方法是如果收到的报文跟发出的报文 Magic Number 值相同，则认为报文存在问题。
其他的 Type 含义参阅 RFC 文档，这里所说的是几个比较重要的 Type 值，这几个是实现 PPPoE 服务器中必须处理和了解的几个值。

LCP Discussion 阶段中双方互相发送 Configuration Request，相互确认对方的信息，同意回复 Configuration ACK 并携带对方的 Request 信息。
如果不同意对方的 Request，这时候回复 NAK，并且在 NAK 中携带自己期待的配置信息，收到 NAK 之后如果接受列出的配置，带上 NAK 中的配置重新 Configuration Request。
如果发现对方报文中列出的配置项不支持，这时候需要回复 Reject，并且携带对方报文中不支持的配置项。
>比如 Configuration Request: Auth Type 0x0000，这时候接收方会回复 Configuration Reject: Auth Type 0x0000，相当于告诉客户端我不支持 0x0000 这种认证，你需要更换一个认证方式。

LCP Discussion 过程可能会来回多次，反复确认双方配置（Request--NAK--Request--Reject--Request--...)，如果在多次 Configuration 依旧无法协商完成只能 LCP Terminate Request 结束这个短暂的邂逅了。虽然这种情况一般很少见，但也不排除，相当于两个人谈一场交易，结果怎么也谈不拢，那交易肯定也没法进行，再死皮赖脸的协商怕不是要打一架了。

*其实这个过程看起来就跟两个人在进行一场协商一样的不是吗？*
*用自然语言表述这个过程可以编这么一个对话*
>A：请求：我可接收最大重量为 1484 的包裹，请确认。
>B：OK 我方收到，已知阁下可接收最大重量为 1484 的包裹。
>B：另外我方最大可接受重量为 1400 的包裹，认证方式编码 C023，请确认。
>A：哈？你现在还在用 C023 认证？你不怕泄密吗？回绝（Reject）认证方式编码 C023，你得换一个。
>B：真是事多……重新请求：我方最大可接受重量为 1400 的包裹，认证方式编码 C223+05，请确认。
>A：等等，1400 最大重量太小了吧，我这边接受不了，拒绝（NAK），你得把配置换成这样：最大可接受重量为 1450 的包裹，认证方式编码 C223+05。
>B：(MMP……)行行行，再次请求：我方最大可接受重量为 1450 的包裹，认证方式编码 C223+05，这回可以了吧？
>A：……
>B：……喂？
>A：……啊，不好意思，我这边刚刚有点事没听清，你再说一遍？
>B：你大爷……时间到了，老子不伺候了！再见！
>------用户 B 断开连接------
>A：？？？？？？

### Auth（认证）
LCP 协商之后接下来就是认证了。
由于认证的方式是在 LCP 阶段确认的，所以这里根据不同的认证方式有两种数据组织方式

#### PAP
PAP 阶段 Code 取值有
>0x01..........Auth Request
>0x02..........Auth ACK
>0x03..........Auth NAK

当 Code 为 Auth Request 时 Data 组织形式如下

| Peer-ID-Length | Peer-ID | Password-Length | Password |
|----------------|---------|-----------------|----------|
| 1 Bytes        | x Bytes | 1 Bytes         | x Bytes  |

其中 Peer-ID 就是用户名，Password 为密码。两个 Length 都是指后面数据长，不包括 Length 所占用的字节。
从这里不难看出，PAP 直接携带用户名密码，并且所支持的最大用户名长和密码长都是 255 字节。
使用 PAP 方式，客户端直接发送 PAP-Auth Request，服务器收到报文后解析用户名密码判断该用户是否合法，认证成功返回 Auth ACK，否则返回 Auth NAK。

当 Code 为 Auth Ack 或者 Auth NAK 的时候，后面 Data 先跟一字节长度提示，然后填入长度控制的字符串作为认证提示，认证提示可以随便填写，就是一个字符串。

_ 比如你可以在 Auth ACK 之后跟上信息：Authentication Failed! Rua!，没错就是可以这么傲娇。 _

#### CHAP
CHAP 阶段 Code 取值有
>0x01..........Challenge
>0x02..........Response
>0x03..........Success
>0x04..........Failure

当 Code 为 Challenge 或者 Response 的时候 Data 组织形式如下：

| Value Size | Value   | Name    |
|------------|---------|---------|
| 1 Bytes    | x Bytes | y Bytes |

其中 Value Size 表示 Value 长度，一般是 16。Name 为用户名，以 0x00 结尾。
Value 一般是 16 字节的数据，各个阶段意义略微有区别。
当时用 CHAP 认证的时候，服务器首先发送 Challenge 消息，里面的用户名可以为空。
客户端收到 Challenge 之后，通过之前商讨的挑战方式，重新计算 Value，并携带用户名，发送 Response 给服务器。
服务器通过客户端答复的 Value 判断认证是否成功，成功返回 Success，否则返回 Failure。

同样的当 Code 为 Failure 或者 Success 的时候，后面 Data 是一个返回给客户端的字符串，表示认证信息，以 0x00 结尾。

MD5挑战模式下，Challenge 的期望值为：MD5(id + password + value)。
其中 id 为 PPPoE Session 阶段携带的 Identifier，value 为服务器 Challenge 发过来的值。计算完之后重新放入 Value 返回给服务器。
服务器收到之后也会用同样的方式处理，对比两者结果是否一致判断用户是否合法。

认证失败一般紧接着就会发送 LCP Terminate Request 断开链接，释放当前 Session。

### IPCP Discussion
认证完成之后就是 IP 协商了。
IPCP 全称 IP Control Protocol，IP 控制协议，用于控制接入设备的 IP 配置信息。
IPCP 阶段 Code 可以取值：
>0x01..........Configuration Request
>0x02..........Configuration ACK
>0x03..........Configuration NAK

Data 组织形式为：

| Type    | Length  | IP Address |
|---------|---------|------------|
| 1 Bytes | 1 Bytes | 4 Bytes    |

Type 可取值
>0x03..........IP
>0x81..........DNS1
>0x83..........DNS2

Length 为总长，Length = 6。
从这里不难发现，PPP 接入下没有子网掩码，因为在 PPP 模式下不存在子网概念，所有设备相互独立，所以客户端一般在配置 PPP 网络的时候习惯性将子网掩码设置为 255.255.255.255。

**IPCP 流程**
服务器首先发出一个 IPCP Request 报文，只带 0x03 字段，这个报文告知客户端服务器的 IP 地址信息。
客户端一般在收到服务器发来的 IPCP Request 报文之后才发送 IPCP Request 请求自己的 IP 地址。
一般情况下，客户端发过来的请求中携带的 IP 信息为0.0.0.0，也有可能是上次认证时分配到的 IP。
服务器收到客户端的 IPCP Request 之后判断配置信息是否正确，正确回复 ACK，否则回复 NAK 并附上正确的 IP 配置信息。

这个过程其实跟 LCP Discussion 过程类似，只是起手动作需要服务器发起，客户端需要先确认服务器 IP 才会进行接下来的确认。
后续流程除了没有 Reject 消息，跟 LCP Discussion 流程是一样的，对了回复 ACK，不对回复 NAK 并携带上所需的信息。

IPCP 协商完成之后，PPP 链接建立过程结束。

### PPP 链接建立之后
PPPoE 链路建立之后，上文也提到后续的数据依旧是以 PPPoE Session 方式发送，不过 P2P Protocol 的值变为 0x0021（IPv4）
这时候，一般的网卡是无法解析该消息的，需要把 PPP 消息中 PPPoE 部分剔除封装成普通的 IPv4 报文，再发送给相关设备，这样网卡才能处理该消息。
这个策略可以考虑使用假 MAC 的方式，PPPoE 服务器通过假 MAC 发送消息给相关设备，相关设备的信息发送回来之后 PPPoE 服务器截获该消息（其实 WinPcap 直接就能收到）然后重新用 PPP 封装在发送给客户端。
换句话说，PPPoE 链路建立之后，通讯过程依旧跟普通的 IPv4 通信相差很大。

### PPPoE 报文总览
一张图总结一下 PPPoE 报文
![PPPoE 报文](/img/PPPoEPacket.png)


### 其他一些注意事项和技巧
- WinPcap 监听会监听到 Destination 非自己的报文，包括自己发的，如果不过滤 Source 会造成报文没有被过滤，自己的报文自己答复接着进入死循环浪费资源。这点需要注意
- LCP 控制报文中如果收到 Echo Request 需要及时回复 Echo Reply，如果没有答复 Echo Request，超过一定次数之后会使对方认为链路断开，链接丢失。所写的 PPPoE 也应该具备这个机制，毕竟 Session 资源紧张。
- Session 报文的 Identifier 变化一般有两种思路，当然这个策略也可以自己设计
>1.当没有回复的时候使 ID + 1 重新发送，直到得到答复之后 ID 重置为 0 发送下一个消息。
>2.当 Package 的类型改变的时候重制 ID 为 0，否则 ID 递增。

- LCP 的 Echo Request 心跳报文中，ID 应该只随心跳次数递增，双方各自记录心跳情况。
- LCP/IPCP 都需要双方各一个 Request 和 ACK 之后才可配置好链接相关信息。
- Session 建立过程中，如果在某阶段耗时过长即可认为流程超时，断开并清理当前 Session。
- 为了保证报文质量，PPPoE 建立过程中，所有报文长度最短应为 60 字节，不够应当补齐。

### 写在最后
看完 PPPoE 协议的内容，是不是有种晕头转向的感觉？
确实 DHCP 协议与 PPPoE 协议比起来简直是小巫见大巫，毕竟链路层协议，复杂程度肯定是要上升的。
不过也不需要太敬畏它，PPPoE 依旧只是一个协议，一个准则而已，有死板的地方，也有可以搞事的地方（喂）。
总的来看其实 PPPoE 协议就是一层一层展开的，理解了 PPPoE 协议其实你也就理解了网络分这么多层到底有什么意义了。
网络报文其实就是约定好的一层一层数据组装到一起，按照一定模式一定能够将其解开，这样的设计还方便了各部分只关注自己所需要关注的部分，省去了很多解析上的麻烦，只能感叹当年这些网络工程师的智慧。
这里面的很多策略都是可以自己发挥的地方，可以尝试一下设计一套自己的高效的策略。
设计 PPPoE 服务器最关键的地方就是学会链路层编程，理解网络报文这种层级关系，理解 PPPoE 协议中各个部分所需要遵循的规则（比如 CHAP 认证时的 Challenge 规则）。
由于 PPPoE 服务器在后期通信中依旧担当一个举足轻重的角色，所以 PPPoE 服务器对服务器要求很高，普通电脑性能不好的话是无法承担这个任务的，这就是为什么很少在 PC 平台下有 PPPoE 服务器软件的原因。
目前商用的 PPPoE 服务器都是各厂商自己设计好的一套解决方案，而不简单是一个软件，里面设计好的各个单元不同分工，这样才有能力承接 PPPoE 的所有工作。
所以我们设计 PPPoE 服务器并不是说要在普通 PC 上实现能用的 PPPoE 服务器，而是通过写 PPPoE 服务器的过程，理解链路层编程，理解网络分层结构，这才是最终目的。

可以拓展的地方：不使用 WinPcap，使用更底层的 C 完成 PPPoE 服务器编程，这样效率更高，说不定性能可以达到和一些 PPPoE 服务器持平。

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
