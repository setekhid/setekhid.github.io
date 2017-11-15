---
layout: post
title: "DC/OS 1.8 负载均衡组件"
categories: dcos
author: "Huitse Tai"
---

衔接前篇 blog，展开讲解 DC/OS 的三个 lb 组件，其中 [marathon-lb](https://github.com/mesosphere/marathon-lb) 和 [minuteman](https://github.com/dcos/minuteman) 由 mesosphere 开发，属于官方组件。[linkerd](https://github.com/linkerd/linkerd) 属于第三方组件，但由于其功能强大，这里略调研一下。

[toc]

marathon-lb (v1.6.0)
----

[marathon-lb](https://github.com/mesosphere/marathon-lb) 通常用于 DC/OS 集群所有公共服务的 l7lb 和服务 gateway。

### mlb 入口配置

marathon-lb 可以从 DC/OS 的 [universe](https://github.com/mesosphere/universe) 快速安装，universe 中的脚本会将 marathon-lb 安装为非 framework，并由 marathon 做 HA 管理。装好后，在 marathon 界面查看 marathon-lb 的配置。

`.args` 入口参数有两个比较重要的，

* `-m http://marathon.mesos:8080` 指明 marathon 的入口地址，域名由 mesos-dns 生成。
* `--group external` 指明该 lb 所监听的标签。
* 由于生产环境对 https 多有需求，还有个参数需手动配置 `--ssl-certs /etc/ssl/certs/` 指明 ssl 证书的存放地址，或逗号分隔的证书列表。

由于 marathon-lb 要直接与物理网络通信，`.container.docker.network` 默认为 `HOST` 类型。`.portDefinitions` 占用了几个常见端口，包括 http 和 https。并占用了 10000 到 10100 端口，用于做端口转发，详见后文。

### mlb 工作方式[^mlb guide]

抄录一张 [mesosphere 的图](https://docs.mesosphere.com/1.8/usage/service-discovery/marathon-lb/)，

![marathon-lb](https://docs.mesosphere.com/wp-content/uploads/2016/06/lb1-3-800x692@2x.png)

marathon-lb 的工作拓扑图如上，通过监听 marathon 发来的 event bus[^marathon event bus]，适时拉取 marathon 上部署的所有应用配置。并根据应用的 `.label` 字段，为应用生成 HAProxy 配置，然后重载 HAProxy，由 HAProxy 实例实现 l7lb。

### HAProxy in mlb

mlb 使用的 HAProxy 的 http 模式，进行 7 层负载。其中负载方式分为两种，一种使用前面配给 mlb container 的端口号进行转发。另一种解析 `http_request.host`, 按照 marathon app 中的 `.label.HAPROXY_N_VHOST` 配置匹配转发。

更详细的 HAProxy 配置，可以使用 `./marathon_lb.py --marathon http://marathon.mesos:8080 --group external` 在本地生成 HAProxy 的配置并查看。

### mlb 常用 label

* `.labels.HAPROXY_GROUP` 与 marathon-lb 的 `--group` 参数相对应，marathon-lb 只会生成其 `--group` 参数指定的 app 的 HAProxy 配置。
* `.labels.HAPROXY_N_VHOST`，marathon-lb 会使用这个参数生成文件 `domain2backend.map` 供 HAProxy 将 domain 映射为某个 backend。其中 `_N_` 指明了该 vhost 对应的入口是 app 的配置 json 中 `.portDefinition` 数组第 N 个端口。
* `.labels.HAPROXY_N_PORT` 类似 vhost，指定映射为 marathon-lb 的某个端口。

https 无需多余配置，在 marathon-lb 启动时指定的证书列表中含有域名信息。这里只需要配置好 vhost 即可。

### mlb 代码入口[^mlb project]

mlb 使用 python 开发，代码比较简单。

`main` 函数在根目录下 [`./marathon-lb.py:L1675`](https://github.com/mesosphere/marathon-lb/blob/v1.6.0/marathon_lb.py#L1675)。通常 marathon-lb 指定 sse 参数运行。接下来启动 `MarathonEventProcessor` 对 marathon 的 `status_update_event`、`health_status_changed_event` 和 `api_post_event` 事件进行监听。

并开启另一个线程拉取 marathon 的所有 app 配置，入口在 [`./marathon-lb.py:L1471`](https://github.com/mesosphere/marathon-lb/blob/v1.6.0/marathon_lb.py#L1471)，在 [`L1510`](https://github.com/mesosphere/marathon-lb/blob/v1.6.0/marathon_lb.py#L1510) 处拉取所有的 app 配置，并在 [`L1511`](https://github.com/mesosphere/marathon-lb/blob/v1.6.0/marathon_lb.py#L1511) 处开始配置生成，这两处处理了几乎所有 marathon-lb 所需的应用 label，比 [`README.md`](https://github.com/mesosphere/marathon-lb/blob/v1.6.0/README.md) 详细。

HAProxy 很早就已经支持了 zero-downtime reload。目前看来比较 concerned 的地方应该就是 app 配置拉取。原以为从 marathon 监听到的事件包含有应用的详细配置，marathon-lb 属于推送式更新，实则不然。app 量大时的响应速度值得一测。

  [^mlb guide]: marathon-lb mesosphere guide https://docs.mesosphere.com/1.8/usage/service-discovery/marathon-lb/

  [^marathon event bus]: marathon event bus https://mesosphere.github.io/marathon/docs/event-bus.html

  [^mlb project]: README.md of marathon-lb project https://github.com/mesosphere/marathon-lb/blob/v1.6.0/README.md

minuteman (dcos 1.8)
----

minuteman 是 DC/OS 1.8 集群内部使用的 l4lb。使用 erlang 开发，通过 ipvs 在内核中实现报文截取。DC/OS 1.8 默认安装于集群的所有节点，且由 systemctl 管理。

### 工作方式

minuteman 直接与 mesos-master 通信，每隔一段时间，拉取 mesos 上的所有 task 状态。并根据 `.port.labels.VIP_$IDX` 生成虚拟 ip 或域名。
