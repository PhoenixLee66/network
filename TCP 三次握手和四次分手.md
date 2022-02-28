#### 1. 网络基础知识

##### 1.1 TCP 头部

![](http://tva1.sinaimg.cn/large/008i3skNgy1gvbm49hyr9j60u00vvn2o02.jpg)

###### 1. 序列号

> 解决网络包乱序的问题

    在初次建立连接的时候，客户端和服务端都会为「本次的连接」随机初始化一个序列号。

###### 2. 确认号

> 解决网络包丢失的问题

    该字段表示「接收端」告诉「发送端」对上一个数据包已经成功接收

###### 3. 标记位

- SYN为1时，表示希望创建连接

- ACK为1时，确认号字段有效

- FIN为1时，表示希望断开连接

- RST为1时，表示TCP连接出现异常，需要断开

##### 1.2 socket

> 套接字（socket）是通信的基石，是支持TCP/IP协议的网络通信的基本操作单元

    它是网络通信过程中端点的抽象表示，包含进行网络通信必须的五种信息：连接使用的协议，本地主机的

#### 2. 三次握手

##### 2.1 本质

> 本质是信道不可靠, 但是通信双发需要就某个问题（双方都有收发的能力）达成一致

    要解决这个问题, 无论你在消息中包含什么信息, 三次通信是理论上的最小值。 如果只有两次握手，只能证明客户端序列号被服务端接收成功，但是服务端无法确认自己的序列号是否被客户端接收成功。所以三次握手不是TCP本身的要求，而是为了满足"在不可靠信道上可靠地传输信息"这一需求所导致的

##### 2.2 流程图

![三次握手流程图](http://tva1.sinaimg.cn/large/008i3skNgy1gvcuaebe9oj619g0u0gox02.jpg)

![三次握手](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/13/1717297c73467e00~tplv-t2oaga2asx-watermark.awebp)

- 第一次握手

    客户端随机生成出序列号（client_isn），并且把标志位设置为SYN（意味着要连接），然后把该报文发送给服务端，同时客户端进入SYNC_SEND 状态，等待服务器的确认

- 第二次握手

    服务端接收到了客户端的请求之后，自己也初始化对应的序列号（ server_isn），然后在「确认号」字段里填上client_isn + 1（相当于告诉客户端，已经收到了发送过来的序列号了） ，并且把 SYN 和 ACK 标记位置为1，服务端进入 SYN-REVD 状态

- 第三次握手

    客户端收到服务端发送的报文后，确认服务端已经接收到了自己的序列号（通过确认号），并且接收到了服务端的序列号(server_isn)。

    此时，客户端也需要告诉服务端自己已经接收到了他发送过来的序列号，所以在「确认号」字段上填上server_isn+1，并且标记位 ACK标记位置 为1。

    客户端在发送报文之后，进入 ESTABLISHED 状态，而服务端接收到客户端的报文之后，也进入 ESTABLISHED 状态。

##### 2.3 其他

###### 1. 序列号初始化随机算法

> 一方面为了安全性（随机ISN能避免非同一网络的攻击），另一方面可以让通信双方能够根据序号将「不属于」本连接的报文段丢弃

isn = M + F(localhost, localport, remotehost, remoteport)

其中M是一个计时器，每4毫秒加1。F是一个Hash算法，比如MD5或者SHA256

#### 3. 四次分手

##### 3.1 流程图

![四次分手](http://tva1.sinaimg.cn/large/008i3skNgy1gvdzz74bokj614q0u0gp002.jpg)

![四次挥手](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/13/1717297c746f6ee2~tplv-t2oaga2asx-watermark.awebp)

> 连接双方都有权利发起，这里以客户端为例

- 第一次分手

    客户端打算关闭连接，设置Sequence Number和Acknowledgment Number后，将 FIN状态位置为1，然后将报文发给服务端，同时客户端进入FIN_WAIT_1状态，即客户端无数据发送给服务端

- 第二次分手

    服务端收到FIN报文后，向客户端发送ACK报文，然后在「确认号」字段里填上client_isn + 1，服务端进入CLOSE_WAIT状态，因为服务端可能数据需要处理，同时客户端进入FIN_WAIT_2状态

- 第三次分手

    服务端数据处理完后，将FIN状态位置为1，然后将报文返回给客户端，服务端进入LAST_ACK状态

- 第四次分手

    客户端收到服务端发送的FIN报文段，向服务端发送ACK报文段，然后客户端进入TIME_WAIT状态；服务端收到客户端的ACK报文段后关闭连接；此时，客户端等待2MSL后依然没有收到回复（网络存在不可靠的情况），则证明Server端已正常关闭，那么客户端也关闭连接

#### 3. 状态分析

##### 3.1 三次握手

###### 1. 客户端单方面 ESTABLISHED 状态

- 客户端与服务端均未发送数据，服务端超时重传自己的SYN+ACK包

- 客户端已经发送数据，服务端接收ACK + Data数据包后，直接进入ESTABLISHED状态

- 服务端发送不了数据，会周期性超时重传SYN + ACK，直到接收到客户端ACK包 

###### 2. SYN_RCVD

- 服务端未收到客户端ACK包，依次等待3秒、6秒、12秒后重新发送SYN+ACK包，若仍然未收到，则主动关闭连接

- 重发次数与tcp_synack_retries参数有关

- 连接将放入“半连接队列”

###### 3. SYN_SEND

- 根据《TCP/IP详解卷Ⅰ：协议》中的描述，此时会尝试三次，间隔时间分别是 5.8s、24s、48s，但总时长是75s

##### 3.2 四次分手

###### 1. TIME_WAIT

1. 目的
   
   - 确保最后的ACK报文一定会被收到，即被动关闭一方可以正确的关闭，如果收不到，2MSL内会重发FIN报文
   
   - 确保在创建新的连接时，原先网络中残余的数据都已无效，Linux 认为数据报文经过MSL（默认为64）个路由器的时间不会超过TTL（默认为30）秒，如果超过，就认为报文已经消失在网络中
   
   - 止历史连接中的数据，被后面相同四元组（local ip, local port, remote ip, remote port）的连接错误的接收

2. 一个TCP连接需要：1. socket 文件描述符；2. IP地址和端口；3. 内存（有数据情况下，读写缓冲区，再加上一个TCP控制块）；4. 线程资源（ **C10K** 问题）

#### 4. Linux命令

##### 4.1 netstat --timers

> 计时器功能，形如：

```bash
Proto Local Address     Foreign Address  State       Timer
tcp   172.21.32.4:57604 172.21.32.6:8848 TIME_WAIT   timewait (39.86/0/0)
```

    1. 第一列值含义

- keepalive：keepalive的时间计时

- on：重发（retransmission）的时间计时

- off：没有时间计时

- timewait：等待（timewait）时间计时

    2. 第二列值含义

- 39.86：计时时间值

- 0：只有第一列值为on，才有效，表示已经产生的重发（retransmission）次数

- 0：keepalive已经发送的探测（probe）包的次数（内核参数跟tcp_keepalive_intvl和tcp_keepalive_probes）

    等待（timewait）时间计时，以Linux为例，通常是半分钟，两倍的MSL就是一分钟，也就是60秒，并且这个数值是硬编码在内核中的，也就是说除非你重新编译内核，否则没法修改它

##### 4.2 killcx

###### 1. 关闭一个 TCP 连接条件

    要伪造一个能关闭 TCP 连接的 RST 报文，必须同时满足「四元组相同」和「序列号正好落在对方的滑动窗口内」这两个条件

###### 2. 流程

    处于 establish 状态的服务端，收到四元组相同的 SYN 报文后，会回复一个 Challenge ACK，这个 ACK 报文里的「确认号」，正好是服务端下一次想要接收的序列号，然后用这个确认号作为 RST 报文的序列号，发送给服务端，此时服务端会认为这个 RST 报文里的序列号是合法的，于是就会释放连接！

#### 5. 优化方案

##### 5.1 TIME_WAIT

###### 1. 调整系统内核参数

> 不建议，存在风险

```properties
# 表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭
net.ipv4.tcp_tw_reuse = 1

# 允许处于 TIME_WAIT 状态的连接被快速回收，该参数在 NAT 的网络下是不安全的
net.ipv4.tcp_tw_recycle = 1

# 必须开启，60s内同一源ip主机的socket connect请求中的timestamp必须是递增的
net.ipv4.tcp_timestamps = 1
```

    net.ipv4.tcp_tw_recycle 存在风险：

- **Linux 会加快客户端和服务端 TIME_WAIT 状态的时间**，也就是它会使得 TIME_WAIT 状态会小于 60 秒，很容易导致数据错乱；

- **Linux 会丢弃所有来自远端时间戳小于上次记录的时间戳（由同一个远端发送的）的任何数据包**。就是说要使用该选项，则必须保证数据包的时间戳是单调递增的。那么，问题在于，此处的时间戳并不是我们通常意义上面的绝对时间，而是一个相对时间。很多情况下，我们是没法保证时间戳单调递增的，比如使用了 NAT，LVS 等情况

- **per-host 是对「对端 IP 做 PAWS 检查」**，而非对「IP + 端口」四元组做 PAWS 检查，所以在NAT或者LVS环境下，SYN包会被丢弃

###### 2. 调整短链接为长链接

> 长连接指建立SOCKET连接后不管是否使用都保持连接，但安全性较差

- nginx 下 keepalive 相关配置
- 使用线程池

###### 3. 在程序中设置 socket 选项

##### 5.2 三次握手优化

![](http://static001.geekbang.org/infoq/56/5690442671b4734fcb75acc80d88cd55.png)

##### 5.3 四次分手优化

![四次分手优化](https://static001.geekbang.org/infoq/47/47f73498851728de18704603f2d82f7f.png)

<h6>参考资料：</h6>

1. [面试官问我TCP三次握手和四次挥手，我真的是 - Java3y - 博客园](https://www.cnblogs.com/Java3y/p/15725895.html)
2. [如何避免TCP的TIME_WAIT状态（高并发） - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/article/1095131)
3. [被微信面麻了，问的太细节了 - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/article/1925533?from=article.detail.1095131)
4. [麻了，被字节问懵逼了！ - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/article/1925531)
5. [提高 TCP 性能的方法，你知多少？_TCP_小林coding_InfoQ写作平台](https://xie.infoq.cn/article/681d093ffc06d594de54992b9)
