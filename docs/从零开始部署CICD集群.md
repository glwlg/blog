# 从零开始部署CICD集群

记录一次从零开始部署一个基于docker的CICD集群，用的是centos7系统，涉及到的组件：docker,jenkins,gitlab,traefik,portainer。用了三台虚拟机（c1,c2,c3,其中c1为master节点)以root用户进行操作。

虚拟机教程可以参考[这篇](https://glwlg.github.io/post/yong-vagrant-zai-ben-di-yun-xing-k3s/)

## 基础环境,所有节点

1. 更新系统基础环境
 首先更新一下系统组件，安装一些必备的工具
 ```
 yum update -y && yum -y install vim bash-completion ntpdate wget git net-tools yum-utils &&  source /etc/profile.d/bash_completion.sh
 ```
2. 同步时间，重要
    
    `ntpdate 1.cn.pool.ntp.org`
    
    vim /etc/crontab
    
    添加一行 `50 4 * * * root ntpdate [1.cn.pool.ntp.org](http://1.cn.pool.ntp.org/)`
    
3. 安装docker
    
    [官方文档](https://docs.docker.com/engine/install/centos/)
    
    ```
    yum-config-manager \
        --add-repo \
        https://download.docker.com/linux/centos/docker-ce.repo
    #这里直接安装最新版(19.03.9)，如果需要低版本可以参考官方文档
    yum -y install docker-ce docker-ce-cli containerd.io
    #启用docker服务
    systemctl enable docker
    #启动docker
    systemctl start docker
    
    ```
    
    - 镜像加速
        
        国内网络环境拉去docker镜像比较慢，建议使用阿里云的镜像加速服务，是免费的，到阿里云的‘容器镜像服务’里找到‘镜像加速器’，有文档说明如何配置。
        
4. c1修改ssh端口号，22端口号后面会用到，先把ssh的改掉释放出来
向SELinux中添加端口：`semanage port -a -t ssh_port_t -p tcp 1022`
    
    `vim /etc/ssh/sshd_config`修改sshd配置，找到`#Port 22`这一行，把#去掉，再加一行`Port 1022`
    
    `systemctl restart sshd`重启sshd，重启没报错的话，用1022端口重新连接c1节点，
    
    再去把`Port 22`重新注释掉，再重启sshd，分两次操作是避免第一次重启失败导致无法连接
    
    ps: 如果用的是vagrant，修改ssh端口之后要去修改Vagrantfile，添加以下两行，否则下次虚拟机无法正常启动
    
    ```json
    c1.vm.network :forwarded_port, guest: 1022,host: 2021
    c1.ssh.port = 2021
    ```
    

## docker集群配置

1. 初始化集群，c1节点
`docker swarm init --advertise-addr 172.28.128.5` 
    
    `--advertise-addr`指定一个swarm使用的ip，一般是和其他节点在同一局域网的那个ip
    
2. 加入集群，c2，c3节点
直接运行上面init之后打印出来的`docker swarm join --token ...`命令即可
3. 部署portainer
    
    ```bash
    curl -L https://downloads.portainer.io/portainer-agent-stack.yml -o portainer-agent-stack.yml
    docker stack deploy --compose-file=portainer-agent-stack.yml portainer
    ```
    
4. 部署traefik
    
    先创建网络 `docker network create --driver overlay --attachable router-net`
    
    - traefik主配置
        
        `vim /srv/docker/traefik/traefik.yml`
        
        ```yaml
        log:
          level: INFO
        accessLog:
          format: "common"
        api:
          dashboard: true
          insecure: true
        entryPoints:
          http:
           address: ":80"
          gitTcp:
           address: ":22"
        providers:
          file:
            directory: /etc/traefik/config
          docker:
            endpoint: "unix:///var/run/docker.sock"
            swarmMode: true
            exposedbydefault: false
        ```
        
        `docker config create traefik.yml /srv/docker/traefik/traefik.yml` 
        
    - gitlab的ssh配置
        
        `vim /srv/docker/traefik/traefik_git.yml`
        
        ```yaml
        tcp:
          services:
            git-tcp-service:
              loadBalancer:
                servers:
                - address: "gitlab:22"
          routers:
            git-tcp-router:
                rule: "HostSNI(`*`)"
                entryPoints:
                  - "gitTcp"
                service: git-tcp-service
        
        ```
        
        `docker config create traefik_git.yml /srv/docker/traefik/traefik_git.yml` 
        
    - 路由配置
        
        `vim /srv/docker/traefik/traefik_routers.yml`
        
        ```yaml
        http:
          routers:
            git-router:
              rule: "Host(`git.local.com`)"
              service: git-service
          services:
            git-service:
              loadBalancer:
                servers:
                - url: "http://gitlab:80"
        ```
        
        `docker config create traefik_routers.yml /srv/docker/traefik/traefik_routers.yml`
        
    - 运行
        
        ```bash
        docker service create \
        --name traefik \
        --mode global \
        --constraint=node.role==manager \
        --publish "mode=host,published=80,target=80" \
        --publish "mode=host,published=22,target=22" \
        --publish "mode=host,published=443,target=443" \
        --publish "mode=host,published=8580,target=8080" \
        --config src=traefik.yml,target="/etc/traefik/traefik.yml"  \
        --config src=traefik_git.yml,target="/etc/traefik/config/traefik_git.yml"  \
        --config src=traefik_routers.yml,target="/etc/traefik/config/traefik_routers.yml"  \
        --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock \
        --network router-net \
        --with-registry-auth \
        --log-driver json-file --log-opt max-size=500M --log-opt max-file=1 \
        traefik:v2.2
        ```
        
    - 更新配置文件的步骤
        
        ```bash
        docker service update --config-rm traefik.yml traefik
        docker config rm traefik.yml
        docker config create traefik.yml ./traefik.yml
        docker service update --config-add  src=traefik.yml,target="/etc/traefik/config/traefik.yml" traefik
        ```
        
    - 补充说明
        
        这里traefik启用了swarm模式，所以不能通过自动发现的方式自动路由gitlab（除非gitlab也用docker service运行，这里是run），所以在providers里额外配置了file，以静态配置的形式配置了gitlab的路由，其他类似Jenkins之类的也可以如此配置。
        

## gitlab部署，c1节点

1. 首先创建三个文件夹
    
    ```bash
    mkdir /srv/docker/gitlab/postgresql -p
    mkdir /srv/docker/gitlab/redis
    mkdir /srv/docker/gitlab/gitlab
    
    ```
    
2. 运行postgresql
    
    ```bash
    docker run --name gitlab-postgresql -d \
    --env 'DB_NAME=gitlabhq_production' \
    --env 'DB_USER=gitlab' --env 'DB_PASS=password' \
    --env 'DB_EXTENSION=pg_trgm' \
    --net router-net \
    --volume /srv/docker/gitlab/postgresql:/var/lib/postgresql \
    --log-driver json-file --log-opt max-size=100M --log-opt max-file=1 \
    --restart always \
    sameersbn/postgresql:10-2
    ```
    
3. 运行redis
    
    ```bash
    docker run --name gitlab-redis -d \
    --net router-net \
    --volume /srv/docker/gitlab/redis:/var/lib/redis \
    --log-driver json-file --log-opt max-size=100M --log-opt max-file=1 \
    --restart always \
    sameersbn/redis:4.0.9-2
    ```
    
4. 编辑配置文件
vim /srv/docker/gitlab/env-file
其中三个SECRETS随机生成，smtp按需配置
    - env-file
        
        ```
        GITLAB_PORT=80
        GITLAB_SSH_PORT=22
        GITLAB_SECRETS_DB_KEY_BASE=32位随机字符串
        GITLAB_SECRETS_SECRET_KEY_BASE=64位随机字符串
        GITLAB_SECRETS_OTP_KEY_BASE=64位随机字符串
        GITLAB_HOST=git.local.com
        SMTP_ENABLED=true
        SMTP_DOMAIN=smtp.mxhichina.com
        SMTP_HOST=smtp.mxhichina.com
        SMTP_PORT=25
        SMTP_USER=devops@xxxxx.com
        SMTP_PASS=password
        SMTP_AUTHENTICATION=login
        SMTP_STARTTLS=true
        SMTP_TLS=false
        TZ=Asia/Shanghai
        GITLAB_TIMEZONE=Beijing
        DB_HOST=gitlab-postgresql
        DB_NAME=gitlabhq_production
        DB_USER=gitlab
        DB_PASS=password
        REDIS_HOST=gitlab-redis
        
        ```
        
5. 运行gitlab
    
    ```bash
    docker run --name gitlab -d \
    --net router-net \
    --env-file /srv/docker/gitlab/env-file \
    --volume /srv/docker/gitlab/gitlab:/home/git/data \
    --log-driver json-file --log-opt max-size=100M --log-opt max-file=1 \
    --restart always \
    --memory 3g \
    --memory-swap 3g \
    sameersbn/gitlab:12.10.6
    docker logs -f gitlab
    ```
    

打开gitlab页面([git.local.com](http://git.local.com/))，刚开始会显示502，等一会进去之后会要求修改密码，然后登录管理员帐户，默认邮箱是admin@example.com，进去之后就可以分配帐户创建项目了

## Docker镜像仓库部署，c1节点

[官方文档](https://docs.docker.com/registry/)

1. 运行
    
    ```yaml
    docker run -d \
    -p 5000:5000 \
    --restart=always \
    --name registry \
    -v /srv/docker/registry:/var/lib/registry \
    --log-driver json-file --log-opt max-size=100M --log-opt max-file=1 \
    registry:2
    ```
    
2. 配置
    
    `vim /etc/docker/daemon.json` 全节点修改
    
    ```json
    {
    "registry-mirrors": ["https://xxxxxxx.mirror.aliyuncs.com"],
    "insecure-registries":["172.28.128.5:5000"]
    }
    ```
    
    `systemctl restart docker`
    
3. 添加一个[web管理服务](https://github.com/kwk/docker-registry-frontend)（可选）
    
    ```json
    docker run \
    -d \
    -e ENV_DOCKER_REGISTRY_HOST=172.28.128.5 \
    -e ENV_DOCKER_REGISTRY_PORT=5000 \
    -p 8081:80 \
    --log-driver json-file --log-opt max-size=100M --log-opt max-file=1 \
    --restart=always \
    konradkleine/docker-registry-frontend:v2
    ```
    

## jenkins部署，c1节点

1. 创建目录
    
    ```bash
    mkdir /srv/docker/jenkins/jenkins_home -p
    mkdir /srv/docker/jenkins/settings
    
    ```
    
2. 运行
    
    ```bash
    docker run -d --name jenkins \
    -p 8080:8080 \
    -p 50000:50000 \
    -u 0 \
    --restart always \
    --add-host git.local.com:172.28.128.5 \
    --log-driver json-file --log-opt max-size=100M --log-opt max-file=1 \
    -v /srv/docker/jenkins/jenkins_home:/var/jenkins_home:rw \
    -v /srv/docker/jenkins/settings:/var/settings:rw \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v $(which docker):/usr/bin/docker \
    jenkins/jenkins:lts
    ```
    
3. 配置
打开jenkins页面(ip:8080),提示输入密码
运行以下命令即可看到
`cat /srv/docker/jenkins/jenkins_home/secrets/initialAdminPassword`
然后选择配置插件，先选择默认，之后需要的再安装

## 项目配置

### 项目准备

一共有三个项目要准备

1. 脚本项目
    
    在gitlab创建一个demo-script项目，用于发布项目，先添加一个脚本  !
	![[update_springboot_traefik_swarm.groovy]]
2. 配置文件项目
    
    创建一个demo-config项目，用于存放不同环境的配置，文件结构就是
    
    demo-config/demo-service/application.properties #按自己项目配置
    
    demo-config/maven/maven.xml #maven私库配置，其中localRepository改成/var/.m2/repository
    
3. 服务项目
    
    这里我用了一个springboot的项目，demo-service，在项目根目录添加一个Dockerfile
	![[Dockerfile]]
    
    在pom文件添加以下内容，这样结合我的发布脚本，maven编译的时候会取出demo-config里的application.properties文件替换项目里的
    
    ```json
    <profiles>
            <profile>
                <id>configs</id>
                <build>
                    <resources>
                        <resource>
                            <directory>src/main/resources</directory>
                            <excludes>
                                <exclude>application.properties</exclude>
                            </excludes>
                        </resource>
                        <resource>
                            <directory>configs/demo-service/</directory>
                            <filtering>false</filtering>
                        </resource>
                    </resources>
                </build>
            </profile>
        </profiles>
    ```
    

### gitlab配置

首先添加一个deploy key，用于jenkins拉取项目

![](attachments/Untitled.png)

这里需要我们生成一个key 

```json
cd /srv/docker/jenkins/jenkins_home/secrets
ssh-keygen -o -t rsa -b 4096 -C ["email@example.com](mailto:%22email@example.com)"
#路径设置输入当前目录即可，后面两项可为空
Enter file in which to save the key (/root/.ssh/id_rsa): ./deploy

生成成功之后复制公钥deploy.pub里的内容填到gitlab
```

添加deploy key之后要在上面三个项目配置中启用这个key，项目主页→Settings→Repository→Deploy Keys→Privately accessible deploy keys→Enable

### Jenkins配置

打开Jenkins→凭据→系统→全局凭据→添加凭据，类型选择SSH Username with private key，id，Username都输入上一步gitlab设置的title即可，Private Key内容就是上一步生成的deploy文件内容，确定保存。

打开Jenkins→系统管理→插件管理→可选插件 搜索dingTalk插件，安装钉钉通知插件。（不需要的可以不安装，需要把脚本里的dingTalk代码删掉，否则会报错）

### 创建项目

新建一个流水线类型的项目

勾选配置项”参数化构建过程“，并添加以下参数

[参数表](https://www.notion.so/5e9c4c3193f14f5987da43dbda897e19)

然后拉到下面”流水线“模块，定义切换为Pipeline script from SCM，SCM选择git，填入脚本项目的仓库地址"[git@git.local.com](mailto:git@git.local.com):demo/demo-script.git",Credentials选择deploy-key，脚本路径修改为"update_springboot_traefik_swarm.groovy"，保存。

保存之后就可以点击"Build with Parameters"试一下发布了。