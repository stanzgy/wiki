## BGP路由在VPN网络上失效问题探究

### 背景

由于科学上网的需求，我在多个云服务提供商处购买了数台云主机，并使用VPN服务使这些云主机运行在
加密的虚拟子网下，网络拓扑如下图示意：

    +------------+     +------------+        +------------+
    |   My_PC    |     |   Cloud_A  |        |   Cloud_B  |
    |            |     |            |        |            |     +----------------+
    |    +-------+     |    +-------+        |    +-------+     |                |
    |    | VPN_A |----------+ VPN_A +---+---------+ VPN_A +--+--+ Target Network |
    |    +-------+     |    +-------+   |    |    +-------+  |  | e.g.: 8.8.8.8  |
    |            |     |            |   |    |            |  |  |                |
    +----------- +     +----------- +   |    +------------+  |  +----------------+
                                        |                    |
                                        |    +------------+  |
                                        |    |   Cloud_C  |  |
                                        |    |            |  |
                                        |    |    +-------+  |
                                        +---------+ VPN_A +--+
                                             |    +-------+
                                             |            |
                                             +------------+


图中有3台分别不同云服务提供商的云主机 `Cloud_A`/`Cloud_B`/`Cloud_C`，通过一个VPN网络 `VPN_A`
连接在一起。我的个人电脑 `My_PC` 也通过接入 `VPN_A` 实现了和这些云主机的连接。
这些云主机中，`Cloud_A` 扮演中间节点的角色，`Cloud_B` 和 `Cloud_C` 扮演网络出口的角色。
当我的个人电脑 `My_PC` 需要通过 `VPN_A` 访问IP地址 8.8.8.8 时，可以通过
`My_PC` --> `Cloud_A` --> `Cloud_B` --> 8.8.8.8 路径到达，也可通过
`My_PC` --> `Cloud_A` --> `Cloud_C` --> 8.8.8.8 路径到达。

为了方便控制VPN转发路径的选择，我通过在 `My_PC` 和 `Cloud_B` 、 `Cloud_C` 之间跑BGP协议，
由 `Cloud_B` `Cloud_C` 广播相同的目标网段，并尝试通过设置不同的 BGP Med 权重属性控制不同线路的
优先级。

### 问题描述

上面的网络设计在传统的物理网络上运行应是没有问题的，但是在我的VPN网络运行则遇到奇怪的路由失效
问题。下面是实际的问题描述。

首先看我个人电脑上的BGP状态：

    # show ip bgp 8.8.8.8
    BGP routing table entry for 8.0.0.0/7
    Paths: (6 available, best #5, table default)
      Advertised to non peer-group peers:
      10.2.10.61 10.2.10.71 10.2.20.61 10.2.20.71 10.2.30.61 10.2.30.71
      ...
      64900
        10.2.30.71 from 10.2.30.71 (10.2.1.71)
          Origin incomplete, metric 510, valid, external, best (MED)
          Last update: Mon Mar  9 03:43:17 2020
      64900
        10.2.30.61 from 10.2.30.61 (10.2.1.61)
          Origin incomplete, metric 520, valid, external
          Last update: Mon Mar  9 02:15:34 2020

根据收到的BGP路由，我的个人电脑访问 8.8.8.8 应通过 10.2.30.71 这个Med最低的网关IP中转。
操作系统内核的路由表也和BGP路由一致：

    ❯ ip r get 8.8.8.8
    8.8.8.8 via 10.2.30.71 dev tun24 src 10.2.30.13 uid 1000
        cache

但是当我 mtr 8.8.8.8 时，奇怪的问题发生了，实际的网络路径并没有通过 10.2.30.71 中转，
而是走了BGP MED优先级低的 10.2.30.61 ！

    ❯ mtr -rn 8.8.8.8
    HOST: xxx                         Loss%   Snt   Last   Avg  Best  Wrst StDev
      1.|-- 10.2.30.91                 0.0%    10    0.8   0.9   0.7   1.1   0.1
      2.|-- 10.2.30.61                30.0%    10   59.6  62.7  37.5  76.6  14.4
     ...
     17.|-- 108.170.236.206            0.0%    10   88.1 103.8  87.0 120.9  14.0
     18.|-- 108.170.238.103            0.0%    10  108.1 102.1  79.4 128.8  14.3
     19.|-- 8.8.8.8                    0.0%    10   93.1 101.7  84.7 120.0  12.6

这个奇怪的现象一下让我楞住了，在各个软件配置、路由表信息都正确的情况下，实际的网络并没有按
路由表生效的规则走。

### 问题分析

对于这个问题，冷静下来分析一下，网关IP到底是如何工作的？
当我们的电脑需要给不在本地网段的IP发送网络报文时，需要将这个报文发送给网关，然后由网关进行转发。
而发送给网关的方式，则是将网络报文的二层目的MAC地址填为网关的MAC地址。

对于我们的问题，抓包查一下到底是什么情况：

    ❯ sudo tcpdump -XX -eni tun24 icmp
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on tun24, link-type RAW (Raw IP), capture size 262144 bytes
    11:20:46.175642 IP 10.2.30.13 > 8.8.8.8: ICMP echo request, id 4, seq 1, length 64
        0x0000:  4500 0054 f327 4000 4001 0f63 0a02 1e0d  E..T.'@.@..c....
        0x0010:  0808 0808 0800 e58f 0004 0001 de8b 655e  ..............e^
        0x0020:  0000 0000 0dae 0200 0000 0000 1011 1213  ................
        0x0030:  1415 1617 1819 1a1b 1c1d 1e1f 2021 2223  .............!"#
        0x0040:  2425 2627 2829 2a2b 2c2d 2e2f 3031 3233  $%&'()*+,-./0123
        0x0050:  3435 3637                                4567
    11:20:46.274575 IP 8.8.8.8 > 10.2.30.13: ICMP echo reply, id 4, seq 1, length 64
        0x0000:  4500 0054 0000 0000 2901 598b 0808 0808  E..T....).Y.....
        0x0010:  0a02 1e0d 0000 ed8f 0004 0001 de8b 655e  ..............e^
        0x0020:  0000 0000 0dae 0200 0000 0000 1011 1213  ................
        0x0030:  1415 1617 1819 1a1b 1c1d 1e1f 2021 2223  .............!"#
        0x0040:  2425 2627 2829 2a2b 2c2d 2e2f 3031 3233  $%&'()*+,-./0123
        0x0050:  3435 3637                                4567

可以看到直接就是4500开头的IP报文，没有二层的MAC信息。
这时我才意识到，我的VPN使用的是 Tun 设备的纯三层网络，报文不包含二层MAC信息，所以依赖二层
MAC地址的网关IP也失效了。
而这样纯三层的网络，转发路径纯粹是由VPN软件本身控制的。

查看一下VPN软件3层转发路径选择的相关代码：

    subnet_t *lookup_subnet_ipv4(const ipv4_t *address) {
        subnet_t *p, *r = NULL;
        avl_node_t *n;
        int i;

        // Check if this address is cached
        ...

        // Search all subnets for a matching one

        for(n = subnet_tree->head; n; n = n->next) {
            p = n->data;

            if(!p || p->type != SUBNET_IPV4) {
                continue;
            }

            if(!maskcmp(address, &p->net.ipv4.address, p->net.ipv4.prefixlength)) {
                r = p;

                if(p->owner->status.reachable) {
                    break;
                }
            }
        }

        // Cache the result

        cache_ipv4_slot = !cache_ipv4_slot;
        memcpy(&cache_ipv4_address[cache_ipv4_slot], address, sizeof(*address));
        cache_ipv4_subnet[cache_ipv4_slot] = r;
        cache_ipv4_valid[cache_ipv4_slot] = true;

        return r;
    }

可以看到VPN软件本身的路径选择算法非常简单，就是选择第一个匹配到的节点。
对于我们这种subnet相同的两个节点，VPN软件是按节点名字的字母顺序排列的：

    Subnet list:
     ...
     10.2.30.61/32#10 owner aws_jpn1
     10.2.30.71/32#10 owner gcp_hkc1
     ...
     0.0.0.0/0#10 owner aws_jpn1
     0.0.0.0/0#10 owner gcp_hkc1
     ...

可以看到 10.2.30.61 节点名字的字母序比 10.2.30.71 靠前，对于同样广播的 0.0.0.0/0 这条路由，
VPN软件自然就选择了 10.2.30.61 作为转发路径，而没有按我们预想的路由表工作。
至此，这个问题终于得到了解答。

### 问题解决

知道了问题的原因后，解决就比较容易了，有两个思路：

1. 将VPN改为支持二层信息的 Tap 设备模式，这样我们通过BGP发布实现的网关优先级就可以工作了
2. 网络转发路径的权重配置，通过配置VPN不同节点的优先级实现，而不是通过BGP Med配置
