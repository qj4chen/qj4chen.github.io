---
layout: post
title: OpenStack 的前世今生
categories:
- OpenStack
---

> 本文简要介绍了云计算、虚拟化以及 OpenStack 产生的背景。

# 云计算的前世今生
## 分时操作系统 Linux
最早的服务器一般是**单主机多用户模式**（分时操作系统 Linux），这种模式下租户和租户间仅有微弱的隔离（用户主目录，文件权限区分），彼此影响，资源互相依赖，没有严格的隔离措施，随着租户增多和单主机多用户模式弊端显现，以及虚拟化技术发展，逐步出现了**单主机多虚拟机模式**，这种模式将底层宿主资源（网络、计算、存储）共享，但不同虚拟机指令、内存和存储等强隔离（Vmware ESXi，KVM + QEMU 等，虚拟机 CPU 本质是宿主机线程模拟的，虚拟机内存也是）。

## Linux 虚拟化技术：Linux Bridge & KVM
单主机多虚拟机有几种类型：Hypervisor 直接安装在物理机上的 **I 型虚拟化**（Xen，VMWare ESXi 等），物理机上先安装常规操作系统，然后 Hypervisor 作为一个模块运行的 **II 型虚拟化**（KVM，VirtualBox 和 VMWare Workstation 等）。I 型虚拟化一般对硬件虚拟化功能进行了特别优化，性能更高，但 II 型虚拟化基于普通 OS，比较灵活（比如支持 KVM 套 KVM）。在这些技术中，开源、热门且应用广泛的数 **KVM - Kernel-Based Virtual Machine**，其包含一个叫做 kvm.ko 的内核模块，只管理虚拟 <u>CPU 和内存</u>（只负责虚拟机调度和内存管理），而 IO 的虚拟化，比如存储和网络，则由 Linux 内核和 QEMU 来实现。<u>KVM 的库叫做 libvirt，服务程序 libvirtd，命令行叫 virsh，GUI 有 vir-manager</u>。一个 KVM 虚拟机本质上是 Linux 的一个 qemu-kvm 进程，宿主机需要支持虚拟化：<small>一般通过查看 /proc/cpuinfo 是否由 vmx 或者 svm 指令支持判断是否支持虚拟化（VMWare ESXi 或 Workstation 需打开 Intel/AMD 虚拟化方案）</small>。

使用 KVM 可以安装如下包：`libvirt, qemu-kvm, qemu-system, virt-install，virt-manager, bridge-utils vlan`。之后启用 virt-manager，其界面类似于 Vmware Workstation，不过可以管理其他 Host 上的 VM，其本质是通过配置发送命令行：/etc/libvirt 的一些配置进行的，修改后重启 libvirt service 即可。也可以使用 virt-install [教程](https://www.cnblogs.com/saryli/p/11827903.html) 来从命令行进行安装，比如一个来自于 cirros-cloud.net 的很小的 Linux 发行版：
`virt-install --connect qemu:///system --ram 200 -n kvm1 --os-type=linux --disk path=/home/abc/project/cirros/cirros-0.3.4-x86_64-disk.img,device=disk,bus=virtio,format=qcow2 --vcpus=1 --nographics --noautoconsole --import`

 之后可使用 `virsh list -all` 来查看（sudo），`virsh start xxx`, `virsh console xxx` 开机并进入命令行，其在后台跑着一个 `/usr/libexec/qemu-kvm` 的进程，其中每个虚拟的 vCPU 都对应此进程中的一个线程。需要注意，vCPU 数量可多于物理 CPU 的数量（起码 KVM 允许），如果在同一时刻不是所有虚拟机都满负荷，那么可以更好的利用 Host CPU 资源（计算密集型不适用）。

而对于**内存虚拟化**，KVM 则需要进行 `VirutalMemory -> PhysicalMemory -> MachineMemory` 的转换，虚拟 OS 控制 VA -> PA，而 KVM 则负责 PA -> MA。注意，内存在 KVM 也是可以 overcommit 的，但需要注意可能导致性能影响。

而对于**存储虚拟化**，KVM 通过 Storage Pool 在宿主机分配一片储存空间（默认 `/var/lib/libvirt/images`），然后在 Pool 中分配 Volume 给 VM，其配置定义在 `/etc/libvirt/storage` 目录下，每个 xml 都对应一个 Pool，默认是 default.xml，比如 Storage Pool 类型是 dir，目录路径是 `/var/lib/libvirt/images`。基于文件的 Volume 的一大优点就是可远程访问，即其可以放在非本地文件系统中，比如 NFS etc 以实现共享和高可用。Volume 的格式有 raw - 移植性好，性能好，大小不定，qcow2 推荐格式，QEMU Copy on Write，可节省磁盘空间，支持 AES 加密，zlib 压缩，快照等。vmdk VMWare 虚拟机格式，其可以直接在 KVM 上运行。通过 `virsh pool-list、pool-define` 可以查看和创建存储。

对于**网络虚拟化**而言，可通过 `brctl` 添加网桥（网桥 IP 为其网卡 IP），然后将外部网卡与之绑定：`brctl addif ens33` 最后配置 `virsh domiflist xxx` 查看当前网络，在 virt-manager 中配置网络为这里的网桥即可访问。默认情况下，QEMU 会自动创建 virbr0 虚拟网桥和 virbr0-nic 虚拟网卡，然后设置了 iptables 规则以进行 SNAT 访问外网（外网不能访问）：

![](https://static2.mazhangjing.com/20211013/db3e_111.png)

> 这里通过 `ip a add` 为 virbr0 分配了 `192.168.122.1/24` 的网段，之后允许 Linux 参考路由表(下图) ip_forward 并且用 iptables 允许此网段 FORWARD 到 virbr0，这里的 KVM Instance 访问外网通过 SNAT 网段伪装（MASQUERADE) 实现。

![](https://static2.mazhangjing.com/20211013/a9d6_222.jpg)
	
默认情况 VM 都会使用此 `virbr0` 网桥(应该是通过一个跨 namespace 的 veth pair，一端连接在 virbr0 的 vnet0 上，通过 ip l 可见，一端连接在 KVM 的 namespace 的 eth0 上)，如右图所示。这里之所以能访问外网，则是通过 peth0 和 peth1 的 Linux `ip_forward` 到 virbr0（通过 ip a add 配置）。

![](https://static2.mazhangjing.com/20211013/89bc_333.png)

除此之外，还有一种桥接模式，其原理如下图所示，直接将一个可以和外部网络连接的网卡 - peth0 挂在 Linux Bridge virbr0 上，而 KVM Instance 还是通过 veth pair 跨 ns 连接到挂在 virbr0 的 vnet0 网络（只有 start KVM instance 后才有，关闭时没有）。

![](https://static2.mazhangjing.com/20211013/1356_444.png)

最后，此网桥使用了 dnsmasq 提供 DHCP 服务（可能这个 virbr0-nic 就是干这个用的），分配的地址参见 `/var/lib/libvirt/dnsmasq` 的 `default.leases`。

## 网格计算、容器和云时代
单主机多虚拟机的问题在于，现代应用峰值流量很多，VM 需要按照最高峰值来配置服务器吗？根据这个需求，引发了“云计算”的热潮：按使用量付费的服务。根据 NIST（美国家标准与技术研究院）定义，云计算是按使用量付费的模式，以提供可用、便捷、按需的网络访问、可配置的计算资源共享池（网络、服务器、存储、应用软件、服务等），而仅需很少的管理工作（或很少的与服务供应商交互）。

为云计算提供支持的云架构一般包含多种服务模式：
- **IaaS 基础设施即服务**(给硬件和网络，比如 KVM 虚拟机，网络)
- **PaaS 平台即服务**(给软件环境：JDK、PHP、Node.js) 
- **SaaS 软件即服务**(给数据：Office365 或者给代码：新浪云)。

而为服务模型提供支持的，则可以是共有、私有或者混合云模式，私有云有更好的控制、安全性和兼容性，但公有云有更好的容量、弹性和成本控制，混合云就是二者混合使用。

而所谓的云计算，依赖的基石有二：**①分布式；②虚拟化**，包括硬件的虚拟化，网络和存储的虚拟化（KVM，OpenvSwitch 等）。构建于之上的云计算，实现了计算能力的灵活伸缩，有效的降低了应用成本。Ps. 分布式和虚拟化定义了云计算，而不是云计算引发了分布式和虚拟化这两个概念，因为我们看到，云计算来自于 Linux 虚拟化 IO、网络（OVS、Linux Bridge）和 CPU（KVM、Container）技术的成熟，以及这个时代特有的 x86 小型机取代大型机，服务虚拟化和数据量急剧上升导致的分布式计算格局。二者合起来，就诞生了所谓的云计算。

# OpenStack 概述
OpenStack 是为管理单一或众多小型机硬件资源，构建统一的 IaaS 平台以及其上的软件设施（PaaS）而生（甚至包括 SaaS 的后端部分）。OpenStack 之于数据中心，就好像 OS 之于服务器一样，它以牺牲性能获得了软件对资源的灵活控制（包括资源逻辑隔离），带来了较高的运行效率，这一优势体现在，OpenStack 对底层硬件进行了可伸缩的抽象支持（虚拟机、卷、块存储、网络、镜像等），使用一套通用 API 进行管理，使得最终用户请求可以高效自动化执行或用户自我供给资源。

![](https://static2.mazhangjing.com/20211013/a11c_555.png)

在现实中，OpenStack 的主要用途之一，就是用来部署私有云（类似于 AWS、Azure）。相比较公有云而言，私有云更多的是资本支出，公有云则是运营支出（在中国，更多的是网络开销）。

OpenStack 的核心在于**计算**，但区别于 **Virtual Machine Monitor（VMM，hypervisor）**，OpenStack 控制 Hypervisor 操作，比如 OpenStack 支持 Kernal-based VM（KVM）、QEMU、ESXi、Hyper-V 等多种虚拟机监控器。区别于 Linux Container，类似于 Docker 的操作系统实例隔离技术运行在系统级别，所有容器共享相同内核，其开销更小。除了虚拟机相关（当然最重要的就是计算）能力，OpenStack 还包含网络组件（比如 Arista Networks，Cisco Nexus，Linux bridging，Open vSwitch - OVS 等），支持 VLAN 和各种隧道等网络类型，可提供 DHCP、路由 L3 等多种服务，相比较直接面向硬件，OpenStack 的抽象层使得前后端解耦，后端厂商变更更加灵活。此外，OpenStack 还包含块存储（基于 Block）和对象存储（分布式对象存储），用于虚拟机快照、镜像仓库等用途。<small>Ps. OpenStack 至于云计算，就好像 Spring(Boot) 至于后端服务一样，它们并不提供具体实现，而是提供一个框架和平台，允许自由插拔组合。</small>

作为一个完整的云平台框架，OpenStack 围绕计算、网络、存储等云设施资源形成了松耦合的多个项目：支持多种 Hypervisor 的 OpenStack Computing - Nova，支持 FC，iSCSI 等块存储协议的块存储 Cinder，统一管理镜像的 Glance、提供 Overlay Network 的 OpenStack Networking - Neutron，支持 Swift、Ceph 等分布式存储技术的对象存储服务 Swift 等，如下所示：

![](https://static2.mazhangjing.com/20211013/1111_666.png)

它们彼此间通过 API 实现调用，通过 Keystone 进行权限认证，通过 OpenStack Dashboard - Horizon 进行展示。

![](https://static2.mazhangjing.com/20211013/c7d5_777.png)