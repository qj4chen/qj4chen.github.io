---
layout: post
title: OpenStack Neutron Plugin 源码解读
categories:
- OpenStack
- Neutron
---

> 本文介绍了 Neutron Plugin (包括 Core Plugin - ML2 和 Router Plugin) 的启动过程和作用原理，基于 OpenStack Rocky 版本。

# Neutron Core Plugins - ML2
Neutron 的核心插件 `Core Plugin` 与扩展插件 `Extension Plugin` 的功能类似：分配资源的逻辑信息，比如 Network 的 Segment ID，Subnet 的 IP 网段等，分配资源的物理信息，比如 Port 接口类型等，然后将资源信息存入数据库表中，通过 RPC 调用进行相应的配置。在 Havana 版本中，Neutron 通过 ML2 Plugins 实现了架构的统一，此插件分为类型驱动（Type Driver）和机制驱动（Mechanism Driver），前者和 Network 的类型密切相关，后者和厂商的实现机制密切相关。在运行时可加载机制驱动的不同实现。

![](https://static2.mazhangjing.com/20211016/cf8a_p1.png)

Ml2Plugin 有一个高达 20 多层的继承树，其中大部分是 Mixin，抽象的核心在于 `NeutronPluginBaseV2` 中，这里定义了增删改查的抽象接口，和数据库相关的实现在 `NeutronDbPluginV2` 中，最后在 Ml2Plugin 中实现了所有的接口。

![](https://static2.mazhangjing.com/20211016/7d44_p2.png)

![](https://static2.mazhangjing.com/20211016/0cd3_p3.png)

## Type Driver & Type Manager
其中类型驱动参见 `egg-info/entry_points`，在 `neutron.ml2.type_drivers` 字段对不同类型的网络定义了不同的 Python 类，比如 `flat = neturon.plugin.ml2.drive.type_flat:FlatTypeDriver` 等。这六种类型的网络（flat、geneve、gre、local、vlan、vxlan）代表了六种网络分片 Segment 技术，TypeDriver 的主要功能就是对 Segment ID 进行分配 —— 表现在 Network 模型的 `provider:network_type` 和 `provider:segmentation_id` 字段差异上。	

![](https://static2.mazhangjing.com/20211016/9847_p4.png)

在之前章节说过，`network.segments` 和 `network.provider extends attributes` 只能二选一，前者代表多 Segment 网络，后者代表单一的运营商网络，对于 Segments 模型而言，其表名为 `networksegments`，外键关联到 `networks` 表，属于多对一关系。除此之外，对于单运营商网络（使用 network provider extends attributes）还是多运营商网络（使用 network segments）在数据库层没有区分。

![](https://static2.mazhangjing.com/20211016/e82b_p5.jpg)
	
对于租户而言，其不能传入 `provider:xxx` 创建租户网络，而是需要通过配置范围：`ml2_conf.ini` 的 `tenant_network_types` 来选定使用哪种网络类型（可以指定多个，如果有多个则随机遍历）。对于管理员而言，其可以通过 `provider:network_type` 设定具体网络类型，或者使用 `external_network_type` 来决定网络类型（如果没有配置，则等同于 `tenant_network_types`）。`neutron/plugins/ml2/managers.py` 的 TypeManager 在初始化时加载了这些配置：

![](https://static2.mazhangjing.com/20211016/986a_p6.png)

对于 segmentID 而言，可在配置中指定不同网络类型使用的不同 SegmentID 范围，其会通过 TypeManager 的 `_allocate_segment` 函数委托 `_allocate_tenant_net_segment` 和 `_allocate_ext_net_segment` 进行分配，后者遍历对应的驱动，逐个尝试，之后通过多态进入驱动的 `allocate_tenant_segment `方法进行分派处理，这个分派过程只要有一个成功即可完成。

![](https://static2.mazhangjing.com/20211016/dbd9_p8.jpg)

比如 VlanTypeDriver，其核心在于 `allocate_partially_specified_segment`，这是 SegmentTypeDriver 的帮助方法，从数据库表 `self.model（ml2_vlan_allocations）`中取出可分配的 SegmentID 并随机选择一个进行分配并写入数据库。

![](https://static2.mazhangjing.com/20211016/0c56_p9.png)

而配置中的数据是何时写入数据库的呢？如果配置修改，数据库会发生什么变化？在 TypeManager 根据配置 `type_driver `初始化对应的 TypeDriver 时，调用了 `initialize` 方法，后者会读取配置中的 SegmentID 范围，然后尝试和数据库进行同步。比如，对于 VxLAN Type Driver 而言，其 `initialize` 方法从配置文件中读取了 `ml2_type_vxlan` 的 `vni_range` 配置，然后和数据库表 `ml2_vxlan_allocations` 数据比较（第一次为空），如果配置文件有而表没有，则插入，反之则删除。
	
![](https://static2.mazhangjing.com/20211016/f92c_p10.jpg)

## Mechanism Driver
对于机制驱动 Mechanism Driver，其公共父类主要用来体现 Neutron 自身实现机制，子类则体现不同厂商实现的，定义在配置文件的机制：`/etc/neutron/plugins/ml2/ml2_conf.ini` 中的 `mechanism_drivers`，这里的值要去 `neutron.egg-info/entry_points.txt` 中的 `neutron.ml2.mechanism_drivers` 下找对应的实现类。

![](https://static2.mazhangjing.com/20211016/067c_p11.png)

这个实现类必须实现 MechanismDriver 父类接口，类似于如下所示的 18 个`{action:create/update..}_{resources:port/network/subnet}_{pre/post}commit` 接口，以及 `bind_port`，如下所示：

![](https://static2.mazhangjing.com/20211016/f3bc_p12.png)

`action_resource_precommit` 接口会在资源操作 `action:create/update/delete` 数据库事务期间调用，如果报错，则触发数据库事务回滚。`action_resource_postcommit` 接口在资源操作数据库完后调用，如果报错，需要显式的去处理异常。

实际上，AgentMechanismDriverBase 一共只实现了两个接口：`create/update_port_precommit`，因为 Network、Subnet 纯逻辑，和厂商的实现机制无关，实现机制主要差异在 `bind_port` 接口上。在仅有的这两个 precommit 中，其均调用了 `_insert_provisioning_block` 函数，此函数用于实现分布式的脏标记（说人话就是基于数据库的状态标记），其在表 `provisioningblocks` 中添加一条记录，entity 字段一般是 L2，standard_attr_id 字段和 `standardattributes` 表关联，代表资源集的基本属性（资源类型，创建时间等）。比如对于 Port，创建资源需要 L2 Agent 设置流表和安全组，DHCP Agent 分配 IP/MAC，之后 `status=ACTIVE` 才能设置，而它们是独立的线程甚至主机，因此在 `neutron/db/provisioning_blocks.py` 中，通过操纵表 `provisioningblocks`，当创建 Port 等资源时，通过 `add_provisioning_component` 函数调用插入对应 Entity 记录，之后在这些 Agent 中 RPC 调用后执行 `provisioning_complete` 删除对应记录，如果记录删除完成，则调用 notify 通知回调，回调函数设置 status=ACTIVE。

![](https://static2.mazhangjing.com/20211016/cd71_p13.png)

除此之外，`bind_port` 函数和 `ml2_port_bindings` 表有关，其表示每个 port、关联的主机 host 的实现技术（虚拟化技术），比如 `vnic_type：normal`、`macvtap`、`direct`、`baremetal`、`directphysical` 等。

![](https://static2.mazhangjing.com/20211016/f587_p14.png)

除此之外，`bind_port` 还和 `ml2_port_binding_levels` 相关。因为 Neutron 并不区分多 Segment 网络和单 Segment 网络（至少在数据库层面），因此对于 VLAN 而言，这里的 `Segment_ID` 指的就是 Tag。`ml2_port_binding_levels` 表的用途可以看做是在 `ports` 和 `networksegments` 之间建立联系。	

![](https://static2.mazhangjing.com/20211016/2fd1_p15.png)

![](https://static2.mazhangjing.com/20211016/9cdc_p16.png)

但问题是，在 Neutron 中，Network 可能仅包含一个 Segment，指二层网络，也可以包含多个 Segment，比如 Network 模型中包含 segments 字段：VTEP 位于 TOR 上，这时候 Network 还是二层网络，单租户感受到的是 Segment ID(此 Segment ID 不在 segments 中)，`network.segments` 仅体现内部组网方式，对租户不可见。比如单独的 Setment 模型，通过 network_id 进行关联，Routed Network 过多虚拟机的租户，此时 Network 指的是二层网络容器，L2 指的是 Segment。这两种模型其实不同，在 VTEP 位于 TOR 上的组网，实际上可以分为两层，其中用户感知最上层，称之为 Level0，这里有一份 Network Type 和 Segment ID，而在下层 Level1，在 Network 的 segments 字段中，有另外的 Network Type 和 Segment ID。

![](https://static2.mazhangjing.com/20211016/c36a_p17.jpg)
	
对于 Routed Network 而言，其拓扑如下所示，每个 Network 都有自己的 Network Type 和 Segment ID，其彼此并列，都属于 Level0。

![](https://static2.mazhangjing.com/20211016/8f89_p18.jpg)
	

## Mechanism Manager
和 TypeDriver 至于 TypeManager 类似，MechismDriver 也有一个 MechanismManager，其也位于 `neutron/plugins/ml2/manager.py` 中。此类在初始化时从 `CONF.ml2.mechanism_drivers` 加载了机制驱动，其 `initialize` 方法会调用 XXXTypeDriver 的 `initialize` 方法实现数据库 SegmentID 同步。其 bind_port 方法需要传入 context 参数（PortContext 类（位于 `driver_context.py`）实例，PortContext 的父类混合了 MechainsmDriverContext 和 driver_api 的 PortContext，其表示 Port 的详细信息）。调用 `bind_port` 函数时会调用 `context._prepare_to_bind(segments_to_bind)`，这里的 `segments_to_bind` 是本网络中可用的 SegmentID，在 Context 中，此方法清空并创建上下文，传入 `_segments_to_bind` 作为可用资源，并将 `_new_bound_segment` 字段设置为空，表示当前正在绑定的 Segment ID，而 `_next_segments_to_bind` 表示需要绑定的下一层 Segmet ID 列表，也设置为空。之后将此 Context 传递给 XXXMechanismDriver 分派，比如 OVSMechanismDriver，其调用 Agent 进行绑定并写入 Context 绑定后的值，MechanismManager 从此 Context 中获取绑定后的值。

![](https://static2.mazhangjing.com/20211016/f905_p19.png)

简而言之，端口绑定的过程，就是将 Network 中 Segment 和 Port 之间关系保存到 `ml2_port_bindings` 表中，其中 segment、port、host 都是已知的，需要确定 Mechanism Driver 和 level，其中 level 的选择是通过 `bind_port` 和 `_bind_port_level` 以及后者递归 level 层次：先对 Context 的 `_new_bound_segment` 获取 segment，如果有 `context._next_segments_to_bind` 则递归调用下一级 level 进行绑定实现的。

![](https://static2.mazhangjing.com/20211016/1478_p20.png)

MechanismDriver 的选择本质是选择哪个厂商的实现方案，如果在 `ml2_conf.ini` 中的 mechanism_drivers 中有指定的 mechanism Driver 名称，则这些 Mechanism Driver 会被加入 Mechanism Manager 的 `ordered_mech_drivers` 并初始化。

![](https://static2.mazhangjing.com/20211016/d450_p21.png)

在 `_bind_port_level` 中，对所有可能的 Mechanism Driver 都进行了遍历，在进入 `driver.obj.bind_port(context)` 后，比如 Linux Agent 的实现 AgentMechanismDriver 会查询 Host 上部署的所有 Agent `context.host_agents`，循环判断每个活着的 agent `try_to_bind_segment_for_agent`，如果返回 True，则返回，反之继续查找。注意这里在检查 Agents 前还对 VNIC_TYPE 进行了判断，如果 Driver 不支持此 Context 的 VNIC_TYPE，则此 Driver 无效。

![](https://static2.mazhangjing.com/20211016/9aa6_p22.png)

`try_to_bind_segment_for_agent` 的核心在于 `check_segment_for_agent` 函数，如果合适，则填充 vifType 和 vifDetails 到绑定上下文 Context。这里的 vifType 和 vnicType 与 vifDetails 可写入厂商的虚拟化信息，其在对应的 XXXMechanismDriver 中的 `__init__` 中进行了初始化。

![](https://static2.mazhangjing.com/20211016/f787_p23.png)

而 `check_segment_for_agent` 的核心逻辑是，判断 agent 允许的网络类型以及试图绑定到的 segment 的网络类型，对于 FLAT 和 VLAN 而言，还要额外检查 Segment 物理网络和 Agent 物理网络是否一致，如果匹配，则完成检查，反之则不通过。

![](https://static2.mazhangjing.com/20211016/a23e_p24.png)

在上述步骤进行之后（Agent 网络类型检查）如果合适则会尝试真正开始绑定，反之则在 `_bind_port_level` 中继续遍历下一个可能的 Mechanism Driver 以重复此过程（因为同一时间可能有不同的端口需要被绑定，其可能使用不同的 Mechanism Driver）。

而调用 bind_port 的源头，则是 ML2Plugin 的 `create_port -> _after_create_port -> _bind_port_if_needed -> _attempt_binding -> _bind_port -> mechanism_manager.bind_port -> _bind_port_level -> driver.obj.bind_port` 到 Mechanism Driver 的具体实现，然后到数据库 `ml2_port_bindings` 表的条目。

## ML2 Plugin：create_network
创建网络从 Ml2Plugin 的 `create_network` 开始，其做了三件事：①分配 SegmentID；②将此 Network 存入 DB；③RPC 调用代理执行实际配置：`_before_create_network`，`_create_network_db` 以及 `_after_create_network`，其中有用的就一个：`_create_network_db`，在这个核心函数中，分为三个子步骤：A. `create_network_db`，其交给父类 NeutronDbPluginV2 进行处理。B. `create_network_segments`，调用 TypeManager 让特定的 TypeDriver 根据网络类型和参数创建 Segment（见下文）。C. `_process_l3_create` 记录网络外部连通性数据。

![](https://static2.mazhangjing.com/20211016/ad87_p26.png)

A. `create_network_db` 基本上就是将 Network 信息存入 `networks` 表，然后根据是否 shared 往 `networkrbacs` 中插入字段。这里的 `networkrbacs` 代表了网络操作权限 RBAC，action 为 `access_as_shared` 表示全租户共享。	

![](https://static2.mazhangjing.com/20211016/5f3f_p27.png)

在 `External_net_db_mixin` 父类中，`_process_l3_create` 方法当 Network 模型中 `router:external` 字段为 true 时，向 `externalnetworks` 插入记录，`netwrokrbac` 插入记录：

![](https://static2.mazhangjing.com/20211016/804c_p28.png)

而 B. `create_network_segments` 则交由 TypeManager 实例先从传入的 network JSON 数据中获取 segments 参数，然后如果有 segments 则①表明用户有管理员权限且创建的是运营商扩展网络，通过 `reserve_provider_segment` （参见 ML2 类型驱动）函数分配 segment，将 segment 信息存入 networksegments 表中，如果②创建的是 Router 外部网关网络，且配置了网络类型，则通过 `_allocate_ext_net_segment` （参见 ML2 类型驱动）调用 TypeDriver 分配 Segment，然后将分配的 Segment 存入 `networksegments` 中。反之，则是③租户创建网络，则调用 Type Driver 直接 `_allocate_tenant_net_segment`（参见 ML2 类型驱动，上述 `_allocate` 都会去 `_allocate_segment` 中通过调用 Type Driver 的 `allocate_tenant_segment` 委托合适的 Type Driver 进行分配，后者通过 SegmentTypeDriver 的 `allocate_partially_specified_segment` 函数进行分配：通过每个 Type Driver 对应的数据库表-见下图，从可选 Segment ID 列表随机挑选分配） 分配 Segment，然后将分配的 Segment 存入表 `networksegments` 中：`_add_network_segment`。

![](https://static2.mazhangjing.com/20211016/5244_p29.png)


## ML2 Plugin：create_subnet
`create_subnet` 的核心在于 `_create_subnet_db`，而后者的核心则是 `_create_subnet_precommit`，在 `_create_subnet_precommit` 中，通过 Subnet Pool 和 IPAM 对子网网段进行分配（见下图），在此子函数中，`_get_subnetpool_id` 函数用于获取一个 Subnet Pool，其数据库表为 `subnetpools` 和 `subnetpoolprefixes`，其中 `prefixes` 表的 cidr 字段定义了从 SubnetPool 里获得的一个可分配的大网段，每个 Subnet 可分配其中一个小网段。

![](https://static2.mazhangjing.com/20211016/4cd9_p30.png)
	
当获取到 `subnetpool_id` 后，通过 IPAM 进行检查，IPAM 服务简写可在 `etc/neutron.conf` 的 `ipam_driver` 中配置，默认实现是 internal，其类映射关系定义在 `egg-info/entry_points.txt` 中，internal 对应的默认实现是 NeutronDBPool，在 `validate_pools_with_subnetpool` 函数中检查后，调用 `allocate_subnet` 函数来分配子网并写入数据库。在 `allocate_subnet` 函数中，调用 `_save_subnet` 方法来写数据库，这里主要涉及如下表：`subnet`、`dnsnameserver`、`subnetroutes`（子网路由表） 和 `subnet_service_type`，`ipallocationpools`（表示子网分配的 IP 地址段）。

![](https://static2.mazhangjing.com/20211016/1546_p31.png)

其关系如下所示：

![](https://static2.mazhangjing.com/20211016/f6ac_p32.png)

`_create_subnet_db` 执行完毕，弹出栈并回到 `create_subnet`，`create_subnet` 执行最后一个调用：`_after_create_subnet`，在 `_after_create_subnet` 函数的 `_create_subnet_postcommit` 子函数中，如果 Subnet 关联的 Network 是 Router 外部网关网络，那么 `_update_router_gw_ports` 更新 Router 外部网关端口信息。

![](https://static2.mazhangjing.com/20211016/2a6c_p33.png)

`_update_router_gw_ports` 会调用 L3RouterPlugin，对于 Router 外部网关网络，`_get_router_gw_ports_by_network` 获取此外部网络的外部网关端口，对于此外部网关端口关联的所有 Neutron Router，更新其外部网关端口信息：`_update_router_gw_port`。

![](https://static2.mazhangjing.com/20211016/2322_p34.png)

![](https://static2.mazhangjing.com/20211016/f19c_p35.png)

更新步骤如下所示，对于每一个 Router，获取其外部网关信息，更新 `external_fixed_ips` 的 `subnet_id `字段，在 `update_router` 方法中另外设置了 `external_fixed_ips` 的 `ip_address` 字段。因为只允许一个 Router 外部网关端口有一个 IPv4 和 IPv6，因此这里进行了 `num_fips` 判定，如果 `fixed_ips` 数量大于 1，表明有一个 IPv4 和 IPv6 地址，就不能绑定了。

![](https://static2.mazhangjing.com/20211016/c23d_p36.png)


## ML2 Plugin：create_port
ML2 Plugin 创建端口的主要工作有：①为 Port 分配 IP 地址；②将 Port 信息写入数据库表；③为 Port 进行 Mechanism Driver 的 `bind_port` 工作（见 Mechanism Driver 部分）；④通过 RPC 调用 Agent 进行具体的配置。这里的核心表是 ipallocations，这张表关联了 networks、subnets 和 ports 三大资源，每个 Port 对应一个 Network，每个 Network 包含多个 Subnet，每个 Subnet 都可以为 Port 提供一个 IP 地址。

![](https://static2.mazhangjing.com/20211016/de55_p37.png)

在创建 Port 的时候，传入参数 `fixed_ips` 包含了 `ip_address` 和 `subnet_id`，如果同时指定，则 Neutron 尝试在相应 Subnet 上分配此地址给 port，如果仅指定了 `subnet_id`，那么 Neutron 从 Subnet 上分配一个有效 IP 给 Port，如果仅指定了 IP，则 Neutron 选择一个合适的 Subnet 将 IP 分配给此 Port。实际的过程很复杂：NeutronDbPluginV2 的 `create_port` 方法如下：

![](https://static2.mazhangjing.com/20211016/a7fc_p38.png)

在 `neutron/db/ipam_pluggable_backend.py` 中，`allocate_ips_for_port_and_store` 的核心在于 `_allocate_ips_for_port` 以及 `IpamPluggableBackend._store_ip_allocation`，`_allocate_ips_for_port` 为 Port 分配端口，根据上述规则进行参数判断和检查，最后调用 `_ipam_allocate_ips` 进行 IP 地址分配，这里根据 `/etc/neutron.conf` 进行了端口 IP 地址上限判断：`max_fixed_ips_per_port`，但如果 Port 的 device owner 是 "network:" 开头，则没有此限制。`_ipam_allocate_ips` 涉及三张表分配地址：subnets，`ipamallocationpools`（表示一个 subnet 可分配的 IP，first_ip 和 last_ip 标识了可分配的 IP 地址范围）、`ipamallocations`（表示 Subnet 已经分配的 IP，subnet 地址、分配的 ip 地址以及 status 状态）

![](https://static2.mazhangjing.com/20211016/3276_p39.png)

而 `_store_ip_allocation` 则操纵表 IPAllocation，将分配的 IP 地址写入 ipallocation 表中。而 Ml2Plugin 的 create_port 方法（区别于上面提到的 NeutronDBPluginV2 这个父类）如下所示：

![](https://static2.mazhangjing.com/20211016/a039_p40.png)

![](https://static2.mazhangjing.com/20211016/c63c_p41.png)

最后在 create_port 的 `_after_create_port` 中通过 Mechanism Manager 调用 Mechanism Driver 执行了 port_binding 操作，至此完成端口的创建。

# Neutron Service & Extension Plugins
Neutron 中 networks，subnets，subnetpools，ports 统一划归为 Core Service，Core Service 的 Plugin 是 ML2 Core Plugin，核心插件的扩展称之为 Service Plugin 业务插件（services），而其他资源统一被称之为 Extensions Service（extensions）在代码中，其分别用 services 和 extensions 表示。		
![](https://static2.mazhangjing.com/20211016/d2d5_p42.jpg)

在 `/etc/neutron.conf` 的 `service_plugins` 中可配置这些 Service Plugins，比如 router，firewall，lbaas，vpnaas，metering，qos 等。同样的，这些 plugin 名称对应在 egg-info 的 `entry_points.txt` 中，比如 router 就是 `neutron.services.l3_router.l3_router_plugin:L3RouterPlugin`。

## Router Plugin：create_router
L3RouterPlugin 也是一个二三十层继承的类，其核心部分如下所示：

![](https://static2.mazhangjing.com/20211016/a028_p43.png)

其中 `create_router` 接口定义在 L3_NAT_db_mixin 中，这里调用 `L3_NAT_dbonly_mixin` 的 `create_router` 接口，如果路由包含外部网关信息，则 RPC 通知 L3 Agent。

![](https://static2.mazhangjing.com/20211016/7188_p44.png)

这里做的主要是创建了三个算子（数据库写入、删除路由器、更新网关），然后通过 `safe_creation` 方法进行路由创建：

![](https://static2.mazhangjing.com/20211016/f478_p45.png)

创建路由器的算子会进行尝试，如果出错了则进行调用删除算子进行删除。

![](https://static2.mazhangjing.com/20211016/a32f_p46.png)

创建算子的核心是`_create_router_db`，其将准备创建的 Router 信息写入数据库表 routers 中：

![](https://static2.mazhangjing.com/20211016/0f26_p47.png)

而 `_update_gw_for_create_router` 函数用来配置 `external_gateway_info` 网关，通过 `_update_router_gw_info` 子函数，提取参数，调用 `_create_gw_port` 函数，此函数又调用 `_create_router_gw_port` 函数，通过 ML2 Plugin 的 `create_port` API 创建一个 Port，将数据写入数据库。如果出错，则调用 delete_router 函数，此函数删除 Router 外部网关端口，清除 routerports 数据，调用 ML2 Plugin 删除外部网关端口，之后获取所有关联到此 Router 上的端口并调用 ML2 Plugin 删除，最后从数据库删除 Router，通知其它模块。	

![](https://static2.mazhangjing.com/20211016/6a45_p48.png)

## Router Plugin：add_router_interface
此 API 用于给 Router 增加接口，先调用父类 `L3_NAT_dbonly_mixin` 同名接口，RPC 通知 L3 Agent，后者先获取 Router 对象，然后根据参数 interface_info 判断增加 Port 还是 Subnet（`port_id` or `subnet_id`），如果是 port_id，则 `_add_interface_by_port`，反之则 `_add_interface_by_subnet`，之后进入 `_add_router_port` 中，在 `routerports` 添加一条记录，表示 Port 和 Router 关系，最后调用 ML2 Plugin 的 `update_port` 修改端口 device_id 和 device_owner 属性，并通知其它模块。

![](https://static2.mazhangjing.com/20211016/5c7c_p49.png)

![](https://static2.mazhangjing.com/20211016/c891_p50.png)

其中 `_add_interface_by_port` 函数大致如下所示，其更新了 device_id 和 device_owner。

![](https://static2.mazhangjing.com/20211016/ce4b_p51.png)

`add_interface_by_subnet` 稍微复杂一些，先获取 Subnet，然后检查其是否与 Router 已经关联的子网网段重复，最后通过 ML2 Plugin 的 create_port 新增一个端口。

![](https://static2.mazhangjing.com/20211016/9200_p52.png)

# Neutron Plugins：RPC and Pub/Sub
在 Neutron 中随处可见 `registry.notify` 这样的代码，其涉及 Callbacks Module 机制，位于 `neutron/api/rpc/callbacks` 目录下，包含了生产者 producer 的 `registry.py` 和消费者 consumer 的 `registry.py` 以及共享的 `events.py`, `resources.py`, `resource_manager.py` 等实现。

![](https://static2.mazhangjing.com/20211016/6ec2_p54.png)

![](https://static2.mazhangjing.com/20211016/e97f_p53.png)

这里的 pull 和 push，都是调用 `resource_manager` 进行处理，后者通过定义在 `events.py` 中的事件和 `resources.py` 中的资源类型进行回调，通过 RPC 消息发送给对应消费者实现通知功能。
