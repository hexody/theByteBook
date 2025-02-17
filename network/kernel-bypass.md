# 3.5 内核旁路技术

相信读者已经认识到“**高并发环境下，网络协议栈的复杂处理，以及内核态与用户态的频繁切换，往往会成为主要的性能瓶颈**”，这一点在网络密集型系统中尤为明显，内核处理能力的局限性将直接制约整个系统的性能表现。

在人们想办法提升内核性能的同时，另外一批人抱着它不行就绕开它的思路，提出了一种“内核旁路（Kernel bypass）”思想的技术方案。其中，DPDK 与 XDP 是主机内“内核旁路”思想的实现代表，RDMA 是主机之间“内核旁路”思想的实现代表。