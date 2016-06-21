title: 搭建Artifactory集群
toc: true
date: 2016-05-05 22:51:19
categories: [devops]
tags:
   - artifactory
   - devops
---

制品仓库系统有很多，例如[Artifactory](https://www.jfrog.com/)、[Archiva](http://archiva.apache.org/)、[Sonatype Nexus](http://www.sonatype.org/nexus)、[Eclipse Package Drone](https://eclipse.org/package-drone)，其中Artifactory拥有很多强大的企业级特性和人性化的用户接口，拥有众多客户群。很多大型的公司都在使用它，通过以下的Google趋势图可以看出，它越来越受青睐。功能对比在此：[Binary Repository Manager Feature Matrix](https://binary-repositories-comparison.github.io/)

![Google趋势图](/img/binary-repository-trends.png)

本文将尝试在阿里云上搭建Artifactory集群。

###Artifactory许可证

官方正版license，3个 License 25900美元(16.7万人民币)一年，贵的离谱。本文以实验学习为主使用最新破解版4.7.4，破解也非常容易就不赘述了。商业用途，请使用正版。

###所需硬件

Artifactory集群需要以下硬件设备：

1. 支持粘性会话的均衡负载(HAProxy/Nginx等)。
2. NFS共享文件夹。
3. 数据库(MySQL等)。

<!-- more -->

###搭建架构图

![Artifactory集群](/img/artifactory-architecture.png)

###网络

集群中所有的节点**最好**处于同一局域网内，节点之间使用固定端口传输数据。

###服务器

本文使用阿里云ECS服务器，申请三台阿里云ECS服务器分别取名artifactory-master,artifactory-slave,artifactory-nfs。

###Artifactory节点配置

artifactory会部署在artifactory-master和artifactory-slave上，需要安装所需的软件。

1. [生成ssh key](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/)并配置authorized_keys方便服务管理。
2. 安装JDK 8。

   ```bash
   apt-get install software-properties-common
   add-apt-repository ppa:webupd8team/java
   apt-get update
   apt-get install oracle-java8-installer
   ```
3. 编辑.bashrc文件`vi ~/.bashrc`在文件尾加入以下内容：

   ```bash
   if [ -f ~/.bash_env ]; then
       . ~/.bash_env
   fi
   ```
4. 创建`.bash_env`文件`touch ~/.bash_env`并添加JAVA_HOME环境变量：

   ```bash
   export JAVA_HOME=/usr/lib/jvm/java-8-oracle
   export JRE_HOME=$JAVA_HOME/jre
   export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
   export PATH=$JAVA_HOME/bin:$PATH
   ```
5. 上传artifactory-pro-4.7.4.zip到artifactory-master和artifactory-slave服务器`/opt`目录并解压，并生成两个不同的`artifactory.lic`许可证，分别放在/opt/jfrog/artifactory-pro-4.7.4/etc目录下。

   ```bash
   scp artifactory-pro-4.7.4.zip root@<ip>:/opt
   ```
6. 分别在artifactory两个节点设置artifactory环境变量，编辑`.bash_env`文件。

   ```bash
   export ARTIFACTORY_HOME=/opt/jfrog/artifactory-pro-4.7.4
   export JAVA_HOME=/usr/lib/jvm/java-8-oracle
   export JRE_HOME=$JAVA_HOME/jre
   export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
   export PATH=$JAVA_HOME/bin:$PATH
   ```

7. 安装artifactory as service。

   ```bash
   sh installService.sh

   passwd artifactory <new password>
   ```

###NFS配置

NFS配置需要在artifactory-nfs上安装NFS服务端，需要在artifactory-master和artifactory-salve上安装NFS客户端。

1. 在artifactory-nfs服务器上安装nfs-kernel-server。

   ```bash
   apt-get install nfs-kernel-server
   ```
2. 在/etc/exports文件里增加一行。

   ```bash
   /artifactory/cluster-home *(rw,sync,no_root_squash,no_subtree_check)
   ```
3. 在artifactory-master和artifactory-salve分别安装NFS客户端。

   ```bash
   apt-get install nfs-common portmap
   ```
4. 在artifactory-master和artifactory-salve分别创建NFS待挂载目录/artifactory/cluster-home。

   ```bash
   mkdir /artifactory/cluster-home
   mount <artifactory-nfs' IP>:/artifactory/cluster-home /artifactory/cluster-home
   ```

5. 将NFS目录分配权限。

   ```bash
   chown -R artifactory:artifactory /artifactory/cluster-home
   ```   

###安装MySQL

在artifactory-nfs上安装MySQL。

1. 安装MySQL。

   ```bash
   apt-get install mysql-server mysql-client

   mysql>
   CREATE DATABASE artdb CHARACTER SET utf8 COLLATE utf8_bin;
   CREATE USER artifactory IDENTIFIED BY 'password';
   GRANT ALL PRIVILEGES ON *.* TO 'artifactory'@'%' IDENTIFIED BY 'password' WITH GRANT OPTION;
   FLUSH PRIVILEGES;
   ```
2. [MySQL性能优化](https://www.jfrog.com/confluence/display/RTF/MySQL)。
3. 允许MySQL远程访问。修改云主机上的/etc/mysql/my.cnf 文件，注释掉 bind_address=127.0.0.1就可以了。

###配置artifactory-master

1. 在`/artifactory/cluster-home`下创建一下目录：

   ```bash
   mkdir ha-etc
   mkdir ha-data
   mkdir ha-backup
   ```
2. 在`./ha-etc`下创建文件`cluster.properties`，内容为：

   ```bash
   ##随机生成的token，保证唯一就行
   security.token=4n4tpxip7spQQu2pKf3811S2W7GY46Yb
   ```
3. 在`./ha-etc`下创建文件`storage.properties`，内容为：

   ```bash
   type=mysql
   driver=com.mysql.jdbc.Driver
   url=jdbc:mysql://<artifactory-nfs' IP>:3306/artdb?characterEncoding=UTF-8&elideSetAutoCommits=true
   username=artifactory
   password=password
   ```
4. 复制`artifactory.system.properties`和`mimetypes.xml`文件

   ```bash
   mv /opt/jfrog/artifactory-pro-4.7.4/etc/artifactory.system.properties /artifactory/cluster-home/ha-etc
   mv /opt/jfrog/artifactory-pro-4.7.4/etc/mimetypes.xml /artifactory/cluster-home/ha-etc
   ```
5. 在/opt/artifactory-pro-4.7.4/etc目录下创建`ha-node.properties`文件，内容如下：

   ```bash
   node.id=art1
   cluster.home=/artifactory/cluster-home
   context.url=http://<artifactory-master's IP>:8081/artifactory
   membership.port=10001
   primary=true
   ```
6. 在`.bash_env`文件添加$CLUSTER_HOME环境变量。

   ```bash
   export ARTIFACTORY_HOME=/opt/jfrog/artifactory-pro-4.7.4
   export CLUSTER_HOME=/artifactory/cluster-home
   export JAVA_HOME=/usr/lib/jvm/java-8-oracle
   export JRE_HOME=$JAVA_HOME/jre
   export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
   export PATH=$JAVA_HOME/bin:$PATH
   ```

###配置artifactory-slave

注意：对于每个artifactory集群节点使用的artifactory.lic是不一样的，否者将会报错。

1. 在/opt/jfrog/artifactory-pro-4.7.4/etc目录下创建`ha-node.properties`文件，内容如下：

   ```bash
   node.id=art2
   cluster.home=/artifactory/cluster-home
   context.url=http://<artifactory-slave's IP>:8081/artifactory
   membership.port=10001
   primary=false
   ```
2. 在.bash_env文件添加`$CLUSTER_HOME`环境变量。

   ```bash
   export ARTIFACTORY_HOME=/opt/jfrog/artifactory-pro-4.7.4
   export CLUSTER_HOME=/artifactory/cluster-home
   export JAVA_HOME=/usr/lib/jvm/java-8-oracle
   export JRE_HOME=$JAVA_HOME/jre
   export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
   export PATH=$JAVA_HOME/bin:$PATH
   ```
   
###安装HAProxy负载均衡

对于负载均衡HAProxy的性能会比Nginx好很多。

#####安装HAProxy

```bash
apt-get install haproxy
```
#####配置HAProxy

```bash
global
    log /dev/log	local0
    log /dev/log	local1 notice
    chroot /var/lib/haproxy
    user haproxy
    group haproxy
    daemon
    maxconn 4096
    nbproc 1

defaults
    log	global
    mode	http
    option	httplog
    option	dontlognull
    option  forwardfor header X-Forwarded-For
    option  redispatch
    retries 2
    maxconn 40000
    balance roundrobin
    timeout connect 5000
    timeout client 50000
    timeout server 50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend http-in
    mode http
    bind *:80
    bind *:443 ssl crt /etc/haproxy/ssl/artifactory.pem
    reqirep ^/(v1|v2)/(.*) /artifactory/api/docker/\1/\2\3
    default_backend artifactory

backend artifactory
    balance roundrobin
    cookie SERVERID insert nocache indirect
    option httpclose
    option forwardfor header X-Forwarded-For
    server artifactory1 <IP>:8081/artifactory maxconn 10000 check cookie
    server artifactory2 <IP>:8081/artifactory maxconn 10000 check cookie

listen stats 
    bind *:8080
    stats enable
    stats refresh 30s
    stats uri /haproxy?stats
    stats realm HAProxy\ Statistics
    stats auth admin:admin
```

####启动

```bash
su - artifactory
service artifactory start
```

###仓库之间复制

Artifactory允许支持不同地区不同项目之间artifactory实例复制。带来的好处有以下几点：

1. 不同地区的开发团队可以使用相同artifacts。
2. 构建的产出artifacts能够及时共享。
3. 缓解远程网络连接不稳定性。
4. 访问远程其他artifactory仓库。

####Push方式

用于本地仓库，上传到某个artifactory实例的某个本地仓库能够同步到其他远程artifactory仓库里面。

####Pull方式

用于远程仓库，将远程artifactory仓库同步到本地artifactory某个仓库。

###安装JFrog Mission Control

服务器有限，在artifactory-master上安装Mission Control。

```bash
wget https://akamai.bintray.com/84/842469ab2f8d53dcd01e99c1f96b39b7580571a20096f741446e5c789ff2bca5?__gda__=exp=1462285257~hmac=76ab0b04df1b8b374bd539b83e246f8fe00ad8be57d7d7e47138b9ffb1b13a78&response-content-disposition=attachment%3Bfilename%3D%22jfrog-mission-control-1.1.deb%22&response-content-type=application%2Fx-debian-package

apt-get install net-tools

dpkg -i jfrog-mission-control-1.1.deb
```

###安装Jenkins

在artifactory-slave上安装Jenkins

```bash
wget -q -O - https://jenkins-ci.org/debian/jenkins-ci.org.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins-ci.org/debian binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins
```

###安装docker

```bash
apt-get install docker.io
```

把jenkins用户加入到docker的group里面。

```bash
gpasswd -a jenkins docker
```

###设置Artifactory的docker repository

####生成ssl

```bash
apt-get install openssl

mkdir /etc/nginx/ssl

openssl genrsa -out "/etc/nginx/ssl/artifactory.key" 2048

openssl req -new -key "/etc/nginx/ssl/artifactory.key" -out "/etc/nginx/ssl/artifactory.csr"

openssl x509 -req -days 365 -in "/etc/nginx/ssl/artifactory.csr" -signkey "/etc/nginx/ssl/artifactory.key" -out "/etc/nginx/ssl/artifactory.crt"

cat /etc/nginx/ssl/artifactory.crt /etc/nginx/ssl/artifactory.key > /etc/nginx/ssl/artifactory.pem
```

####配置HAProxy

```
global
    log /dev/log	local0
    log /dev/log	local1 notice
    chroot /var/lib/haproxy
    user haproxy
    group haproxy
    daemon
    maxconn 4096
    nbproc 1

defaults
    log	global
    mode	http
    option	httplog
    option	dontlognull
    option  forwardfor header X-Forwarded-For
    option  redispatch
    retries 2
    maxconn 40000
    balance roundrobin
    timeout connect 5000
    timeout client 50000
    timeout server 50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend http-in
    mode http
    bind *:80
    bind *:443 ssl crt /etc/haproxy/ssl/artifactory.pem
    reqirep ^/(v1|v2)/(.*) /artifactory/api/docker/\1/\2\3
    default_backend artifactory

backend artifactory
    balance roundrobin
    cookie SERVERID insert nocache indirect
    option httpclose
    option forwardfor header X-Forwarded-For
    server artifactory1 <IP>:8081/artifactory maxconn 10000 check cookie
    server artifactory2 <IP>:8081/artifactory maxconn 10000 check cookie

listen stats 
    bind *:8080
    stats enable
    stats refresh 30s
    stats uri /haproxy?stats
    stats realm HAProxy\ Statistics
    stats auth admin:admin
```

###Artifactory生态链

![Artifactory生态链](/img/artifactory-eco-system.png)

###Artifactory集群性能优化

1. [Artifactory Performance Tuning](http://mkuthan.github.io/blog/2011/03/10/artifactory-performance-tuning/)
