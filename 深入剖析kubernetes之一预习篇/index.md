# 深入剖析Kubernetes之一（预习篇）


## 01 | 预习篇 · 小鲸鱼大事记（一）：初出茅庐

Cloud Foundry开启PaaS为核心构建平台层服务能力的变革

PaaS在当时被接纳的主要原因是提供了“应用托管”的能力，当时用户普遍用法是租AWS或者OpenStack虚拟机，像管理物理机一样，用脚本/手工方式部署应用，导致了云端环境和本地环境不一致，PaaS项目通过打包和分发机制，Cloud Foundry为每种主流语言都定义了一种打包格式，使用cf push命令上传到云上Cloud Foundry存储中，然后通过调度器将应用调度到虚拟机上启动，并且也提供Cgroups和Namespace机制为应用单独创建一个“沙盒”的隔离环境——**这，正是 PaaS 项目最核心的能力。** 而这些 Cloud Foundry 用来运行应用的隔离环境，或者说“沙盒”，就是所谓的“容器”

dotCloud开源Docker容器项目

Docker项目的实现，本质也是同样使用Cgroup和Namespace实现的"沙盒"，但是Docker有着另外的功能——Docker镜像

凭借着Docker镜像的打包方式比PaaS平台便捷的优点：

1. 无需为每种语言，每个框架维护一个打好的包
2. 无需做很多修改和配置工作才能运行起来

在2014年迅速占领了所有云计算领域头条技术

## 02 | 预习篇 · 小鲸鱼大事记（二）：崭露头角

 Docker 项目在短时间内迅速崛起的三个重要原因：

1. Docker 镜像通过技术手段解决了 PaaS 的根本性问题；
2. Docker 容器同开发者之间有着与生俱来的密切关系；
3. PaaS 概念已经深入人心的完美契机。

## 03 | 预习篇 · 小鲸鱼大事记（三）：群雄并起

CoreOS 是一个基础设施领域创业公司。 它的核心产品是一个定制化的操作系统，用户可以按照分布式集群的方式，管理所有安装了这个操作系统的节点。从而，用户在集群里部署和管理应用就像使用单机一样方便了。

Docker 项目发布后，CoreOS 公司很快就认识到可以把“容器”的概念无缝集成到自己的这套方案中，从而为用户提供更高层次的 PaaS 能力。所以，CoreOS 很早就成了 Docker 项目的贡献者，并在短时间内成为了 Docker 项目中第二重要的力量。

然而，这段短暂的蜜月期到 2014 年底就草草结束了。CoreOS 公司以强烈的措辞宣布与 Docker 公司停止合作，并直接推出了自己研制的 Rocket（后来叫 rkt）容器。

这次决裂的根本原因，正是源于 Docker 公司对 Docker 项目定位的不满足。Docker 公司解决这种不满足的方法就是，让 Docker 项目提供更多的平台层能力，即向 PaaS 项目进化。而这，显然与 CoreOS 公司的核心产品和战略发生了严重冲突。

相较于 CoreOS 是依托于一系列开源项目（比如 Container Linux 操作系统、Fleet 作业调度工具、systemd 进程管理和 rkt 容器），一层层搭建起来的平台产品，Swarm 项目则是以一个完整的整体来对外提供集群管理功能。而 Swarm 的最大亮点，则是它完全使用 Docker 项目原本的容器管理 API 来完成集群管理，比如：

- 单机 Docker 项目：

```shell
$ docker run "我的容器"
```

- 多机 Docker 项目：

```shell
$ docker run -H "我的 Swarm 集群 API 地址" "我的容器"
```

所以在部署了 Swarm 的多机环境下，用户只需要使用原先的 Docker 指令创建一个容器，这个请求就会被 Swarm 拦截下来处理，然后通过具体的调度算法找到一个合适的 Docker Daemon 运行起来。

后来，Docker又对Fig项目进行收购，Fig项目是第一次提出“容器编排”的项目，衍生出了Compose。

直到Google Kubernetes项目的开源...

## 04 | 预习篇 · 小鲸鱼大事记（四）：尘埃落定

随着Docker项目的爆火，大量网络、存储、监控、CI/CD甚至UI项目纷纷出台，也涌现了很多Rancher、Tutum这样在开源与商业上均取得了巨大成功的创业公司。

Docker项目兴起的时候，Google也推出了经历过生产环境验证的Linux容器：Imctfy，然而面对Docker的崛起，Google向Docker表示了合作愿望，共同推进container runtime项目作为Docker项目的核心依赖，但Docker公司并没有认同，在不久后发布了容器运行时库Libcontainer，但匆忙的重构导致了代码阅读性差、可维护性不强的缺点，开始让社区叫苦不迭。

于是，2015年由Docker牵头，将Libcontainer捐出，衍生出RunC的项目，交由一个完全中立的基金会管理，以RunC为依据，大家共同制定一套容器和镜像的标准和规范。这就是OCI（Open Container Initiative），OCI旨在将容器运行时和镜像的实现从Docker项目中完全剥离出来，改善了Docker在技术上一家独大的现状，给其他玩家提供了不依赖于Docker构建各自平台层能力提供了可能。

但Docker并不担心与OCI的威胁，也因本身Docker已经是容器生态的事实标准而无心推动标准建设。于是Google、RedHat等共同牵头发起了名为CNCF（Cloud Native Computing Foundation）的基金会，以Kubernetes为基础，建立一个独立基金会主导的平台级社区来对抗Docker公司为核心的容器商业生态。

面对Kubernetes社区的崛起和壮大，Docker放弃了开源社区而专注于商业化转型，从2017年开始，Docker将容器运行时部分Containerd捐赠给CNCF社区，并将Docker项目改名为Moby，交由社区维护。

