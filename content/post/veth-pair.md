+++
date = "2015-03-21T15:18:12+08:00"
draft = false
title = "veth pair"
tags = ["openstack"]
topics = ["Development"]
+++

Virtual Ethernet Pair简称veth pair,是一个成对的端口,所有从这对端口一 端进入的数据包都将从另一端出来,反之也是一样.

<!--more-->

下面用例子说明veth pair的创建和使用:

现在有这样一个环境,两个网桥,一个是Linux内核网桥br1,另一个是ovs网桥 br-eth1,现在想把两个网桥连接起来,就可以用veth pair.

    +------------------+              +------------------+
    |                  |              |                  |
    |   ovs bridge     |              |   Linux bridge   |
    |   br-eth1        |              |   br1            |
    |                  |              |                  |
    +------------------+              +------------------+

首先创建一对veth pair:

    [root@compute-195 ~]# ip link add eth1-br1 type veth peer name phy-br1
    [root@compute-195 ~]# ip link list
    
    ...
    
    25: phy-br1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
        link/ether 22:83:01:db:37:b4 brd ff:ff:ff:ff:ff:ff
    26: eth1-br1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
        link/ether 32:b2:aa:1a:8a:13 brd ff:ff:ff:ff:ff:ff

创建成功后,可以通过ip link看到一对端口,可以通过ethtool查看他们是否成 对:

    [root@compute-195 ~]# ethtool -S phy-br1
    NIC statistics:
         peer_ifindex: 26

创建成功后,就可以分别将两个端口添加到不同的网桥上:

# Linux网桥

    [root@compute-195 ~]# brctl addif br1 phy-br1
    [root@compute-195 ~]# brctl show
    bridge name     bridge id               STP enabled     interfaces
    br0             8000.e41f136dc9f0       no              enp11s0f0
                                                            vnet0
                                                            vnet2
                                                            vnet3
    br1             8000.deef26d9c76a       no              phy-br1
                                                            vnet1
                                                            vnet4

可以看到phy-br1添加到了br1网桥上.

# ovs网桥

    [root@compute-195 ~]# ovs-vsctl add-port br-eth1 eth1-br1
    [root@compute-195 ~]# ovs-vsctl show
    e61b93e6-a701-4b6e-86c2-05f883885ab8
        Bridge br-int
            fail_mode: secure
            Port "qvof01b51bf-71"
                tag: 1
                Interface "qvof01b51bf-71"
            Port "int-br-eth1"
                Interface "int-br-eth1"
            Port br-int
                Interface br-int
                    type: internal
        Bridge "br-eth1"
            Port "eth1-br1"
                Interface "eth1-br1"
            Port "eth1"
                Interface "eth1"
            Port "phy-br-eth1"
                Interface "phy-br-eth1"
            Port "br-eth1"
                Interface "br-eth1"
                    type: internal
        ovs_version: "2.3.0"

可以看到eth1-br1也添加到了br-eth1网桥上了.这样整个网络就完成了网桥的 连接:

    +------------------+              +------------------+
    |  ovs bridge      |              |   Linux bridge   |
    |  br-eth1         |              |   br1            |
    |       +----------+              +----------+       |
    |       | eth1-br1 +--------------+ phy-br1  |       |
    +-------+----------+              +----------+-------+

