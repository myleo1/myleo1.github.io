# KubeEdge组件之EdgeMesh


## 背景

### 云原生容器网络发展阶段

<img src="/KubeEdge_Component_Of_EdgeMesh/edgeMesh_background_cloudnative_network.png" alt="edgeMesh_background_cloudnative_network" style="zoom:35%;" />

- 第一阶段：Docker 容器网络，在 Docker 出现之后，也有自己的 4 种容器网络模型，比如 Host 模式、Content 模式，None 模式以及 Bridge 模式。但原生 Docker 容器无法解决容器之间的跨级通信问题，后来 Docker 推出 CNM 以及对应的实现 libnetwork 解决了这个问题。
- 第二阶段：容器网络接口(CNI), 后来由于各种原因 Kubernetes 主推的 CNI 热度反超了 CNM，CNI 是一个接口更简单、而且兼容性更高的容器网络接口规范。
- 第三阶段: 服务网格 + CNI, 随着服务网络的发展，它与 CNI 进行了配合，一些服务网格的插件，会在每个 pod 启动时往 pod 里注入 Sidecar 代理，来提供 4 层或 7 层的流量治理功能。

### Kubernetes服务发现

Kubernetes 服务发展其实与容器网络是依赖的关系，首先用户会通过 Deployment 创建一组应用的后端实例对其进行管理。但 Pod 的生命周期是非常短暂的，可能会随着 Pod 更新升级或迁移等，它的 Pod IP 都会发生变化，这个现象称之为 Pod IP 的漂移。

为了解决这个问题, Kubernetes 社区提出了 Service 的概念，每个 Service 都会对应到一组后端的一个应用实例上，通过提供恒定不变的 Cluster IP 来解决 Pod IP 漂移的问题，同时也提供了 proxy 组件，基于控制面提供的一些信息维护 Cluster IP 到 Pod IP 的转换规则。当 Client 需要访问该服务时，一般只需要访问这个不变的 Cluster IP 即可。这个流量会经过本机的网络栈，被替换成了 mysql 后端实例的 Pod IP，通过容器网络请求发送。

<img src="/KubeEdge_Component_Of_EdgeMesh/edgemesh_service_discovery.png" alt="edgemesh_service_discovery" style="zoom: 33%;" />

### 边缘场景下的挑战

- 边缘计算细分领域众多，互操作性差
- 边云通信网络质量低，时延高，且边缘经常位于私有网络，难以实现双向通信
- 边缘资源受限，需要轻量化的组件管理运行边缘应用
- 边缘离线时，需要具备业务自治和本地故障恢复等能力
- 边缘节点高度分散，如何高效管理，降低运维成本
- 如何对异构资源进行标准化管理和灵活配置

以上这些关键挑战，边缘计算平台 KubeEdge 均可以实现，也解决了基本上所有的问题。但除了这些问题外，还有一些其他问题，举个例子：比如边缘有一个视频流应用，需要与云上的 AI 应用进行交互。首先边缘的网络是无法直接与云上网络互相连通的，所以无法从边缘对云上进行访问。不过这个问题其实可以通过给云上的 AI 应用配置一个公网 IP 来解决，但如果云上的每一个应用都配置一个公网 IP，那 IP 将无法收敛。而且云上的应用想要主动访问边缘的应用，其实也是做不到的，因为边缘上的应用一般都处于**私网**里，它一般不会有公共 IP，所以就无法做到正确路由和双向通信。

<img src="/KubeEdge_Component_Of_EdgeMesh/edgemesh_problem.png" alt="edgemesh_problem" style="zoom: 50%;" />

**总的来说，有以下几个问题：**

1. 边云网络割裂，微服务之间无法跨子网直接通信
2. 边缘侧缺少服务发现能力
3. 边缘环境下组网配置管理复杂

基于以上的问题，EdgeMesh 应运而生。

## EdgeMesh

### 定义

EdgeMesh 作为 KubeEdge 集群的数据面组件，为应用程序提供了简单的服务发现与流量代理功能，从而屏蔽了边缘场景下复杂的网络结构。

### 设计原则

**轻量化:** 每个节点仅需部署一个极轻的代理组件，边缘侧无需依赖 CoreDNS、Kube-Proxy 和 CNI 插件等原生组件

**云原生体验:** 为 KubeEdge 集群中的容器应用提供与云原生一致的服务发现与流量转发体验

**跨子网通信:** 屏蔽复杂的边缘网络环境，提供容器间的跨子网边边和边云通信能力

**高可靠性:** 通过打洞建立点对点直连，转发效率极高；在不支持打洞时通过中继转发流量，保障服务之间的正常通讯

**分层式架构:** 采用分层式设计架构，各模块能够与原生组件兼容并支持动态关闭

### 实现功能

<img src="/KubeEdge_Component_Of_EdgeMesh/edgemesh_func.png" alt="edgemesh_arch" style="zoom:50%;" />

### 架构

<img src="/KubeEdge_Component_Of_EdgeMesh/edgemesh_arch.png" alt="edgemesh_arch" style="zoom: 50%;" />

#### EdgeMesh-Agent

- Proxier：负责配置内核的 iptables 规则，将请求拦截到 EdgeMesh 进程内
- DNS：内置的 DNS 解析器，将节点内的域名请求解析成一个服务的集群 IP
- Traffic：基于 Go-Chassis 框架的流量转发模块，负责转发应用间的流量
- Controller：通过 KubeEdge 的边缘侧 Local APIServer 能力获取 Services、Endpoints、Pods 等元数据
- Tunnel-Agent：利用中继和打洞技术来提供跨子网通讯的能力

#### EdgeMesh-Server

- Tunnel-Server：与 EdgeMesh-Agent 建立连接，协助打洞以及为 EdgeMesh-Agent 提供中继能力

### 工作流程

<img src="/KubeEdge_Component_Of_EdgeMesh/edgemesh_working_process.png" alt="edgemesh_working_process" style="zoom: 45%;" />

### 实现原理

- 利用P2P打洞技术，来打通边缘节点间的网络
- 将边缘节点间的通信分为局域网和跨局域网
  - 局域网内的通信：直接通信
  - 跨局域网通信：打洞成功时Agent之间建立直连通道，否则通过Server中继转发
- 离线场景下通过EdgeMesh内部实现轻量级DNS服务器，域名请求在节点内闭环
- 极致轻量化，每个节点只有一个Agent

#### P2P打洞实现（NAT）

<img src="/KubeEdge_Component_Of_EdgeMesh/edgemesh_nat.png" alt="edgemesh_nat" style="zoom:50%;" />

##### NAT类型

1. 锥型
   - 完全锥形（NAT1）：任何一个外部主机发送到（eAddr:ePort）的报文将会被转换后发送到（iAddr:iPort）
   - 限制锥形（NAT2）：只有（iAddr:iPort）向特定的外部主机hAddr发送过数据，主机hAddr从任意端口发送到（eAddr:ePort）的报文将会被转发到（iAddr:iPort）
   - 端口限制锥形（NAT3）：只有（iAddr:iPort）向特定的外部主机端口对（hAddr:hPort）发送过数据，由（hAddr:hPort）发送到（eAddr:ePort）的报文将会被转发到（iAddr:iPort）
2. 对称型
   - 对称NAT（NAT4）：对称式NAT把内网IP和端口到相同目的地址和端口的所有请求，都映射到同一个公网地址和端口；同一个内网主机，用相同的内网IP和端口向另外一个目的地址发送报文，则会用不同的映射（比如映射到不同的端口）。

注：NAT类型与运营商相关

##### 结论

- 穿透能力只与NAT类型强相关，从上到下穿透性下降，安全性上升
- 一端是对称型，另一端是对称型或端口限制型则无法穿透

关于NAT，更详细的介绍请戳：[https://zhuanlan.zhihu.com/p/335253159](https://zhuanlan.zhihu.com/p/335253159)

### 官方性能测试

<img src="/KubeEdge_Component_Of_EdgeMesh/edgemesh_benchmark_1.png" alt="edgemesh_benchmark_1" style="zoom: 40%;" />

<img src="/KubeEdge_Component_Of_EdgeMesh/edgemesh_benchmark_2.png" alt="edgemesh_benchmark_2" style="zoom:40%;" />

<img src="/KubeEdge_Component_Of_EdgeMesh/edgemesh_benchmark_3.png" alt="edgemesh_benchmark_3" style="zoom:40%;" />

<img src="/KubeEdge_Component_Of_EdgeMesh/edgemesh_benchmark_4.png" alt="edgemesh_benchmark_4" style="zoom:40%;" />

### 目前存在的问题

- EdgeMesh目前不支持多点部署，在某些情况下会出现问题

  - 情况1：

  <img src="/KubeEdge_Component_Of_EdgeMesh/edgemesh_problems_1.png" alt="edgemesh_problems_1" style="zoom:50%;" />

  当连接数过大或通信流量过大时，都会导致edgemesh-server的单点故障问题

  - 情况2：

  <img src="/KubeEdge_Component_Of_EdgeMesh/edgemesh_problems_2.png" alt="edgemesh_problems_2" style="zoom:50%;" />

  edgemesh-server 的位置会影响流量转发的延迟。如果中继服务器的位置太远，会大大增加延迟。

  - 情况3：

    <img src="/KubeEdge_Component_Of_EdgeMesh/edgemesh_problems_3.png" alt="edgemesh_problems_3" style="zoom:50%;" />

在某些私有网络的情况下，edgemesh-agent 无法连接到外网的 edgemesh-server，从而导致 edgemesh-agent 无法正常工作。

### EdgeMesh RoadMap

**2021.06：** EdgeMesh 项目开源

**2021.09：** EdgeMesh 支持微服务跨局域网边云/边边通信

**2021.12：** EdgeMesh 支持一键化部署

**2022.03：** EdgeMesh 支持 Pod IP 的流量跨边云转发

**2022.06：** EdgeMesh 对接标准的 Istio 进行服务治理控制

**2022.09：** EdgeMesh 支持跨集群服务通信

