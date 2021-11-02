---
layout: post
title: OpenStack Neutron 网络实战 (LinuxBridge, OVS)
categories:
- OpenStack
- Neutron
---

> 本文介绍了 OpenStack Horizon 界面操作网络相关功能以及操作分别基于 LinuxBridge 和 OVS 的底层实现原理，基于 OpenStack Rocky 和 CentOS 7。

# OpenStack 检查 
安装情况：在 Controller 节点 `systemctl | grep neutron-*、openstack-*` 可以查看起的相关 OpenStack 服务，注意检查下是否有没启动的，去 `/var/log` 对应的日志文件看一下原因。

|  |  |
|--|--|
|`systemctl / grep openstack`	| `systemctl / grep neutron`|
|openstack-glance-api.service	| neutron-dhcp-agent.service|
|openstack-glance-registry.service	| neutron-l3-agent.service|
|openstack-nova-api.service	| neutron-linuxbridge-agent.service|
|openstack-nova-conductor.service	| neutron-metadata-agent.service|
|openstack-nova-consoleauth.service	| neutron-server.service|
|openstack-nova-novncproxy.service | |
|openstack-nova-scheduler.service | |
| | |

在 Horizon，可以通过如下路径查看服务、网络和计算激活信息：`Admin > System > System Information` 的 Services、Compute Services 以及 Network Agents。此外，可以在`网络 > 网络 > 端口 > (对于所有端口)编辑端口 > "端口安全"`选项关闭，以简化网络调试过程（有些时候，Ping 不通就是这里的原因，根据我的经验，在 Flat 网络中，此处无所谓，在 VxLAN 中还好，但在 VLAN 网络中，此处影响连通性）。此外，注意设定 `网络 > 安全组 > default`，打开所有的 TCP/UDP/ICMP 连通，这里很奇怪的只影响 VLAN 网络的 ping 连通性，即便没有为 VLAN 网络附加此安全组。

在安装的时候，`systemctl start neutron-l3-agent` 的时候，没有任何提示，但是服务会启动失败，因此还是很有必要按照上述步骤检查一下，比如典型的官方文档的 neutron-l3-agent 配合 LinuxBridge 会提示找不到 TypeDriver，需要查看 `/var/log/neutron/l3-agent.log` 并且修改对应配置： `/etc/neutron/l3_agent.ini`
```
#interface_driver = linuxbrdige
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
```

`[非官方]` 对于 Controller 节点和 Compte 节点，需要打开 ip 转发，设置 NAT 伪装，关闭 rp_filter 等，这些都要额外检查：修改 `/etc/sysctl.conf` to contain the following 然后 `sysctl -p /etc/sysctl.conf` 进行加载：
```
net.ipv4.ip_forward=1
# Controls source route verification
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.all.rp_filter=0
```
并且设置 iptables 规则：
`iptables -t nat -A POSTROUTING -o ens37 -j MASQUERADE`

有时候起的 Instance VNC 连接不上，查看控制节点 `/var/log/nova/novncproxy.log` 发现似乎是被阻塞了，可以修改允许 Nova 节点 VNC 流量以放行（每次重启后失效）：
`iptables -I INPUT -p tcp -m multiport --ports 5900:6100 -m comment --comment "vnc ports" -j ACCEPT`

有时候，起来的 Instance 卡在 GRUB 上，比如 VMWare 虚拟环境下的 cirros，在 `Compute > Images > Edit Image > Metadata` 修改其 `disk_bus=ide` 和 `vif_model=e1000` 即可。参见此处（如果 cirros 还是起不来，重新创建实例即可）。或者使用命令：
```
cd /var/lib/glance/images/
ll #查找到镜像 id：9f35c24a-62c1-48b4-869d-5c15bbac2de8
openstack image set --property hw_disk_bus=ide --property hw_vif_model=e1000 9f35c24a-62c1-48b4-869d-5c15bbac2de8
```

Cirros 应该是 Debain 系 Linux，其可以一次性的写入配置文件使用静态 IP，只用在 `/etc/network/interface` 填入如下的静态地址，而不是 dhcp 即可：
```
auto eth0
iface eth0 inet static
	address 172.16.1.6
	netmask 255.255.255.0
	gateway 172.16.1.1
```

# Neutron 配置概述
Neutron 的结构较为复杂，部署灵活，配置文件众多，大致关系如下所示，其中位于 Controller 的 ① neutron.conf 负责为 Neutron Server & Plugin 服务提供联系其他服务的密码（比如 nova），联系数据库和 MQ 的凭证，以及提供自身服务的 url。类似的还有 Compute 节点的 nova.conf 为 nova-compute 提供服务。L2 Agent 和 L3 Agent 可位于网络节点 or 控制节点，其使用主 neutron.conf 中的 MQ 凭证通过 MQ 和 Neutron Server 进行通信，此外有自己独立的 xxx_agent.ini 配置文件，用于具体干活，比如 interface_driver 直接绑定网卡，xxx_range 等。对于 L3 Agent，一般在 Network/Controller 节点，对于 L2 Agent，一般 Network/Controller 和 Compute 节点都有一份，当然配置文件也都要各有一份。注意，从 Agent 独立出的 ml2_conf.ini 这个 Plugin 配置文件仅有 Controller 节点的 Neutron Server & Plugins 有，在 Network 和 Compute 节点虽然包会安装，但是一般不配置（或者压根没有），这里定义的内容会通过 Neutron Server 通过 MQ 下发给各 Agent，因此各 Agent 只需要 ① 总的 neutron.conf 以连接 MQ，③/④ 具体的 Agent 实现以执行 MQ 下发的来自 Neutron Server & Plugin 的 ② 配置即可。

![](https://static2.mazhangjing.com/20211016/020e_m1.png)

ML2  Plugin 配置：`/etc/neutron/plugins/ml2/ml2_conf.ini` 中保存着 ML2 Plugin 的配置。

![](https://static2.mazhangjing.com/20211016/c419_m2.jpg)

L2 Agent 配置：`/etc/neutron/plugins/ml2/openvswitch_agent.ini` or `linuxbridge_agent.ini` 中保存 L2 Agent 的配置，其在每个网络和计算节点都有此配置。此外，SRIOV Nic Switch 参见`sriov_agent.ini`，MacVTap 参见`macvtap_agent.ini`。

L3 Agent 配置：`/etc/neutron/l3_agent.ini` 中保存 L3 Agent 的配置，其仅存在于网络节点。`/etc/neutron/dhcp_agent.ini` 保存 DHCP Agent 的配置，仅存在网络节点。`/etc/neutron/metadata_agent.ini | /etc/neutron/metering_agent.ini` 分别代表了元数据和计量配置，仅存在网络节点。注意，ovs 和 linuxbridge 支持 l3 agent，dhcp agent，metadata agent，l3 metering agent，而 sriov nic 和 macvtap 均不支持 L3 Agent。

刚安装好的各包的样本配置文件参见此处：[Neutron 配置文件](https://docs.openstack.org/neutron/rocky/configuration/)

# LinuxBridge - Local Network
所谓的 local network 指的是多个 VM 在不同 namespace 通过 veth pair 在 linuxBridge 上进行 tap 的连接，如下图所示。在同一个 local network 下的主机彼此可互访（demo 和 demo2），但多个 local network 之间不能通信，且不能访问外部网络。为实现租户 Local Network，需修改 `/etc/neutron/plugins/ml2/ml2_conf.ini` 的 type_drivers 确保有 local，tenant_network_types 确保只有 local。然后 `systemctl restart neutron-*` 使配置生效，并且通过 systemctl 或者 Horizon 确认服务都顺利启动。
	
![](https://static2.mazhangjing.com/20211016/a052_m3.jpg)

![](https://static2.mazhangjing.com/20211016/45c1_m4.jpg)

![](https://static2.mazhangjing.com/20211016/b406_m5.jpg)

其实现原理比较简单：当创建一个带 DHCP 的有一台 VM attach 的网络时，在 Controller/Network 节点就会创建了一个 brq-58 网桥，然后 tap-c7 被接入此网桥（brctl show）。这里的 tap-c7 是 dhcp Agent 创建的端口（DHCP 工作原理见下文，简而言之就是 DHCP 进程 dnsmasq 位于一个 namespace 中，通过 veth pair 连通这个 tap-c7 端口，插入网桥 brq-58 以提供 DHCP 服务）。在 Compute 节点，localNetwork 也创建了 brq-58 网桥，tap-15 被接入此网桥，这里的 tap 设备通过 veth pair 跨越 namespace 连接到 VM 的  fa:16::79:99 MAC 地址的 eth0 上（`brctl show; virsh list; virsh domiflist instance-xxxx`）。

此处创建的 Port、Subnet 和 Network 和实际实现存在对应关系：Dashboard 上显示的子网的 Port 默认 Name 就是这里的两个 tap 名称，即 DHCP 端口 tap-c7 和 VM 端口 tap-15，这两个端口都是位于 root namespace 中的，分别和位于某 ns 的 DHCP 服务器通过 veth pair 连通，位于某 ns 的 VM eth0 通过 veth pair 连通。在开发中可以据此查看 Linux 实际信息以确定 OpenStack 工作状态。

再创建一个 Instance，分配 IP 172.16.1.6。通过对 Cirros 设置静态 IP，可以看到这两台 VM 可以 Ping 通了，但是其并没有办法连接外网（可以为 LinuxBridge 绑定一块能连接外网的网卡）。此外还需要注意，不同 local 网络之间也不互通。

# LinuxBridge - Flat Network
Flat Network 和 Local Network 类似，区别是，Flat 网络可以将物理机的网卡绑定到 LinuxBridge 上，以实现网络联通。但问题是，每个 Flat Network（一个租户） 都需要一块物理网卡（注意，不能是 VLAN 逻辑接口，VLAN 模式对应 VLAN Network）。这里 Flat 的含义就是所有租户的 VM 都透过 veth pair 跨越 ns 连接到一个大的 Flat Network 的 LinuxBridge 上，然后这个 LinuxBridge 再绑定到一块网卡上。

![](https://static2.mazhangjing.com/20211016/0a57_m6.png)

其配置方式类似 Local Network，修改 `ml2_conf.ini` 文件，type_drivers 确保有 flat，设置 tenant_network_types 租户网路类型为 flat 即可。因为 flat 网络涉及 Provider Network - 物理网卡，因此要根据管理员网络和租户网络分别设置。在 `[ml2_type_flat]` 段落中，设定 flat_network 的 label（比如 default），此外还要将 label 对应到物理网卡名称，在 `/etc/neutron/plugins/ml2/linux_bridge_agent.ini` 设定 physical_interface_mappings 指定 label（比如 default）对应的物理网卡名称（注意计算节点和控制节点都要设置），这样租户就可以自动的根据配置在对应网卡上连通创建 Flat Network。

对于管理员而言，可以自定义 physical_network 物理网络名称，其同样的在 `linux_bridge_agent.ini` 中找到物理网络对应的网卡。Horizon 的操作如下：通过管理员 > 网络 > 网络 > 创建网络，物理网络填入的是 flat_network 的 label，其在 `linux_bridge_agent.ini` 中对应了一块网卡，之后在控制节点可以 `brctl show` 看到 ens37 网卡和一个虚拟的 tap-46（用于连通 DHCP dnsmasq） 被绑定到新创建的 brq-c7 上了。同样的，计算节点 ip a 显示这里创建了一个  br-c7 网桥，以及一个  fe:16:3e:44:86:13 MAC 地址的 tap-bb 端口（用于连通 VM eth0），此外 `brctl show` 看到 br-c7 网桥还绑定了这个 ens37 网卡。

# LinuxBridge - DHCP 服务
**配置**：DHCP Agent 位于网络节点，每次创建网络，都会创建一个 dnsmasq 轻量级 DNS 服务器进程，为此 network 的所有启用了 DHCP 的子网提供服务。DHCP Agent 的配置文件在 `/etc/neutron/dhcp_agent.ini` 下（这里面可选择 DHCP 服务实现以及 DHCP 服务的接口连接到网络，这里选择的是 LinuxBridge。
```
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

**进程**：可以看到两个子网被 DHCP Agent 启动的两个 dnsmasq 进程。这里面可以看到 pid-file，dhcp-hostsfile，addn_hosts 和 DHCP 配置文件 dhcp-optsfile，dhcp-range 等信息，大部分配置在 `/var/lib/neutron/dhcp` 下。dhcp-hostfile 定义了 host 的 IP 和 MAC 对应关系，数据来自 Neutron 的数据库，interface 定义了监听的 DHCP 请求接口。

![](https://static2.mazhangjing.com/20211016/92b8_m7.jpg)

**机制**：每个 dnsmasq 进程都位于独立的 namespace，名称为 `qdhcp-<network-id>`，其中物理 interface 位于根 ns，虚拟设备则一般位于某个 ns，Neutron 通过 veth pair 将 root ns 的 brq 网桥连接到一个 tap，然后将其和在某个 ns 中的另外一个 tap 通过 veth pair 相连，实现 dnsmasq 隔离的同时能够访问根 ns 的 brq Linux Bridge 网桥。

> 在 OpenStack 环境中，`ip netns list` 能显示出来藏在 ns 中的 dhcp server 和 router，固定命名 + ID。

当一个 VM 发出 DHCPDISCOVER 广播后，其消息在整个 flat_net 传播，经过 VM 的 eth0 ns 转换出 tap，到达 Controller 的 brq4b 网桥的 tap0e 后，后者被 dnsmasq 监听，dnsmasq 检查 host 文件，找到对应项，DHCPOFFER 将 IP、NETMASK 和地址租期发送给 VM，VM 发送 DHCPREQUEST 并接受，dnsmasq 发送 DHCPACK 结束过程。需要注意，在上述部署和服务的过程中，DHCP Agent 的日志在 `/var/log/neutron/dhcp-agent.log` 中，dnsmasq 的日志在 `/var/log/syslog` 中，如果遇到错误可结合日志查看（比如当关闭 Port 安全策略后 DHCP Agent 突然打印很多日志，表明此 Port 被阻塞了）。

# Linux Bridge - VLAN Network
VLAN 网络和 Flat 类似，区别在于，多个不同 VID 的 brq 可共享一个 eth，这些 brq 被分隔打上不同的 VLAN Tag 即可：eth1.100，eth1.101 等。需要注意，和物理网卡相连的物理交换机的端口必须设置为 Trunk 模式而不是 Access 模式，因为此 eth1 网卡需要传输多个打了不同 VLAN Tag 的数据，此 eth1 网卡无需配置 IP 地址，其走的是 L2。

![](https://static2.mazhangjing.com/20211016/8d92_m8.png)

VLAN 网络的配置需要修改 `/etc/neutron/plugins/ml2/ml2_conf.ini` 的 tenant_network_types 选项，指定租户可创建 vlan 网络（这里仅管租户网络创建，但是如果这里不选择 vlan，那么管理员创建 vlan 网络会报错），然后在 [ml2_vlan_type] 段落指定 network_vlan_ranges 为一个物理网络 label 对应的 vlan id 范围（比如 provider:1:1000），当然，因为 vlan 需要和物理网卡对应，因此在 linuxbridge_agent.ini 的 physical_interface_mapping 需要将网络名 label 和物理网卡对应起来（比如 provider:ens37）。

启动一个 VM 后，分配此网络，Nova 节点新建了一个桥 brq-b6，桥连接到了 ens37.200 VLAN 端口，VM 通过 tap-2e 接口跨越 ns 连接到此桥，即可从此 ens37.200 出网（`brctl show，virsh list; virsh domiflist instance-xxxxxx`）。再启动一个 VM，绑定到同样的网络 VLAN200，现在 brq 桥有会一个新的 tap-9b 连接到此 VM，上述是 172.16.200.0/32 的网络，再创建一个 172.16.100.0/32 的网络，VLAN ID 为 100，创建一个 VM，将其绑定到此网络上，现在在计算节点会有一个新的 br-9b 的桥，此桥上的 tap-f4 与 VM 相连，ens37.100 VLAN 端口与外网相连，参见上图。注意，虽然 OpenStack 说现在创建了网关：172.16.100.1，但实际上，ens37.100 或者 brq 都没有绑定这个网关地址，即网关是 ping 不通的!!!，只有将其绑定了路由后，网关才能 ping 通，因为网关被绑定在了路由 ns 对应的 qr tap 设备上（进入 `router ns ip a` 查看），参见下文。通过 bridge fdb show 可以看到来自 ens37 网卡或外部 DHCP 的 vlan 100、vlan 200 和 master 报文将会转发到对应的 brq 上实现基于 VLAN 的隔离和通信。

# LinuxBridge - Route
上述 VLAN100 和 VLAN200 网络是不通的，其可以在 Internal Network 通过物理路由器实现连通，此外 Neutron 也提供了 L3 Agent 实现路由功能，配置位于 `/etc/neutron/l3_agent.ini` 中，需要设置 interface_driver（对于 LinuxBridge 和 OVS 需要不同设置）：
```
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver #LinuxBridge
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver #OVS
```

在网络 > 路由中新建一个路由，然后将 VLAN100 和 VLAN101 加入接口，现在二者应该可以通信了。

![](https://static2.mazhangjing.com/20211016/33a1_m10.png)

拓扑结构如上图所示，这里的路由是虚拟的 L3（位于 Controller 中）Agent 创建的，其通过两个 Tap 连通了两个 brq 网桥，如下图所示。

![](https://static2.mazhangjing.com/20211016/dabe_m11.png)

这里的机制是，虚拟的 Router 在多个子网上创建了 tap（见下图），但这些 tap 并未分配 IP（`brctl show` 和 `ip a` 可以确认）。因为这里的 Router 是租户创建的，通过 Namespace 实现的隔离，所以依旧是通过 ns 的 veth pair 将两个 root namespace 的 tap 连接到 router，`ip netns list` 找到 router 的 ns 名称，然后 `ip netns exec ns-name-here <command>`，这里的 `<command>` 输出 `ip a` 和 `ip r` 就可以看到两个 brq 的网关地址，以及其静态路由条目：到不同网段从哪个 ns 内的 tap 端口走。总的来说，网关位于 ns 隔离的 Router 接口处，这也是为什么不创建 Router 不能 ping 网关的原因

路由的另一个作用就是访问外部网络，首先需要在配置中为 Plugin 和 Agent 指定外部网络 label 名称和对应的网卡。如果外部网络是 flat，在 `/etc/neutron/plugins/ml2/ml2_conf.ini` 中 flat 段落中配置 `flat_network = external` 将外部网络命名为 external，在 `linuxbridge_agent.ini` 中配置 physical_interface_mappings 添加类似 external:ens33 的配置（比如：provider:ens37,external:ens33）。如果外部网络是 vlan，那么在 `ml2_conf.ini` 的 vlan 段落配置 `network_vlan_range = provider:100:300,external`，然后在 `linuxbridge_agent.ini` 的 physical_interface_mappings 中同样添加指定 external:ens33 即可。配置完成后，重启网络服务使其生效。

之后在 Horizon 中创建外部网络：在 Admin -> Networks 菜单 Create Network 选项下创建一个 Provider Network Type 为 flat 的，Physical Network 指定物理网络名称（这里是 external）的，勾选了☑️ External Network 的网络。然后创建一个子网，比如分配 10.10.10.0/24 的地址，设定 10.10.10.1 为网关地址（通常需要询问管理员外部 subnet 网关地址，填写在此处），外网不需要 DHCP 服务，可不勾选，然后执行网络创建。可以看到，在 Network 节点新建了一个 br-53 网桥，其绑定了一个物理网卡 ens33。在前面创建的 Router 中，点选 Add Gateway，然后将此网络添加并设置为网关，可以看到，此 Router 多了一个新的 Interface，比如 10.10.10.2，brctl show 可以看到 br-53 网桥不仅有 ens33 网卡，还有这个 Router 的 10.10.10.2 的端口的外部 veth pair 端口 tap-03，通过找到此 Router 的 ns，`ip netns exec ns_name ip a` 查看到这个 10.10.10.2 的 ns 内部 qg-03 端口（此分配的外部网络地址同样位于 Router ns 内部接口上，如果为主机分配 float IP，则 float IP 也位于此处）。

![](https://static2.mazhangjing.com/20211016/7617_m9.png)

如上图所示，在 Compute 节点，数据包从 VM 发出，从 eth0 跨 ns 到 root ns 的 brq，经过 VLAN 封包到达 Controller 节点，解包进入 brq，从和 Router 相连的 tap 进入 ns5 的 Router，在这里的 qr 处（即内部网关 172.16.100.1）被 ip_forward 路由到 qg（10.10.0.2）到达外部网络，出 ns5 后，经过 tap 到达 brq 外部网关网桥，到达 ens33 然后抵达外部网关（10.10.0.1）。在 Route 的 qr -> qg 的时候进行了一次 SNAT 源地址转换，可先找到 router 的 ns 名：`ip netns list`，然后 `ip netns exec ns5 iptables -t nat -s` 查看其 SNAT 规则。在 ns5 中对 qr 和 qg 间进行 tcpdump 也能看到相关地址变化的情况。

为了实现外部访问内部，需要 Float IP，在控制面板中，可以从外部网络分配 Float IP 并将其绑定到指定 VM（比如将 10.10.0.3 绑定到 172.16.100.3。注意，选项是分配给此 VM，但实际其被配置在 Router 上，这里在 Linux 中，Floating IP 已经被分配到这里的 qg 上了，`ip netns list`，找到 router ns 名，`ip netns exec ns5 ip a` 然后 `iptables -t nat -s` 可以看到新添加的规则当外部到达内部，如果目的地是 10.10.0.3，则将目的地改为 172.16.100.3，其通过 qr 172.16.100.1 转发到 VM。数据出网，则将源地址 172.16.100.3 转换为 Floating IP 10.10.0.3 并发出）。

# LinuxBridge - VxLAN*
Virutal Extensible Local Area Network，VxLAN，指的是 VLAN 的可扩展协议，相比较 12bit 最大 4094 的 VLAN 区段，VxLAN 使用 24bit 标记，能容纳更多网段。VLAN 使用 STP 避免环路，而 VxLAN 数据包封装到 UDP 三层传输和转发。因为其采用隧道机制，因此 Top on Rack 交换机无需在 MAC 表记录过多虚拟机信息。总的来说，VxLAN 是一个大二层协议，即将 L2 建立在 L3 之上，将 L2 数据包封装到 UDP 中以扩展 L2 网段（IP + UDP）来支持大规模租户网络环境。VxLAN 定义了一个 MAC in UDP 的格式，如下所示：将 L2 包加上 VxLAN header，放到 UDP 和 IP 包中，在 L3 建立了 L2 隧道：

![](https://static2.mazhangjing.com/20211016/e570_m13.png)

这种封装使用 VXLAN tunnel endpoint VTEP 设备进行封装实现和解封操作，每个 VTEP 都有一个 IP Interface，即一个 IP 地址，VTEP 使用此 IP 封装 L2 Frame，然后使用此 IP Interface 传输封装后的 VxLAN 数据包。在外部的 Transport IP Network 中，两个 VTEP 基于 Outer IP header（即 VTEP 的 IP 地址）进行路由，在两端的 VTEP 端点进行传输。

![](https://static2.mazhangjing.com/20211016/f2e2_m12.png)

在上面的例子中，Host-A 向 Host-B 发送数据，在 VTEP-1 映射表查到 MAC-B 对应的 VTEP-2 的 IP（关于如何找到 Host-B 的 MAC-B 以及 MAC-B 所在的 VTEP-2 先不提），然后将 L2 报文作为数据封装 UDP、IP（填写 VTEP-2)，添加 Router-1 MAC 地址进行发送，经过 IP 网络传输到 VTEP-2 后，解包 UDP 和 VxLAN header，拿到发送给 Host-B 的 MAC-B 的原始数据，传送给 Host-B。

其中 VTEP 可由 带 VxLAN 内核模块（Linux Bridge）的 Linux 或者 OpenvSwitch 支持，或者硬件实现。对于 Linux Bridge 而言，每个连接到不同 VxLAN 区段的 Bridge 发送数据包给一个 VTEP 实现：UDP Socket（8472），后者接收到虚拟机发出的数据包后，封包作为多播 UDP 包从网卡发出。在另一侧的 VTEP 上，收到 VxLAN 包后，解包并根据 VxLAN ID 将其转给某个 VxLAN Interface，然后通过 Linux Bridge 转发给虚拟机。

在 `ml2_conf.ini` 设置 tenant_network_type 为 vxlan，type_driver 包含 vxlan，mechanism_drivers 包含 linuxbridge 和 l2population。在 [ml2_type_vxlan] 区段指定 vni_ranges 比如 1001:2000（针对租户网络，管理员网络可自由设置），在 [VXLAN] 区段指定 enable_vxlan 为 True，l2_population 为 True，设定 local_ip （VTEP IP）。VxLAN 隧道无需指定网卡，但需要配置 OVERLAY NETWORK IP（可以是 MANAGEMENT NETWORK IP）。

同样的步骤在控制台创建网络、子网，设置类型为 VxLAN，分配 VID，指定 IP 区段。在底层会发现创建了一个 brq-d4 的网桥，以及 vxlan-100 的 VxLAN 接口，一个 DHCP 服务器外部接口 tap-59（ip -d l show dev vxlan-100 可查看此接口详细配置，比如 VNI 和对应的网口）。创建两个 VM 并将其加入网络，在计算节点，同样的 brq-d4 桥包含了这两 VM 的 tap-cd，tap-aa 接口，vxlan-100 接口。多 VxLAN 的路由和浮动 IP 绑定和 VLAN 类似。

![](https://static2.mazhangjing.com/20211016/f2e2_m12.png)

## l2_population
在 VxLAN 中，如果有大量网络通过隧道互联，那么要想知道一个 VM MAC，需要向所有 VxLAN 网络发送 ARP 报文，L2 Population 在 VTEP 上提供了 Proxy ARP 功能，使得 VTEP 可以得知整个 VxLAN 网络的 VM 的 IP 和 MAC 对应信息，VM 和 VTEP 的对应关系。而这一数据来自于 Neutron，Neutron 保存了每个 port 的状态，port 则保存了 IP 和 MAC 的相关数据，每次启动 VM，Port 状态都会从 Down -> Active，Neutron 会通过 RPC 通知各节点 Neutron Agent 更新 VTEP 消息(基于 OVS 的话，OVSNeutronAgent 会监听 br-int 桥端口变化，并向 NeutronServer 发送消息，这种机制实现了 VTEP 后主机的发现和集中管理，方便实现 Proxy ARP 功能 - 根据 IP 反查 MAC 然后发送到指定 VTEP)。这个机制称之为 L2 Population，L2 Population 支持 VXLAN LinuxBridge，VXLAN/GRE OVS。相比较 VXLAN BGP EVPN 的方案，L2 Population 在大量 MAC 地址（主机）的情况下有性能问题 —— 就好像使用 OVS 而非硬件交换机一样，但 L2 Population 的优点是集中管理，可定制程度高，配置简单。

在 `/etc/neutron/plugins/ml2/ml2_conf.ini` 中设置启用 `mechanism_drivers = linuxbridge,l2population`，在 `[VXLAN]` 配置中配置 `l2_population = True`，之后可以看到 `ip -d link show dev vxlan-100` 会带有一个 proxy ageing 的代理。`bridge fdb show dev vxlan-100` 可以看到对于特定 VM Port MAC 信息，将会 Forward 到 VTEP 保存的 VM 的 Port 的 IP 上（VTEP MAC 和 IP 的对应关系）。

# LinuxBridge - Security Group & FWaaS
安全组基于 iptables 实现，如果仅有 default 安全组，那么强制所有 VM Instance 使用，默认的安全组是允许所有外出流量，禁止所有进入流量。iptables 规则应用在 Controller/Neutron 的 Port 上，比如 tap-xx，ingress 命名为 neutron-linuxbri-ixxx，egress 命名为 neutron-linuxbri-oxxx 的 Chain 中，比如 allow ping & ssh 的安全组会增加：
```
-A neutron-linuxbri-ixxx -p tcp -m tcp --dport 22 -j RETURN
-A neutron-linuxbri-ixxx -p icmp -j RETURN
```
上述两条规则。

对于 FaaS 而言，其交由 L3 Agent 负责实现，需要在 `/etc/neutron/fwaas_driver.ini` 中定义 driver（iptables 的驱动），enabled = True，然后在 /etc/neutron/neutron.conf 的 service_plugins 中开启 FWaaS 服务（fwaas 插件）。在 Horizon 上通过 Project -> Network -> Firewalls 可打开 Firewall Policies 标签页面，创建一个 Prolicy，然后将其关联到 Router 上，在 Router NS 的 iptables 上就能够看到防火墙规则的变化。比如下面的一个规则，对所有 qr-* 开头接口发出的流量应用右侧规则：

![](https://static2.mazhangjing.com/20211016/1665_m15.jpg)

比如一条规则 iv4e..601 定义了对于 INVALID 数据包 DROP，RELATED 或 ESTABLISHED 数据包 ACCEPT：

![](https://static2.mazhangjing.com/20211016/dddc_m16.png)

而对于正常传输的数据，则执行最下面的 fwaas-defau，全部丢弃。

![](https://static2.mazhangjing.com/20211016/7d4d_m17.png)

在控制界面 Add Rule，选择协议、来源和目的 IP 以及行为，添加到 Prolicy 后，可以看到添加了两条：

![](https://static2.mazhangjing.com/20211016/2788_m18.png)

# OVS - 替换
> OVS 的替换本质就是将 neutron-linuxbridge-agent 替换为 neutron-openvswitch-agent，但实际稍微复杂一些，比如需要停止 neutron-linuxbridge-agent 服务，删除和重新配置 neutron 数据库，安装 neutron-openvswitch-agent 包，修改 `/etc/neutron/neutron.conf` 配置，添加 `[oslo_messaging_rabbit]`，注释掉 `[oslo_concurrency]` 的 lock_path，上述修改 ml2_conf 的 mechanism_drivers（保留 l2population），对于 `/etc/nova/nova.conf` 增加一些选项，配置并修改 /ml2/openvswitch_agent.ini 的 OpenvSwitch 相关配置：安全组、防火墙、L2_pop、OVS Local IP 和 bridge_mappings 等选项，修改 `/etc/neutron/l3_agent.ini` 和 `/dhcp_agent.ini` 的 interface_driver 为 `neutron.agent.linux.interface.OVSInterfaceDriver`，然后填充数据库并重启相关服务，对于计算节点修改 openvswitch_agent.ini 对应配置，重启计算服务和 OVS 即可。

> OVS 替换的具体操作参见 OpenStack Rocky 安装的 Nova 和 Neutron 章节。

OVS 的模型和 LinuxBridge 略微不同，其会创建 br-ex/br-tun 和 br-int 网桥，被用于非隧道外网、隧道和集成服务（集成服务之所以叫集成，是因为不仅各个 VM，包括 DHCP、Router 等服务都是在此处接驳）通信，ovs-vsctl show 现在替代 brctl show 查看网桥状态。计算节点有 br-int 和 br-tun，没有 br-ext。早期的 OVS 不支持端口安全组，因此通常配合 LinuxBridge 使用（现在则是为了兼容性继续使用这个模型），VM 通过 tap interface（tapxxx）attach 到 Linux Bridge （qbrxxx）上，经过 veth pair（qvbxxx 和 qvoxxx）attach 到 OVS Integration Bridge（br-int），经过 OVS patch ports（int-br-ethX 和 phy-br-ethX）到达 OVS provider bridge（br-ethX）（如果使用 VxLAN 等隧道技术，则 br-ethX 会替换为 OVS tunnel bridge（br-tun）），然后到达物理 Interface（ethX）。

## Local Network
一个 Local Network 会在 br-int 上创建一个 tap 用于连接 DHCP 设备，如果在 Controller 上起一个 VM，则此 VM 会创建对应的 LinuxBridge（qbr），然后此桥在一端通过 tap 跨 ns 连到 VM 的 eth0，另一端通过 veth pair 连到 br-int，veth pair 的 LinuxBridge 端 tap 通常叫做 qvb，br-int 端叫做 qvo。

![](https://static2.mazhangjing.com/20211016/eaca_m19.png)

如果起多个 VM，那么和 LinuxBridge 创建 tap 挂在 qbr 不同，这里会创建多个 LinuxBridge，将其通过 qvb 和 qvo 挂在到同一个 br-int 上。如果起多个 Local Network，其也会挂在同一个 br-int 上，和多个 VM 不同的是会有新的 DHCP 进程加入进来。正所谓 br-integration 名字中的整合的含义。

这里绿色和红色标识的两个 LocalNetwork 并不能互相通信，因为 `ovs-vsctl show` 可以看到 br-int 为这三台 VM 相连的 qbr 的 qvo 创建了不同的 tag，前两个是 1，后一个是 2，这意味着其属于不同的 VLAN，是互相隔离的。这里的 “VLAN" 只用于隔离网桥 Port，不是物理网络的 VLAN。

## Flat Network
Flat Network 在 ml2_conf.ini 中设置 tenant_network_types 为 flat 即可，指定 ml2_type_flat 的 flat_networks 标签，将其对应到 `[ovs]` 下的 bridge_mappings 物理网卡。

`ovs-vsctl show` 可以看到多了一个 br-ethX 网桥，此网桥有一个 phy-br-ethX 接口，br-int 增加了一个 int-br-ethX，两者通过 Patch Port 联通（区别于 veth pair，Patch Port 只能在 OVS 内部使用，Linux Bridge 和 OVS 连接不能使用，LinuxBridge 和 LinuxBridge 连接不能使用，Patch Port 的性能更好，可在配置中修改此行为）。这里的 br-ethX 网桥联通了物理端口 ethX。在 Controller 创建 VM 后（下图左半部分），其拓扑结构如下所示，先 tap 联通 qbr，qbr 通过 qvb 和 qvo 联通 br-int，后者通过 int-br-ethX 和 phy-br-ethX 联通 br-ethX，后者联通 ethX 物理端口。如果在 Compute 节点继续创建 VM，那么后者也会起 qbr、qvb、qvo、br-int、int-br、phy-br 和 br-eth 并联通到 ethX 上。

![](https://static2.mazhangjing.com/20211016/abeb_m21.png)

## VLAN Network
在 `ml2_conf.ini` 配置 tenant_network_types 为 vlan，指定 network_vlan_ranges 和 bridge_mappings（OVS 配置文件和标签下），注意，这里指定在 bridge_mappings 的，比如 br-eth1，是 OVS 网桥，而不是一个网卡，需要我们通过 ovs-ovctl 创建 br-eth1，并将 eth1 绑定到其上。这个网卡是 PROVIDER NETWORK，无需 IP 绑定，但必须对应物理交换机口是 TRUNK 模式，因为要出去 VLAN ID 的报文。

```
ovs-vsctl add-br br-eth1
ovs-vsctl add-port br-eth1 eth1
```

同样的在 Controller 和 Compute 都创建一个 VM 属于 VLAN100（控制面板创建第一个网络），其结构大致和 Flat Network 类似，vm 通过 tap 联通 qbr，qbr 通过 qvb，qvo 联通 br-int，后者通过 int-br-ethX 和 phy-br-ethX 联通 br-ethX。在 Compute 起另外一个 VM，将其分配到 VLAN101 区段（控制面板创建第二个网络），可以看到位于 Controller 的 VM1 和 Compute 的 VM2 可通，与 VM3 均不通。

![](https://static2.mazhangjing.com/20211016/cf78_m22.png)

OpenVSwitch 的 VLAN 实现其实不是通过 VLAN 逻辑端口，而是通过 Flow Rule 流规则对 br-int 的数据进行转发，`ovs-ofctl dump-flow` 可以查看 flow rule，其中每条 rule 定义了 priority 优先级（越高越好），in_port inbound 端口编号，`ovs-ofctl show <bridge>` 可查看每个 Port 的端口编号，dl_vlan 数据包原始的 VLAN ID，actions 数据包操作，比如：

![](https://static2.mazhangjing.com/20211016/00b6_m24.png)

上述含义为从 br-eth1 的 phy-br-eth1（port 2）进入的包，如果 vlan id 为 1，则将其改为 100，此条目优先级为 4。如果 vlan id 为 5，则将其改为 101。此外，对于从 br-eth1 出的流量，如果来自 port1（int-br-eth1），那么将内部标签 100 和 101 转换为 1 和 5 再出去，就 br-ethX 完成了内外标签的转换，之所以有内外标签转换的原因参考这里。

## Routing
`l3_agent.ini` 必须配置 interface_driver 为 `neutron.agent.linux.interface.OVSInterfaceDriver`，此外需要制定通往外网的网桥，其定义在 external_network_bridge 中，比如 br-ex。步骤和 LinuxBridge 创建 Router 的过程类似，创建并且绑定两个 VLAN 网络网关，Route 依旧位于 namespace 中，指定 Gateway IP 的 tap 在 ns 中，跨越 veth pair 连接到 root ns 的两个 qr 到 br-int（这里区别于 LinuxBridge veth pair 联通出的 tap 连接在不同网络的 qbr 上）。

![](https://static2.mazhangjing.com/20211016/adc6_m25.png)

对于访问外网而言，和 LinuxBridge Route 访问外网类似，如果外网是 flat，在 `ml2_conf.ini` 指定 flat_network 为 external 标签，然后在 bridge_mappings 后添加 external:br-exN 对应的网桥，对于外网是 vlan 而言，在 ml2_conf.ini 指定 network_vlan_ranges 添加 external，然后指定 bridge_mappings 添加 external:br-exN，之后重启 neutron 相关服务，提前使用 ovs-vsctl 创建 br-exN 将 ethN 添加到此 br-exN 网桥。

在控制面板创建一个网络，标记 √ External Network，指定网络类型和物理网络标签（external），关闭 DHCP，填写网关，然后在路由中将这个网络添加进去即可。其拓扑如下所示，和 LinuxBridge 访问外网创建了一个 qbr 类似，这里创建了一个 br-ex（我们自己创建的，在配置中指定给了 external 外部网络），被连接到 eth2 上，然后通过 qg 端口联通到 route 上，完成了外网的联通，自然地，两个内部网络的网关位于 route 的 qr 相连的内部 ns tap 上（比如 172.16.100.1 和 172.16.101.1)，且在与 qg 相连的 ns 内部端口（10.10.10.2）进行了 SNAT 地址转换，到达了外部网关 10.10.10.1。Floating IP 和 LinuxBridge 类似，被添加到和 qg 相连的 route ns 内部的端口上，通过 route ns 内部的 iptables NAT 规则进行转换。

![](https://static2.mazhangjing.com/20211016/076d_m26.png)
	
## VxLAN
在 `ml2_conf.ini` 中配置 tenant_network_types 为 vxlan，设定 vni_ranges 并在 `[agent]` 中配置 `tunnel_types = vxlan，l2_population = True` 开启通道，在 `[ovs]` 指定 `tunnel_bridge = br-tun`，local_ip 为 VTEP 的 IP 地址即可。和 VLAN 类似，不过这里的 br-ethX 被替换为了 br-tun，且其通过 path-int 和 patch-tun 端口和 br-int 相连。分别在两个节点创建两个 VM，将其加入 VxLAN 网络，通过 `ovs-vsctl show` 查看其结构如下所示，eth0 通过 tap 联通 qbr，qbr 通过 qvb 和 qvo 联通 br-int，br-int 通过 patch-tun 和 patch-int 联通 br-tun，后者联通 ethX（注意，这里无需指定 `ovs-vsctl add-port br-tun ens37` 以让 VxLAN 能够通过 OVERLAY NETWORK 通信，不像 VLAN 在 OVS 上手动绑定 ens37 通向外网的 PROVIDER NETWORK，Neutron 配置的 ovs;ip 会自动从对应的接口出，在测试环境可以是 MANAGEMENT NETWORK，对比 VLAN 的图可以看出，VxLAN 这里的 eth1 有 IP，且 `ovs-vsctl show` 中 br-tun 没绑定网卡，而 VLAN 的网卡则被 br-ethX 手动进行了绑定），并且通过 Tunnel 打洞实现 VxLAN 的互通。

![](https://static2.mazhangjing.com/20211016/35aa_m27.png)

这里需要注意 br-tun 的 vxlan-xxx Port，这是 VxLAN 的隧道端点，其指定了本地和远端的 VTEP IP，实现包的封装，不论起多少 VxLAN，都会创建对应多个 qbr，并联通到 br-int 上，但只有一个 vxlan-xxx 在 br-tun 上进行隧道传输（因为其本地和远端地址都是一样的）。OVS 通过 Flow Rule 实现 VxLAN，br-int 被当做一个 L2 交换机，其根据 VLAN 和 MAC 地址进行转发：

![](https://static2.mazhangjing.com/20211016/fdc3_m28.png)

br-tun 的 Flow Rule 比较复杂，大致而言，如果是内部的包，对于多播，匹配 VLAN 号则去 VLAN 号 Tunnel 发，对于单播，根据规则 Tunnel 发，外部进来的，Tunnel 号匹配，则添加 VLAN 号，传送给 br-int（内外 VID 转换）。在这个过程中，进行了自学习（VLAN 号/tag - 目标 MAC 地址），以加快转换和向对应 br-int 端口转发的效率。VxLAN 的 Route 和 Floating IP 和 VLAN 类似，不再赘述。