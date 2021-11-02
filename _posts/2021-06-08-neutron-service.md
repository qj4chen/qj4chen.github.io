---
layout: post
title: OpenStack Neutron Service 源码解读
categories:
- OpenStack
- Neutron
---

> 本文介绍了 Neutron Service (neutron-service 进程) 的启动过程和作用原理，基于 OpenStack Rocky 版本。

Neutron 主要管理“软”网元，即 Linux Bridge、OVS 等软件实现的网络组件，以及厂商提供的闭源产品，此外，也包含一些硬网元，比如 VTEP 位于 TOR 交换机上的场景(multiSegments)。RESTful API 首先经过 Python 的 WSGI，之后通过 Python API 调用 Neutron 的 Plugin 模块，后者通过 RPC 调用 Neutron 的 Agent，Agent 通过某种协议对 VNF 虚拟网络功能进行配置（注意，这里的 Agent 包含 L2 和 L3 的，Nova 节点包含 L2 Agent，Neutron 节点包含 L2，L3 Agent 以及 Plugin，相关配置文件在多个节点都要配置）。

![](https://static2.mazhangjing.com/20211016/c327_s1.jpg)
	
如上所示，控制节点和网络节点上一般部署 `neutron-server`（包含 Plugin）、RDBMS、MQ，L2 Agent，L3 Agent（因为有些网络功能需要在网络节点和计算节点均设置），计算节点部署 L2 Agent，比如 L2 的 neutron-openswitch-agent、neutron-linuxbridge-agent 等，neutron-dhcp-agent、neutron-l3-agent 等以及 Agent 底层控制的 Bridge（LinuxBridge、OVS）。

从服务的具体实现上而言，Controller 上的 `neutron_server` 本质是一个 Python WSGI Server，其可通过 `systemctl status/start/stop neutron-server（service neutron-server start/stop/status）`启停，服务脚本位于 `/usr/lib/systemd/neutron-server.service` 中，这里定义了 Python 脚本的位置（`/usr/bin/neutron-server`），启动使用的参数文件（`/etc/neutron/neutron.conf`，`/etc/neutron/plugin.ini` 等），日志记录位置（`/var/log/neutron/server.log`）等信息。其各个 WSGI App 和负责的 Plugin 如下所示：

![](https://static2.mazhangjing.com/20211016/8c6a_s2.png)


## Neutron Server 依赖的基础技术
### WSGI & RPC
WSGI 是 Python 的一套 Web 框架规范，可供 Web Server 通过 WSGI 接口调用，其必须满足：①可调用对象；②其包含 `environ`、`start_response` 参数；③有一个可迭代的返回值。

所谓可调用，可以是带有参数（参数为 ①environ：包含 CGI 环境变量、OS 环境变量、WSGI 环境变量、服务器自定义环境变量，②`start_response`：一个 Callable Object，其本身是一个函数，包含 status 状态码、`response_headers` HTTP 头、`exc_info` 错误处理信息）的函数，可以是带有 `__call__` 的类实例（或者直接是一个类），如下所示：

![](https://static2.mazhangjing.com/20211016/f8f8_s3.png)

此函数/类方法调用后必须有一个可迭代的返回值，比如调用 start_response 直接写入 Header、Body 并返回，或者交给另一个 your_app 进行链式处理。

![](https://static2.mazhangjing.com/20211016/b4a1_s4.png)

一般而言，OpenStack 使用 RabbitMQ 实现 AMQP 实现 RPC 通信，其过程如下所示，RPC Client 发送请求需要生产消息，然后写入主题，RPC Server 通过 MQ 接收到消息，发送回一个 Direct Exchange 直接回应，通过 MQ 传递给 RPC Client。

![](https://static2.mazhangjing.com/20211016/8b7e_s5.png)

### greenlet & eventlet
Neutron 是一个网络阻塞型 Web 应用，因此不必使用 Servlet 等多线程技术，其采用的是轻便的用户上下文切换的协程技术：基于 greenlet 封装的在网络阻塞时自动切换的 eventlet 库。greenlet 使用 switch 进行启动，并且在执行过程中可使用 switch 切换上下文（类似于 goroutine 的多路复用）。

![](https://static2.mazhangjing.com/20211016/febf_s6.png)

eventlet 在其基础上进行了封装，GreenPool 创建了一个协程池，通过 spawn 来运行多个协程，协程将在网络阻塞时自动切换上下文。

![](https://static2.mazhangjing.com/20211016/0df0_s7.png)

### OSLO
最后，OpenStack 通过 oslo 抽取各个项目的公共库，以供其他项目使用。从 github 上下载的 neutron 缺少 oslo，一般需要额外安装 pip install -r requirement.txt。

## Neutron Server 启动一览
调用 `/usr/bin/neutron_server.py` 后，Neutron Server 的核心启动顺序如下所示：

![](https://static2.mazhangjing.com/20211016/50a5_s8.png)


`[SERVE]` 根据 setup.cfg，neutron-server 的 entryPoint 位于 `neutron.cmd.eventlet.server:main`，之后的核心位于 `neutron/server/wsgi_eventlet.py` 中，`eventlent_wsgi_server` fun 中，serve_wsgi 启动了一个 Web Server（此处的 service 是 neutronApiService)，第二句话创建了一个 RPC Consumer。

![](https://static2.mazhangjing.com/20211016/5368_s9.png)

`[LOAD]` 之后 `neutron/service.py` WsgiService 的 _run_wsgi 方法创建了 Neutron Server RESTful API 的核心，以及 Plugins 的加载（load_paste_app）：

这里的 load_paste_app 创建了一个 Loader 对象，其 load_app 方法加载了 WSGI APP。为加载此 APP，其从 `/etc/neutron.conf` 的 `api_paste_config` 中或 neutron 配置目录的 `api-paste.ini` 中加载配置（`/usr/share/neutron/api-paste.ini`），paste 是一个 新颖的 Python Web 框架，其通过配置文件的方式定义请求的处理流程并委托特定可调用对象根据不同策略提供服务。

![](https://static2.mazhangjing.com/20211016/7091_s10.png)

比如刚开始的 `composite:neutron`，其调用 Paste 包的 urlmap 组件对 URL 过滤并分流，类似于其他 Web 框架中的路由，use = egg 表示 egg 包，call 表示某个模块，config 表示其它配置文件。对于 neutronapi_v2_0 而言，其调用了 `pipeline_factory` 对象，后者根据两种策略：noauth 和 keystone （依赖配置文件的 auth_strategy 选项标识）分别获取其 composite - filters，取出 `neutronapiapp_v2_0`（创建了一个 APIRouter 对象），并且将其按照过滤链的方式从前往后层层将其包裹，因此，当实际调用时，会从 APIRouter 开始，逐个应用 cors、http_proxy_to_wsgi、request_id、catch_errors .. 的顺序进行处理。

![](https://static2.mazhangjing.com/20211016/43bb_s11.jpg)


`[START]` 此 APIRouter 包裹后，在 `server.py` NeutronApiService 的 `run_wsgi_app` 中传入，通过配置传入绑定的 host 和 port，确定配置（如果有）worker 数，进入 `server.start`，之后在 wsgi.Server 的 `_get_socket` 中获取 Socket 并开始工作。

![](https://static2.mazhangjing.com/20211016/4b21_s12.png)

在 Server 中 start 方法：

![](https://static2.mazhangjing.com/20211016/e49b_s13.png)

`[LAUNCH]` 这里的配置在 `etc/neutron.conf` 的 bind_host、bind_port 中配置，默认是 0.0.0.0:9696 端口。在 start 方法中，调用了 `_launch`，后者根据计算机 CPU 决定是在当前线程启动服务还是创建新线程。

![](https://static2.mazhangjing.com/20211016/3bfb_s14.png)

最后到达核心的 `wsgi.py` 的 WorkerService 中，start 方法通过 pool.spawn 创建绿色线程，此处在协程中调用 `service（Server 实例）._run` 方法:

![](https://static2.mazhangjing.com/20211016/64fe_s15.png)

`Server._run` 方法，此处真正创建了 NeutronApiService 实例并在协程执行：

![](https://static2.mazhangjing.com/20211016/9c41_s16.png)


对于上述 LOAD 过程而言，Neutron Server 需要对核心服务和扩展服务分别使用对应 Plugin 进行处理。HTTP 请求先经过 app_urlmap，如果是 / 则调用 neutronversions 构造 versions_factory  通过 pecan.make_app(RootController()) 来返回版本号。如果是 /v2.0，则进入 neutronapi_v2_0 后，经过授权后依次处理 extensions 和 neutronapiapp_v2_0 模块。

对于核心服务 Core Service 而言，关键在于后者，其调用 APIRouter.factory 方法，通过 pecan.make_app(V2Controller()) 构造了 APP，在 V2Controller 中，通过委托 NeutronManager 来对指定路径前缀 get_resources_for_path_prefix 获取对应资源模型，然后 get_controller_for_resource 以构造这些资源对应的控制器方法。在这个方法中，manager 从 CONF.core_plugin 中找到核心插件名称，从 CONF.service_plugins 获取核心扩展插件名称，在 `neutron.egg-info/entrypoints.txt` 中找到对应的类（或直接定义在 neutron-server.conf 的 service_providers 中，比如 `service_provider = FIREWALL:inspur-ice:networking_odl.fwaas.xxx`），一般是 ml2 和 `neutron/plugins/ml2/plugin.py` 的 ML2Plugin，实例化以为 APIRouter Path -> Resource -> Controller 请求链条提供服务（类似于 ODL 通过 RESTCONF -> Yang Model -> Service Impl RPC 调用一样）。

> Ps. 在早期版本中，核心插件及其扩展通过如下方式加载：APIRouter、RouterMiddleware 和 Controller，这些方法位于 neutron/api/v2 包下。过程参见 Extension Service。
> ![](https://static2.mazhangjing.com/20211016/238a_s17.png)

对于 Extension Service（/v2.0/extensions) 而言，extensions 位点定义的 filter_factory 是 plugin_aware_extension_middleware_factory，其位于 Core Service 的外层，即先当做扩展服务处理，处理不了再通过核心服务尝试处理。此工厂函数在 neutron/api/v2/extensions.py 中 `PluginAwareExtensionManager.get_instance`() 后，将其注入 ExtensionMiddleware 中，对于 get_instance() 方法，其主要是加载了内置的 Extension Plugin —— 通过模块和类路径找到并实例化，此外也加载了 CONF.api_extensions_path 路径下的扩展插件（映射参见 neutron.egg-info/entrypoint.txt 下 neutron.service_plugins 对应关系）。

![](https://static2.mazhangjing.com/20211016/12ec_s18.png)

根据上述逻辑，对于自定义 Extension Service 而言，需要实现 neutron_lib.api.extensions.ExtensionDescriptor 接口，通过配置 `etc/neutron.conf` 的 api_extensions_path 来找到可执行动作，或者将动作直接放在 neutron or neutron-xxaas/extensions 下进行加载即可：

![](https://static2.mazhangjing.com/20211016/1a17_s19.png)

ExtensionMiddleware 做的事情就是（在 `__init__` 中）将这些找到的 Extension Plugins 并进行解析，将其和对应的 Resource 联系起来并生成 URL 路由规则，然后当 `__call__` 时交给 RouteMiddleWare 让后者对 HTTP 请求路径解析并映射到对应 Resource 上并委托 RouteMiddleWare 通过 ExtensionMiddleware 的 `_dispatch` 调用资源的控制器方法实现调用。大致过程如下所示：

![](https://static2.mazhangjing.com/20211016/f85c_s20.png)

最后，我们提到 Neutron 通过 AMQP Server 实现 RPC 调用，实现 Plugins 和 Agents 的交互，Neutron Server 启动后，在 wsgi_eventlet.py 中（上述 `[SERVE]` 步骤），`start_api_and_rpc_workers` 方法即启用了 RPC Consumer，然后在 `service.start_all_workers` 函数中让 Plugin 成为 RPC Consumer。

![](https://static2.mazhangjing.com/20211016/6eaa_s21.jpg)

`start_all_workers` 方法中的 `_get_rpc_workers()`：①在 `neutron/service.py` 的 RpcWorker 中，对于所有 plugins 进行遍历，找到有 start_rpc_listeners 方法的插件（对于 ML2 Plugin 而言， Ml2Plugin 的 start_rpc_listeners 方法、L3RouterPlugin 的 start_rpc_listeners 方法）②同样的在 `neutron/service.py` 的 RpcReportsWorker 中也遍历了带有 `start_rpc_state_reports_listener` 方法的插件。对于 `_get_plugins_workers()` 方法而言，其检查每一种 plugin 中是否实现了 get_workers 方法，然后将所有上述的 workers 收集起来，为其创建了 RPC Consumer。这个过程可在 `etc/neutron.conf` 中 `rpc_workers` 和 `rpc_state_report_workers` 中进行设置其允许的最大线程，如果超出，则按照协程运行。
