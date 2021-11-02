---
layout: post
title: OpenStack Neutron 简介
categories:
- OpenStack
- Neutron
---

> 本文介绍了 Neutron 的功能、架构和抽象模型。

# Neutron 概述
最初 OpenStack 的网络功能寄生在 Nova 项目中，Nova-network 模块可提供简单网络拓扑（扁平、带 DHCP 的扁平、VLAN 网络，如下所示）和基本 L2、L3 网络服务（可为 VM 分配私有固定和浮动 IP 地址，且需借助外部物理路由器实现网络功能），并通过 ebtables 和 iptables 实现安全控制，支持 multi-host 部署在多个计算节点实现高性能和高可用，但是其不是一个灵活的框架，对多个厂商不同网络设备支持不好，随着云计算对网络复杂度提升，逐步孵化出了独立的 Neutron 项目（支持更复杂网络拓扑，包括 GRE、VxLAN 大二层网络隧道技术，插件式支持多种网络后端技术：OpenvSwitch, Cisco, LinuxBridge, NEC 等，支持 SDN 集成）。

Nova-network 支持的 flat/vlan 没有租户隔离的单一网络平面：

![](https://static2.mazhangjing.com/20211016/a238_n1.png)	

Neutron 在整个 OpenStack 的定位如下：

![](https://static2.mazhangjing.com/20211016/3466_n2.png)

简单来说，Neutron 为 OpenStack 提供了一个**统一的网络资源模型**（统一的北向编程接口）以及**插件 Plugins 与代理 Agent** 模块化可插拔架构（二层网络的 ML2 Plugin + L2 Agent，三层网络的 L3 Agent/dvr Agent/未来也有 ML3 Plugin，高级网络服务框架以及其 VPN 服务插件、负载均衡服务插件等）。基于这种可插拔架构，Neutron 可提供基于租户隔离的 L2 - L7 虚拟网络，并提供了比如 VPN、防火墙等 xAAS 高级网络服务。现在，网络设备厂商可扩展插件，在插件中加入自己驱动来实现对 Neutron 的支持，比如将完整的 SDN 产品和 OpenStack 集成起来。此外，这种微内核框架，使得插件和代理可通过消息通信解耦（AMQP），从而让 OpenStack 每一个进程都可以运行在任何节点上支持大规模部署。

## 可扩展的 Plugin 和 Agent 架构
Neutron 的可扩展架构图如下所示，包括核心的 **Network、Subnet 和 Port 网络资源 API，扩展的 ProviderNetwork，PortBinding，Router、Quotas，SecurityGroups，AgentScheduler，LBaas，FWaaS，VPNaaS** 等 网络服务 API。

这些北向 API 由 Neutron Server 负责提供（neutron-server 进程，一个位于 controller 上的控制网络服务的 Python Web 程序），而 Neutron Server 则通过提供网络核心能力 **Core Plugins** 和网络高级服务的 **Service Plugins** 提供，其中 Core Plugin 有 **ML2 Plugins**（负责管理虚拟交换机）和 **Vendor Plugins**（① 设备商可在此处对接物理交换机）两种，Core Plugin 用于实现 Core API，除了 Neutron 内置的 ML2 实现，还可提供自行实现，比如像 networking-odl、networking-cisco 等提供对 Core API 的扩展并对接设备商自己的网络设备和业务。对于高级网络服务，其被单独暴露为 ExtensionAPI ，设备商可提供 **Service Plugin** 以实现这些 API，比如可在此处对接自己的 SDN 控制器（② Service Plugins、③ Extension API）。	

![](https://static2.mazhangjing.com/20211016/48b9_n3.png)

![](https://static2.mazhangjing.com/20211016/abb3_n4.png)

这些北向 API 包括其可插拔的 Plugin 实现通过 MQ 和 **DHCP、L2、L3 Agent** 等进行解耦，Agent 用于具体实现 Plugin 的指令以创建和维护网络拓扑。

### L2 Network
ML2 Plugin 插件包含了 TypeDriver（包括 Flat、VLAN、VxLAN、GRE 等不同的网络类型）和 MechamismDriver（负责和二层网络真正打交道，可以有比如 LinuxBridge、OpenVSwitch、SDN 或者不同网络设备的实现）两部分。这里的 MechamismDriver 提供了多种网络实现，可以在 plugins/ml2/ml2_conf.ini 配置中通过定义 L2 agent_type（LinuxBridge 或 OpenvSwitch） 和 vif_type 虚拟界面类型来实现和具体实现的绑定。
	
> L2 层因为缺乏控制平面（不像 L3 有路由表），因此 L3 下来的报文必须 ARP 找到 IP 和 MAC 对应关系，以提供 MAC 头，这容易引发广播风暴。网卡通过中断 OS 内核来处理 L2 数据，如果不匹配 MAC 地址（如果使用了 VLAN，则 VLAN + MAC）则直接丢弃。VLAN 需要手动配置并且最多支持 4096 个端口，在数据中心远远不够

Neutron 通过 L2pop 特性解决了 L2 无控制平面的问题，可适应更大规模部署（GRE 在隧道使用广播，VxLAN 使用多播，L2pop 则禁用广播，其数据库包含了 IP 和 MAC 对应，通过 ebtables 截获 ARP 请求并直接将结果返回虚拟机），这种方式适合中等大小网络，MAC 一多还是不行。

### L3 Network
对于 L3 而言，可以由硬件实现路由，也可以由 Linux 系统通过 sysctl -w net.ipv4.ip_forward=1 充当路由器，在内网和外网进行 L3 通信，需要通过 **Source Network Address Transform SNAT** 源网络地址（外出）转换和 **DNAT 目的网路地址转换**（入内）实现，这个内部 IP 被称之为浮动 IP。Neutron 还在 L3 支持 dnsmasq 的 DHCP 服务，启动后，VM 的 IP 和 MAC 映射在 /var/lib/neutron/dhcp/NETID/host 中。

L3 使用 **DVR（分布式虚拟路由）**—— 每个 Nova 计算节点部署一个三层网络虚拟路由器，这样计算节点之间的东西流量（同租户相同子网虚拟机）可避免集中式经过 Neutron 转发，避免了集中化网络（计算节点的 DVR 可设置局部可见，支持为不同虚拟机相同子网使用相同网关，类似于 VXLAN BGP EVPN 方案的任播网关，这种方式可节省 IP，相比较 kubernetes，后者假定每台主机不重复唯一子网，不会有不同计算节点相同子网问题）。每个计算节点支持 DNAT 服务，支持访问浮动 IP，使得南北流量 DNAT 也可以避免走集中式网络节点。此外，Neutron 通过 **VRRP - Virtual Router Redundancy Protocol** 实现了虚拟路由服务的高可用。

### Network Service
对于其它高级网络服务，一般通过 neutron.conf 的 `service_provider` 指定，比如 LBaaS 负载均衡即服务（在租户 namespace 下安装 haproxy，netscaler 等），VPNaaS，FWaaS（区别 neutron 的 security group 在计算节点定义访问规则，firewall as service 在 L3 上限制对整个网络的控制 —— 租户所有虚拟机，一般基于 iptables）、Metering（网络计量服务，通过为 IP 段打标签方式计算内部和外部流量，通过 iptables 完成）、DNSaaS（处于孵化的 Designate 项目），Neutron 会根据配置的名称找到对应的 Extention Plugin 和 Agent，打通 Extension API 和网络拓扑并提供服务。

`FWaaS` 通过 L3 Agent 的 iptables 规则实现，其包含两个版本，第一个版本如右图所示，当其作用于一个路由器，则所有内部端口都收到保护。在 v2 版本中，则包含入口策略和出口策略，其工作在端口级别（使用公开 NameSpace 的作用于 tap 的 iptables 实现，而不再是由 L3 Agent 的位于 NameSpace 的 router 的 iptables 实现。	

VM 启动后还需要额外外部数据进一步配置，比如 `metadata 元数据服务`（大部分 VM 镜像都支持 cloud-init，后者可自动配置 VM 初始设置，比如主机名、网卡和密钥。元数据通过 `169.254.169.254/latest/meta-data` 访问，此 metadata 请求被转发到 9697 端口，即 `neutron-ns-metadata-proxy` 进程，通过 Socket 到 `neutron-metadata-agent` 后，通过配置文件的 nova-metadata_host 找到真正的 `nova-metadata-api`，`X-Forwarded-For` 到此地址实现的元数据服务）、`config-drive` 配置驱动服务、`inject` 文件注入服务（用于修改 root 密码、注入 ssh 密钥、个性化定制实例等）。

总结而言，Neutron 支持的特性如下所示：
![](https://static2.mazhangjing.com/20211016/d343_n5.png)

## 对多种网络拓扑的支持
对于网络拓扑而言，Neutron 支持很多类型，比如左图无隔离的静态 IP 分配的单张 Single FLAT Network（家庭网络），以及右图这种无隔离的静态 IP 分配的多子网 Multi FLAT Network（工作室网络）。

![](https://static2.mazhangjing.com/20211016/fcea_n6.jpg)
	
如果允许用户创建私网，那么就能够实现网段重叠以支持更多租户，通过某种机制，将这些私有网段合并在一张共享网络上，这就是 Mixed FLAT and Private Network（隔离实验网络），当然，可以提供一个路由，给租户分段，实现 L3 的租户隔离，但也因为路由，所以私有网段不能重合：Provider Router with Private Network（部门协作网络）。

![](https://static2.mazhangjing.com/20211016/d3b9_n7.jpg)
	
以及更深隔离和更多扩展性的 Per-tenant Routers with Private Network 实现：每个租户可自定义一个或多个网段，并有一个自己的 Router 以实现 L2 和 L3 的隔离（60% 以上生产实践都采用“多平面租户私有网络”网络拓扑）。这种方案需要 L3 Agent 提供 namespace 隔离的每租户虚拟 Router，L2 Plugin + L2 Agent 提供每租户 OVS 或 Linux Bridge 实现的虚拟交换机。	

![](https://static2.mazhangjing.com/20211016/7860_n8.png)

这里实现租户私有网络和物理平面网络隔离的方案是 `Underlay + Overlay`，即通过 VLAN、VXLAN 或 GRE 等隧道技术在 Overlay 覆盖层网络上实现不同租户（VRF）跨主机、跨机架、跨数据中心的私有网络，看起来好像每个租户私有一个网络，都独占一个路由器，可通过 SNAT/DNAT 实现外部网络访问，而实际则是通过 Underlay 底层网络负责东西向流量、南北向流量（出外网）的工作。从本质上来说 —— 所谓的 Neutron 网络就是一堆交换机：包括物理的和虚拟的，以及这些交换机实现的资源隔离。

## VLAN、Overlay & Big L2 Network
传统的 L2 没有控制平面， L3 的 IP 报文段下到 L2 后需要确定 MAC 地址，通过 ARP 协议广播来确定 IP 对应的 MAC，ARP 广播在大型 L2 网络中洪泛导致了广播风暴。一般而言，可通过 VLAN - Virtual Local Area Network 虚拟局域网来将 L2 隔离为不同广播域，每个广播域对应一个特定用户组，默认互相隔离，不同广播域如果要通信，则必须通过路由器，在 VLAN 的 L2 中，交换机通过 VLAN 号和 MAC 地址确定转发端口，ARP 广播会匹配特定 VLAN 组，避免了广播风暴。问题是，VLAN 这种方式修改了帧头，其要求手动配置所有物理交换机，工作量大，且难以大规模部署，此外，VLAN 号最大 4094，对于公有云有不切实际的数量限制。

基于 VLAN 思想，诞生了许多其它技术，比如📚多协议标签交换技术 MPLS Multiprotocol Label Switching，MPLS 在发包前先发送一个 hello 包，通过 L3 分布式路由协议确定好路径，之后在 L2 打好 MPLS 标签，之后 L2 直接由硬件根据 MPLS 标签转发，避免每一跳都需要在 L3 软件 TCP/IP 解包，找到 IP 进行转发，大大提升了性能。此外还有 GRE、VxLAN、VPN 等隧道技术，这些技术基本都是重新定制 L2 和 L3 头部，通过 L3 网络将数据传送到远端，在远端重新解包，即 Overlay Network 覆盖网络。覆盖网络是 SDN 的重要基础。GRE 克服了 VLAN 人工配置的缺点，其往数据帧添加了 GRE 帧头，但可能导致数据帧大于网卡 MTU，因此需要数据帧分片 —— 导致了性能下降，此外 GRE 隧道点对点，N 节点需要 N(N-1) GRE 隧道，浪费 L4 端口资源，从这一点来看，GRE 和 VPN 基本都使用 UDP 的原因在这里，UDP 用完后端口可立即释放 —— 无连接。VxLAN 类似于 GRE，其定义了一种在 L2 定制数据帧头的标准格式，提供多播缩小局域网广播范围。即便如此，VXLAN 洪泛查找 MAC 在大型网络中也是沉重的负担，为通告不同隧道端点背后的主机信息，可使用集中式控制器或者 BGP EVPN，为了实现多目的路由，可使用 IP 组播或者 IR 头端复制。对于 Neutron 而言，数据库记录了 IP 和 MAC 映射关系，通过 L2pop 的这些信息可禁用广播，ML2 Plugin 在这里替代 ARP 实现了 L2 控制平面角色。

Neutron 更关注单数据中心的网络管理，但不可否认的是，多数据中心的大二层互联技术正日益重要：比如大二层多路径技术，其可以解决 L2 东西流量，传统的 STP 生成树协议用于配合无控制平面的 L2 使用，但是在大二层中，网络是网状的，Cisco FabricPath 将三层基于 IP 的 ISIS 引入到二层，基于交换机 ID 创建全网交换机网络拓扑，之后就不需要像 STP 网络分层部署交换机，网络中交换机地位是对等的，无需 STP 生成树协议，无需 MAC 学习 —— 因为现在已经有了 ISIS 交换的二层拓扑信息，建立了控制平面，之后数据帧可直接基于此路径转发，不再依赖 MAC 地址。Cisco FabircPath 在大二层中取得了广泛的应用，但因为其是私有协议，且基于 MAC 标记的隧道扩展性不佳，现在多被 IP in UDP 的 VXLAN BGP EVPN 取代。

对于跨越数据中心的 L2 互联，比如混合云，目前主要依赖 VPN，GRE 隧道不具备地址学习能力，VxLAN 可以多播隔离广播域并进行地址学习，但对于广域网无能为力。因此这里 —— 即 SDN 的核心：覆盖网络技术中的覆盖网络传输虚拟化 Overlay Transport Virtualization OTV 技术，通过 MAC in IP 方式，让 IP 协议传输 OTV 自定义二层帧，相比较 VxLAN，其可以在广域网进行地址学习，建立了控制平面，OTV 在发送第一个数据帧的时候已经确定了路径，同时优化了二层网关，因此可内置 HSRP Host Standby Router Protocaol 热备份路由协议、VRRP Virtual Router Redundancy Protocol 虚拟路由冗余协议，GLBP Gateway Load Balancng Protocol 网关负载均衡协议。当然，不依赖 OTV 这种 DCI 的方式也有，比如 VXLAN BGP EVPN 提供的 multiPod、multiFabric 和 multiSite 等方案。

如果一个虚拟机想要跨越数据中心保持 IP 不变，其网关需要跟着移动，LISP Locator/Identifier Separation Protocol 位置身份分离协议可用来实现网络资源移动性问题，其在控制层使用 eBGP 进行 L2 地址学习。另一种方式是使用 VXLAN BGP EVPN 的 RT-3 路由通告移动性。

# Neutron 网络模型
简单来说，Neutron 包含三类节点 Node，也称作 Host。其中**计算节点 Compute Node** 构成了 OpenStack 最重要的部分之一 —— 计算，Host 中包含若干个虚拟出来的 VM，这些 VM 就是云的基础，相同 Host 的不同 VM、不同 Host 的 VM 之间的二层通信，可通过 Host 的 Bridge(虚拟出来的) 进行通信，而如果想要访问 Internet，则必须通过 Router 先到达数据中心网关出去，Router 也是 Linux 虚拟的，其位于**网络节点 Network Node** 中，网络节点还包含 DHCP 等网络服务，不论是计算节点还是网络节点，都可以部署在单个或者多个 Host 中。

对于计算节点而言，其包含了 OpenStack 可控制的基于虚拟网元实现的 Security Layer - 防火墙，以及 Integration Layer - 实现综合网络功能（交换/路由），位于其上的是 Top of Rack 机架顶端交换机，即 DC — Physical Network，不属于 OpenStack 管理。

对于二层网络而言，Neutron 支持 Local，Flat，VLAN，GRE，VXLAN，Geneve 等网络类型，其中以 VLAN、VxLAN 和 GRE 使用最多。

## 计算节点 - VLAN 实现模型（OVS）
图 3-5 是 VLAN 实现的一个典型模型，其中 br 和 qbr 开头的设备表示网桥，VM 表示虚拟机。从用户的角度看，这四个虚拟机被划分为两个 VLAN 100 和 200（图 3-7），但从实现的角度看，这四个虚拟机则分明是被划分为四个 VLAN（图 3-6）。这里的网桥，其中和 VM 直接相连的是通过 tap 实现的 —— VM 和 qbr 的连接（qbr 一般是基于 Linux Bridge 的网桥），之后此 qbr 通过 veth pair 和 br-int，然后 br-int 和 br-ethx 相连，后两者都是 OVS（OpenVSwitch 网桥），OVS 之间可通过 veth pair 相连，或者通过 patch 相连，后者性能更好。这里混合使用 bridge 和 ovs 的原因是，早期的 OVS 没有 Stateful openflow 规则，即不支持 iptables 实现安全控制，因此需要 Bridge，现在则完全是处于兼容的原因进行保留，br-int 的含义是 Bridge Integration（负责 attach 多个 VM 的 qbr Linux Bridge 网桥，并通过管理 VID 区分，负责 attach vRouter - DVR 等）。br-ethx 也是一个 Bridge，基于 OVS 实现，其含义是 Bridge Ethernet External 负责和外部通信，因此这里的 G 和 H 就是网卡 NIC Interface（Interface in Network Interface Card）。

![](https://static2.mazhangjing.com/20211016/ff75_n9.jpg)
	
而至于上述 VLAN 标签问题，OpenStack 是通过如下方案解决的：对于出网，在 br-int 中 D 处进行内部（管理） VID 打标签，br-ethx 中 F 处进行外部（客户）VID 打标签，对于入网，在 br-ethx 中先从 F 处将外部（客户） VID 转换为内部 VID，在 br-int 中在 D 处从内部（管理）VID 进行解包。

![](https://static2.mazhangjing.com/20211016/9267_n10.jpg)
	

## 计算节点 - VxLAN 和 GRE 实现模型（OVS）实现参见此处
VxLAN 模型的实现和 VLAN 模型很像，这里的 br-tun 指的是 Bridge Tunnel，和 br-ethx 区别并不大，它们都是 OVS，br-tun 执行的是 VxLAN 中 VTEP（VxLAN Tunneling End Point VXLAN 隧道终结点）功能。GRE 的 br-tun 执行的是 GRE Tunnel 功能，其余大致类似，但注意，GRE 虽然没有提供一个可见网络 ID（VLAN ID/VNI），但是内部还是有一个 Tunnel ID，即进行的是 VLAN ID 和 Tunnel ID 的转换。

![](https://static2.mazhangjing.com/20211016/401d_n11.jpg)
	
而 VxLAN 方案也是要进行内外 VID 转换的，和 VLAN 类似，在 br-tun G 处进行 VLAN-VXLAN 转换，在 br-int D 处进行 VLAN 到 Untag 的转换。

![](https://static2.mazhangjing.com/20211016/576a_n12.jpg)
	
计算节点的模型将上述流程分为用户网络层 User Network - 即 OpenStack 用户创建的网络（外部网络），对应 br-ethx 和 br-tun，一般使用 OVS 实现的本地网络层 Local Network（不论用户网络层采用何种技术 - VxLAN or GRE，本地网络层全部按照 VLAN），本地网络层又分为 qbr 负责 iptables 实现的安全层和 br-int 一般基于 OVS 的负责内部交换的 Bridge 层（Bridge 层用于屏蔽 VM 层，负责 VM Untag 报文和管理 VID 的报文转换，以及负责配合和各种不同实现的用户网络层用户 VID 进行协同工作。最后需要注意，同一 Host 的不同 VM 之间的流量（东西流量）只经过 qbr 和 br-int（这也是本地网络层之本地的含义），是不会上升到 br-tun/br-ethx 的。

![](https://static2.mazhangjing.com/20211016/d9e3_n13.jpg)	

## 计算节点 - SDN 实现模型（OpenDayLight）
略。

## 网络节点和控制节点的实现模型
一般而言，VM 访问因特网需要经过网络节点（Juno 之后 DVR 直接部署在计算节点，计算节点可直接访问 Internet，无需经过网络节点），此处的网络节点被看做第一层网关，即下图所示 GW1，注意对于 Neutron 而言，外层 GW2 和 GW3 被统称为外部网络 External Network。Neutron 在网络节点中通过虚拟路由器 vRouter 实现此网关功能。从网络视角看，网络节点分为 4 层，类似于计算节点的用户网络层、本地网络层以及底部提供服务的网络服务层：包括 ①采用 dnsmasq 进程提供 dns、dhcp、tftp 等服务的 DHCP Service（通过 namespace 实现隔离，每个网络运行在一个 namespace 中），②Router 内核模块实现的路由转发，包括 SNAT/DNAT 功能（同样的，每个 Router 运行在一个 namespace 中），注意，这里的核心在于 Router 而非 br-ex，br-ex 仅仅是一个 Bridge，一般是 OVS，其核心在于 Router。就像 Nova Host 的最终在于 VM 计算一样，Neutron Host 的最下层在于连通外网，即外部网络层。

![](https://static2.mazhangjing.com/20211016/86f4_n14.jpg)
		
控制节点的 Neutron 进程 - neutron-server 通过 RESTful 或 CLI 接收外部请求后，Plugin 通过 RPC 和 Agent 交互以部署网络。总的来说，控制节点并没有具体实现网络功能，其仅仅对各种虚拟网元做配合管理的工作。因为控制节点（网络节点）和计算节点都需要做具体网络部署工作，因此很多 agent 在两边都有，但也存在差异，比如在安装 OpenStack 中，我们为 Nova Node 安装了 libvirt 和 nova-compute 服务，以及 neutron-linuxbridge-agent，而在 Controller Node 则安装了 nova-server，neutron-linuxbridge-agent，neutron-l3-agent，neutron-dhcp-agent，neutron-metadata-agent 等服务（这里 Controller Node 同时扮演 Networking Node 角色）。

![](https://static2.mazhangjing.com/20211016/15b9_n15.png)

以上节点构成了 Neutron 的整体 - Network As a Service - NaaS，其中控制节点 Neutron 进程 Plugin 通过 RPC 和网络、计算节点的各 Agent 配合，向外提供 RESTful 服务接口。
	
其中计算节点各 Bridge 构成了 Neutron 的 L2 网络（GRE、VLAN、Flat、VxLAN 等），br-ethx/tun 对外构建了用户网络，对内 br-int 负责屏蔽 VM，提供 VM 本地网络，qbr 则提供了额外的 VM 安全功能。
	
其中网络节点则提供了其他网络服务，比如 DHCP，Router 负责提供三层服务，SNAT/DNAT 等。

# Neutron 资源模型
Neutron 的资源模型包括租户隔离下的 Network、Subnet、Port 和 Router 等，其通过一组模型进行分配和消费，这个模型的概念有：区域 Region、服务 Service、端点 EntryPoint、域 Domain、项目 Project、组 Group、用户 User、角色 Role、凭证 Token，见安装 Keystone 章节。

Newton 之前使用 tenant_id，之后使用 project_id 表示租户。租户隔离意味着，在数据层，网络不能互通，一个租户感知不到另一个租户的网络，且可以重复网段。在故障处理方面，意味着一个租户网络故障将不会影响另一个租户网络。根据实际，管理面和数据面的故障，必须要做到租户隔离，而物理资源层面，则无法实现故障隔离，比如不能为每个租户实现一个网络节点。对于计算节点来说，br-tun/br-ethx 和 br-int 是共享的，通过 VLAN 或 tunnel 实现正常流量隔离，qbr 则是租户各有的，其用来实现异常隔离控制，DVR Router 也在计算节点，其通过 namespace 实现隔离。对于网络节点而言，DHCP 和 Router 都是 namespace 实现隔离的，DVR 和 Router 的隔离另外保证了逻辑资源（IP）冲突的问题。

![](https://static2.mazhangjing.com/20211016/df1e_n16.jpg)
	
从管理层面，在硬件/OS 上，不存在租户隔离，管理面（控制节点）部署在一个 Host，多租户共享，在应用程序上，多个租户也是共享的，在数据库层面，Neutron 采用共享表，通过表中字段实现弱隔离，数据库本身还是共享的。	

![](https://static2.mazhangjing.com/20211016/e0d4_n17.png)

## Network
Neutron 的网络资源模型（支持 Local、Flat、VLAN、VxLAN、GRE、Geneve）如下所示。注意新版本 project_id 替换了 tenant_id 表示租户。此外比较常见的有 id，project_id，name，mtu，availability_zones，qos_policy_id，shared，router:external，subnets。

### Provider Network
由租户完全负责创建管理的网络称之为租户网络（比如 Local 网络无需和任何本地网卡绑定，全部是虚拟的，可租户管理，就是一种租户网络），其他的涉及不在 Neutron 管理范围内的网络被统称为外部网络，即运营商网络 Provider Network（比如 Flat 网络需要绑定一个外部网卡，这就涉及了运营商网络） 。

![](https://static2.mazhangjing.com/20211016/a96b_n16.png)

上图所示为 Project Network 和 Provider Network 的关系，Provider Network 就是涉及真实网卡和线路、交换机、路由器的网络，而项目的网络则是完全虚拟的，包括 L2 层和 L3 层的虚拟。在 OpenStack 语义下，只要涉及配置涉及物理网络设备的部分就称之为 Provider Network，反之则是 Neutron 自行管理的普通 Network。

管理员可创建运营商网络，创建运营商网络必须传入 provider: 开头的三个参数，而租户网络不需要（租户可自行创建租户网络），其自动分配。运营商网络的一个目的是为了混合云，比如在下图中，为了和客户本地云 VLANID 100 融合，OpenStack 通过创建一个运营商网络与之匹配：provider:network_type 表示另一网络类型，provider:segmentation_id 表示另一网络 SegmentationID。而不需要这些参数的租户网络则通过 /etc/neutron/plugins/ml2/ml2_conf.ini 中的 tenant_network_types 和 vni_ranges 提供了网络类型和自动分配的网络 ID。

![](https://static2.mazhangjing.com/20211016/dde4_n17.jpg)
	
而对于 provider:physical_network 字段，其代表一个可读字符串，在运营商网络中表示运营商网络需要匹配的外部网络名称，对于运营商网络 or 租户网络，其都意味着 br-ethx 的选择（主机网卡的选择），即走哪条路出去（比如 network_vlan_range(user) 或 provider:physical_network(admin) + bridge_mapping/OVS； flat_network(user) 或 provider:phsical_network(admin) + physical_network_mapping/LinuxBridge) 。

![](https://static2.mazhangjing.com/20211016/4fde_n18.png)

对于非隧道型网络而言，如果 br-ethx 使用 OVS 实现的，那么在 etc/neutron/plugins/ml2/openvswitch_agent.ini 的 bridge_mappings 中可配置物理网络和网卡的映射，比如：physnet1:br-ethx1, physnet2:br-ethx2，之后创建运营商网络时传入 provider:physical_network 参数（比如 physnet1）即可走正确的网卡（br-ethx1）。而对于租户网络而言，其无法传入 provider:physical_network 参数，需要配置 /etc/neutron/plugins/ml2/ml2_conf.ini，在 ml2_type_vlan 的 network_vlan_ranges 或 ml2_type_vxlan 的 vni_range 中配置 physnet1:1000:2999, physnet2:3000:4000 来通过 VID 间接获取到 physical_network 网络名称，进而决定选择哪个 br-ethx 出。

### Trunk Network & VLAN Aware VM
Bridge 有三种 VLAN 接口模式，分别是：
![](https://static2.mazhangjing.com/20211016/4baf_n19.jpg)
	
**Access 接口模式**，对于 Tag 报文直接丢弃，Untag 报文打上默认 VID Tag，然后进入交换模块，交换后去除 Default VID Tag，然后出去。其主要目的是使得同一 VLAN 内部可以彼此通信。

**Trunk 接口模式**增加了对于入口的 VLAN ID 控制，对于 Untag 报文打上默认 VID Tag 进入，对于 Tag 报文，过滤，且只允许满足条件的 VID Tag 报文进入，不满足则直接丢弃，哪怕等于 Default VID，只要有 Tag 且不在范围内都丢弃。已经有 VLAN ID 的满足过滤条件的报文不再打 Default VID Tag，直接进，直接出（出去不去除原本的 VID Tag）。Untag 的报文进入打 Default VID Tag，出去解除 Default VID Tag。Trunk 是为了让不同 VLAN 之间彼此可以通信。

**Hybrid 接口模式**和 Trunk 接口模式类似，区别在于，除了增加了进入时对于 VID 的过滤，还增加了出去时对于 VID 的处理，如果在需要去除 VID 范围内，则先去除 Tag 再出接口。

默认情况下，计算节点（OVS 实现）的各个网桥采用如左图所示的 VLAN 连接方式，但是也有例外情况：比如一些应用需要连接到上百个 neutron 网络、一个 VM 需对容器可能需要连接到不同的子网络、一些遗留的应用可能希望使用不同的 VLAN 连接到不同网络，因此 VM 应该能够发送和接受带有 VLAN Tag 的报文，即 VLAN aware VM —— 即一个 VM 可以接入多个 Network。VM 通过 Port（vNIC）接入网络，而一个 Port 只能属于一个 Network。

解决方案有：

1. 一个 VM 多个 vNIC/Port，这并不现实，对接到上百个 Network 需要上百个虚拟网卡。
2. 使用 Bridge VLAN 的 Trunk 模式，一个 Port 对应多个 Network，这看似没有什么问题，但因为上文提到的 VLAN ID 内外问题，如果我们的 VM 需要 VLAN aware，且指定需要一个 VLAN ID 10，那么其可能和用于管理的内部 Host VLAN 冲突，且租户并不知道。
3. 在 Kilo 之后，Network 有了一个 vlan_transparent 字段表示支持 VLAN 透传，但是并未实现，Host 内部网络、Host 之间物理网络如何解决透传都很难解决，处于这样的原因，OVS 不支持 VLAN 透传。
4. Trunk Networking：Newton 版本引入，总的思想是：VM aware 的 VLAN ID 在 Host 内部不能冲突，VM aware 的 VLAN ID 不在 Host 之间物理网络透传。其解决方案是，在需要 VLAN aware 的 VM 前加了一个 br-trunk，其使用 Trunk/Hybird 模式，br-trunk 自动对用户定义的 VM3 aware VLAN ID 进行了转换避免内部 VLAN ID 冲突（图4-24）。这里假想的模型是“一个 Port 对应多个 Network”，但这种方案需要破坏兼容性。实际上，Neutron 引入了一个叫做 trunk 的模型，其包含 ParentPort：port_id 字段，用来定义父端口的 ID（走 Untag 报文），以及 SubPortList：sub_ports 字段，用来表示 Trunk 关联的子端口列表（走 VLAN aware 带有 VLAN ID 的 Tag 报文）。从 VM 视角来看，其在一个端口可发送带有 VLAN ID 的报文，即实现了多个网络，从 Neutron 的视角看，其保持了一个 Port 对应一个 Network 的惯例。


对于普通报文而言，其走的是 br-trunk 的 Trunk Parent Port vNIC，和之前没有区别 —— 除了多走了一段路，还是在 br-int Untag 转换内部 VID 30，在 br-ethx 转换外部 VID 300。

![](https://static2.mazhangjing.com/20211016/89c1_n20.png)

对于 VLAN aware 报文而言，其经过 br-trunk 后剥离 VLAN VID，进入 Trunk SubNet，然后使用内部 ID 1010 和外部 ID 100。对于 Neutron 而言，其保持了原有结构，一个 Port 对应一个子网（需要配置 br-int 上 ParentPort 和 SubPortList 对应的 VID 信息，但从代码上来看，并没有发生变化，还是为每个 Port 配置对应的 VID 信息），对于 VM 而言，其从一个 Port 发送出了多个 VLAN 的报文，实现了联通多网。	

![](https://static2.mazhangjing.com/20211016/94a7_n21.png)

## Subnet
子网表示属于某个网络的一批 IP 地址的集合，其模型如下所示。核心参数有：id，network_id，ip_version，cidr，dns_nameservers，enable_dhcp，allocation_pools，host_routes，gateway_ip。

子网提供 IP CoreNetwork Services，又称之为 DDI 服务：DNS、DHCP、IPAM（IP Address Management IP 地址管理系统）enable_dhcp，allocation_pools（DHCP），dns_nameservers（DNS），subnetpool_id（IPAM）提供了 DDI 服务的相关配置。这里 subnetpool_id 和 allocation_pools 是重复的，但 DHCP 和 IPAM 都是可选服务，因此重复是有必要的。所谓 Subnet Pool 资源池，在 Kilo 版本加入，指的是定义一个大的网段，Subnet 从其中分配一个小的网段。相关的字段有：default_quota（可选，定额分配。表示一个 Project 最多可以申请的 IP 个数，默认为 IPv4 32 位掩码子网网段的个数），prefixes（数组，资源池子网前缀），min_prefixlen，default_prefixlen，max_prefixlen（分配的地址长度）。一般情况下，通过 openstack subnet create 创建子网，要选定子网池，从 prefixes 选择，一般提供 prefixlen 要求 Neutron 自动提供这么多个 IP 地址，如果不提供，则默认是 default_prefixlen 长度个 IP 地址，其默认是 min_prefixlen（默认 IPV4 8，IPV6 64）。

## Port
Port 指的是 Neutron 中的虚拟网口，虚拟机、路由器都需要绑定 Port，它也有两个基本属性：一个 MAC 地址和多个 IP 地址。Port 必须属于一个 Network，通过 network_id 来表示。fixed_ips 表示 Port 的多个 IP 地址，以及其各自所属的子网。理论上，如果一个 Network 有 N 个 Subnet，则 Port 最多可以有 N 个 IP 地址，分属 N 个 Subnet（一个 Subnet 必须属于一个 Network）。	
一般而言，Port 会有一个 MAC 地址，表示在 mac_address 字段中，但可以为每个 IP 分配一个 MAC 地址，表示在 allowed_address_pairs 中。此外，Port 必须依附于一个设备，通过 device_id 和 device_owner 来表示设备 ID 和设备类型，比如 compute:nova。

![](https://static2.mazhangjing.com/20211016/45a3_n22.png)


## Router
Router 资源的核心在于端口和路由表，其中路由表使用 routes 表示，其包含了目的 CIDR 网段、下一跳 IP 地址。而端口并不采用字段标识，而采用 /v2.0/routers/xxxid/add(remove)_router_inteface 来负责添加和删除端口。对于外部网络，Router 采用 external_gateway_info 表示外部网关信息。

![](https://static2.mazhangjing.com/20211016/0198_n23.png)

这里如果要访问外网，需要路由记录外部网络 104.20.110.0/24 指向下一跳地址 182.24.4.1，从 Port1(182.24.4.6) 出去，external_gateway_info 包含了 network_id（必须为所代表的 Network，其 Router：external 字段必须为 true）、enable_snat、external_fixed_ips（ip_address 表示出口 Port1 IP：184.24.4.6， subnet_id 表示与之相连的外部网络 Router2 的子网信息，从中提取 Subnet 的 gateway_ip：184.24.4.1） 等字段，如果启用了 enable_snat，那么 SNAT 将会在 Router_1 的 Port1 进行转换，源 IP 将会变成 Port1 的 IP 地址：184.24.4.6。

![](https://static2.mazhangjing.com/20211016/8bb6_n24.png)

当需要添加 Router 接口时，通过 API 调用添加，传入 router_id + subnet_id 或者 router_id + port_id，如果传入的是 port_id，则 Router 会给自己增加一个 Port 并使用 Port 的 IP 地址。如果传入 subnet_id 则需要从其 gateway_ip 中获取 IP 地址并分配给网关。增加端口有一些限制，比如 Router 绑定的端口关联的子网（指的不是直连子网的端口，而是带有外部路由器的外部子网的端口）只能有一个 IPv4 Subnet，一个 Router 可以绑定多个端口，这些端口可能属于多个 Network，对于同一 Network 所有端口，只能关联一个 IPv6 Subnet。	

在 Router 增加端口 Port，意味着此端口 Port 背后所有 Subnet 流量都只能从此端口 Port 进入路由器，从此端口 Port 出去流量能到达背后 Subnet，这称之为直连路由（即此端口背后就是一张网，没有其他路由器阻隔），其无需 Router 上创建路由表项，Router 通过链路层协议发现。对于外部网关 external_gateway_info 信息，Router 会增加默认静态路由，但是这些都不存在 routes 字段（路由表）中，因为其不包含目的网段，除了直连的和配置在 routes 文件中的，其余都走这条路，因此也称之为默认静态路由（路由表找不到默认跳转的外部网络，default，一般为公共网络）。routes 字段的路由则被称之为静态路由（即此端口背后还有路由器，必须记录才可跳转），一个例子如下所示，nexthop 指的是下一个“外部”路由器的接口 IP，这里一般指的是私网，没有启用 SNAT 功能（区别于默认静态路由 external_gateway_info）。

此外，Router 还有 DNAT 功能，但是其被放在 Floating IP 模型(做 DNAT/SNAT 转换)中，而不在 Route 模型中，Floating IP 资源包含 router_id 路由编号、fixed_ip_address 与浮动 IP 关联的 IP 地址、floating_ip_address 浮动 IP 地址、port_id Router 上占用 fixed_ip_address 的 PortID，以及 Floating IP 本身的 id。如下图所示，这里的浮动 IP 指的就是和 Router_1 关联的资源， port_id 是 Port3，floating_ip_address 是 Port3 的 IP 地址 182.34.4.2，fixed_ip_address 是 10.0.0.3，其互相关联以实现 DNAT：源是10.0.0.3 则转换为 182.34.4.2，目的是 182.34.4.2，则转换成 10.0.0.3。

![](https://static2.mazhangjing.com/20211016/f66e_n25.png)

下面是关于默认静态路由、静态路由和直连路由的一个示意图：

![](https://static2.mazhangjing.com/20211016/3292_n26.png)
![](https://static2.mazhangjing.com/20211016/5d85_n27.png)

这里的 DNAT 绑定在 Port6 上，floating_ip_address 是 200.10.20.2，fixed_ip_address 是 10.0.20.3。

注意，Kilo 版本后，只允许一个 Router 外部网关接口（即接外部路由器的网关接口，并非直接直连子网的接口）只能有一个 IPv4 和一个 IPv6 地址，即 Port5、6、7 只是示意图，实际上只会有一个。

## Multi-Segments
Neutron 的 Network 中包含多个 Subnet，这些 Subnet 需要借助路由器实现互通。Network 包含一个字段 segments，其不能和 provider extended attributes 混用，原因是，启用 Segment 后，Network 就变成了一个 L2 容器，而每个 Segment（对应一个 Subnet） 都是一个独立的 L2 Network。	

![](https://static2.mazhangjing.com/20211016/5e8e_n28.png)

### Segment as Subnet
在一般的 Neutron 环境中，Network 一般包含一个 Subnet，但是如果一个 Network 中 VM 或 Host 太多，那么将面临二层广播风暴、ARP 表的交换机容量限制问题。可以将一个 Network 划分多个子 Network 解决，此时 Network 称之为 Routed Network，子 Network 称之为 Segment，每个 Segment 有不同的 Subnet，其内部只二层通，互相可通过物理路由器实现三层通。

实现方法可以是 Network 模型中的 segments 字段，也可以是独立的 Segment 模型，区别是，Network 模型中的 segments 字段指向其关联的多对一的多端 Segment，而 Segment 模型的 network_id 字段则指向其关联的多对一的一端网络。这里的一个 Segment 就对应一个物理网络-机架，也对应一个隔离的 L2 Network - Subnet（1:1 关系）。

这里有限制，这里的路由器 - 即在多个 Subnet/Segment 之间 L3 互通的必须是物理路由器，一个机架连接到一个物理路由器后网口固定，其网口 IP 也固定，因此，此机架对应的 Host/VM 的 Subnet 需要这个物理路由器上网口的同一网段。

### VTEP on TOR
Multi-Segment 的第二种用法是 VTEP（VXLAN Tunnel End Point） 位于 TOR 交换机上，而不是位于计算节点 br-tun 上，这里 Network 指的还是二层网络，TOR 需要做 VNI（VXLAN ID）和 VLAN ID 的映射(就好像之前 br-tun 做的那样，将内部管理 VLAN ID 映射为用户 VXLAN VNI)，这个映射并不能随意修改，可以基于 segments 字段的列表进行处理：这里 network_type 为 vlan，其指的是内部网络 ID，其 segmentation_id 为 VLAN ID，VNI 由 Neutron 内部根据此处 physical_network 和配置文件映射规则自动生成。

![](https://static2.mazhangjing.com/20211016/2ac8_n29.png)

总的来说，Neutron 资源模型核心在于 Network、Subnet 和 Port，VM 通过 Port 连接到 Router，后者通过静态默认路由（实现 SNAT 外网访问）、默认路由和直连路由（实现 Host Port 互通）实现转发，通过 Floating IP 实现 DNAT 内网访问。	