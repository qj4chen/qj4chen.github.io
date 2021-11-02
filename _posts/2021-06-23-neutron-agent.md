---
layout: post
title: OpenStack Neutron Agent 源码解读
categories:
- OpenStack
- Neutron
---

> 本文介绍了 Neutron Agent（包括 OpenVSwitch Agent 和 L3 Agent） 的启动过程和作用原理，基于 OpenStack Rocky 版本。

Neutron Plugins 通过 RPC 调用 Neutron Agent，后者部署在各个 Compute/Network 节点上，用来配置业务对象，被配置的对象称之为网元，其包含**物理网络功能 Physical Network Function, PNF** 和**虚拟网络功能 Virtual Network Function, VNF**。VNF 一般指的在 X86 和 Linux 之上软件实现的网络功能，比如 L2 交换和 L3 路由等。每一个 Neutron Agent 都是一个进程，也是几个 RPC Consumer，每一个种类的 Neutron VNF 都有与之对应的 Neutron Agent，Agent 用于根据 Plugin 的指令对其进行操控：L2 Agent（OVS Agent or LinuxBridge Agent） 和 L3 Agent（Router Agent、DHCP Agent 等） 分别用于控制网络的 L2 和 L3 功能。

![](https://static2.mazhangjing.com/20211016/2339_a1.png)

# OVS Agent
OVS Agent 通过 RPC 和控制节点进行交互，其作用位点大概如下图所示，通过 CLI 下发流表对 OpenVSwitch 进行配置。其主要干了两件事情：Bridge 的创建、内外 VID 的转换。代码入口是 `plugin/ml2/openvswitch/agent/main`，其调用了 `../openflow/ovs_ofctl/main`，而后者又调用了 `../ovs_neutron_agent.py` 在这里实例化了 OVSNeutronAgent 并 RPC 通知初始化消息，之后进入 `daemon_loop` 阶段。	

![](https://static2.mazhangjing.com/20211016/496c_a2.png)

## Bridge 创建和配置
### Bridge 结构和功能
在 `openflow/ovs_ofctl/main` 中有基于 OVSAgentBridge 的三种不同 OVS 桥子类，其集成关系如下图所示：

![](https://static2.mazhangjing.com/20211016/88a9_a3.jpg)
	
BaseOVS 定义了桥相关操作的 API，比如 `add_bridge`，`delete_bridge`，`bridge_exists`，`get_bridges`，这些操作大部分都是调用 ovsdb 或委托给子类执行的。所谓的 ovsdb，来自于配置文件 `cfg.CONF.OVS.ovsdb_interface`，其位于 `etc/neutron/plugins/ml2/openvswitch_agent.ini` 中，默认的值是 vsctl，具体实现是 OvsdbVsctl 类，此外可选 native 值，具体实现是 OvsdbIdl 类。OvsdbVsctl 主要用于生成 ovs-vsctl 命令行，然后提供 execute 方法让 BaseOVS 可调用以下发交换机执行。

![](https://static2.mazhangjing.com/20211016/f3e5_a4.jpg)
	
OVSBridge 用于配置 SDN Controller（`set_controller`），配置 Bridge/Port（`create_port`，`add_port`），配置 OpenFlow 流表（`remove_all_flows`，`run_ofctl`），前两个基本就是 ovs-vsctl，第三个是 ovs-ofctl 命令行。
OVSAgentBridge 仅包含两个函数：`setup_controllers` 会调用 `del_controller` 函数以执行 ovs-vsctl 命令并删除控制器（选择 ovs-ofctl 模块而非 native 后，就不需要控制器，因此这里可以删除），`drop_port `会调用 `install_drop` 函数，后者在 OVSAgentBridge 的 OpenFlowSwitchMixin 父类中的 add_flow 函数增加了一个流表项，即所有从 Bridge（OVSAgentBridge br_name）的端口（drop_port 的参数 in_port）进入的流都丢弃。	

![](https://static2.mazhangjing.com/20211016/e4dd_a5.png)

OVSIntegrationBridge 基于上述父类提供对 br-int 桥的处理，基本都是流表操作，比如默认流表的安装：`setup_default_table`，`setup_canary_table`，`check_canary_table`，比如内层 VLAN 配置：`provision_local_vlan`，`reclaim_local_vlan`，DVR 相关配置：`_dvr_to_src_mac_table_id`，`install_dvr_to_src_mac`，ARP 反欺骗配置：`install_icmpv6_na_spoofing_protection`,`install_arp_spoofing_protection`, `delete_arp_spoofing_protection`, `delete_arp_spoofing_allow_rules` 等。

OVSPhysicalBridge 基于上述父类提供对 br-phys 非隧道桥，即 br-ethx 和 br-ex 桥的处理，全部是流表相关操作，比如默认流表安装、内层 VLAN 配置、DVR 相关设置等。

OVSTunnelBridge 基于上述父类提供对 br-phys 隧道桥的处理，全部是流表相关操作，比如默认流表安装、内层 VLAN 配置、DVR 相关设置、flood 相关配置、UNICAST 相关配置、ARP 代答相关配置、TUNNEL PORT 相关配置等。

### Bridge 的初始化和配置
上面说了，OVSNeutronAgent 是 Neutron OVS Agent 的入口类，在它的 `__init__` 中创建了 br-int：`setup_integration_br`（通过 ovs-vsctl 创建一个 br-int Bridge，`set_secure_mode` 设置其安全模式，`delete_port` 删除 peer port，`drop_flows_on_start` 删除所有流表如果有配置的话，`setup_default_table` 设置默认流表），br_phys：`setup_physical_bridges`（针对每个 bridge_mappings 如果尚未创建好对应 bridge clazz 则通过 OVSPhysicalBridge 创建 —— 注意，不去调用 ovs-vsctl 创建实际的桥，这些连接到物理网卡的桥都是事先创建好的，并设置安全模式、控制器信息，初始化流表等。此外，设置 br-int 和 br-phys 的对接，如果设置 `use_veth_interconnection` 则使用 veth 模式对接，反之使用 patch port 模式对接），br-tun：`setup_tunnel_br` 和 `setup_tunnel_br_flows`（创建 OVSTunnelBridge 实例，调用 ovs-vsctl 创建 br-tun，设置控制器信息，通过在 `int_br` 上增加 `patch_port` 来对接 br-int 和 br-tun，删除所有流表并设置默认流表等），相关的步骤在这些方法中可详细查看。

此外，因为 OVSNeutronAgent 是实际负责 OVS 操作的类，在 `__init__` 中存在很多配置项映射，比如 `agent;veth_mtu，agent;tunnel_types`（隧道类型，比如 vxlan），`agent;l2_population，agent;enable_distributed_routing，ovs;integration_bridge` 映射创建的 int_br（eg. integration_bridge=br-int 会创建名字为 br-int 的网桥），`ovs;bridge_mappings` 映射创建的 `physical_bridges`（eg. `bridge_mappings=default:br-eth1` 会创建名字为 br-eth1 关联物理网络 default 的网桥），`ovs;local_ip` 即 VTEP 外层隧道 IP，`agent;vxlan_udp_port`，如果启用了 DVR，这里还实例化了 OVSDVRNeutronAgent，如果启用了安全组，这里实例化了 SecurityGroupAgentRpc。

## 内外 VID 转换
租户隔离中需要将内层管理 VID 和外层租户 VID 进行适时的转换，其发生在 br_int 和 br_tun/ethx 中，如下图所示。OVS 将内外 VID 映射关系保存在 OVS Bridge 类中，比如 br-int 的端口表 other_config 字段。	

![](https://static2.mazhangjing.com/20211016/4d27_a6.png)

这里的含义是：如果 Network ID 为 `12345..9` 时，网络类型为 VLAN，物理网络为 `physical_netwrok` 字段类型时，那么内层 ID 为 vlan 字段值，外层 ID 为 `segmentation_id` 值（Neutron Plugin 分配）。

![](https://static2.mazhangjing.com/20211016/67af_a7.png)

内层 VID 是由 OVS Agent 分配的，OVSNeutronAgent 通过 `available_local_vlans` 存储有效的 Local VLAN ID 字段，`MIN_VLAN_TAG - MAX_VLAN_TAG [1-4094]` 之间都是有效的。OVSNeutronAgent 使用 `_local_vlan_hints` 这个 map 存储 Network ID 和内层 VID 关系：`_local_vlan_hints[networkID] = localVlanID`，当给 Network 分配一个 Local VLAN ID 后，将关系记录在其中，下次再分配则直接查找即可。

一般而言，数据报进入 Host，在 br-ethx/br-tun F 处将外层 VID 转换为内层 VID，在数据报出 Host，则在 br-int 上打上内层 VID，在 br-ethx/br-tun F 处将内层 VID 转换为外层 VID。`provision_local_vlan` 函数用于内外 VID 映射和部署，其根据网络类型，比如 tunnel 隧道，则调用 `tun_br.provision_local_vlan` 处理，如果是 vlan，则 `_local_vlan_for_vlan` 处理（VLAN 内外转换位点和 VxLAN 不同，从内到外 VID 转换在 E 处发生，从外到内 VID 转换在 F 处发生），如果是 flat，则 `_local_vlan_for_flat` 处理（类似于 VLAN，不过外层 VID 不存在 - 对应 Untag 报文），不管哪种类型的网络，桥的标签转换最终总是通过调用 ovs-ifctl add-flows（`add_flow` 函数)配置流表进行的。

## OVS Agent 启动分析
OVS Agent 在 `neutron.egg-info/entry_points.txt` 中的 `console_scripts` 小节的 `neutron-openvswitch-agent` 字段定义了 main 方法，其根据 `etc/neutron/plugins/ml2/openvswitch_agent.ini` 的 of_interface（默认是 ovs-ofctl）找到 egg-info 对应的主类：`openflow.ovs_ofctl.main`，后者实例化 OVSNeutronAgent 实例对象，进入 `daemon_loop` 中，此进程的名字是 `neutron-openvswitch-agent`。初始化 Bridge 实例发生在其 `__init__` 函数中（见上文描述，基本就是加载配置，创建了几个桥）。相关的代码在 `neutron/plugin/ml2/openvswitch/agent` 文件夹中。OVSNeutronAgent 是一个进程，也是一个 RPC Consumer（其 `__init__` 方法中调用了 `setup_rpc` 并 `create_consumers` 创建了消费者)。

RPC 调用接口中没有 `port_add`，这是因为创建 VM 的流程中 Nova 调用 Neutron Server RESTful API 通过 ML2 Plugin 去 create port 后，RPC 告知 OVS Agent `port_update` 接口。Nova 自己会在 br-int 上创建端口，之后将此端口通过 ovs-vsctl add-port 绑定到 br-int 上，因此 OVS Agent 只会看到 br-int 上多了一个端口，其仅负责 update 它（加入端口字段，配置 br-int 和 VM 对接的端口 tag，配置 br-int/br-tun/br-ethx 流表规则使得可正确进行内外 VID 转换）。不仅如此，br-int 上所有的网元都不是 OVS Agent 负责创建的，比如 DHCP Agent Port - DHCP Agent 负责，Router Port - L3 Agent 负责。

![](https://static2.mazhangjing.com/20211016/13f1_a8.jpg)
		
此外，RPC 也没有 `tunnel_add` 的调用，之所以没有 tunnel 的创建，原因是隧道本质上并不存在，只是对 VxLAN 报文的 UDP 封装，在 `openvswitch_agent.ini` 中的 `local_ip` 和 `tunnel_types` 配置后，`daemon_loop` 函数会将自身 Tunnel 信息通过 RPC 通知 Neutron Server，在这个时候进行的 Tunnel 隧道配置，如下图所示，OVS Agent 1 启动后告知自己 tunnel 信息，neutron server 返回并告知 OVS Agent 关于 OVS Agent 2 和 3 的 Tunnel 信息，且通知 Agent 2 和 3 关于 Agent 1 Tunnel 的信息，通过这个过程实现 VTEP 信息共享。当每个 OVS Agent 收到此 `tunnel_update` 消息后，会在 `_setup_tunnel_port` 中调用 `br.add_tunnel_port` 函数以及 `br.setup_tunnel_port` 函数来为 br-tun 添加一个 Tunnel Port 且设置此 Tunnel 的流表。

![](https://static2.mazhangjing.com/20211016/efcf_a9.jpg)
	
OVS Agent 的 main 函数除了创建 OVSNeutronAgent 外，就执行了一行函数：`daemon_loop`，此函数的核心在 `rpc_loop` 这个死循环中，其通过 `pooling_Manager` 对 br_tun 进行轮询，处理 br_int 桥上的 Port 变化：处理方法通过两个函数完成：`treat_device_added_or_updated` 函数更新流表以映射内外 VID 转换规则，bind_device 当端口当前配置的 VLAN ID 和理论上端口应该具备的内层 VID 不一致时重新配置端口 Tag：删除原来的流表规则并配置此理论上端口应该具备的 VID 对应的流表规则。

![](https://static2.mazhangjing.com/20211016/0f22_a10.png)

总的来说，作为进程，OVS Agent 部署在 Neutron 每一个计算和网络节点上，一共有三种 OVS Bridge：br-int，br-tun，br-phys（对应 neutron 的 br-ethx 和 br-ex），OVS Agent 通过调用 ovs-vsctl 和 ovs-ofctl 命令行对这些网桥进行操纵。具体来说，OVS Agent 在进程启动时，创建了三类网桥，并初始化它们（`__init__`)，在 `__init__` 函数中，也完成了 br-int 和 br-ethx 的接口对接，br-int 和 br-tun 的接口对接，而 br-int 和 VM 接口对接则是通过 Nova 在 boot VM 时进行的。br-ex 和其他网元，比如 Router 对接接口，是通过这些 L3 Agent 完成的。如果是隧道协议，Neutron 创建 Host 隧道也是在 OVS Agent 启动后，RPC 通报自身 Tunnel Type 和 Tunnel IP 后完成的，Neutron Server 接收后通过 RPC 广播给其他 OVS Agent（tunnel_update），OVS Agent 接到后，比较自身隧道信息，如果 br-tun 外接口吻合流表规则则使此端口具有隧道功能。对于 Nova 创建的 br-int，Nova 并未通知 OVS Agent，后者会轮询 br-int 上端口变化，以自动更新流表规则，完成内外 VID 转换（以及配置 br-int 和 VM 对接端口的 Tag）。

# L3 Agent
L3 Service 路由转发由 L3 Agent 负责配置，本质上，L3 Agent 创建了一个 namesapce，为每个 Router（Linux 本身）创建了隔离，然后将这些 Router 通过 br-int 和 L2 层互联。除了基本路由功能、SNAT 和 DNAT（Floating IP），Router 还支持 DVR，Firewall，HA，IPv6 特性。L3 Agent 是一个进程，一个 RPC Consumer，接收 Neutron Server 的 RPC 消息。	

![](https://static2.mazhangjing.com/20211016/8707_a11.png)

## OVSInterfaceDriver
OVSInterfaceDriver 初始化了 Router 和 OVS Bridge 接口，还通过 Linux CLI 初始化路由器端口（配置 IP 地址和路由表）。`plug` 函数用来创建 Router 和 OVS Bridge 对接的接口，`unplug` 则删除接口，`init_router_port` 初始化路由器端口，`plug_new` 被 plug 调用，当对接的接口不存在时执行创建，OVSInterfaceDriver 重载了此函数。

![](https://static2.mazhangjing.com/20211016/3c15_a12.png)

首先判断 Router 与 Bridge 对接的接口名称 `device_name` 存在，如果不存在则 `plug_new` 创建一个新接口，反之则设置 mtu 属性（如果有），`plug_new` 函数的核心过程如下所示：

![](https://static2.mazhangjing.com/20211016/19d3_a13.png)

而 `init_router_port` 接口则用来初始化路由器端口，包括配置端口 IP 地址、路由表等，其核心过程如下所示：

![](https://static2.mazhangjing.com/20211016/bb26_a14.png)

其中 `init_l3` 会重新配置端口上的 IP 地址，删除不需要的，增加需要的。`set_onlink_routes` 则用来在 `init_l3` 的基础上更新 Router 的路由信息。

## RouterInfo
RouterInfo 实际调用 OVSInterfaceDriver 完成路由操作，其 `process` 函数是核心，这里主要做了三件事：①处理 Router 内部接口（Neutron 通过 `add_router_interface` API 增加的接口）：`_process_internal_ports`，②处理 Router 外部网关：`process_external`，③更新 Router 路由表：`routers_updated`。此外，还要额外存储 ex_gw_port，Floating IP，enable_snat 信息。

`_process_internal_ports` 函数根据配置从 router 参数中获取内部接口，对比当前 Router 端口，构建需要新增的、删除的和更新的端口列表，然后调用 `internal_netwrok_added/remove/update` 等函数进行处理。以 `internal_netwrok_added` 函数为例，其调用 `_internal_network_added` 函数，通过 OVSInterfaceDriver 的 `plug` 连到 Router，通过其 `init_router_port` 来初始化端口（分配 IP 地址，配置路由表）。除了增加接口以外，另外两个函数对于需要删除的内部接口，需要修改的内部接口都进行了处理，最后刷新了路由表。	

![](https://static2.mazhangjing.com/20211016/79ff_a15.png)

对于 `process_external` 而言，①先从端口列表获取 `ex_gw_port` 外部网关端口 ②然后 `_process_external_gateway` 处理此外部网关端口：plugin 并设置 IP 和路由表 ③通过 `process_snat_dnat_for_fip` 处理浮动 IP 的 NAT 转换（本质就是 IPTABLES 下 SNAT/DNAT）④将 Floating IP 配置在 Router 外部网关端口（Port2）上：`get_external_device_interface` & `configure_fip_address`。
其中传入的配置文件大致如下所示：

![](https://static2.mazhangjing.com/20211016/6dd7_a16.png)

![](https://static2.mazhangjing.com/20211016/01e1_a17.png)

其中 ②`_process_external_gateway` 指的就是创建 Port2 这个端口，如果启用了 SNAT 则还需要进一步配置，首先这里要获取外部网关端口名字，创建此端口并配置默认路由：`external_gateway_added`（调用 `_external_gateway_added` 函数，此函数先在 `_plug_external_gateway` 中通过  OVSInterfaceDriver 实例的 plug 去创建外部网关端口 Port2，然后 `_get_external_gw_ips` 获取外部网关 IP - Port3 的 IP，然后通过 `init_router_port` 初始化路由端口，通过 `route.add_gateway` 默认增加静态路由：调用 ip route replace default via.. 给 Port2 设定 IP），之后 `_handle_router_snat_rules` 通过调用 iptables 增加 SNAT 规则，以对默认出网的私网地址进行 Dynamic NAT 转换。

其中 ③ `process_snat_dnat_for_fip` 会调用 `process_floating_ip_nat_rules`，后者 `get_floating_ips` 后，对于每个浮动 IP，都为 `iptables_manager.ipv4['nat'].add_rule` 添加规则，`floating_forward_rules` 调用 iptables 命令行进行处理，然后 `.apply()` 应用规则 SNAT/DNAT 规则。

最后对于 `routes_updated` 更新路由表项目，需要注意：通过外部网关端口的静态默认路由，在上述 `_plug_external_gateway` 中已经自动创建了，而 Router 内部的端口会直接成为直连路由，除此之外，还可以直接修改 Router 模型的 routes 字段，这就是“更新 Router 的路由表”，首先通过 `helpers.diff_list_of_dict` 比较现有路由表和用户传入的路由表，然后调用 `ip route replace/delete` 命令该删删，该加加。

## L3 Agent 启动流程
在 egg-info 的 `entry_points` 中定义了 `neutron-l3-agent` 的主类是 `neutron.cmd.eventlet.agents.l3:main`，其本质上启动了一个 `neutron.agent.l3.agent.L3NATAgentWithStateReport` 对象实例，然后创建了一个 RPC Consumer，通过 `_process_router_update`（定义在父类 L3NATAgent，WithStateReport 相比较 L3NATAgent 增加了周期性的向 Neutron Server 报告 L3 Agent 状态的功能） 对 Router 的变更信息（来自 Neutron Server 的 RPC 消息）进行整理、合并和下发，其主要利用了一个队列，通过多个协程从队列获取未过期消息然后调用对应业务方法（比如下图 `_process_added_router`），委托 RouterInfo 进行处理。

![](https://static2.mazhangjing.com/20211016/d750_a18.png)

这个分布式消息处理机制原理如下：L3 Agent 通过 Neutron Server 的 RPC 消息通知、自身向 Neutron Server 周期轮序或不定期查询来获取 Router 变更消息，然后对 Router 进行配置。因为可能多个协程同时收到 Router 变更消息，但 Router 配置却仅需一份，且是串行的，Neutron 通过队列和 ERP 实例，保证和一个 router_id 有关的 Router 变更消息全部由一个协程处理，保证同一个 Router 串行处理。而不同 router_id 的 Router 消息变更则交给不同协程处理，保证效率，如下所示：

![](https://static2.mazhangjing.com/20211016/484c_a19.png)

注意，RPC 的 Router 变更消息比轮询的优先级更高，其会具备较高处理优先级；同优先级的 Router 变更消息，早发生早处理，如果过期，则直接丢弃不再处理。为实现这个机制，L3 Agent 先 RouterUpdate 对 Router 消息变更进行了包装，通过 RouterProcessingQueue（PriorityQueue 对变更消息排序，基于优先级和消息到达时间）和 ExclusiveRouterProcessor（按照 RouterID 分为多个子队列，每个子队列按照 RPQ 排序结果）以保证上述机制。