# 术语

RTO 是指超时重传时间（Retransmission TimeOut）
RTT（Round Trip Time，往返时间）

# HTTP3

QUIC（Quick UDP Internet Connections，快速 UDP 网络连接）

# 零 RTT 建立连接

HTTP/2 的连接需要 3 RTT，如果考虑会话复用，即把第一次握手算出来的对称密钥缓存起来，那么也需要 2 RTT，更进一步的，如果 TLS 升级到 1.3，那么 HTTP/2 连接需要 2 RTT，考虑会话复用则需要 1 RTT。
HTTP/3 首次连接只需要 1 RTT，后面的连接更是只需 0 RTT，意味着客户端发给服务端的第一个包就带有请求数据，这一点 HTTP/2 难以望其项背。

# 连接迁移

TCP 连接基于四元组（源 IP、源端口、目的 IP、目的端口），切换网络时至少会有一个因素发生变化，导致连接发生变化。
QUIC 连接不以四元组作为标识，而是使用一个 64 位的随机数，这个随机数被称为 Connection ID，即使 IP 或者端口发生变化，只要 Connection ID 没有变化，那么连接依然可以维持。

# 队头阻塞/多路复用

HTTP/1.1 和 HTTP/2 都存在队头阻塞问题（Head of line blocking）

## HTTP/1.1 的队头阻塞

一个 TCP 连接同时传输 10 个请求，其中第 1、2、3 个请求已被客户端接收，但第 4 个请求丢失，那么后面第 5 - 10 个请求都被阻塞，需要等第 4 个请求处理完毕才能被处理，这样就浪费了带宽资源。

## HTTP/2 的队头阻塞

HTTP/2 的多路复用解决了 HTTP/1.1 的队头阻塞问题。不像 HTTP/1.1 中只有上一个请求的所有数据包被传输完毕下一个请求的数据包才可以被传输，HTTP/2 中每个请求都被拆分成多个 Frame 通过一条 TCP 连接同时被传输，这样即使一个请求被阻塞，也不会影响其他的请求。
HTTP/2 虽然可以解决“请求”这个粒度的阻塞，但 HTTP/2 的基础 TCP 协议本身却也存在着队头阻塞的问题。
在一条 TCP 连接上同时发送 4 个 Stream，其中 Stream1 已正确送达，Stream2 中的第 3 个 Frame 丢失，TCP 处理数据时有严格的前后顺序，先发送的 Frame 要先被处理，这样就会要求发送方重新发送第 3 个 Frame，Stream3 和 Stream4 虽然已到达但却不能被处理，那么这时整条连接都被阻塞。
不仅如此，由于 HTTP/2 必须使用 HTTPS，而 HTTPS 使用的 TLS 协议也存在队头阻塞问题。TLS 基于 Record 组织数据，将一堆数据放在一起（即一个 Record）加密，加密完后又拆分成多个 TCP 包传输。一般每个 Record 16K，包含 12 个 TCP 包，这样如果 12 个 TCP 包中有任何一个包丢失，那么整个 Record 都无法解密。

## QUIC 是如何解决队头阻塞问题的呢？

QUIC 的传输单元是 Packet，加密单元也是 Packet，整个加密、传输、解密都基于 Packet，这样就能避免 TLS 的队头阻塞问题。

QUIC 基于 UDP，UDP 的数据包在接收端没有处理顺序，即使中间丢失一个包，也不会阻塞整条连接，其他的资源会被正常处理。

## 多路复用的原理

HTTP/2 的每个请求都会被拆分成多个 Frame，不同请求的 Frame 组合成 Stream，Stream 是 TCP 上的逻辑传输单元，这样 HTTP/2 就达到了一条连接同时发送多条请求的目标，这就是多路复用的原理。

## 队头阻塞会导致 HTTP/2 在更容易丢包的弱网络环境下比 HTTP/1.1 更慢！

原因见 HTTP/2 对头阻塞，TCP 协议本身却也存在着队头阻塞的问题和 TLS 协议也存在队头阻塞问题。

# 拥塞控制

拥塞控制的目的是避免过多的数据一下子涌入网络，导致网络超出最大负荷。
TCP 拥塞控制由 4 个核心算法组成：慢启动、拥塞避免、快速重传和快速恢复。
慢启动：发送方向接收方发送 1 个单位的数据，收到对方确认后会发送 2 个单位的数据，然后依次是 4 个、8 个……呈指数级增长，这个过程就是在不断试探网络的拥塞程度，超出阈值则会导致网络拥塞；

拥塞避免：指数增长不可能是无限的，到达某个限制（慢启动阈值）之后，指数增长变为线性增长；

快速重传：发送方每一次发送时都会设置一个超时计时器，超时后即认为丢失，需要重发；

快速恢复：在上面快速重传的基础上，发送方重新发送数据时，也会启动一个超时定时器，如果收到确认消息则进入拥塞避免阶段，如果仍然超时，则回到慢启动阶段。

## QUIC 对 TCP 拥塞控制的改进特性

### 热插拔

TCP 中如果要修改拥塞控制策略，需要在系统层面进行操作。QUIC 修改拥塞控制策略只需要在应用层操作，并且 QUIC 会根据不同的网络环境、用户来动态选择拥塞控制算法。

### 前向纠错 FEC

QUIC 使用前向纠错(FEC，Forward Error Correction)技术增加协议的容错性。一段数据被切分为 10 个包后，依次对每个包进行异或运算，运算结果会作为 FEC 包与数据包一起被传输，如果不幸在传输过程中有一个数据包丢失，那么就可以根据剩余 9 个包以及 FEC 包推算出丢失的那个包的数据，这样就大大增加了协议的容错性。
这是符合现阶段网络技术的一种方案，现阶段带宽已经不是网络传输的瓶颈，往返时间才是，所以新的网络传输协议可以适当增加数据冗余，减少重传操作。

### 单调递增的 Packet Number

#### TCP 为了保证可靠性，使用 Sequence Number

由于 TCP 原始请求和重传请求都使用了相同的 Sequence Number，超时发生后客户端发起重传，后来接收到了 ACK 确认消息，但因为原始请求和重传请求接收到的 ACK 消息一样，所以客户端就郁闷了，不知道这个 ACK 对应的是原始请求还是重传请求。如果客户端认为是原始请求的 ACK，但实际上是左图的情形，则计算的采样 RTT 偏大；如果客户端认为是重传请求的 ACK，但实际上是右图的情形，又会导致采样 RTT 偏小。

#### QUIC 采用原始请求和重传请求不同的 Packet Number 避免改问题

QUIC 解决了上面的歧义问题。与 Sequence Number 不同的是，Packet Number 严格单调递增，如果 Packet N 丢失了，那么重传时 Packet 的标识不会是 N，而是比 N 大的数字，比如 N + M，这样发送方接收到确认消息时就能方便地知道 ACK 对应的是原始请求还是重传请求。

### ACK Delay

TCP 计算 RTT 时没有考虑接收方接收到数据到发送确认消息之间的延迟，这段延迟即 ACK Delay。QUIC 考虑了这段延迟，使得 RTT 的计算更加准确。

### 更多的 ACK 块

接收方收到发送方的消息后都应该发送一个 ACK 回复，表示收到了数据。但每收到一个数据就返回一个 ACK 回复太麻烦，所以一般不会立即回复，而是接收到多个数据后再回复。
TCP SACK 最多提供 3 个 ACK block，而 QUIC 最多可以捎带 256 个 ACK block。

# 流量控制

拥塞控制主要是控制发送方的发送策略，流量控制反映接收方的接收能力。
TCP 会对每个 TCP 连接进行流量控制，流量控制的意思是让发送方不要发送太快，要让接收方来得及接收，不然会导致数据溢出而丢失，TCP 的流量控制主要通过滑动窗口来实现的。

## 单条 Stream 的流量控制

Stream 还没传输数据时，接收窗口（flow control receive window）就是最大接收窗口（flow control receive window），随着接收方接收到数据后，接收窗口不断缩小。在接收到的数据中，有的数据已被处理，而有的数据还没来得及被处理。
随着数据不断被处理，接收方就有能力处理更多数据。当满足 (flow control receive offset - consumed bytes) < (max receive window / 2) 时，接收方会发送 WINDOW_UPDATE frame 告诉发送方你可以再多发送些数据过来。这时 flow control receive offset 就会偏移，接收窗口增大，发送方可以发送更多数据到接收方。

## Connection 级别的流量控制

理解了 Stream 流量那么也很好理解 Connection 流控。Stream 中，接收窗口(flow control receive window) = 最大接收窗口(max receive window) - 已接收数据(highest received byte offset) ，而对 Connection 来说：接收窗口 = Stream1 接收窗口 + Stream2 接收窗口 + ... + StreamN 接收窗口 。

> 参考文档：
> [HTTP/3 原理与实践](https://mp.weixin.qq.com/s/wrOXO5MH4wtbuvrCPCQNQA)
