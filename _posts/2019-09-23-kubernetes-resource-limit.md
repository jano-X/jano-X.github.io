---  
layout: post                          # 表明是博文  
title: "kubernetes的资源配额和限制"           # 博文的标题  
subtitle: "通过ResourceQuota和LimitRange来实现对进群资源的限制"    # 博文的小标题  
date: 2019-09-23                     # 博文的发表日期，此日期决定主页上博文的先后顺序  
author: "jano X"                       # 博文的作者  
header-img: "img/post-bg-cka.jpg"     # 博文页面上端的背景图片  
header-mask: "0.1"                    # 博文页面上端的背景图片的亮度，数值越大越黑暗  
catalog: True                         # 开启catalog，将在博文侧边展示博文的结构  
tags:                                 # 本篇博文的标签，该标签将在archive页面中对博文进行分类  
  - kubernetes
  - ResourceQuota
  - LimitRange
  - 资源限制
---  

# k8s的资源配额和限制
kubernetes 作为当下最流行的的容器集群管理平台，需要统筹集群整体的资源使用情况，将合适的资源分配给pod容器使用，既要保证充分利用资源，提高资源利用率，又要保证重要容器在运行周期内能够分配到足够的资源稳定运行。
#### 系统版本
```bash
#物理机版本
[root@k8s-test-101 ~]# uname -a
Linux k8s-test-101 3.10.0-862.el7.x86_64 #1 SMP Fri Apr 20 16:44:24 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux

[root@k8s-test-101 ~]# cat /etc/redhat-release 
CentOS Linux release 7.5.1804 (Core) 

#docker 版本
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

#k8s版本
[root@k8s-test-101 ~]# kubectl  version
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.0", GitCommit:"e8462b5b5dc2584fdcd18e6bcfe9f1e4d970a529", GitTreeState:"archive", BuildDate:"2019-07-03T04:01:12Z", GoVersion:"go1.12.6", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.0", GitCommit:"e8462b5b5dc2584fdcd18e6bcfe9f1e4d970a529", GitTreeState:"clean", BuildDate:"2019-06-19T16:32:14Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}

```
#### 资源限制类型
k8s现阶段包括2种类型的资源限制：
- LimitRange

LimitRange限制容器的资源使用，作用于命名空间[namespace]，给单个容器[every container]设置最大，最小，默认的资源限制
```yml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-cpu-limit-range
  namespace: default
spec:
  limits:
  #如果容器没有指定request,limit,将会使用该默认配置
  #默认limit如下：
  - default:
      memory: 200Mi
      cpu: 1000m
    #默认request如下:
    defaultRequest:
      memory: 100Mi
      cpu: 100m
    #最大限制,容器 limit不能超过该值
    max:
      memory: 512Mi
      cpu: 1000m
    #最小请求,容器request不能少于该值
    min:
      memory: 10Mi
      cpu: 10m
    type: Container
```

在pod中给单个容器指定资源限制如下：
```yml
...
    spec:
      containers:
      - name: mynginx
        image: 192.168.76.104/hd-ops/mynginx:1.2.1
        imagePullPolicy: Always
        ...
        resources:            
          requests:
            cpu: 100m
            memory: 150Mi
          limits:
            cpu: 100m
            memory: 150Mi
...
```
- ResourceQuota
  
  resourceQuota 限制该命名空间内所有容器[all containers]能申请的请求[request]、限制[limit]、pod的总数
  
示例如下:
```yml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: hd-resource-quota
  namespace: default
spec:
  hard:
    #该命名空间内所有容器不能超过该request
    requests.cpu: "1"
    requests.memory: 1Gi
    #该命名空间内所有容器不能超过该limit
    limits.cpu: "2"
    limits.memory: 2Gi
    #该命名空间内pod总数不能超过该值
    pods: "2"
```

如果超出限制,会出现类似消息:
```yml
kubectl  get deployment  -oyaml
message: 'pods "mynginx-7cd54874c7-8z9jl" is forbidden: exceeded quota: hd-resource-quota,
        requested: pods=1, used: pods=2, limited: pods=2'
```