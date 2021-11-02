---
layout: post
title: OpenStack 的安装 (Rocky, CentOS 7)
categories:
- OpenStack
---

> 本文简要介绍了在 CentOS 7 下安装 OpenStack Rocky 的步骤，包含了基于 Linux Bridge 实现的网络和基于 OVS 实现的网络两种类型。

[Rocky 安装文档](https://docs.openstack.org/rocky/install/) | [Rocky 部署文档](https://docs.openstack.org/rocky/deploy/)（Docker，Kolla-Ansible）| [Rocky 配置文档](https://docs.openstack.org/rocky/admin/)（[Neutron 配置文档](https://docs.openstack.org/neutron/rocky/admin/)，[Neutron 配置文件](https://docs.openstack.org/neutron/rocky/configuration/)）| [Rocky 操作文档](https://docs.openstack.org/operations-guide/)（[Neutron 排错文档](https://docs.openstack.org/operations-guide/ops-network-troubleshooting.html)）| [Rocky API 文档](https://docs.openstack.org/rocky/api/)([Neutron REST API](https://docs.openstack.org/api-ref/network/#)) | [OpenStack Client 安装、配置、使用](https://docs.openstack.org/python-openstackclient/latest/#getting-started) | [Horizon 安装、配置、使用](https://docs.openstack.org/horizon/rocky/)

如下是整个 OpenStack 各个组件的交互方式，其中最重要的有 Nova，Neutron，Keystone，Horizon，Swift/Cinder。

![](https://static2.mazhangjing.com/20211013/44e4_in1.png)

根据主要提供的服务在 L2 还是 L3 分为提供者网络和自服务网络（前者以来物理网络 L3，后者多了 Networking L3 Agent，可提供 L3 层服务）。此外，下列提到的密码，均列举在这里：

![](https://static2.mazhangjing.com/20211013/3dc8_in2.png)

# 基础准备工作
首先要规划网络拓扑，这里使用一个 Controller 节点负责 Keystone、Horizon、Neutron、Glace 等服务，且包含 DB、AMQP、NTP 等基础组件，一个 Compute 节点负责 Nova 计算服务（KVM、网络服务 Agent），以及可选的 Block Storage Cinder 和 Object Storage Swift 服务。

![](https://static2.mazhangjing.com/20211013/4347_in3.png)

官方 InstallGuide 中管理网络是 10.0.0.31/24，这里使用的是 172.20.103.234/16。

| 主机 | IP |
| --- | --- |
| Cm-cent1	| 管理&外部网络 ens160：821D：172.20.103.234，内部网络 ens192：8227 |
| Cm-cent2	| 管理&外部网络 ens33：8e16：172.20.103.235，  内部网络 ens160：8e0c |
| Cm-cent3	| 管理&外部网络 ens33：3911：172.20.103.236，  内部网络 ens160：3907 |

> Ps. 在 ESXi 中，我们使用了上述 IP，而在本地 Vmware Workstation 中，则使用了下面的 IP(在 OpenStack 中，Instance Tunnels Network 又被称之为 overlay，External Network / 非 Tunnel 的 Instance Network 被称之为 provider。为了简便起见，我们可使用 ①management-需要互相通，且管理者从外部可访问，即配置 my_ip，像是配置中各种 http://xxx:xx URL，走的就是 management 网络流量，overlay 走 management 流量即可，即配置 vxlan 的 local_ip； ②provider 网络-需要互相通，如果 VM 需要访问外网，则此网络需要和外网通，一般做一个 NAT 即可，但不要求从外网可以访问)：

![](https://static2.mazhangjing.com/20211013/72c5_in4.png)


| 主机名      |	管理网络地址	   | 内部网络地址 | 配置		| 管理网络信息 |
|-----------|---------------|------------|-------------|-----------------------------|		
|Controller |192.168.222.5  |172.16.0.5	 |2U 1.5G 1N 100GB	|ens33	                |
|Neutron 	|192.168.222.6	|172.16.0.6	 |2U 2G 3N 20GB	|00:0C:29:69:01:40 ens33		|
|Compute 	|192.168.222.10	|172.16.0.10 |2U 4G 2N 100GB|	00:0C:29:8C:82:4C ens33	|	
|Block	    |192.168.222.20	|172.16.0.20 |2U 1.5G 1N 100GB|	ens33               |	

VMNET10 管理网络 
VMNET11 实例网络
VMNET12 外部网络

编辑 /etc/hosts，并测试连通性（ping）。

对于管理网络而言，如下进行配置（Ubuntu 在 /etc/network/interfaces 下，RHEL 在 /etc/sysconfig/network-scripts/ifcfg-xxx 下），主要是设置静态 IP，一定要设置 BOOTPROTO，ONBOOT，IPADDR，GATEWAY，PREFIX 可用 NETMASK 替代，不要修改 NAME，DEVICE 和 UUID，最好加上 HWADDR（网卡 MAC 地址）。对于 Vmware，使用 NAT 网络，在虚拟网络编辑器中设置网关为 172.20.103.1，注意检查 Windows 的控制面板 > 网络 >对应虚拟网卡 > IPv4 的网关、IP 地址（分配为 172.20.103.2 即可）、DNS，否者不能联通外部网络。
	
对于内部网络而言，如下进行配置（右图），这里主要是关闭 DHCP，并且设置 ONBOOT 开启启动，添加 HWADDR 防止我们修改物理网卡导致和虚拟机 ensxx 对应出错。

![](https://static2.mazhangjing.com/20211013/d60c_in5.jpg)
	
之后为 Node 安装 chrony，配置 /etc/chrony.conf，修改 server NTP_SERVER iburst，以及 allow 172.20.103.0/16（控制节点 NTP_SERVER 为某个 NTP 服务器，计算节点 NTP_SERVER 为控制节点），并 systemctl enable chronyd 和 systemctl restart chronyd。通过 chronyc sources 来查看配置，计算节点必须返回控制节点。

## 安装 OpenStack Packages [All Node]
`yum -y install centos-release-openstack-rocky`

这个命令及其依赖会往 /etc/yum.repos.d 下放置多个 OpenStack 以及依赖的 ceph、qemu、virt 的源并配置 rpm-gpg（rpm -ql)。

`yum -y upgrade`

因为上述步骤本质放置了源，所以这里顺便 upgrade 更新所有组件

`yum -y install python-openstackclient openstack-selinux vim`（CentOS7，CentOS8 安装 python3-openstackclient）

安装 Python OpenStack SDK 和其依赖的组件，安装 OpenStack 和 SELinux 交互组件。

## 安装 MySQL、MQ 和 Cache，ETCD [Controller]
`yum -y install mariadb mariadb-server python2-PyMySQL`

`vim /etc/my.cnf.d/openstack.cnf`，将 bind-address 改为 Controller 的 IP 地址：

![](https://static2.mazhangjing.com/20211013/e910_in6.png)

`systemctl enable/start mariadb.service`
`mysql_secure_installation`

```bash
yum -y install rabbitmq-server
systemctl enable/start rabbitmq-server.service
rabbitmqctl add_user openstack RABBIT_PASS
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```
RABBIT USER: openstack

```bash
yum -y install memcached python-memcached
vim /etc/sysconfig/memcached
OPTIONS="-l 127.0.0.1,::1,controller"
systemctl enable/start memcached.service
```
MEMCACHED USER：memcached

`yum -y install etcd`

设置 `/etc/etcd/etcd.conf` 中 `ETCD_LISTEN_PEER_URLS` 和 `ETCD_LISTEN_CLIENT_URLS` 与 `ETCD_NAME`，`ETCD_INITIAL_ADVERTISE_PEER_URLS`，`ETCD_ADVERTISE_CLIENT_URLS`，`ETCD_INITIAL_CLUSTER`，`ETCD_INITIAL_CLUSTER_TOKEN`，`ETCD_INITIAL_CLUSTER_STATE`。

![](https://static2.mazhangjing.com/20211013/11ec_in7.png)

`systemctl enable/start etcd`

# Keystone 鉴权验证服务 文档地址 | 安装 CentOS 地址
> Keystone 被设计通过 OpenStack Identity API 实现客户端认证 Authentication、分布式多租户授权 Authorization 和服务发现（目录），其位于用户交互的最上层，通过身份认证后才能访问其他 OpenStack 服务，可与 LDAP 或 AD 集成。

![](https://static2.mazhangjing.com/20211013/cd1f_in8.png)

大致而言，用户请求 API 时，会首先去 Keystone 获取一个令牌环 Token，然后带着这个 Token 去请求各服务的 API（类似于 CAS）。如右图所示，每个服务都假定 Token 是无效的，当我们请求 Nova 创建 VM 时，其携带 Token 请求 Glance 分配镜像，Glance 去 Keystone 鉴权后继续处理，全部成功后返回。

OpenStack 还有**域 Domain、项目 Project、用户 User、组 Group 和角色 Role** 的概念：用户是域唯一的，而不是全局唯一的。在用户之外还有组的概念，组包含了多个用户，其也是域唯一的。项目代表了 OpenStack 所有权基本单位，所有资源都应该被某个项目拥有，项目属于域，其也是域唯一的（如果未指定，则属于默认域）。域包含了项目、用户和组，其代表一个命名空间，默认为 Default。角色 Role 决定了用户或者组在域或项目范围的授权级别，其也是域唯一的。具有特定资源访问角色的身份认证后将有 Token。

![](https://static2.mazhangjing.com/20211013/c549_in9.jpg)

OpenStack 的**服务 Service** 可由多个**端点 Entrypoint** 提供，每个端点都代表可访问某一服务的 URL 地址，可是三种类型之一：**管理 admin、内部 internal 或公共 public**，分别用于管理云基础架构的管理员操作、OpenStack 内部服务通信、客户私有云和外网可见。OpenStack 支持多个区域实现扩展，下面使用一种端点类型和 RegionOne 区域。区域、服务和实现的端点共同构成了服务目录 Service catalog（比如 /wh/compute/compute101/createVM）。	

![](https://static2.mazhangjing.com/20211013/688c_in10.png) 

总的来说，你可以认为一个**用户**（张三）携带**凭证**（身份证）在 KeyStone 进行**验证**，然后得到一个**令牌**（房卡），带着令牌访问**租户**（宾馆）提供的位于**域**的**项目**（住宿、餐饮等）的**服务**，其沿着**区域**和**端点**（路径）找到这些服务的实现（某个房间），找到后根据自己的**角色**（VIP 等级）进行使用。右图展示了“用户”和“服务”的关系，即上上图中 User 通过 Role 连接到 Project 中的 Service 中的详情图。	

![](https://static2.mazhangjing.com/20211013/e2da_in11.png)

## Bootstrap keystone
首先要在 DBMS 创建好数据库和账号：
```bash
mysql -u root -p
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'xxx';
```

之后安装 `openstack-keystone` 以及 Http REST 代理 `httpd` 和 `mod_wsgi`：

`yum -y install openstack-keystone httpd mod_wsgi`

这里的 openstack-keystone 向 `/etc/keystone` 写入了一堆比如 `logging.conf` 和 `keystone.conf` 以及 keystone-paste.ini 配置文件，然后在 `/etc/logrotate.d` 中定义了日志轮转，提供了 `/usr/bin/keystone-manage, keystone-wsgi-admin` 和 `keystone-wsgi-public` 等命令，日志位于 `/var/log/keystone` 目录下。之后在 keystone.conf 配置文件写入 DBMS 地址和账号密码信息，注意这里的 bootstrap-password 不是 Keystone 的密码，而是 ADMIN_PASS：

```bash
vim /etc/keystone/keystone.conf
[database]
connection = mysql+pymysql://keystone:xxx@controller/keystone
[token]
provider = fernet
```

```bash
su -s /bin/sh -c "keystone-manage db_sync" keystone

keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone

keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

keystone-manage bootstrap --bootstrap-password xxx \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

之后配置 Apache 和 wsgi，使 keystone 和 apache httpd 协同工作：

```bash
vim /etc/httpd/conf/httpd.conf
ServerName controller
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
systemctl enable httpd.service
systemctl start httpd.service
```

这里 keystone 工作在 5000 端口，起的 Http 服务器工作在 80 端口。注意，keystone 配置有一个 verbose 选项，其可以输出详细信息，方便调试。`firewall-cmd --zone=public --add-port=80/tcp --permanent; firewall-cmd --reload` 可以看到网页。

## Configure keystone
一个笨方法，先在环境中暴露如下变量，其中包含了用户名 admin、凭证 ADMIN_PASS、项目 Project 名等信息，在上一步 Bootstrap 时已经有了 projectName 和 domainName，因此这里直接使用 Default。
```bash
export OS_USERNAME=admin
export OS_PASSWORD=xxx
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```

一个完整的配置通过 openstack 命令来创建，openstack + domain、project、user 和 role 可创建域、项目、用户和角色，比如：

创建一个叫做 example 的域：
```bash
openstack domain create --description "An Example Domain" example 
openstack domain list
+----------------------------------+---------+---------+--------------------+
| ID                               | Name    | Enabled | Description        |
+----------------------------------+---------+---------+--------------------+
| 2ab8c6c6871741a4940c2f3d04b09b52 | example | True    | An Example Domain  |
| default                          | Default | True    | The default domain |
+----------------------------------+---------+---------+--------------------+
```
为 default 域创建项目 service：
```bash
openstack project create --domain default --description "Service Project" service
openstack project show service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
+-------------+----------------------------------+
```
为 default 域创建项目 myproject：
```bash
openstack project create --domain default --description "Demo Project" myproject
openstack project list
+----------------------------------+-----------+
| ID                               | Name      |
+----------------------------------+-----------+
| 261188ec58fd48888832de67ad7a7896 | admin     |
| 568f952dd3fd4215ac528cb626399042 | service   |
| 8942e736c34548c29a5c9f16707c9d72 | myproject |
+----------------------------------+-----------+
```
为 default 域创建用户 myuser：
```bash
openstack user create --domain default --password-prompt myuser
openstack user show myuser
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| name                | myuser                           |
+---------------------+----------------------------------+
```
创建一个不关联域的角色 myrole：
```bash
openstack role create myrole
openstack role list	openstack role show myrole
+----------------------------------+--------+	+-----------+----------------------------------+
| ID                               | Name   |	| Field     | Value                            |
+----------------------------------+--------+	+-----------+----------------------------------+
| 422374887d294e4596469d11b70de5e3 | admin  |	| domain_id | None                             |
| 8cdd92bf33574f709653f40722bcd328 | reader |	| id        | a9ce71501719426db792b09aab4b96de |
| a9ce71501719426db792b09aab4b96de | myrole |	| name      | myrole                           |
| ad1544b8d76849cc8a7f0eb5f188a390 | member |	+-----------+----------------------------------+
+----------------------------------+--------+
```
为项目 myproject 的 myuser 用户设定 myrole 角色：
```bash
openstack role add --project myproject --user myuser myrole
openstack user list
+----------------------------------+--------+
| ID                               | Name   |
+----------------------------------+--------+
| 40e99e65a3e34e879c14f11c33e81483 | myuser |
| 5f0aaf4f0ead47e58c53a3b23f843511 | admin  |
+----------------------------------+--------+
```
`unset OS_AUTH_URL OS_PASSWORD`

之后可通过 `openstack token issue` 命令来为特定用户 admin、项目 admin 以及用户 myuser 与项目 myproject 获取 Token，其相当于上述 export 后执行 `openstack token issue`，只不过这里将变量全部传入 CLI：
```bash
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name myproject --os-username myuser token issue
```

此外，OpenStack 还可以使用脚本：OpenRC 或 clouds.yaml 此处 来完成上述步骤，比如创建一个 admin-openrc，写入如下左图认证信息，demo-openrc，写入如下右图认证信息：
|  |  |
|--|--|
|export OS_PROJECT_DOMAIN_NAME=Default	|export OS_PROJECT_DOMAIN_NAME=Default|
|export OS_USER_DOMAIN_NAME=Default	|export OS_USER_DOMAIN_NAME=Default|
|export OS_PROJECT_NAME=admin	|export OS_PROJECT_NAME=myproject|
|export OS_USERNAME=admin	|export OS_USERNAME=myuser|
|export OS_PASSWORD=xxx	|export OS_PASSWORD=MYUSER_PASS|
|export OS_AUTH_URL=http://controller:5000/v3	|export OS_AUTH_URL=http://controller:5000/v3|
|export OS_IDENTITY_API_VERSION=3	|export OS_IDENTITY_API_VERSION=3|
|export OS_IMAGE_API_VERSION=2	|export OS_IMAGE_API_VERSION=2|
| | |

在 shell 执行 `. admin-openrc` 后（即获取访问 admin-only CLI 命令），`openstack token issue` 即可获取 AuthenticationToken。

最后不要忘了 `firewall-cmd` 放行 80/tcp 和 5000/tcp 端口（`firewall-cmd --zone=public --add-port=5000/tcp --permanent; firewall-cmd --reload` 如果想 Debug 的话）。在前端，httpd 通过 Python WSGI 接口通过如下管道实现服务：
```bash
[pipeline:api_v3]
pipeline = healthcheck cors sizelimit http_proxy_to_wsgi osprofiler url_normalize request_id build_auth_context token_auth json_body ec2_extension_v3 s3_extension service_v3
```

# Glance 镜像服务
镜像服务允许用户通过 REST API 方式发现、注册和获取 VM 镜像，其支持简单文件存储，S3，HTTP、Vmware datastore、Object Storage 等多种存储方式。如果是 file 类型，则位于 /var/lib/glance/images 下，Glance 服务可以位于 Controller 节点，如果简单使用的话，也可以多 Node 分布式部署。Glance 服务并不像想象中那么简单，其需要支持缓存、复制的集群一致性和高可用，审计等。glance-api 用于提供 API 以进行发现、获取和存储动作，glance-registry 则用于存储、处理和获取镜像的元信息，包括类似镜像类型和大小等，glance-registry 不应该暴露给用户，在 Q 版本中被遗弃，在 S 版本被移除。

创建数据库表：
```bash
mysql -u root -p
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'xxx';
```

创建用户、角色、服务和端点，注意这里的密码是 glance 密码：
```bash
. admin-openrc
openstack user create --domain default --password-prompt glance
openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image" image
openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292
```

安装并配置 glance，其配置位于 `/etc/glance` 文件夹下，提供了 `/usr/bin/glance-api` 和 `glance-cache-xxx` 以及 `glance-control`，`glance-manage`，`glance-registry`，`glance-wsgi-api`，`glancce-replicator` 等二进制程序，以及 `openstack-glance-api` 和 `openstack-glance-registry` 两个位于 `/usr/lib/systemd/system` 的服务，日志位于 `/var/log/glance` 下。

`yum -y install openstack-glance`

修改 `/etc/glance/glance-api.conf`：
```bash
[database]
connection = mysql+pymysql://glance:xxx@controller/glance

[keystone_authtoken] #除此之外，全部注释掉
www_authenticate_uri  = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = xxx

[paste_deploy]
flavor = keystone

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

之后修改  `/etc/glance/glance-registry.conf`：
```bash
[database]
connection = mysql+pymysql://glance:xxx@controller/glance

[keystone_authtoken] #注释掉所有非此处出现的行
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = xxx

[paste_deploy]
flavor = keystone
```

之后来初始化数据库，开机自启和启动 glance 服务：
```bash
su -s /bin/sh -c "glance-manage db_sync" glance
systemctl enable openstack-glance-api.service openstack-glance-registry.service
systemctl start openstack-glance-api.service openstack-glance-registry.service
```

具体 Glance API 使用可参见管理员文档，cirros 地址 `openstack image create --disk-format qcow2 --container-format bare --public --file ./cirros-0.5.2-x86_64-disk.img cirros-0.5.2-x86_64`：
```bash
[root@controller ~]# openstack image list
+--------------------------------------+---------------------+--------+
| ID                                   | Name                | Status |
+--------------------------------------+---------------------+--------+
| 9f35c24a-62c1-48b4-869d-5c15bbac2de8 | cirros-0.5.2-x86_64 | active |
+--------------------------------------+---------------------+--------+
openstack image set --property hw_disk_bus=ide --property hw_vif_model=e1000 3e983592-85xxxxxxxxxx
```

# Nova 计算服务
OpenStack Nova 计算通过和 Identity 交互进行身份验证后，可访问 OpenStack Glace 镜像服务，OpenStack Horizon 仪表盘，其可以在标准硬件上扩展，从镜像启动实例。总的来说，Nova 包含如下部分：
nova-api service	响应终端用户请求，比如创建实例（即各种云的界面管理后台执行的 API）
| | |
|--|--|
|nova-api-metadata service	|响应实例元信息请求|
|nova-conductor module	|用于协调 nova-compute 和数据库解耦	|
|nova-scheduler service	|用于从队列获取虚拟机实例并确定其在哪个服务器主机运行，其包含一系列的 Filter 然后分配 Instance|
|nova-compute service	|	守护进程，用于内部起停 KVM，Vmware，Xen 实例，更新状态到数据库等|
|nova-placement-api service	|用于追踪供应商设备状态和用量 Placement|
|nova-consoleauth daemon	|配合 nova-novncproxy、spicehtml5proxy、xvpvncproxy daemon 实现代理服务|

此外，Nova 还需要 MQ 和 SQL 来传递消息以及存储当前可用实例类型、当前使用的实例、可用的网络和项目 Project 等信息。具体来说，Nova 的服务主要位于 Controller 和 Compute 节点，其中 nova-api，nova-scheduler 等服务位于控制节点，nova-compute 等和虚拟机密切相关的位于计算节点。

![](https://static2.mazhangjing.com/20211013/250e_in12.png)

一个流程大致如下所示：通过 Nova-API 来进行操作，nova-api 负责和 nova-scheduler 交互，后者选择合适的节点，联系其 nova-compute 组件，通过 Compute Manager 和 Compute Driver 来操作底层的 libvirt（virtsh 等），以控制 KVM、Xen 等实现虚拟化实例管理工作。其跨节点交流的方式是通过 AMQP 进行的：

![](https://static2.mazhangjing.com/20211013/1a39_in13.png)
	
对于 nova-scheduler 而言，其工作流程如下所示，一方面，其通过 libvirt 监控虚拟机资源并定时写入 DB，另一方面，其接受新的虚拟机创建请求，经过多个 filter 后找到合适的计算节点，进行创建（同时触发 libvirt 更新数据库操作，以防止信息过时）。

![](https://static2.mazhangjing.com/20211013/174f_in14.png)


## Bootstrap Nova in Controller
首先当然是在 DBMS 创建数据库，并设置用户：
```sql
mysql -u root -p
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
CREATE DATABASE placement;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'xxx';
```

之后 `. admin-openrc` 后，开始使用 openstack 创建用户、角色并分配给项目：
创建一个叫 nova 的用户：
`openstack user create --domain default --password-prompt nova`

将其分配 admin 角色：
`openstack role add --project service --user nova admin`

创建一个叫 nova 的服务 Service，定义类型为 compute：
`openstack service create --name nova  --description "OpenStack Compute" compute`

为这个服务分配端点 EntryPoint：
```bash
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
```
之后为 PlacementAPI 创建用户，添加到 admin 角色，创建服务并定义实现端点：
```bash
openstack user create --domain default --password-prompt placement
openstack role add --project service --user placement admin
openstack service create --name placement --description "Placement API" placement
openstack endpoint create --region RegionOne placement public http://controller:8778
openstack endpoint create --region RegionOne placement internal http://controller:8778
openstack endpoint create --region RegionOne placement admin http://controller:8778
```
之后安装 Nova 核心服务，nova-api 和 nova-api-metadata 都是 /usr/bin 的可执行程序，此外还有 /usr/lib/systemd/system 的 openstack-nova-api/metadata-api/os-compute-api.service 服务。
```bash
yum -y install openstack-nova-api openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler openstack-nova-placement-api
```

修改一个上万行的配置文件：`/etc/nova/nova.conf`，添加如下内容，对应的 XXX_PASS 要填入实际值。
```bash
[DEFAULT]
enabled_apis = osapi_compute,metadata
my_ip = 10.0.0.11
# CHANGE IP HERE
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
transport_url = rabbit://openstack:xxx@controller
# RABBIT_PASS HERE

[api_database]
connection = mysql+pymysql://nova:xxx@controller/nova_api
# NOVA PASS HERE

[database]
connection = mysql+pymysql://nova:xxx@controller/nova
# NOVA_DBPASS HERE

[placement_database]
connection = mysql+pymysql://placement:xxx@controller/placement
# PLACEMENT_DBPASS HERE

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = xxx 
# NOVA PASS HERE

[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = xxx
# PLACEMENT PASS HERE
```

之后追加 /etc/httpd/conf.d/00-nova-placement-api.conf，添加如下内容以访问 PlacementAPI
```xml
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
```
`systemctl restart httpd`

之后执行 nova-api 和 placement 数据库的同步，cell0 和 cell1 的同步：
```bash
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova

su -s /bin/sh -c "nova-manage db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

最后自启和开启后台服务：
```bash
systemctl enable openstack-nova-api.service \
  openstack-nova-consoleauth openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl start openstack-nova-api.service \
  openstack-nova-consoleauth openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
```

如果出错，注意关闭服务（其会无限重试），然后查找 `tailf /var/log/nova/nova-api.log`，查看哪里配置错误了（很容易发生，因为同样条目在多个标签下都可能出现）。

## Configure Nova in Compute Node
因为还需要配 neutron，所以先进行“开始 Compute 节点的 Nova 配置”后。当 neutron 配置完毕后再开始 Compute 节点的 Neutron 配置 以及完成 Compute 节点的全部配置，验证 Nova 到 Controller 的联通。
### 开始 Compute 节点的 Nova 配置
安装 nova-compute 包：
`yum -y install openstack-nova-compute`
配置  `/etc/nova/nova.conf`：
```bash
[DEFAULT] #绿色为在 Controller 节点的 nova.conf 存在的配置
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:xxx@controller
# RABBIT_PASS HERE

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = xxx
# NOVA_PASS HERE

[vnc] #红色为冲突的配置，蓝色为新加的配置
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = xxx
# PLACEMENT_PASS HERE
```

之后查看是否支持虚拟化：`egrep -c '(vmx|svm)' /proc/cpuinfo`，要 非 0，否则调整 `/etc/nova/nova.conf`:
```bash
[libvirt]
virt_type = qemu
```

### 开始 Compute 节点的 Neutron 配置（* 完成 neutron 后）
配置  `/etc/nova/nova.conf`：
```bash
[neutron] #这部分在 Controller 中的配置 在 Neutron 安装时添加 Neutron Service 后写入（见下文）
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = xxx
# NEUTRON_PASS
```

安装 linuxbridge ebtables 和 ipset，
`yum -y install openstack-neutron-linuxbridge ebtables ipset`

如果是 OVS，则不安装 linuxbridge agent，而安装：
`yum -y install openstack-neutron-openvswitch ebtables ipset`

配置 `/etc/neutron/neutron.conf`：
```bash
[DEFAULT]
transport_url = rabbit://openstack:mi960032@controller
# RABBIT_PASS
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = xxx
# NEUTRON_PASS

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
# OVS 注释掉此行
```

之后根据网络类型，配置  `/etc/neutron/plugins/ml2/linuxbridge_agent.ini` 文件：
```bash
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

[vxlan]
enable_vxlan = true
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
如果是 OVS，配置 `/etc/neutron/plugins/ml2/openvswitch_agent.ini` 文件：
```bash
[agent]
tunnel_types = vxlan
l2_population = True
prevent_arp_spoofing = True

[ovs]
local_ip = OVERLAY_IP_MAY_MANGAGEMENT_IP #for Tunnel network type
bridge_mappings = provider:br-ex #for VLAN and flat

[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

同时将内核参数调整：
```
net.bridge.bridge-nf-call-iptables
net.bridge.bridge-nf-call-ip6tables
```
对于 OVS，配置：
```
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```

### 完成 Compute 节点的 Neutron 和 Nova 配置（* 完成 neutron 后）
重启 nova-compute，设置 neutron-linuxbridge-agent 自启和启动即可：

```bash
systemctl restart openstack-nova-compute.service
systemctl enable neutron-linuxbridge-agent.service
systemctl start neutron-linuxbridge-agent.service
```

如果是 OVS，则重启计算服务，将 OVS-agent 设置为自启和启动即可：
```bash
systemctl restart openstack-nova-compute.service
systemctl enable neutron-openvswitch-agent.service
systemctl start neutron-openvswitch-agent.service
```

最后开机并启动 libvirtd 和 openstack-nova-compute 服务：
```bash
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
```

### 验证和注册 Nova 到 Controller（* 完成 neutron 后）
最后一步，检查 Controller 是否发现了 Compute 节点，并更新 cell_v2 数据库，或者可以配置 /etc/nova/nova.conf 的 `scheduler;discover_hosts_in_cells_interval=300` 来开启自动发现（否则每个新加入的 Compute 节点都需要 `nova-manage cell_v2 discover_hosts`）。
```bash
$ . admin-openrc
$ openstack compute service list --service nova-compute
# su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
OR ============================================================================
[scheduler]
discover_hosts_in_cells_interval = 300
```

此外，检查 `openstack network agent list` 查看 OVS 或 LinuxBridge 是否工作正常。

# Neutron 网络
Neutron 是 OpenStack 的网络组件，其用来通过 neutron-server 组件接受传入 API 请求，并进行动作以为 OpenStack Compute（Nova）实例提供网络和连通性 —— 其中插件 Plugin 作为提供创建网络或子网（ML2），提供 IP 地址（L3）的接口，比如最常用的 ML2 插件（用于在节点大二层间进行链路，比如 VLAN，VxLAN，GRE 技术的 LinuxBridge，OVS 实现），此外 OpenStack 支持多种不同厂商规格的插件扩展，有比如 Cisco Virtual 和 Physical Switches，NEC OpenFlow，Open vSwitch，Linux bridging，Vmware NSX 等。插件提供的标准由代理 Agent 实现，比如 L3 代理（L3 路由服务，通过 Linux ns 隔离的 Router 实现路由，安装在计算节点）、DHCP 代理（DNSmasq 提供 DHCP 服务，安装在网络节点）和元数据代理（cloud-init 之类服务，安装在网络节点）。Neutron 也使用 MQ，其用来在 neutron-server 的各种 plugin 和 agents 之间通信，以及存储相关信息。
	
插件和代理的逻辑划分比较清晰：插件提供 API，通过 AMQP 和代理沟通，代理负责实现，如右图所示。但是实际部署上比较灵活，如下图所示，Network 节点一般部署 ML2 插件用于响应 neutron-server 的 L2 API 请求，L3 代理用于处理 neutron-server 的 L3 路由，DHCP 和 METADATA 代理等，计算节点一般部署 ML2 代理，用于实际操作 OVS 或 LinuxBridge 以实现网络拓扑变更。	
总的来说，neutron 负责管理 VNI（Virtual Networking Infrastructure）和 PNI（Physical Networking Infrasturcture）的访问层（Access layer）方面，可为项目提供高级虚拟网络拓扑，比如防火墙、负载均衡和 VPN 等服务。从实现上看，其通过抽象网络 Network、子网 Subnet 和路由 Router 实现虚拟，路由用于在子网和网络间通信，neutron 至少包含一个外部网络（物理），以用来访问其中的资源，以及若干个内部网络，用来进行虚拟机之间的通信，外部网络访问虚拟机需要通过路由器（路由器有一个网关连接到外部网络，一个网关连接到内部网络），同样，虚拟机通过路由访问外部网络。如果将外部网络 IP 地址绑定内部网络端口，那么就可以在外部网络访问 VM 对应端口应用（所谓浮动 IP：Float IP）。此外，还可定义安全组和防火墙规则（通过 iptables），以阻止或放行端口、流量类型或特定范围端口。每个 neutron 的插件都有自己的概念，比如有的提供安全组插件，有的 NoOp，有的提供 FWaaS 防火墙即服务和 LBaaS 负载平衡器即服务。

> 注意，Linux 作为路由器有一些需要进行配置：如果 sysctl 启用了 net.ipv4.ip_forward 后，内核将转发非本设备地址的流量，而不是直接丢弃。默认情况下 Linux 内核如果不能确定数据包的源路由，其将会丢弃数据包（为了防止 DDOS 的非对称路由），但在 OpenStack 环境中，则一般需要禁用非源路径过滤，让 OpenStack 管理：net.ipv4.conf.all/default.rp_filter 可对现在/未来所有接口禁用反向路径过滤。

Neutron 的结构较为复杂，部署灵活，配置文件众多，但是大致关系如下所示，其中位于 Controller 的 neutron.conf 负责为 Neutron Server & Plugin 服务提供联系其他服务的密码（比如 nova），联系数据库和 MQ 的凭证，以及提供自身服务的 url。类似的还有 Compute 节点的 nova.conf 为 nova-compute 提供服务。L2 Agent 和 L3 Agent 可位于网络节点 or 控制节点，其使用主 neutron.conf 中的 MQ 凭证通过 MQ 和 Neutron Server 进行通信，此外有自己独立的 xxx_agent.ini 配置文件，用于具体干活，比如 interface_driver 直接绑定网卡，xxx_range 等。对于 L3 Agent，一般在 Network/Controller 节点，对于 L2 Agent，一般 Network/Controller 和 Compute 节点都有一份，当然配置文件也都要各有一份。注意，从 Agent 独立出的 ml2_conf.ini 这个 Plugin 配置文件仅有 Controller 节点的 Neutron Server & Plugins 有，在 Network 和 Compute 节点虽然包会安装，但是一般不配置（或者压根没有），这里定义的内容会通过 Neutron Server 通过 MQ 下发给各 Agent，因此各 Agent 只需要 ① 总的 neutron.conf 以连接 MQ，③/④ 具体的 Agent 实现以执行 MQ 下发的来自 Neutron Server & Plugin 的 ② 配置即可。

![](https://static2.mazhangjing.com/20211013/07ba_in15.png)

## 在 Controller/Network 节点初始化 Neutron
> Note:如果是在 LinuxBridge 和 OVS 之前切换，则 DROP DATABASE neutron; 之后再新建数据库并设置用户。
首先还是配置 DBMS 数据库：
```sql
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'xxx';
```

导入管理员 CLI 权限：
`. admin-openrc`
创建 neutron 的 OpenStack 用户，将其加入 admin 角色：
```bash
openstack user create --domain default --password-prompt neutron
openstack role add --project service --user neutron admin
```
然后创建 neutron 的 network 类型服务：
```bash
openstack service create --name neutron --description "OpenStack Networking" network
```
为此服务定义端点：
```bash
openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696
```

对于 Self-Service Network 而言，在 Controller 需要安装 neutron、neutron-ml2、neutron-linuxbridge、ebtables 组件，如果是 OVS，则安装 openvswitch-agent 组件：
`yum -y install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables `

如果使用 OVS 那么需要安装 openvswitch agent：
`yum -y install openstack-neutron-openvswitch`

然后修改 /etc/neutron/neutron.conf 配置文件：
```bash
[database]
connection = mysql+pymysql://neutron:xxx@controller/neutron
# NEUTRON_DBPASS

[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true

transport_url = rabbit://openstack:xxx@controller
# RABBIT_PASS

auth_strategy = keystone

notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = xxx
# NEUTRON_PASS

[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = xxx
# NOVA_PASS

[oslo_concurrency]|
lock_path = /var/lib/neutron/tmp
# 如果是 OVS，则注释掉此句
```

之后配置 /etc/neutron/plugins/ml2/ml2_conf.ini 文件：
```bash
[ml2]
type_drivers = flat,vlan,vxlan
# After you configure the ML2 plug-in, removing values in the type_drivers option can lead to database inconsistency.

tenant_network_types = vxlan

mechanism_drivers = linuxbridge,l2population
# 如果是 OVS，则启用如下配置：
mechanism_drivers = openvswitch,l2population

extension_drivers = port_security

[ml2_type_flat]

flat_networks = provider

[ml2_type_vxlan]
vni_ranges = 1:1000

[securitygroup]
enable_ipset = true
```

如果是 Linux bridge agent，修改 `/etc/neutron/plugins/ml2/linuxbridge_agent.ini`：
```bash
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

[vxlan]
enable_vxlan = true
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

如果是 OVS，修改 /etc/neutron/plugins/ml2/openvswitch_agent.ini:
[agent]
tunnel_types = vxlan
l2_population = True
prevent_arp_spoofing = True
# 官方文档中没有 prevent_arp_spoofing

[ovs]
local_ip = OVERLAY_IP_HERE_MAY_MANAGEMENT_IP
bridge_mappings = provider:br-ex

[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

之后 `vi /etc/sysctl.conf` 进行如下配置，启动 `br_netfilter kernel module`，可能需要 modprobe br_netfilter，然后 `sysctl -p /etc/sysctl.conf` 使其生效。
```
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```
对于 OVS，配置：
```
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```

之后配置 `/etc/neutron/l3_agent.ini` L3 层以启用路由和 NAT：
```bash
[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
# 如果是 OVS，则修改为：
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
# or interface_driver = openvswitch
```

之后配置 DHCP agent，修改 `/etc/neutron/dhcp_agent.ini`，然后修改：
```bash
[DEFAULT]
interface_driver = linuxbridge
# or interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
# 如果是 OVS，则修改为：
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

之后配置 metadata agent，修改 `/etc/neutron/metadata_agent.ini`，然后修改：
```bash
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = xxx
# METADATA_SECRET HERE
```

最后修改 `/etc/nova/nova.conf` 文件，然后修改：
```bash
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = xxx
# NEUTRON_PASS
service_metadata_proxy = true
metadata_proxy_shared_secret = mi960032
# METADATA_SECRET
```

最后，检查 `/etc/neutron/plugin.ini` 是否指向 `/etc/neutron/plugins/ml2/ml2_conf.ini`，如果没有则创建：
`ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini`

将配置文件应用更改到数据库：
```bash
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

重启 nova 服务：`systemctl restart openstack-nova-api.service`

如果是 Linux Bridge 实现的，则使用如下步骤，自启 neutron-server、neutron-linuxbridge-agent、neutron-dhcp-agent、neutron-metadata-agent 服务，然后将这些服务启动：

```bash
systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
systemctl start neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
```

如果是 OVS 实现的，则使用如下步骤：
```bash
systemctl enable neutron-server.service \
  neutron-openvswitch-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
systemctl start neutron-server.service \
  neutron-openvswitch-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
```

最后自启和启动 neutron-l3-agent 服务：
```bash
systemctl enable neutron-l3-agent.service
systemctl start neutron-l3-agent.service
```

对于 OVS 而言，则还要新建上述所需的 br-ex 网桥，将其连接到外部网卡，然后重启服务：
```bash
ovs-vsctl add-br br-ex
ovs-vsctl add-port br-ex ensXX #外部网卡

systemctl restart neutron-server.service \
  neutron-openvswitch-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service neutron-l3-agent.service
```

## 在 Compute 节点配置 Neutron
配置好 Neutron 在 Networking 节点后，配置其在 Compute 节点并继续完成 Nova 的 Compute 配置。

# Horizon
Horizon 仪表盘需要 Identity Service - Keystone，可独立搭配 Object Storage 等使用，也可以配合 Image Service、Compute、Networking 服务使用，其本质是一个 Django Web App。

`yum -y install openstack-dashboard` 

然后配置 `/etc/openstack-dashboard/local_settings` 文件，设置 OPENSTACK 的 Controller 地址，此外可以添加允许访问仪表盘的地址，['*'] 表示所有。可配置 memcached 缓存，设置 KEYSTONE URL、默认 DOMAIN、默认 ROLE，NEUTRON 的配置、时区、多域名支持、各个组件的 API 版本。最后让 django 配合 httpd 工作：修改 `/etc/httpd/conf.d/openstack-dashboard.conf` 添加少许配置即可。最后重启 memcached 和 httpd 即可（参见文档）。

具体参见 [OpenStack Rocky 官方文档 Horizon 配置](https://docs.openstack.org/horizon/rocky/install/install-rdo.html)

> 对于 Neutron Fwaas、Lbaas 以及其 Horizon 插件的安装见其它博文。