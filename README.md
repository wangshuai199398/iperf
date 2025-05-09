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

./iperf3 -s -p 9999 -U 1
./iperf3 -c 10.10.254.10 -i 1 -t 0 -p 9999 -u -U 1000 -l 4000
查看日志：
./iperf3 -d -V

宏定义
---------
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

