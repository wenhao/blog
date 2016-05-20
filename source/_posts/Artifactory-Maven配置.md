title: Artifactory Maven配置
toc: true
date: 2016-05-20 12:13:24
categories: [devops]
tags:
   - artifactory
   - devops
---

###配置Maven本地仓库

1. `maven-snapshot-local` 开发版本仓库。
2. `maven-staging-local` 预发布版本仓库。
3. `maven-release-local` 准发布版本仓库。
4. `maven-plugins-snapshot-local` 插件开发版本仓库。
5. `maven-plugins-release-local` 插件发布版本仓库。
6. `maven-virtual` maven虚拟仓库，maven项目的仓库入口。
7. `maven-plugins-virtual` maven插件虚拟仓库，maven插件项目的仓库入口。

![maven local](/img/maven-local.png)

<!-- more -->

###配置Maven代理仓库

Artifactory自带的`jcenter`能够满足大部分Java项目的开发，没有必要再添加其他代理仓库如[apache maven repository](https://repo1.maven.org/maven2)。

![maven remote](/img/maven-remote.png)

###Maven Snapshot类型仓库配置

Maven Snapshot仓库是开发人员用得最为频繁的仓库类型，产生的Artifact量也是最大的，占用的磁盘空间同样也是最多的。但是对于Snapshot类型的Artfiact的生命周期反而是最短的，下个Snapshot版本一出来当前的Snapshot基本上就没什么用了。

Artifactory支持对Snapshot仓库进行自动清理：

1. 通过配置Max Unique Snapshots，0为不限制，5代表最多可保存5个同版本的Snapshot Artifact。
2. Maven Snapshot Version Behavior，选择Unique,Maven3也强制使用这个。
3. Suppress POM Consistency Checks， 强制限制每个Artifact的唯一性。
4. Include / Exclude Patterns，为了避免上传其他不应当Maven仓库管理的内容通常可以加一些包含模式，对于Maven仓库可以是：`**/*.jar`和`**/*.pom`。

   ```
   注意: 如果上传错误的类型到maven类型的仓库，Artifactory不会索引此artifact也不会更新它的元数据(metadata)，如同一般类型的仓库一样(General Repository)
   ```
5. Handle Snapshots只允许存Snapshot，相对maven release仓库推荐只选择Handle Releases。

![maven snapshot](/img/maven-snapshot.png)

###虚拟仓库

使用虚拟仓库(Virtual Repository)的好处是使用者不用关心虚拟仓库后端的具体实现，虚拟的任何修改对于使用者来说都是透明的。使用虚拟仓库默认的搜索顺序为先本地仓库然后远程仓库的缓存最后才是远程仓库。

所有仓库都应该添加到虚拟仓库里面。

![maven virtual](/img/maven-virtual.png)


###配置供developer访问的用户及权限

1. 添加`developer`Users。
2. 添加`developer`Groups。
3. 添加`developer`Permissions，能够访问所有artifactory maven仓库，拥有Deploy/Cache，Annotate，Read权限。

###配置Apache Maven

1. 选择maven的某个仓库，点击`Set Me Up`，复制所有内容到maven设置文件`settings.xml`。

![maven set me up](/img/maven-set-me-up.png)

###发布流程

1. 选择jenkins mavenstyle job。
2. 配置VCS。
3. Artifactory 插件 Enable Release Management。
4. 配置Artifactory 插件。
5. build完成之后点击Artifactory Release Staging发布到maven-staging-local。
6. 当QA完成测试之后，点击Artifactory Promotion从maven-staging-local发布到maven-release-local。

###使用推(PUSH)同步

场景：对于分布式的开发团队A和B，两个地区分别大奖了Artifactory集群，A团队的开发依赖于B，所以B上传的Artifacts在A区也要能够同时看到。

配置本地仓库，在Replications设置：

1. 添加`Cron Expression *`这个是定时同步。
2. Enable Event Replication，对于Artifact增删改查立即在其他仓库生效，即保证一致性，上传Aritfact能保证多个仓库同时出现。
3. Sync Deleted Artifacts，一般不做同步删除。
4. Path Prefix，相当于一个简单的过滤器，只同步推某个路径下的。

![maven push sync](/img/maven-push-sync.png)

###使用拉(PULL)同步

场景：对于分布式的开发团队，两个地区的Artifactory集群需要同步某个仓库里面的所有内容。但是这样务必会增加带宽和磁盘的压力。最好的情况还是根据实际情况只同步需要同步的内容。

拉同步其实就是把另外一个Artifactory的仓库当做本地的远程(Remote)仓库使用。

###Jenkins Pipeline配置

####配置Artifactory Jenkins插件全局配置

![artifactory global config](/img/artifactory-global-config.png)

####配置jenkins build job

![artifactory plugin config](/img/artifactory-plugin-config.png)

####使用Parameterized trigger plugin传递参数

![parameterized trigger plugin](/img/parameterized-trigger-plugin.png)

####设置Artifactory build artifacts插件

![artifactory build artifacts plugin](/img/artifactory-build-artifacts-plugin.png)

####获取下载链接

下载链接会使用已定义的过滤表达式来排序，`ARTIFACT_DOWNLOAD_URL_1`到`ARTIFACT_DOWNLOAD_URL_N`。

####使用Delivery Pipeline 插件

![pipeline](/img/pipeline.png)
