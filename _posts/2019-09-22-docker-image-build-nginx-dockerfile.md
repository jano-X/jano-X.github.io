---  
layout: post                          # 表明是博文  
title: "docker 镜像制作"           # 博文的标题  
subtitle: "基于centos7.5制作nginx镜像"    # 博文的小标题  
date: 2019-09-20                      # 博文的发表日期，此日期决定主页上博文的先后顺序  
author: "jano X"                       # 博文的作者  
header-img: "img/post-bg-cka.jpg"     # 博文页面上端的背景图片  
header-mask: "0.1"                    # 博文页面上端的背景图片的亮度，数值越大越黑暗  
catalog: True                         # 开启catalog，将在博文侧边展示博文的结构  
tags:                                 # 本篇博文的标签，该标签将在archive页面中对博文进行分类  
  - docker
  - dockerfile
  - nginx
  - image
---  

# 基于centos7.5制作nginx镜像

#### 系统版本
```bash
[root@k8s-test-101 ~]# uname -a
Linux k8s-test-101 3.10.0-862.el7.x86_64 #1 SMP Fri Apr 20 16:44:24 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux

[root@k8s-test-101 ~]# cat /etc/redhat-release 
CentOS Linux release 7.5.1804 (Core) 

[root@k8s-test-101 ~]# docker version
Client:
 Version:           18.09.6
 API version:       1.39
 Go version:        go1.10.8
 Git commit:        481bc77156
 Built:             Sat May  4 02:34:58 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          18.09.6
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.10.8
  Git commit:       481bc77
  Built:            Sat May  4 02:02:43 2019
  OS/Arch:          linux/amd64
  Experimental:     false
```

构建过程的第一件事是将整个上下文（递归地）发送到docker守护进程.因此要保持该目录的精简，只放需要用到的文件，另外可以通过【.dockerignore】来忽略某些文件。

```bash
cd /data/docker_images/nginx

tree -La 2 
.
├── Dockerfile
├── .dockerignore
└── src
    ├── html
    ├── nginx-1.12.2
    ├── nginx.conf
    ├── pcre-8.42
    └── vhost
```
主要包括3部分:
- Dockerfile 
构建镜像的主要文件，每条指令都是独立运行的,会导致创建新镜像层，因此尽量合并来减少镜像层数，缩小镜像体积。

```bash
cat Dockerfile

#
# Hd-ops Nginx Dockerfile
#

# Pull base image.
FROM centos:7.5.1804

LABEL maintainer="xiongjl0318@gmail.com"

#Install Nginx.
COPY src/ /tmp/src/

RUN  \
     cd  /tmp/src/nginx-1.12.2  && \
     useradd ops  && \
     yum -y install gcc gcc-c++ deltarpm openssl openssl-devel && \
     mkdir /opt/nginx -p && \
     ./configure --prefix=/opt/nginx --with-http_ssl_module --with-pcre=../pcre-8.42 && \
     make && make install  && \
     mkdir -p /opt/nginx/conf/vhost /log/nginx /data/html && \
     mv ../nginx.conf /opt/nginx/conf/  && \
     chown -R ops:ops  /log/ /opt /data && \
     yum clean all && rm -rf /var/cache/yum  && \
     rm -rf /tmp/src

WORKDIR /opt/nginx/

CMD ["/opt/nginx/sbin/nginx", "-g", "daemon off;"]
```
- 自定义目录src 
自定义目录src，主要用来存放需要导入到镜像中的文件。
- .dockerignore
通过指定该文件来排除文件和目录
.dockerignore 配置如下
```bash
cat .dockerignore
./Dockerfile*
```
#### 构建镜像过程
构建镜像(build image):
```bash
docker build -t 192.168.76.104/hd-ops/nginx:1.12.2-centos7.5 .
```

推送镜像(push image) 到已有的harbor仓库
```bash
docker push 192.168.76.104/hd-ops/nginx:1.12.2-centos7.5
```
将镜像拉取（pull image）到本地
```bash
docker pull 192.168.76.104/hd-ops/nginx:1.12.2-centos7.5
```
查看镜像大小
```bash
docker images |grep 1.12.2-centos7.5
192.168.76.104/hd-ops/nginx                                                   1.12.2-centos7.5    34172e515e2a        2 hours ago         367MB
```

查看镜像历史构建信息(可以看到每一层的大小)
```bash
docker history  192.168.76.104/hd-ops/nginx:1.12.2-centos7.5
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
662398ff27b5        2 hours ago         /bin/sh -c #(nop)  CMD ["/opt/nginx/sbin/ngi…   0B                  
7f5e762a254b        2 hours ago         /bin/sh -c #(nop) WORKDIR /opt/nginx/           0B                  
397e8087577c        2 hours ago         /bin/sh -c cd  /tmp/src/nginx-1.12.2  &&    …   138MB               
511ab897d630        2 hours ago         /bin/sh -c #(nop) COPY dir:e1026dbf8746281ae…   29.2MB              
69333eab6f2a        23 hours ago        /bin/sh -c #(nop)  LABEL maintainer=xiongjl0…   0B                  
cf49811e3cdb        4 months ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B                  
<missing>           4 months ago        /bin/sh -c #(nop)  LABEL org.label-schema.sc…   0B                  
<missing>           4 months ago        /bin/sh -c #(nop) ADD file:9f1be15dfdf2c68a6…   200MB 
```
#### 在k8s中使用自定义镜像
使用k8s调度后，可以在容器的定义中指定image和指定将容器的某些端口暴露出来，并可以在容器定义中,覆盖image的默认entrypoint和Cmd。dockerfile定义中的命令(ENTRYPOINT,CMD)与k8s定义的命令如下：

Description Docker | field name | Kubernetes field name
---|---|---
The command run by the container | Entrypoint | command
The arguments passed to the command | Cmd | args

容器部分配置如下 :
```conf
  spec:
      containers:
      - name: mynginx
        image: 192.168.76.104/hd-ops/nginx:1.12.2-centos7.5
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        command: 
        - /opt/nginx/sbin/nginx
        args: 
        - -g
        - daemon off;
```
> 仅截取部分配置