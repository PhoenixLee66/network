#### 1. 概述

    服务端收到客户端发起的 SYN 请求后，**内核会把该连接存储到半连接队列**，并向客户端响应 SYN+ACK，接着客户端会返回 ACK，服务端收到第三次握手的 ACK 后，**内核会把连接从半连接队列移除，然后创建新的完全的连接，并将其添加到 accept 队列，等待进程调用 accept 函数时把连接取出来**

![](http://pic3.zhimg.com/80/v2-351442fbc23ab0af4980141cf140001a_720w.jpg)

#### 2. 半连接队列

> 也称 SYN 队列

##### 2.1 TCP 半连接队列溢出的处理逻辑

> 针对 Linux 2.6.32 版本分析的「理论」半连接最大值的算法

![CP 半连接队列溢出的处理逻辑](http://pic1.zhimg.com/80/v2-2326c6df59e30888e668cd3727b6ec60_720w.jpg)

1. 如果半连接队列满了，并且没有开启 tcp_syncookies，则会丢弃
2. 若全连接队列满了，且没有重传 SYN+ACK 包的连接请求多于 1 个，则会丢弃
3. 如果没有开启 tcp_syncookies，并且 max_syn_backlog 减去 当前半连接队列长度小于 (max_syn_backlog >> 2)，则会丢弃

##### 2.2 半连接队列的最大值 max_qlen_log

> Linux 2.6.32 内核版本

###### 1. 理论值

![理论值](http://pic1.zhimg.com/80/v2-00246c3957e12fd149b82d40745bcb04_720w.jpg)

- 当 max_syn_backlog > min(somaxconn, backlog) 时， 半连接队列最大值 max_qlen_log = min(somaxconn, backlog) * 2;
- 当 max_syn_backlog < min(somaxconn, backlog) 时， 半连接队列最大值 max_qlen_log = max_syn_backlog * 2;

###### 2. 实际可能值

> 如果没有开启 tcp_syncookies，并且 max_syn_backlog 减去 当前半连接队列长度小于 (max_syn_backlog >> 2)，则会丢弃

![](https://pic1.zhimg.com/80/v2-dbfbf46876b8421912e2a1decc433910_720w.jpg)

    其中  net.ipv4.tcp_max_syn_backlog 值可通过 /etc/sysct.conf 修改

##### 2.3 syncookies

###### 1. 流程图

> /ect/sysctl.conf 配置开关 net.ipv4.tcp_syncookies

    服务器根据当前状态计算出一个值，随着SYNC+ACK报文一起返回给客户端。当客户端返回ACK报文时，验证该值。如果合法，则连接建立成功 。

![syncookies 流程](http://img2020.cnblogs.com/blog/1179840/202006/1179840-20200624133338181-1354086500.png)

###### 2. SYN 攻击防御

- 增大半连接队列：增大 tcp_max_syn_backlog 的值，还需一同增大 somaxconn 和 backlog

- 开启 tcp_syncookies 功能

- 减少 SYN+ACK 重传次数：/etc/sysctl.conf 配置 net.ipv4.tcp_synack_retries

#### 3. 全连接队列

> 也称 accepet 队列

##### 3.1 队列大小配置

> TCP 全连接队列大小取决于 min(somaxconn, backlog)

- somaxconn：Linux 内核的参数，默认值是 128，可通过如下方式修改

```bash
# vim /etc/sysctl.conf
net.core.somaxconn = 2000

# 生效
sysctl -p
```

- backlog：nginx配置中Listen配置项，默认值为 511

```nginx
server {
    listen 808 default backlog=6000;
    server_name localhost slbcheck.test.actumtech.com;
    access_log off;
    location / {
        root /etc/nginx/conf.d;
        index index.html index.htm;
    }
}
```

##### 3.2 查看队列大小

> 使用 `ss` 命令查看 

###### 1. Listen Status

```bash
ss -lnt
State      Recv-Q Send-Q   Local Address:Port  Peer Address:Port
LISTEN     0      100       *:38967             *:*
```

- Recv-Q：当前全连接队列的大小，也就是当前已完成三次握手并等待服务端 `accept()` 的 TCP 连接

- Send-Q：当前全连接最大队列长度

###### 2. Not Listen Status

```bash
ss -lnt
State      Recv-Q Send-Q   Local Address:Port  Peer Address:Port
ESTAB      39     0         127.0.0.1:808       127.0.0.1:35016
```

- Recv-Q：已收到但未被应用进程读取的字节数

- Send-Q：已发送但未收到确认的字节数

##### 3.3 压测

> wrk 工具：[GitHub - wg/wrk: Modern HTTP benchmarking tool](https://github.com/wg/wrk.git)



- session 1

```bash
./wrk -t 2 -c 3000 -d 60s http://172.22.1.106:808
```

- session 2

```bash
[root@ttt ~]# ss -lnt
State      Recv-Q Send-Q   Local Address:Port  Peer Address:Port
LISTEN     0      511        *:808                 *:*
[root@ttt ~]# ss -lnt
State      Recv-Q Send-Q   Local Address:Port  Peer Address:Port
LISTEN     512    511        *:808                 *:*
[root@ttt ~]# ss -lnt
State      Recv-Q Send-Q   Local Address:Port  Peer Address:Port
LISTEN     512    511        *:808                 *:*
[root@ttt ~]# ss -lnt
State      Recv-Q Send-Q   Local Address:Port  Peer Address:Port
LISTEN     510    511        *:808                 *:*
[root@ttt ~]# netstat -s |grep overflow
    104862 times the listen queue of a socket overflowed
[root@ttt ~]# netstat -s |grep overflow
    105283 times the listen queue of a socket overflowed
[root@ttt ~]# netstat -s |grep overflow
    105930 times the listen queue of a socket overflowed


```

    当服务端并发处理大量请求时，如果 TCP 全连接队列过小，就容易溢出。发生 TCP 全连接队溢出的时候，后续的请求就会被丢弃，这样就会出现服务端请求数量上不去的现象。

![全连接队列溢出](http://pic2.zhimg.com/80/v2-59396b0f9eb18eca18fff60398558dc1_720w.jpg)



    tcp_abort_on_overflow 两个值表示含义：

- 0：服务端丢弃客户端 ACK 包

- 1：服务端发送 RST 包，在客户端异常中可以看到很多 `connection reset by peer` 的错误



<h6>参考资料：</h6>

1. [TCP 半连接队列和全连接队列满了会发生什么？又该如何应对](https://zhuanlan.zhihu.com/p/144785626)


