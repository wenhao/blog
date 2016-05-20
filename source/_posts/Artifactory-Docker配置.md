title: Artifactory Docker配置
toc: true
date: 2016-05-20 12:55:01
categories: [devops]
tags:
   - artifactory
   - devops
---


Artifactory支持多个Docker仓库，并且每个Docker仓库都支持Docker Registry API。

###配置Docker本地仓库

1. `docker-dev-local` 开发版本仓库V1。
2. `docker-dev-local2` 开发版本仓库V2。
3. `docker-prod-local` 准发布版本仓库V1。
4. `docker-prod-local2` 准发布版本仓库V2。
5. `docker-virtual`虚拟仓库统一入口。

![docker local](/img/docker-local.png)

<!-- more -->

###配置Docker代理仓库

![docker-puh](/img/docker-hub.png)

###为Docker配置反向代理

Docker必须使用SSL，配置Subdomain。使用Subdomain的好处在于不用每加一个Docker仓库就重启一次Nginx服务器。

![docker reverse proxy](/img/docker-reverse-proxy.png)

###使用推(PUSH)同步

![docker push sync](/img/docker-push-sync.png)

**注意：**由于Docker Client的限制，Artifactory不支持Docker仓库的拉(PULL)同步。

###Jenkins Docker Slave配置

1. 安装docker。
2. 文件/etc/hosts添加`<IP> docker-dev-local2.<IP>`。
3. 文件/etc/default/docker添加：

    ```bash
    DOCKER_OPTS="-H tcp://0.0.0.0:4243 -H unix:///var/run/docker.sock"
    DOCKER_OPTS="$DOCKER_OPTS --insecure-registry docker-dev-local2.<IP>"
    ```
4. 添加文件~/.dockercfg内容为(举例)

    ```json
    {
    	"docker-dev-local2.<IP>" : {
    		"auth": "YWRtaW46QVAybVlKN1FwdXI3Q2NzZUJxdnpBOEY4SkI5",
    		"email": "youremail@email.com"
    	}
    }
    ```
5. 重启docker。

###Jenkins Push Docker镜像

jenkins构建项目同步Dockerfile打包镜像：

```bash
docker build -t docker-dev-local2.<IP>/$NAME:$VERSION -f ./dockerFile/Dockerfile。
docker push docker-dev-local2.<IP>/$NAME:$VERSION
```

###Pull Docker镜像

在Dockerfile里面需要下载docker镜像。镜像都来自于`docker-virtual`，所以/etc/hosts和/etc/default/docker需要相应的配置。

1. 文件/etc/hosts:

    ```bash
    <IP> docker-dev-local2.<IP> docker-virtual.<IP>
    ```
2. 文件/etc/default/docker:

    ```bash
    DOCKER_OPTS="-H tcp://0.0.0.0:4243 -H unix:///var/run/docker.sock"
    DOCKER_OPTS="$DOCKER_OPTS --insecure-registry docker-dev-local2.<IP>  --insecure-registry docker-virtual.<IP>"
    ```

3. Dockerfile：

    ```
    FROM docker-virtual.<IP>/ubuntu:latest
    
    ...
    
    CMD echo 'Hello Docker'
    ```


