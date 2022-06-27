# KubeEdge架构及其核心组件


## 核心理念

- 云边协同
  - 双向多路复用消息通道，支持边缘节点位于私有网络（无公网IP环境）
  - Websocket+消息封装，大幅减少通信压力，高时延下仍可正常工作
- 边缘离线自治
  - 节点元数据持久化，实现节点级离线自治
  - 节点故障恢复无需List-watch，降低网络压力，快速ready
- 极致轻量
  - 重组Kubelet功能模块，极致轻量化（~10mb内存占用）
    - 移除内嵌存储驱动，通过CSI接入
  - 支持CRI集成Containerd、CRI-O，优化runtime资源消耗

## 架构

<div align=center><img src="/KubeEdge_Arch_And_Core_Component/kubeedge_arch.png" alt="kubeedge_arch" style="zoom: 33%;" /></div>

KubeEdge总体由云上部分（CloudCore）和边缘部分（EdgeCore）构成:

### 云上组件

<div align=center><img src="/KubeEdge_Arch_And_Core_Component/kubeedge_cloud.png" alt="kubeedge_cloud" style="zoom: 33%;" /></div>

- EdgeController（扩展的K8s控制器）
  - 边缘节点管理
  - 应用状态元数据云边同步
- 设备抽象API/DeviceController（扩展的K8s控制器）
  - 接入和管理边缘设备
  - 设备元数据（信息、状态）云边同步

- CSI Driver
  - 同步存储数据到边缘
- Admission Webhook
  - 校验进入KubeEdge对象的合法性

- CloudHub
  - WebSocket服务端，监听云端变化，缓存并发送消息到EdgeHub

### 边缘组件

<div align=center><img src="/KubeEdge_Arch_And_Core_Component/kubeedge_edge.png" alt="kubeedge_edge" style="zoom: 33%;" /></div>

- EdgeHub
  - WebSocket客户端，负责与云端（EdgeController）交互，同步云端资源更新；并向上报告边缘主机和设备状态变化
- MetaManager
  - 消息处理器，位于 Edged 和 Edgehub 之间，它负责向轻量级数据库 (SQLite) 存储/检索元数据
- DeviceTwin
  - 负责存储设备状态（开关值、传感器值等）到EdgeStore并将设备状态同步到云，另外为应用程序提供查询接口
- EventBus
  - 与MQTT服务器（mosquitto）交互的MQTT客户端，为其他组件提供订阅和发布功能
- ServiceBus
  - 运行在边缘的 HTTP 客户端，接受来自云上服务的请求，与运行在边缘端的 HTTP 服务器交互，提供了云上服务通过 HTTP 协议访问边缘端 HTTP 服务器的能力
- Edged（Kubelet-lite）
  - 轻量的Kubelet，用于管理容器化的应用程序

## 关键能力

### 支持CRI接口

<div align=center><img src="/KubeEdge_Arch_And_Core_Component/kubeedge_abi_cri.png" alt="kubeedge_abi_cri" style="zoom: 33%;" /></div>

早期KubeEdge集成了Docker Client，v1.0之后加入CRI，可自行选择容器运行时

- 通过CRI，把相关组件和容器运行时进行解耦，可实现选择适合自己的容器运行时

### 支持CSI接口

<div align=center><img src="/KubeEdge_Arch_And_Core_Component/kubeedge_abi_csi.png" alt="kubeedge_abi_csi" style="zoom: 33%;" /></div>

Kubernetes的数据存储在云上数据中心，每个节点都能轻松访问到，并且组件与Master之间的交互采用List、Watch机制同步数据。

以云边协同为基础的KubeEdge部分节点在云上，部分节点在边缘，以List、Watch机制同步数据需要跨越云边实现List、Watch，开销很大。

KubeEdge的做法是：在云上使用CSI Driver from KubeEdge做劫持，通过CloudHub发送到边缘，真实的存储后端实际上分布在边缘。

### 边缘设备管理

<div align=center><img src="/KubeEdge_Arch_And_Core_Component/kubeedge_abi_device_manager1.png" alt="kubeedge_abi_device_manager1" style="zoom:33%;" /></div>

- DeviceModel：定义设备通用的属性字段，是否只读，是否需要做数据处理
- Device：定义需要接入的设备实体，会从DeviceModel继承属性字段，只要配置设备访问方式等就能实现与设备的交互

<div align=center><img src="/KubeEdge_Arch_And_Core_Component/kubeedge_abi_device_manager2.png" alt="kubeedge_abi_device_manager2" style="zoom:33%;" /></div>

- 南向Mapper组件：把设备的其他协议（OPC-UA、Modbus等）转换成MQTT协议，推荐每个边缘节点以Demonset形式部署Mapper

### EdgeMesh:ServiceMesh At Edge

#### EdgeMesh作用

实现简单的服务发现和流量代理功能，从而屏蔽了边缘场景下复杂的网络结构

#### EdgeMesh实现原理

- 利用P2P技术，来打通边缘节点间网络
- P2P打洞不成功情况下，通过Server中继转发

### EdgeSite

<div align=center><img src="/KubeEdge_Arch_And_Core_Component/kubeedge_edgesite-arch.png" alt="kubeedge_edgesite-arch" style="zoom:33%;" /></div>

#### EdgeSite作用

KubeEdge默认在云端部署EdgeController和DeviceController，然后通过Websocket/Quic隧道连接云端和边缘端，通过云端一个中心来统一调度应用到特定Edge Node上运行。但是，就像Rancher K3s的应用场景，有些时候边缘端也希望运行一套完整的K8s集群，K3s的方案只是提供了一套精简的K8s集群，而Kubeedge的EdgeSite模式，除了运行K8s集群之外，还提供了对IoT设备的适配和支持。

有些场景用户需要在边缘运行一个独立的 Kubernetes 集群来获得完全控制并提高离线调度能力，有两种情况用户需要这样做：

- CDN场景

CDN 站点通常遍布全球，无法保证边端节点间网络连接和质量；另一个因素是部署在 CDN 边缘的应用程序通常不需要与中心交互。对于那些在 CDN 资源中部署边缘集群的人来说，他们需要确保集群在没有与中央云连接的情况下也能正常工作。

- 用户需要部署资源有限且大部分时间离线运行的边缘环境

在一些物联网场景中，用户需要部署一个边缘环境并离线运行。

对于这些用例，需要一个独立的、完全受控的、轻量级的 Edge 集群。通过集成 KubeEdge 和标准 Kubernetes，这个 EdgeSite 使客户能够运行一个高效的 Kubernetes 集群来进行 Edge/IOT 计算。

From:[https://www.bookstack.cn/read/kubeedge/1171574c81cee03f.md](https://www.bookstack.cn/read/kubeedge/1171574c81cee03f.md)

