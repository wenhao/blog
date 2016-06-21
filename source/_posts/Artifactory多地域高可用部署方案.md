title: Artifactory多地域高可用部署方案
toc: true
date: 2016-06-21 10:27:47
categories: [devops]
tags:
   - artifactory
   - devops
---

引入Artifactory之后，相当一段时间还处于验证阶段，单集群解决方案会比较普遍。随着运维的成熟度和团队对Artifactory的接受度不断提升，各地域给单集群Artifactory带来的压力会越来越大，此时多地域高可用方案会逐步实施。

###单节点集群

解决的问题：

1. 支持多仓库maven，python，docker等等。
2. 支持多地域上传下载，华北，华东和华南等。
3. 避免Artifactory机房停电，引入“Remote Backup Artifactory”实例，即把Artifactory集群中的一台部署到不同机房或者地域。
4. 避免磁盘故障，NFS增量备份。
5. 避免数据库故障，MySQL多实例同步。
6. HAProxy高可用集群。

![Artifactory单集群](/img/artifactory_single_cluster.png)

<!-- more -->

###多地域集群

1. 支持更多的请求，缓解单集群压力。
2. 根据不同地域的特点配置各自的Artifactory集群。
3. 引入“Central Share”中心仓的概念解决分布式团队开发依赖的问题。中心仓可以是某个Artiactory集群中的某个仓库，如果流量很大也可以是一个独立的Artifactory集群。

![Artifactory多地域集群](/img/artifactory_multi_region_cluster.png)
