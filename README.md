iperf3:  A TCP, UDP, and SCTP network bandwidth measurement tool
================================================================

Summary
-------
iperf 是一个用于在 IP 网络上进行最大可达带宽主动测量的工具。它支持调整与时间控制、协议、缓冲区相关的各种参数。
每次测试都会报告测得的吞吐量 / 比特率、丢包情况及其他参数。

本版本通常被称为 iperf3，它是对最初由 NLANR/DAST 开发的原始版本的重新设计。
iperf3 是从零开始实现的一个新版本，目标是构建一个更小、更简单的代码库，并开发一个可以在其他程序中使用的功能库版本。
iperf3 还包含一些在其他工具（如 nuttcp 和 netperf）中具备但原始 iperf 中缺失的功能。
例如：零拷贝模式（zero-copy mode） 和 可选的 JSON 输出格式。
请注意，iperf3 与原始版本的 iperf 不兼容。

iperf3 的主要开发平台为：Ubuntu Linux、FreeBSD 和 macOS。目前，这些是官方支持的唯一平台，
但也有用户在 OpenBSD、NetBSD、Android、Solaris 以及其他 Linux 发行版 上成功运行的报告。

iperf3 主要由 ESnet / 劳伦斯伯克利国家实验室（Lawrence Berkeley National Laboratory） 开发，并以 三条款 BSD 许可证 进行发布。

### Building 
    ./configure; make; make install

(Note: If configure fails, try running `./bootstrap.sh` first)

Invoking iperf3
---------------

在使用默认选项的情况下，iperf 的目标是展示典型的、设计良好的应用程序的性能。所谓“典型的设计良好的应用程序”，是指避免使用那些仅在测试中有效的人工优化手段（例如将数据通过 splice() 直接传输到 /dev/null）
iperf 也提供了一些用于**“极限最佳情况”优化的参数，但这些优化必须由用户显式启用

These flags include:

    -Z, --zerocopy            use a 'zero copy' sendfile() method of sending data
    -A, --affinity n/n,m      set CPU affinity


uso
---------
cd /src

./src/iperf3 -s -p 9999 -U 1
./src/iperf3 -c 10.10.254.10 -i 1 -t 0 -p 9999 -u -U 1000 -l 4000
查看日志：
./src/iperf3 -d -V

./src/iperf3 -s -p 1234 -d -V
./src/iperf3 -c 1.1.1.1 -p 1234 -V -d


--port 指定客户端数据连接的客户端端口
-l     每次发送数据的大小，会设置test->settings->blksize，tcp默认131072, udp默认1440
-b     带宽，tcp中设置Target Bitrate，1M是1000000，udp中设置bandwidth，1G是10000000000



连接
---------
```shell
客户端控制socket进行连接iperf_connect
客户端数据socket进行连接iperf_tcp_connect
```


宏定义
---------
```shell
HAVE_TCP_CONGESTION 服务端1
    当前系统的TCP协议栈是否支持设置或获取TCP 拥塞控制算法
    TCP 拥塞控制算法是 TCP 协议中用于控制网络拥塞的核心机制，常见的算法包括：
	•cubic（默认，Linux 上常用）
	•reno
	•bbr（Google 开发的高性能算法）
	•westwood 等
    查看当前使用的算法：
    cat /proc/sys/net/ipv4/tcp_congestion_control

HAVE_TCP_CONGESTION 客户端1

HAVE_TCP_USER_TIMEOUT 默认1
    设置超时时间

HAVE_DONT_FRAGMENT 1

HAVE_SSL           1

HAVE_FLOWLABEL     1
HAVE_SO_MAX_PACING_RATE  1
```



客户端创建2个socket
----------
```shell
一个数据连接 + 一个控制连接
服务端都是同一个端口，客户端两个连接有两个端口

1. 第一个 TCP 连接：控制连接
- 用于客户端与服务器之间的 控制消息传递
- 包括启动、停止、配置、统计信息等
- 这个连接在整个测试过程中都保持打开
- 走的是服务器监听的默认端口（5201）

2. 第二个 TCP 连接：数据连接
- 用于真正的 测试数据流量
- 客户端会根据测试类型（TCP流、UDP流、多线程等）发起额外的连接来传输数据
- 数据连接通常也是TCP，但可以配置成UDP

工作原理
	1.	客户端连接服务器的 5201端口（默认），建立 控制连接。
	2.	控制连接建立后，客户端和服务器协商测试参数（流量类型、时长、并发流数等）。
	3.	协商后，客户端再发起 数据连接（TCP流时是新的TCP连接）。
	4.	数据连接传输测试流量。
	5.	测试结束后，通过 控制连接 汇报测试结果，关闭数据连接。
	6.	最后关闭控制连接。

为什么要分成两个连接？
原因                   解释
控制与数据分离           保证测试数据流量不影响控制命令的可靠传输。
更灵活                 可以支持更复杂的测试场景（如并发流、多路复用等）。
结果收集方便            结束后通过控制连接可靠收集统计数据。


对比：iperf2 只用一个连接
	•	iperf2 时代，控制信息和数据是复用在一个连接上的。
	•	iperf3 设计上更接近 HTTP2/QUIC 等现代协议的理念，采用控制/数据分离。
```

```shell
客户端发送                                                 服务端
1. test->cookie                 37B         收到cookie，发送09状态   1B    PARAM_EXCHANGE
2. 参数大小                       4B         获取参数大小
3. 参数send_parameters          144B         获取参数，发送状态0a      1B    CREATE_STREAMS
4. 创建数据socket
5. 数据socket发送test->cookie    37B         获取cookie，发送状态01    1B    TEST_START
6. Starting Test                            发送状态02              1B     TEST_RUNNING
7. pthread_create发送数据       131072B      读数据

8. 结束后                                    发送交换结果13           1B     EXCHANGE_RESULTS
9. 发送结果  4B+数据      
10. 获取服务端结果 
11.                                         发送显示结果14           1B     DISPLAY_RESULTS
12. 发送IPERF_DONE              1B                                         IPERF_DONE
4次挥手
tcp_send_fin
``` 

```shell
 * 序列号: 发送数据的位置，告诉接收方这个数据从哪里开始
 * 确认号: 告诉对方“我已经成功接收到哪个序列号之前的数据，请从这个位置继续发送”

客户端 seq ack                服务端 seq ack
81a15e61  0        SYN       5b499d3d  81a15e62 SYNACK  发送给对方
81a15e62  5b499d3e ack                                 3次握手

81a15e62  5b499d3e snd 25B   5b499d3e 81a15e87 ack      客户端开始发送数据
                             5b499d3e 81a15e87 snd  1B
81a15e87  5b499d3f ack

81a15e87  5b499d3f snd 4B    
81a15e8b  5b499d3f snd 89B   5b499d3f 81a15f14 ack

seq是上一次的seq+数据长度，同时也是对方的ack数据
```