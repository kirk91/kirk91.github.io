---
title: 饿了么透明代理 Samaritan
tags:
  - samaritan-proxy
  - eleme
categories:
  - samaritan-proxy
abbrlink: b2fdd6c0
date: 2020-01-14 11:37:00
---

在饿了么，一个应用如果要访问MySQL、Redis、MQ等基础组件，都是通过一个本地的透明代理来完成的。该代理将基础组件服务化，屏蔽了分布式环境中的集群发现、健康检查和负载均衡等细节，让应用使用起来如单机时一样简单，能够直接使用各语言现有的 SDK，无需任何改动。

<!--more-->

## 背景

在内部最初迁移至 [SOA] 架构的时候，我们在应用的每台机器上部署了 [HAProxy]，用于代理到基础组件的流量。起初由于内部服务的数量和变更都比较少，这个方案工作的还不错；但随着服务的增多和实例频繁的变更发布，这种方案的弊端逐渐显现出来，主要是如果一个服务的实例有增加或删除，就需要更新依赖该服务的所有机器上的配置文件并重启 HAProxy 。

这些变更和重启的运维工作在当时占据了我们大量的工作时间，除此之外因为主要依赖人的手工操作，极其容易出现错误；还有每次重启会有一小段时间的服务不可用。这些问题在当时迫切地需要解决掉，不然就会影响到大家的订餐体验。

经过一番调研之后，发现有很多公司遇到同样的问题，甚至 [Aribnb] 和 [Qubit] 还开源了自己的解决方案 [Synapse] 和 [Bamboo]，这两个项目的思路比较类似，简单说是通过自动生成配置文件 + 管理 HAProxy 进程。方案简单明了，无需改造侵入 HAProxy, 开发和维护成本都可控，但每次重启进程还是会影响到服务的可用性，尤其在变更比较频繁的时候就更加明显。

我们希望能够有更好的方案，比如: 服务节点发生变更的时候，不用重启进程能立即生效，不会影响到服务的可用性。经过内部的讨论和交流，最终我们决定编写自己的面向 SOA 的 [Load Balancer] (这个决定在当时是非常大胆冒险的), 用于替代 HAProxy, 它就是 **Samaritan**, 为了简单我们在内部也称之为 **Sam**。

之所以用 *Samaritan* 命名这个项目，也是希望通过它能够把我们从繁重的运维工作中解救出来:

>A charitable or helpful person (with reference to Luke 10:33).
>
> "suddenly, miraculously, a Good Samaritan leaned over and handed the cashier a dollar bill on my behalf"
>
> refer: https://www.lexico.com/definition/samaritan

![Samaritan](https://samaritan-proxy.github.io/images/logo.png)


[SOA]: https://en.wikipedia.org/wiki/Service-oriented_architecture
[HAProxy]: http://www.haproxy.org/
[Aribnb]: https://airbnb.io/
[Qubit]: https://www.qubit.com/
[Synapse]: https://github.com/airbnb/synapse
[Bamboo]: https://github.com/QubitProducts/bamboo
[Load Balancer]: https://en.wikipedia.org/wiki/Load_balancing_(computing)


## 介绍

**Samaritan** 是一个工作在客户端模式下面向 SOA 的透明代理，有着良好的性能和可用性。 它被广泛应用于饿了么的生产环境，部署在每一个 container 或者 vm 上，代理了全网到基础组件的所有流量包括 Redis、MySQL 和 MQ等，当前在生产环境运行着的实例数有 5w+。

![](https://i.imgur.com/h0kRsBt.png)

它具有如下特性:

- **轻量，高性能**

  使用 [Go](https://golang.org/) 编写而成，有着比较高的性能，资源占用比较少，作为一个单独的进程与应用程序一起运行。

- **支持热更新配置，无需重启**

  在运行时能够从远端获取代理策略和服务实例信息的实时变更，并且立即生效。这意味着配置变更无需额外的重启，也就不会有因重启导致的服务不可用。另外也大大地降低了运维的成本，使得其能够大规模部署和适应云环境成为可能。

- **故障的自动检测、处理和恢复**

  Sam 会定期对服务的节点实例进行健康检查，支持 TCP、ATCP、MySQL 和 Redis 等，能够自动剔除故障节点并转移对应流量，当节点恢复后会再次加入。

- **RESTful Admin API**

  提供了 RESTful Admin API，支持获取运行时信息包括 config 和 stats， 调整日志等级，性能分析 pprof 等。

- **良好的观测性**

  在 Sam 内部，会记录与连接和请求有关的频次、耗时和成功失败等指标并进行上报，当前支持 [statsd] 和 [prometheus]。这些指标可以帮助我们了解服务和网络的实时状况，并在需要的时候作出决策。

[statsd]: https://github.com/statsd/statsd
[prometheus]: https://prometheus.io

## 如何工作

![how it works](https://samaritan-proxy.github.io/docs/images/how-to-work.svg)

- Sam 订阅代理配置和服务实例的变更，并按照配置的策略在用户端对流量进行转发。
- Sash 是一个管理服务器，提供代理配置和服务实例变更的实时推送，以及一些基础的运维工具如部署升级等。

备注:

1. Sash 是 `Sam` 和 `Dashboard` 的组合
2. Sash 只是一个管理服务器的实现，不强制绑定，如有需要可自行实现

## 演进

在早期的时候，Sam 只有四层代理即简单的流量拷贝，虽然工作地蛮好，也的的确确解决了最初我们遇到的问题，但是比较缺少有效的应用层指标，无法为业务提供更多行之有效的帮助。

另外很多人知道，在饿了么我们是使用 [Redis Cluster] 作为缓存的，为了兼容已有的的 Redis SDK 和应用(不支持集群协议), 我们构建了一个支持集群协议的代理 [Corvus]，用于将 Redis 2 的请求透明地翻译成 Redis 3 请求。 Corvus 是一个非常出色的项目，很快地在生产环境落地，也被很多其他互联网公司使用。但随着公司业务的快速发展，Corvus 在生产环境部署的实例越来越多，随之而来的机器成本和维护成本都越来越高, 迫切的需要想个办法来解决。

基于上面提到的两点，我们想既然 Sam 代理了到基础组件的所有流量，那是不是可以把 Corvus 的功能做进去，这样一方面可以节省掉 Corvus 的成本，另外一方面可以让 Sam 往七层应用层方面发展，更好地服务于业务。 如果能够实现的话，整个请求的路径将变成下面的样子:

![](https://i.imgur.com/U64Gl9X.png)

看起来是挺简单的，但改造的难度和挑战还是蛮大的，比如:

1. 如何保证与 corvus 命令兼容，并性能相差不大?
2. 支持了七层，会不会占用较多的资源影响到业务应用?
3. 四层到七层，复杂度大幅度提升，如何保持项目质量与稳定?
4. 生产环境有上千个 corvus集群，如何悄无声息地下线掉并且不影响业务?

这些展开来每个都可以单独写成一篇文章，篇幅问题这里就先不做讨论了，后面有机会的话会专门写。让人可喜的是经过团队成员的不懈努力和配合，在19年中期的时候我们就下线了全网的 corvus, 全部 Redis Cluster 的代理替换为 Sam，至此不但解决了一直上升的成本问题，也让 Sam 支持了七层代理。

[Redis Cluster]: https://redis.io/topics/cluster-tutorial
[Corvus]: https://github.com/eleme/corvus

## 开源

19年下半年的时候，我们团队内部讨论决定开源 Sam，主要考虑的因素如下:

1. *回馈社区*: 我们通过开源项目构建了很多内部的系统和平台，也希望能够尽自己的力量，通过 Sam 的开源来分享我们的解决方案，如果在这个过程中能够帮助其他公司和个人解决问题就更好了。
2. *更好的发展*: 当前我们虽然完成了 Redis Cluster 代理的开发，但还有很多东西可以优化，可以适配支持更多的协议如 MySQL、MQ等，希望大家可以参与进来一起完善共建。

通过一段时间内部依赖的剥离和一些地方的重新设计，项目已经开源了，地址是: https://github.com/samaritan-proxy/samaritan 欢迎大家踊跃 star、提 pr 和使用 😊

## FAQ

1. 有没有更详细的文档介绍?

   所有的文档都可以在 https://samaritan-proxy.github.io/docs/ 找到，除了基本的介绍外还有架构、实现细节、数据指标和配置管理等, 后面我们也会随着项目的演讲及时更新。

2. 除了兼容 corvus 的命令外，有什么不一样的地方吗?

   除了最基本的命令兼容外，在 sam 里面我们还实现了一些比较有意思且能够切实解决业务痛点的功能, 比如:
   - 命令级别打点: 统计每种命令的执行时间P99、请求参数和返回结果长度等
   - 集群级 scan: 通过普通的 scan 命令可以对整个集群进行数据扫描
   - 透明(解)压缩: 对大 key 进行透明压缩，节省缓存空间和机器带宽占用
   - 热 key 实时收集: 实现了 `hotkey` 命令，使得业务可以快速定位热点 key

3. 社区已经有了 Envoy, 为啥要自己造 Sam?

   主要原因其实是那时候没有 Enovy, 后面有了后更换的 ROI 比较低，然后 Enovy 本身也有复杂度

4. 除了开源 sam 外，后续还有其他的动作没?

   后续我们会把内部的分布式缓存系统 *FlexiCache* 进行开源，并且这个已经在进行中了，这两个项目的结合是非常紧密的，到时候欢迎大家一起使用。


最后的最后，由于个人能力和经验的有限，不可避免地会有或多或少的错误，项目也不是百分百完美的，有很多可以改进优化的地方，欢迎大家一起交流共建 🤝 ❤️
