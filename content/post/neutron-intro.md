+++
date = "2015-04-21T15:22:18+08:00"
draft = false
title = "neutron 简介"
Tags = ["openstack"]
Categories = ["Development"]
+++

首先通过nova查看当前虚拟机的id：

    [root@controllera-30 ~(keystone_admin)]# nova show win7-desk25
    +--------------------------------------+----------------------------------------------------------+
    | Property                             | Value                                                    |
    +--------------------------------------+----------------------------------------------------------+
    | OS-DCF:diskConfig                    | MANUAL                                                   |
    | OS-EXT-AZ:availability_zone          | nova                                                     |
    | OS-EXT-SRV-ATTR:host                 | host-27                                                  |
    | OS-EXT-SRV-ATTR:hypervisor_hostname  | host-27                                                  |
    | OS-EXT-SRV-ATTR:instance_name        | instance-00000006                                        |
    | OS-EXT-STS:power_state               | 1                                                        |
    | OS-EXT-STS:task_state                | -                                                        |
    | OS-EXT-STS:vm_state                  | active                                                   |
    | OS-SRV-USG:launched_at               | 2015-01-31T13:37:38.000000                               |
    | OS-SRV-USG:terminated_at             | -                                                        |
    | accessIPv4                           |                                                          |
    | accessIPv6                           |                                                          |
    | config_drive                         |                                                          |
    | created                              | 2015-01-31T13:37:15Z                                     |
    | flavor                               | cdsp-2048-2-60 (128c133a-6c16-4558-a495-b5e73640526a)    |
    | hostId                               | 47c748662e01e339b4a942f7c1c7fcc9f90313bbd6bcf24c1d2ffa5a |
    | id                                   | e4f65799-42a4-47d5-bdc6-5fb51b942d64                     |
    | image                                | win7template (d280d0c1-d239-4a3b-8b64-8b58dbe94628)      |
    | key_name                             | -                                                        |
    | metadata                             | {}                                                       |
    | name                                 | win7-desk25                                              |
    | net1 network                         | 192.168.1.8                                              |
    | os-extended-volumes:volumes_attached | []                                                       |
    | progress                             | 0                                                        |
    | security_groups                      | default                                                  |
    | status                               | ACTIVE                                                   |
    | tenant_id                            | e7fd033bb1e546248fb530baaf83e891                         |
    | updated                              | 2015-02-02T10:51:45Z                                     |
    | user_id                              | 1910f945b7c34e50be16e0be51bc13aa                         |
    +--------------------------------------+----------------------------------------------------------+

可以看到这个虚拟机的实例名为instance-00000006，运行在host-27主机上。根 据这些信息，在host-27上查询该虚拟机的domid：

    [root@host-27 ~]# virsh list | grep instance-00000006
    5     instance-00000006              running

通过dumpxml相关信息，可以查看到对应的网卡信息：

    [root@host-27 ~]# virsh dumpxml 5 | grep tap -b3
    2081-    <interface type='bridge'>
    2111-      <mac address='fa:16:3e:99:4e:20'/>
    2152-      <source bridge='qbraadd565d-89'/>
    2192:      <target dev='tapaadd565d-89'/>
    2229-      <model type='virtio'/>
    2258-      <driver name='qemu'/>
    2286-      <alias name='net0'/>

可以看到虚拟机网卡设备是tapaadd565d-89，连接到网桥qbraadd565d-89，网桥 信息可以通过brctl来查看：

    [root@host-27 ~]# brctl show
    bridge name     bridge id               STP enabled     interfaces
    qbraadd565d-89          8000.7a998003561a       no              qvbaadd565d-89
                                                            tapaadd565d-89
    virbr0          8000.000000000000       yes

由于现在OpenStack使用Linux网桥的安全策略，因此创建虚拟机时，会先在 Linux网桥上创建port，并将虚拟机连接上去，然后再将该网桥上的端口连接到 对应的ovs网桥上去。

通过ovs命令可以查看对应的br-int上的port端口：

    [root@host-27 ~]# ovs-vsctl show
    057b47ea-32c4-4a5f-8762-6fe377c7c2a5
        Bridge br-int
            fail_mode: secure
            Port "qvoaadd565d-89"
                tag: 3
                Interface "qvoaadd565d-89"
            Port br-int
                Interface br-int
                    type: internal
            Port "int-br-eth1"
                Interface "int-br-eth1"
        Bridge "br-eth1"
            Port "br-eth1"
                Interface "br-eth1"
                    type: internal
            Port "phy-br-eth1"
                Interface "phy-br-eth1"
            Port "eth1"
                Interface "eth1"
        ovs_version: "2.3.0"

可以看到，在ovs网桥上有一个对应的端口qvoaadd565d-89，这个端口和Linux网 桥上的qvbaadd565d-89端口是一对veth pair，相当于一根网线的两头，数据分 别从一端进入然后从另一端出来。这两个端口一端连接到Linux网桥，一端连接 到ovs内部网桥br-int上。

通过上面ovs的信息，br-int和br-eth1两个网桥之间也有一对veth pair，分别 是int-br-eth1和phy-br-eth1。这对veth pair负责将内部网桥和外部网桥连接 起来，内部网桥负责连接各个虚拟机，以及网桥内部虚拟机之间的消息通信。而 外部网桥负责和外部进行网络通信。

为了查看veth pair可以通过命令来查看：

    [root@host-27 ~]# ethtool -S qvoaadd565d-89
    NIC statistics:
         peer_ifindex: 35

查看qvoxxxxxx对应的配对，通过上述命令可看到对应的index信息，再通过ip link来查看对应的信息

    [root@host-27 ~]# ip link | grep aadd565d-89
    33: qbraadd565d-89: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT
    34: qvoaadd565d-89: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master ovs-system state UP mode DEFAULT qlen 1000
    35: qvbaadd565d-89: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master qbraadd565d-89 state UP mode DEFAULT qlen 1000
    37: tapaadd565d-89: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master qbraadd565d-89 state UNKNOWN mode DEFAULT qlen 500

可以看到qvoaadd565d-89和qvoaadd565d-89是一对veth pair。

在计算节点上大致的一个连接示意图如下所示：

           +------------------------+
           |  instance-00000006     |
           |                        | 虚拟机
           |           +------------+
           |           | eth0       |
           +-----------+-----+------+
                             |
                             |
           +-----------+-----+------+
           |           | tapxxxxxx  |
           |           +------------+
           |   qvrxxxxx             |  Linux 网桥
           |           +------------+
           |           | qvbxxxxxx  |
           +-----------+-----+------+
                             |
                             |
           +-----------+-----+------+
           |           | qvoxxxxxx  |
           |           +------------+
           |  br-int                | ovs内部网桥
           |           +------------+
           |           | int-br-eth1|
           +-----------+-----+------+
                             |
                             |        vlan tag 3
           +-----------+-----+------+
           |           | phy-br-eth1|
           |           +------------+
           |  br-eth1               | ovs外部网桥
           |           +------------+
           |           | eth1       |
           +-----------+-----+------+
                             |        vlan tag 1000
                             |         Network
    -------------------------+-------------------------------

如果配置了vlan，那么在br-int内部，所有网络包会打上内部的tag标签，如 tag3，但是通过br-eth1时，会将这个内部标签修改为外部标签，例如tag1000。 这个转换是根据openstack配置信息，当前网络允许的vlan tag范围来进行映射 的。

通过tcpdump抓包可以发现，在通过phy-br-eth1端口上的包仍然是tag3的内部 tag，但是从eth1流出的包就被转换成了tag1000，可以了解到这个转换是在 br-eth1内部进行的。
