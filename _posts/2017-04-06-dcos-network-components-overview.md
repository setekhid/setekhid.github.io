---
layout: post
title: "DC/OS 1.8 网络组件介绍"
categories: dcos
author: "Huitse Tai"
---

本 overview 有效期有限，据 slack 上一位兄弟言，1.8的 VIPs 由 dcos-minuteman 实现，1.9为 dcos-navstar ，暂不清楚。

[toc]

## DNS

DC/OS 提供三层 DNS 服务，主要用于对集群内部进行服务发现。

### Mesos-DNS

[Mesos-DNS](http://mesosphere.github.io/mesos-dns/) 是 DC/OS 中负责进行服务 DNS 解析的组件，能够将诸如 `hello-service.marathon-framework.mesos` 的域名通过标准 DNS 查询，解析为服务的 IP 地址（A Record）和服务的端口（[SRV Record](https://en.wikipedia.org/wiki/SRV_record)）。

![mesos-dns-stream](http://mesosphere.github.io/mesos-dns/img/architecture.png)

上图来自 mesosphere，描述了 Mesos-DNS 项目的内部结构。Record Generator 扮演 fetcher 的角色，定时从 MesosMaster 拉取集群信息及任务信息，解析后生成相应的 DNS 记录。DNS Resolver 响应 DNS 查询请求。

Mesos-DNS 部署在 DC/OS 集群的所有 Master 节点上。

### Navstar-DNS

[Navstar-DNS](http://github.com/dcos/navstar) 负责进行对 Mesos-DNS 进行代理查询，并对内提供 DNS 域名动态注册服务。

### Spartan

由于 Mesos-DNS 要定时从 Mesos Master 拉取集群信息，部署实例数不能随 DC/OS 集群规模增长。此时 DNS 查询需要一个本地代理，将公共 DNS 的查询工作从 Mesos-DNS 中分离出去。

[Spartan](https://github.com/dcos/spartan) 产生的另一个原因是需要对 Mesos-DNS 进行并发查询，避免查询超时。

Spartan 对外暴露一个健康检查的 DNS 记录，名为 `ready.spartan`，供其他服务进行健康检查。

DC/OS 常见部署中，除了一个名为 `dcos-spartan.service` 的服务，还有一个 `dcos-spartan-watchdog.service` 的服务，对 Spartan 定时做健康检查，并负责修改 `/etc/resolv.conf` 文件，保证 Spartan 不可达时，公网域名依旧可以正常解析。

一次完整的 DNS 查询会产生类似下图的一张时序图，

![one-dns-query](https://g.gravizo.com/svg?%40startuml%3B%0A%0Aactor%20client%3B%0Aparticipant%20spartan%3B%0Aparticipant%20%22navstar-dns%22%20as%20navstar%3B%0Aparticipant%20%22mesos-dns%22%20as%20mdns%3B%0Aparticipant%20%22mesos-master%22%20as%20mesos%3B%0Aparticipant%20%22public%20dns%22%20as%20dns%3B%0A%0Anavstar%20-%3E%20mdns%3A%20dns%20polling%3B%0Amdns%20-%3E%20mesos%3A%20dns%20polling%3B%0A%0Aclient%20-%3E%20spartan%3A%20dns%20query%3B%0Aactivate%20spartan%3B%0Aspartan%20-%3E%20navstar%3A%20batch%20dns%20query%3B%0Aactivate%20navstar%3B%0Aspartan%20-%3E%20dns%3A%20batch%20dns%20query%3B%0Aactivate%20dns%3B%0Anavstar%20-%3E%20spartan%3A%20query%20response%3B%0Adeactivate%20navstar%3B%0Aspartan%20-%3E%20client%3A%20first%20response%3B%0Adns%20-%3E%20dns%3A%20processing%3B%0Adns%20-%3E%20spartan%3A%20query%20response%3B%0Adeactivate%20dns%3B%0A%0Adeactivate%20spartan%3B%0A%0A%40enduml%3B)

Spartan 会选择性并发查询 DC/OS 内部 DNS 或者外部 DNS 服务器。

## overlay 网络

[overlay 网络](https://dcos.io/docs/1.8/overview/design/overlay/)实现了 IP-per-container。保证网络应用移植的透明性。

![ip-allocation](https://dcos.io/docs/1.8/overview/design/img/overlay-fig-1.png)

上图取自 dcos.io，DC/OS 的 overlay 网络实现为 VxLAN 机制。VxLAN 相关封包方式及报文结构可以参考[cisco 的这篇文章](http://www.cisco.com/c/en/us/products/collateral/switches/nexus-9000-series-switches/white-paper-c11-729383.html)。

在 DC/OS 配置文件中，使用 [dcos_overlay_network](https://dcos.io/docs/1.8/administration/installing/custom/configuration-parameters/#-a-name-dcos-overlay-enable-a-dcos_overlay_enable) 参数规划 overlay 网络。subnet 参数规划了该集群可以容纳 agent 节点个数，及每个节点能够分配到的子网号。prefix 规划了每个 agent 节点能够容纳的所有容器个数。其中主机位最高一位为0的 IP 分配给 Mesos Container，为1的 IP 分配给 Docker Container，即两类 Container 平分地址池。

![overlay-network-sequence](https://g.gravizo.com/svg?%40startuml%3Bparticipant%20%22mesos%20framework%22%20as%20framework%3Bparticipant%20%22mesos%20master%22%20as%20master%3Bparticipant%20%22mesos%20agent%22%20as%20agent%3Bparticipant%20navstar%3Bgroup%20agent%20bootstrap%3Bagent%20-%3E%20master%3A%20get%20subnet%20of%20agent%3Bmaster%20-%3E%20agent%3A%20subnet%20and%20vtep%20address%3Bagent%20-%3E%20agent%3A%20create%20docker%20network%20and%20cni%20network%3Bend%3Bframework%20-%3E%20master%3A%20launch%20task%3Bagent%20-%3E%20navstar%3A%20config%3Bcreate%20task%3Bagent%20-%3E%20task%3A%20launch%3B%40enduml%3B)

以上为 Agent 启动时 IP 子网分配过程。

DC/OS 中 container2container 的通信方式比较值得一提，由 Mesos Master 的 overlay module 向 Mesos Slave 的 overlay module 发放容器子网的拥有者 Agent。其他部分如图所示，CNI 提供 Mesos Container 的网络实现。Navstar 进行网络编排……

## DC/OS 内部及外部负载均衡

* [Minuteman](https://github.com/dcos/minuteman) 作为 DC/OS 内部 LB

Minuteman 通过检测 Mesos 的 `task.port` 结构体中的 label，匹配 `VIP_$IDX` 字段相同的服务作为后端服务。

Minuteman 使用 [IPVS](https://en.wikipedia.org/wiki/IP_Virtual_Server) 技术，通过在 iptables 的 raw 表中接入 filter，将匹配相应 IP 的数据包放入 [NFQueue](https://home.regit.org/netfilter-en/using-nfqueue-and-libnetfilter_queue/)。Minuteman 通过监听 NFQueue，截获报文，匹配端口信息，适当修改为 DNat/SNat 后重新传入内核进行路由。

```
# iptables -vL -t raw
Chain PREROUTING (policy ACCEPT 27M packets, 8170M bytes)
 pkts bytes target     prot opt in     out     source               destination         
    3   180 NFQUEUE    tcp  --  any    any     anywhere             anywhere             match-set minuteman dst,dst tcp flags:FIN,SYN,RST,ACK/SYN NFQUEUE balance 50:58

Chain OUTPUT (policy ACCEPT 27M packets, 7898M bytes)
 pkts bytes target     prot opt in     out     source               destination         
    7   420 NFQUEUE    tcp  --  any    any     anywhere             anywhere             match-set minuteman dst,dst tcp flags:FIN,SYN,RST,ACK/SYN NFQUEUE balance 50:58

```

* [marathon-lb](https://github.com/mesosphere/marathon-lb) 作为 DC/OS 对外 LB

marathon-lb 通过定时拉取 Marathon 的 task 信息，生成 HAProxy 的配置文件进行负载均衡。
