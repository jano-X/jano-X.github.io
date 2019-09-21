---  
layout: post                          # 表明是博文  
title: "kubernetes多主集群安装"           # 博文的标题  
subtitle: 通过kubeadm自定义目录初始化    # 博文的小标题  
date: 2019-09-21                     # 博文的发表日期，此日期决定主页上博文的先后顺序  
author: "jano X"                       # 博文的作者  
header-img: "img/post-bg-cka.jpg"     # 博文页面上端的背景图片  
header-mask: "0.1"                    # 博文页面上端的背景图片的亮度，数值越大越黑暗  
catalog: True                         # 开启catalog，将在博文侧边展示博文的结构  
tags:                                 # 本篇博文的标签，该标签将在archive页面中对博文进行分类  
  - git  
  - github  
  - markdown  
  - kubernetes  
---  

## 环境说明
#### 系统版本
```bash
[root@k8s-test-104 ~]# uname -a
Linux k8s-test-104 3.10.0-862.el7.x86_64 #1 SMP Fri Apr 20 16:44:24 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux

[root@k8s-test-104 ~]# cat /etc/redhat-release 
CentOS Linux release 7.5.1804 (Core) 
```
#### 准备安装的集群版本信息
- docker Version: `18.09.6`
- kubernetes Version: 
```conf
client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.0", GitCommit:"e8462b5b5dc2584fdcd18e6bcfe9f1e4d970a529", GitTreeState:"clean", BuildDate:"2019-06-19T16:40:16Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.0", GitCommit:"e8462b5b5dc2584fdcd18e6bcfe9f1e4d970a529", GitTreeState:"clean", BuildDate:"2019-06-19T16:32:14Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
```
- 集群负载均衡类型: nginx 1.12.2
  
- 集群访问入口(Apiserver VIP地址): 172.16.20.244:6443
  
- 镜像版本
```
192.168.76.104/k8s.gcr.io/kube-proxy                v1.15.0             d235b23c3570        7 weeks ago         82.4MB
192.168.76.104/k8s.gcr.io/kube-apiserver            v1.15.0             201c7a840312        7 weeks ago         207MB
192.168.76.104/k8s.gcr.io/kube-controller-manager   v1.15.0             8328bb49b652        7 weeks ago         159MB
192.168.76.104/k8s.gcr.io/kube-scheduler            v1.15.0             2d3813851e87        7 weeks ago         81.1MB
192.168.76.104/k8s.gcr.io/etcd                      3.3.10              2c4adeb21b4f        8 months ago        258MB
192.168.76.104/k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        20 months ago       742kB
192.168.76.104/calico/node                                         v3.8.1              4fcdc789ef1a        2 weeks ago         189MB
192.168.76.104/calico/pod2daemon-flexvol                           v3.8.1              6f36255ccd3f        2 weeks ago         9.37MB
192.168.76.104/calico/cni                                          v3.8.1              f89121d9c764        2 weeks ago         143MB
192.168.76.104/calico/kube-controllers                             v3.8.1              214ddcd2a33e        2 weeks ago         46.8MB
192.168.76.104/coredns/coredns                      1.5.0               7987f0908caf        4 months ago        42.5MB
```
  
- 集群拓扑
本次安装涉及到的服务器列表:

| Hostname | IP | Role | Comment |
| --- | --- |  --- | --- |
| k8s-test-101 | 192.168.76.101 | Master | ControlPanel节点 | 
| k8s-test-102 | 192.168.76.102 | Master | ControlPanel节点 | 
| k8s-test-103 | 192.168.76.103 | Master | ControlPanel节点 | 
| k8s-test-104 | 192.168.76.104 |node,Harbor | worker node及harbor私有仓库 |
| hd-test-244 | 172.16.20.244 | LoadBalancer | 负载均衡节点(nginx)

- 自定义目录规范

1. docker
```
   数据目录： /var/lib/docker
```
2. kube-apiserver
```
   审计日志： /log/kube-apiserver-audit.log
   日志目录： /log/kube-apiserver
```
3. kube-controller-manager
```
   日志目录： /log/kube-controller-manager
```
4. scheduler
```
   日志目录： /log/kube-scheduler
```
5. kubelet
```
   日志目录： /log/kubelet
   数据目录： /var/lib/kubelet
   配置目录： /opt/kubernetes/conf
   安装目录： /opt/kubernetes
   证书目录： /opt/kubernetes/conf/pki
```
6. etcd
```
   数据目录：/data/etcd
   证书目录：/opt/kubernetes/conf/pki/etcd
```
7. kube-proxy
```
   代理模式： ipvs
   调度算法： rr
```

#### 环境配置（所有节点执行，master 以及 node）
所有节点服务器均关闭防火墙\SELINUX\SWAP
完整的内核参数[/etc/sysctl.conf]如下:
```conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
kernel.sysrq = 0
kernel.core_uses_pid = 1
kernel.shmmni = 4096
kernel.shmall = 4294967296
kernel.shmmax = 8589934592
kernel.msgmnb = 65536
kernel.msgmax = 65536
net.core.netdev_max_backlog = 32768
net.core.somaxconn = 32768
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_max_syn_backlog = 65536
net.ipv4.tcp_tw_reuse = 0
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_fin_timeout= 30
net.ipv4.tcp_keepalive_probes= 3
net.ipv4.tcp_keepalive_intvl= 20
net.ipv4.tcp_keepalive_time= 1800
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 87380 16777216
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_max_tw_buckets = 2000
net.ipv4.tcp_window_scaling = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.ip_local_port_range = 30000  65000
net.ipv4.icmp_echo_ignore_broadcasts = 1
vm.swappiness = 0
#vm.overcommit_memory = 1
kernel.core_pattern = /data/corefile/core-%e-%s-%u-%g-%p-%t
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
使生效:

```bash
sysctl -p
```

> **kernel.core_pattern**: 设置coredump路径 **net.ipv4.ip_forward = 1,net.bridge.bridge-nf-call-ip6tables = 1,net.bridge.bridge-nf-call-iptables = 1**为kuberneter必须设置的参数

设置 coredump（内存参数已配置 coredump 存储路径）

```bash
mkdir /data/corefile -p
cat <<EOF >> /etc/security/limits.conf
*               soft    core            unlimited
*               hard    core            unlimited
EOF

#应用当前 bash 环境：
ulimit -c unlimited
```

selinux配置如下:
```bash
cat /etc/sysconfig/selinux  | grep -v ^# | grep -v ^$
SELINUX=disabled
SELINUXTYPE=targeted
```
禁用swap分区
```bash
swapoff -a

sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
确保每个节点的**MAC**和**product_uuid**是唯一的
```bash
ifconfig -a

cat /sys/class/dmi/id/product_uuid
```
> 更详细的系统要求可参考官网[Installing kubeadm]

[Installing kubeadm]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

#### 安装runtime docker-ce 
安装脚本[install_docker_ce.sh]如下:
```bash
# Install Docker CE
## Set up the repository
### Install required packages.
    yum install -y yum-utils device-mapper-persistent-data lvm2

### Add docker repository.
yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

## Install docker ce.
yum install -y docker-ce-18.09.6 docker-ce-cli-18.09.6

## Create /etc/docker directory.
mkdir /etc/docker

# Setup daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart docker.
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
```
#### IPVS安装
代理模式采用**ipvs**，需要在**每个节点**安装 `ipvsadm`
```bash
yum -y install ipvsadm  ipset

cat > /etc/sysconfig/modules/ipvs.modules <<EOF
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

chmod 755  /etc/sysconfig/modules/ipvs.modules 
sh /etc/sysconfig/modules/ipvs.modules

systemctl stop iptables
systemctl stop firewalld
systemctl disable iptables
systemctl disable firewalld

iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

lsmod | grep ip_vs
ip_vs_sh               12688  0 
ip_vs_wrr              12697  0 
ip_vs_rr               12600  0 
ip_vs                 141432  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          133053  9 ip_vs,nf_nat,nf_nat_ipv4,nf_nat_ipv6,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack_ipv4,nf_conntrack_ipv6
libcrc32c              12644  4 xfs,ip_vs,nf_nat,nf_conntrack
```
#### 负载均衡nginx配置
安装部分不再描述
主要需要加载stream模块,版本信息如下:
```bash
[root@hd-test-244 ~]# /opt/nginx/sbin/nginx -V
nginx version: nginx/1.12.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/opt/nginx --with-http_ssl_module --with-pcre=/tmp/pcre-8.42 --with-stream
```
主要配置[nginx.conf]如下：
```conf 
stream{
upstream apiserver{
  server 192.168.76.101:6443;
  server 192.168.76.102:6443;
  server 192.168.76.103:6443;
}

server {
    listen 6443;
    proxy_connect_timeout 2s;
    proxy_pass apiserver;
      }
}
```
#### harbor私有仓库配置
> 请自行安装或使用公共镜像

---
## kubernetes集群部署
#### 下载安装包
- 下载二进制安装包

- 服务端
URL: `https://dl.k8s.io/v1.15.2/kubernetes-server-linux-amd64.tar.gz`
```bash
#sha256 校验码： 
faa734636ca18ae8786552eab317c0288cf958525df9236ce9b734d5b13809c002bb1538763bc0f8cbb2efa4340152f2cc78630e6158ed7edee163ac53570e22
```
- 客户端
URL: `https://dl.k8s.io/v1.15.2/kubernetes-client-linux-amd64.tar.gz`
```bash
#sha256 校验码： 
8a5f25e74a317169687abd65c499fc4f7cc2ff0824a7a9830acd38bfeec9923d8a0d653c3b1eec147c44a853ceeaa62bbb741f17e69b4df614008e53a2fe5b08
```

#### 安装命令行工具
- 安装kubernetes二进制文件{kubectl,kubelet,kubeadm}

```bash
mkdir /root/kubeadm
cd /root/kubeadm
```

上传二进制包及配置到目录`/root/kubeadm`

```bash
[root@k8s-test-101 kubeadm]# ls
audit-policy.yml  calico.yaml  initCluster.yaml-template  k8s-cluster-init.sh  kubernetes-server-linux-amd64.tar.gz k8s.gcr.io-k8s-images.tar

[root@k8s-test-101 kubeadm]# tar xzf kubernetes-server-linux-amd64.tar.gz 
```

复制二进制文件到安装目录`/opt/kubernetes/bin`
```bash
mkdir /opt/kubernetes/{bin,conf} -p
cp  /root/kubeadm/kubernetes/server/bin/{kubectl,kubelet,kubeadm} /opt/kubernetes/bin/
```

- 配置环境变量

```bash
echo 'export PATH=$PATH:/opt/kubernetes/bin/' >> /etc/profile
echo 'export KUBECONFIG=/opt/kubernetes/conf/admin.conf' >> /etc/profile
source /etc/profile
```

- 因为初始化时开启了审计，需上传 `audit-policy.yml` 审计日志策略文件到 `/opt/kubernetes/conf/ `

```bash
cp /root/kubeadm/audit-policy.yml /opt/kubernetes/conf/
```
> 可选，如不需要开启apiserver审计，需修改下面的安装脚本,去掉审计相关配置

#### 下载镜像 (只需要在一台机器push镜像)

因官方镜像需翻墙，这里通过`docker save`及`docer load`来导入镜像
```bash
cd /root/kubeadm
mkdir k8s.gcr.io
cd k8s.gcr.io/
tar xf  ../k8s.gcr.io-k8s-images.tar 
for i in $(ls *.tar);do docker load -i $i;done

docker images |grep -E '^k8s.gcr.io'
k8s.gcr.io/kube-scheduler                           v1.15.2             88fa9cb27bd2        9 days ago          81.1MB
k8s.gcr.io/kube-proxy                               v1.15.2             167bbf6c9338        9 days ago          82.4MB
k8s.gcr.io/kube-apiserver                           v1.15.2             34a53be6c9a7        9 days ago          207MB
k8s.gcr.io/kube-controller-manager                  v1.15.2             9f5df470155d        9 days ago          159MB
k8s.gcr.io/coredns                                  1.3.1               eb516548c180        7 months ago        40.3MB
k8s.gcr.io/etcd                                     3.3.10              2c4adeb21b4f        8 months ago        258MB
k8s.gcr.io/pause                                    3.1                 da86e6ba6ca1        20 months ago       742kB

for IMAGE in $(docker images |grep -E '^k8s.gcr.io' |awk '{image=$1":"$2}{print image}');
do
  #docker tag ,生成tag命名来执行。
  docker tag ${IMAGE} 192.168.76.104/${IMAGE}
  #push到私有仓库
  docker push 192.168.76.104/${IMAGE}
done  

docker images |grep -E '^192.168.76.104/k8s.gcr.io'
192.168.76.104/k8s.gcr.io/kube-apiserver            v1.15.2             34a53be6c9a7        4 weeks ago         207MB
192.168.76.104/k8s.gcr.io/kube-scheduler            v1.15.2             88fa9cb27bd2        4 weeks ago         81.1MB
192.168.76.104/k8s.gcr.io/kube-proxy                v1.15.2             167bbf6c9338        4 weeks ago         82.4MB
192.168.76.104/k8s.gcr.io/kube-controller-manager   v1.15.2             9f5df470155d        4 weeks ago         159MB
192.168.76.104/k8s.gcr.io/coredns                   1.3.1               eb516548c180        7 months ago        40.3MB
192.168.76.104/k8s.gcr.io/etcd                      3.3.10              2c4adeb21b4f        9 months ago        258MB
192.168.76.104/k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        20 months ago       742kB

```
> 如果不使用私有仓库拉取镜像，可以将镜像`docker load`手动载入到每个节点。

#### 生成配置及证书文件
- 生成证书

通过脚本`k8s-cluster-init.sh`生成所有master节点的证书

**脚本中以下部分配置必须修改:**
```bash
HOSTS=(
  ['k8s-test-101']=192.168.76.101
  ['k8s-test-102']=192.168.76.102
  ['k8s-test-103']=192.168.76.103
)
APISERVER_VIP=172.16.20.244
```
> 如果不使用私有仓库拉取镜像,需要注释`k8s-cluster-init.sh`脚本的`imageRepository: "192.168.76.104/k8s.gcr.io"`

执行初始化脚本
```bash
sh k8s-cluster-init.sh
#查看证书
[root@k8s-test-101 kubeadm]# cd /tmp/k8s-cluster-init/
[root@k8s-test-101 k8s-cluster-init]# ls
k8s-test-101  k8s-test-102  k8s-test-103
[root@k8s-test-101 k8s-cluster-init]# ls k8s-test-101
k8s-cluster-init.yaml  pki
[root@k8s-test-101 k8s-cluster-init]# ls k8s-test-101/pki/
apiserver.crt              apiserver.key                 ca.crt  front-proxy-ca.crt      front-proxy-client.key
apiserver-etcd-client.crt  apiserver-kubelet-client.crt  ca.key  front-proxy-ca.key      sa.key
apiserver-etcd-client.key  apiserver-kubelet-client.key  etcd    front-proxy-client.crt  sa.pub
```

- 拷贝安装包、证书以及初始化配置文件到其他 Master 节点
```bash
[root@k8s-test-101 ~]# ls /opt/kubernetes/conf/
[root@k8s-test-101 ~]# 
[root@k8s-test-101 ~]# 
[root@k8s-test-101 ~]# cd /opt/
[root@k8s-test-101 opt]# scp -r kubernetes 192.168.76.102:/opt
kubectl                                                                                                                 100%   41MB 109.5MB/s   00:00    
kubelet                                                                                                                 100%  114MB  46.0MB/s   00:02    
kubeadm                                                                                                                 100%   38MB  86.3MB/s   00:00    
[root@k8s-test-101 opt]# scp -r kubernetes 192.168.76.103:/opt
kubectl                                                                                                                 100%   41MB 108.0MB/s   00:00    
kubelet                                                                                                                 100%  114MB 114.9MB/s   00:00    
kubeadm                                                                                                                 100%   38MB  94.0MB/s   00:00    
[root@k8s-test-101 opt]# cd /tmp/k8s-cluster-init/
[root@k8s-test-101 k8s-cluster-init]# ls
k8s-test-101  k8s-test-102  k8s-test-103
[root@k8s-test-101 k8s-cluster-init]# scp -r k8s-test-102 192.168.76.102:/tmp/
k8s-cluster-init.yaml                                                                                                   100% 5402     2.5MB/s   00:00    
ca.key                                                                                                                  100% 1675   913.3KB/s   00:00    
ca.crt                                                                                                                  100% 1017   702.1KB/s   00:00    
peer.key                                                                                                                100% 1675     1.0MB/s   00:00    
peer.crt                                                                                                                100% 1143   644.3KB/s   00:00    
healthcheck-client.key                                                                                                  100% 1675     1.0MB/s   00:00    
healthcheck-client.crt                                                                                                  100% 1094   655.8KB/s   00:00    
server.key                                                                                                              100% 1679     1.0MB/s   00:00    
server.crt                                                                                                              100% 1143   735.2KB/s   00:00    
ca.key                                                                                                                  100% 1679     1.1MB/s   00:00    
ca.crt                                                                                                                  100% 1025   687.6KB/s   00:00    
front-proxy-ca.key                                                                                                      100% 1675     1.2MB/s   00:00    
front-proxy-ca.crt                                                                                                      100% 1038   720.6KB/s   00:00    
sa.key                                                                                                                  100% 1675     1.1MB/s   00:00    
sa.pub                                                                                                                  100%  451   345.7KB/s   00:00    
apiserver-kubelet-client.key                                                                                            100% 1675     1.0MB/s   00:00    
apiserver-kubelet-client.crt                                                                                            100% 1099   725.9KB/s   00:00    
apiserver.key                                                                                                           100% 1679     1.0MB/s   00:00    
apiserver.crt                                                                                                           100% 1237   791.1KB/s   00:00    
front-proxy-client.key                                                                                                  100% 1679   955.9KB/s   00:00    
front-proxy-client.crt                                                                                                  100% 1058   486.9KB/s   00:00    
apiserver-etcd-client.key                                                                                               100% 1679   847.3KB/s   00:00    
apiserver-etcd-client.crt                                                                                               100% 1090   748.8KB/s   00:00    
[root@k8s-test-101 k8s-cluster-init]# scp -r k8s-test-103 192.168.76.103:/tmp/
k8s-cluster-init.yaml                                                                                                   100% 5402     3.9MB/s   00:00    
ca.key                                                                                                                  100% 1675     1.5MB/s   00:00    
ca.crt                                                                                                                  100% 1017   896.2KB/s   00:00    
server.key                                                                                                              100% 1675     1.5MB/s   00:00    
server.crt                                                                                                              100% 1143     1.1MB/s   00:00    
peer.key                                                                                                                100% 1679     1.6MB/s   00:00    
peer.crt                                                                                                                100% 1143     1.1MB/s   00:00    
healthcheck-client.key                                                                                                  100% 1679     1.4MB/s   00:00    
healthcheck-client.crt                                                                                                  100% 1094     1.0MB/s   00:00    
apiserver-etcd-client.key                                                                                               100% 1675     1.5MB/s   00:00    
apiserver-etcd-client.crt                                                                                               100% 1090   915.6KB/s   00:00    
ca.key                                                                                                                  100% 1679     1.5MB/s   00:00    
ca.crt                                                                                                                  100% 1025   861.8KB/s   00:00    
apiserver-kubelet-client.key                                                                                            100% 1675     1.6MB/s   00:00    
apiserver-kubelet-client.crt                                                                                            100% 1099     1.1MB/s   00:00    
apiserver.key                                                                                                           100% 1679     1.4MB/s   00:00    
apiserver.crt                                                                                                           100% 1237     1.2MB/s   00:00    
front-proxy-ca.key                                                                                                      100% 1675     1.5MB/s   00:00    
front-proxy-ca.crt                                                                                                      100% 1038   1000.0KB/s   00:00   
front-proxy-client.key                                                                                                  100% 1675     1.4MB/s   00:00    
front-proxy-client.crt                                                                                                  100% 1058     1.1MB/s   00:00    
sa.key                                                                                                                  100% 1675     1.7MB/s   00:00    
sa.pub 
```

- 配置 kubelet 启动服务(Master 节点)

```bash
cat > /usr/lib/systemd/system/kubelet.service <<"EOF"
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/

[Service]
ExecStart=/opt/kubernetes/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

mkdir -p /usr/lib/systemd/system/kubelet.service.d

cat > /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf <<"EOF"
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/opt/kubernetes/conf/bootstrap-kubelet.conf --kubeconfig=/opt/kubernetes/conf/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
EnvironmentFile=-/opt/kubernetes/conf/kubelet.env 
ExecStart=
ExecStart=/opt/kubernetes/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_EXTRA_ARGS $KUBELET_KUBEADM_ARGS
EOF

echo 'KUBELET_EXTRA_ARGS=' > /opt/kubernetes/conf/kubelet.env 
```

复制kubelet启动配置到其他master节点
```bash
[root@k8s-test-101 k8s-cluster-init]# scp -r  /usr/lib/systemd/system/kubelet.service /usr/lib/systemd/system/kubelet.service.d  192.168.76.102:/usr/lib/systemd/system/ 
kubelet.service                                                                                                         100%  233   341.8KB/s   00:00    
10-kubeadm.conf                                                                                                         100%  448    26.8KB/s   00:00    
[root@k8s-test-101 k8s-cluster-init]# scp -r /usr/lib/systemd/system/kubelet.service /usr/lib/systemd/system/kubelet.service.d 192.168.76.103:/usr/lib/systemd/system/
kubelet.service                                                                                                         100%  233    12.9KB/s   00:00    
10-kubeadm.conf  
[root@k8s-test-101 kubeadm]# scp /opt/kubernetes/conf/kubelet.env  192.168.76.102:/opt/kubernetes/conf/
kubelet.env                                                                                                             100%   20    17.1KB/s   00:00    
[root@k8s-test-101 kubeadm]# scp /opt/kubernetes/conf/kubelet.env  192.168.76.103:/opt/kubernetes/conf/
kubelet.env 
```
- 配置第一个master节点
1. 复制证书文件及集群初始化文件`k8s-cluster-init.yaml`到安装目录`/opt/kubernetes/conf/`
```bash
[root@k8s-test-101 k8s-cluster-init]# cd /tmp/k8s-cluster-init/k8s-test-101/
[root@k8s-test-101 k8s-test-101]# ls
k8s-cluster-init.yaml  pki
[root@k8s-test-101 k8s-test-101]# mv pki k8s-cluster-init.yaml  /opt/kubernetes/conf/
[root@k8s-test-101 k8s-test-101]# 
```
2. 执行初始化之前的检查
```bash
[root@k8s-test-101 k8s-test-101]# cd /opt/kubernetes/conf/
[root@k8s-test-101 conf]# kubeadm init phase preflight  --config k8s-cluster-init.yaml 
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
```
3. 生成 ETCD yaml静态pod配置文件
```bash
[root@kubernetes-master-134 kubernetes-master-134]# kubeadm init phase etcd local --config=k8s-cluster-init.yaml 
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
```
4. 生成 k8s master 组件 yaml 静态 pod 配置文件
```bash
[root@k8s-test-101 conf]# kubeadm init phase control-plane all --config=k8s-cluster-init.yaml
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[controlplane] Adding extra host path mount "audit-policy-file" to "kube-apiserver"
[controlplane] Adding extra host path mount "log-dir" to "kube-apiserver"
[controlplane] Adding extra host path mount "log-dir" to "kube-controller-manager"
[controlplane] Adding extra host path mount "kubeconfig" to "kube-controller-manager"
[controlplane] Adding extra host path mount "k8s-certs" to "kube-scheduler"
[controlplane] Adding extra host path mount "log-dir" to "kube-scheduler"
[controlplane] Adding extra host path mount "kubeconfig" to "kube-scheduler"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[controlplane] Adding extra host path mount "audit-policy-file" to "kube-apiserver"
[controlplane] Adding extra host path mount "log-dir" to "kube-apiserver"
[controlplane] Adding extra host path mount "log-dir" to "kube-controller-manager"
[controlplane] Adding extra host path mount "kubeconfig" to "kube-controller-manager"
[controlplane] Adding extra host path mount "k8s-certs" to "kube-scheduler"
[controlplane] Adding extra host path mount "log-dir" to "kube-scheduler"
[controlplane] Adding extra host path mount "kubeconfig" to "kube-scheduler"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[controlplane] Adding extra host path mount "audit-policy-file" to "kube-apiserver"
[controlplane] Adding extra host path mount "log-dir" to "kube-apiserver"
[controlplane] Adding extra host path mount "log-dir" to "kube-controller-manager"
[controlplane] Adding extra host path mount "kubeconfig" to "kube-controller-manager"
[controlplane] Adding extra host path mount "k8s-certs" to "kube-scheduler"
[controlplane] Adding extra host path mount "log-dir" to "kube-scheduler"
[controlplane] Adding extra host path mount "kubeconfig" to "kube-scheduler"
```

5. 生成 k8s master 以及 kubelet 组件的 kubeconfig 配置文件
```bash
[root@k8s-test-101 conf]# kubeadm init phase kubeconfig all --config=k8s-cluster-init.yaml 
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
```
6. 将配置文件移动到k8s程序目录
```bash
[root@k8s-test-101 conf]# mv /etc/kubernetes/* /opt/kubernetes/conf/
[root@k8s-test-101 conf]# rm -rf /etc/kubernetes/
[root@k8s-test-101 conf]# mkdir /log/kubelet -p
```

7. 生成 kubelet 配置文件以及启动 kubelet 进程
```bash
#因为修改了kubelet系统启动配置,重载守护进程
[root@k8s-test-101 conf]# systemctl daemon-reload
[root@k8s-test-101 conf]# kubeadm init phase kubelet-start --config=k8s-cluster-init.yaml
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[root@k8s-test-101 conf]# systemctl enable kubelet
Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /usr/lib/systemd/system/kubelet.service.
```

#### 配置其余2个master节点
登录`节点k8s-test-102`开始配置
```bash
yum install socat -y
echo 'export PATH=$PATH:/opt/kubernetes/bin/' >> /etc/profile
echo 'export KUBECONFIG=/opt/kubernetes/conf/admin.conf' >> /etc/profile
source /etc/profile
cd /tmp/k8s-test-102/
ls 
mv pki k8s-cluster-init.yaml  /opt/kubernetes/conf/
cd /opt/kubernetes/conf/
kubeadm init phase etcd local --config=k8s-cluster-init.yaml
kubeadm init phase control-plane all --config=k8s-cluster-init.yaml
kubeadm init phase kubeconfig all --config=k8s-cluster-init.yaml  
mv /etc/kubernetes/* /opt/kubernetes/conf/
rm -rf /etc/kubernetes/
mkdir /log/kubelet -p
systemctl daemon-reload
kubeadm init phase kubelet-start --config=k8s-cluster-init.yaml
systemctl enable kubelet
```
登录`节点k8s-test-103`开始配置
```bash
echo 'export PATH=$PATH:/opt/kubernetes/bin/' >> /etc/profile
echo 'export KUBECONFIG=/opt/kubernetes/conf/admin.conf' >> /etc/profile
source /etc/profile
cd /tmp/k8s-test-103/
ls 
mv pki k8s-cluster-init.yaml  /opt/kubernetes/conf/
cd /opt/kubernetes/conf/
kubeadm init phase etcd local --config=k8s-cluster-init.yaml
kubeadm init phase control-plane all --config=k8s-cluster-init.yaml
kubeadm init phase kubeconfig all --config=k8s-cluster-init.yaml  
mv /etc/kubernetes/* /opt/kubernetes/conf/
rm -rf /etc/kubernetes/
mkdir /log/kubelet -p
systemctl daemon-reload
kubeadm init phase kubelet-start --config=k8s-cluster-init.yaml
systemctl enable kubelet
```

#### 检查3个控制平面是否正常
- 检查`control-plane`组件是否都起来了,正常情况下因看到以下结果:
```bash
[root@k8s-test-101 conf]# kubectl  get pods -n kube-system -owide
NAME                                   READY   STATUS    RESTARTS   AGE     IP               NODE           NOMINATED NODE   READINESS GATES
etcd-k8s-test-101                      1/1     Running   0          2m44s   192.168.76.101   k8s-test-101   <none>           <none>
etcd-k8s-test-102                      1/1     Running   0          2m35s   192.168.76.102   k8s-test-102   <none>           <none>
etcd-k8s-test-103                      1/1     Running   0          96s     192.168.76.103   k8s-test-103   <none>           <none>
kube-apiserver-k8s-test-101            1/1     Running   2          95s     192.168.76.101   k8s-test-101   <none>           <none>
kube-apiserver-k8s-test-102            1/1     Running   2          2m45s   192.168.76.102   k8s-test-102   <none>           <none>
kube-apiserver-k8s-test-103            1/1     Running   0          113s    192.168.76.103   k8s-test-103   <none>           <none>
kube-controller-manager-k8s-test-101   1/1     Running   0          2m38s   192.168.76.101   k8s-test-101   <none>           <none>
kube-controller-manager-k8s-test-102   1/1     Running   0          2m19s   192.168.76.102   k8s-test-102   <none>           <none>
kube-controller-manager-k8s-test-103   1/1     Running   0          113s    192.168.76.103   k8s-test-103   <none>           <none>
kube-scheduler-k8s-test-101            1/1     Running   0          2m51s   192.168.76.101   k8s-test-101   <none>           <none>
kube-scheduler-k8s-test-102            1/1     Running   0          2m34s   192.168.76.102   k8s-test-102   <none>           <none>
kube-scheduler-k8s-test-103            1/1     Running   0          114s    192.168.76.103   k8s-test-103   <none>           <none>
```

- mark-control-plane 创建标签和污点
 
```bash
#需每个master节点执行
mkdir /etc/kubernetes -p
cp /opt/kubernetes/conf/admin.conf /etc/kubernetes/
kubeadm init phase mark-control-plane   --config k8s-cluster-init.yaml
rm  -rf /etc/kubernetes/
```
- uploadconfig 上传配置到集群
```bash
kubeadm init phase uploadconfig  all --config k8s-cluster-init.yaml --kubeconfig /opt/kubernetes/conf/admin.conf 
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.15" in namespace kube-system with the configuration for the kubelets in the cluster
```
- bootstrap-token 
```bash
kubeadm init phase bootstrap-token  --config k8s-cluster-init.yaml --kubeconfig /opt/kubernetes/conf/admin.conf     
[bootstrap-token] Using token: 783bde.3f89s0fje9f38fhf
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
```

- addon 创建CoreDNS和kube-proxy

```bash
kubeadm  init phase addon all --config k8s-cluster-init.yaml --kubeconfig /opt/kubernetes/conf/admin.conf 
```

- calico

```bash
cd /root/kubeadm 
kubectl  apply -f calico.yaml
```

#### 加入worker节点
```bash
mkdir /opt/kubernetes/{bin,conf} -p
mkdir /opt/kubernetes/conf/{pki,kubelet,manifests} -p
mkdir /var/lib/kubelet/ -p
```
- 创建命令集
```bash
scp -r 192.168.76.101:/opt/kubernetes/bin /opt/kubernetes/
```

- 将kubelet配置为系统服务
```bash
scp -r 192.168.76.101:/usr/lib/systemd/system/{kubelet.service,kubelet.service.d} /usr/lib/systemd/system/
```

- 通过**kubeadm join**来生成引导文件和配置文件
执行`kubeadm join `来加入,格式如下：
```bash
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```

令牌(token)获取,登录能访问集群的机器`k8s-test-101`
```bash
#k8s-test-101 操作
kubeadm token list
#默认情况下，令牌在24小时后过期。如果在当前令牌过期后将节点加入群集，则可以运行以下命令来创建新令牌：
kubeadm token create
```

如果没有值`--discovery-token-ca-cert-hash`，可以通过在控制平面节点上运行以下命令链来获取它：
```bash
#k8s-test-101 操作
openssl x509 -pubkey -in /opt/kubernetes/conf/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

<master-ip>:<master-port> 为负载均衡的IP和端口,对应端点为*apiserver*的安全端口

```bash
k8s-test-104 操作
kubeadm  join phase kubelet-start --token bp11t6.5zmwxk09fo11lgl0 172.16.20.244:6443 --discovery-token-ca-cert-hash sha256:35b5fa6085177e0c874cb048d4bf68adffd502102d1a153ecfc5e26f610cb13f 
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
#出现这个就可以直接[CTRL+C]了
#将生成的证书和配置拷贝到自定义目录
mv /etc/kubernetes/*  /opt/kubernetes/conf/ -f
```

- 启动kubelet

```bash
systemctl  daemon-reload 
systemctl restart kubelet
```

- 检查worker节点是否配置成功

```bash
#在master节点操作
kubectl  get nodes
NAME           STATUS   ROLES    AGE   VERSION
k8s-test-101   Ready    master   57m   v1.15.2
k8s-test-102   Ready    master   57m   v1.15.2
k8s-test-103   Ready    master   57m   v1.15.2
k8s-test-104   Ready    <none>   29m   v1.15.2

kubectl label node k8s-test-104 node-role.kubernetes.io/node=
```
#### 最后部署结果

```bash
#在master节点操作
[root@k8s-test-101 conf]# kubectl  get pods -n kube-system -owide
NAME                                       READY   STATUS    RESTARTS   AGE    IP               NODE           NOMINATED NODE   READINESS GATES
calico-kube-controllers-7bd78b474d-w4nrn   1/1     Running   0          20m    10.100.147.66    k8s-test-101   <none>           <none>
calico-node-2lds7                          1/1     Running   0          2m8s   192.168.76.104   k8s-test-104   <none>           <none>
calico-node-5djlw                          1/1     Running   0          20m    192.168.76.103   k8s-test-103   <none>           <none>
calico-node-hc8gr                          1/1     Running   0          20m    192.168.76.102   k8s-test-102   <none>           <none>
calico-node-xhwb9                          1/1     Running   0          20m    192.168.76.101   k8s-test-101   <none>           <none>
coredns-68ccfcb756-lhbkk                   1/1     Running   0          22m    10.100.147.64    k8s-test-101   <none>           <none>
coredns-68ccfcb756-lwlsk                   1/1     Running   0          22m    10.100.147.65    k8s-test-101   <none>           <none>
etcd-k8s-test-101                          1/1     Running   0          30m    192.168.76.101   k8s-test-101   <none>           <none>
etcd-k8s-test-102                          1/1     Running   0          29m    192.168.76.102   k8s-test-102   <none>           <none>
etcd-k8s-test-103                          1/1     Running   0          29m    192.168.76.103   k8s-test-103   <none>           <none>
kube-apiserver-k8s-test-101                1/1     Running   2          28m    192.168.76.101   k8s-test-101   <none>           <none>
kube-apiserver-k8s-test-102                1/1     Running   2          30m    192.168.76.102   k8s-test-102   <none>           <none>
kube-apiserver-k8s-test-103                1/1     Running   0          29m    192.168.76.103   k8s-test-103   <none>           <none>
kube-controller-manager-k8s-test-101       1/1     Running   0          30m    192.168.76.101   k8s-test-101   <none>           <none>
kube-controller-manager-k8s-test-102       1/1     Running   0          29m    192.168.76.102   k8s-test-102   <none>           <none>
kube-controller-manager-k8s-test-103       1/1     Running   0          29m    192.168.76.103   k8s-test-103   <none>           <none>
kube-proxy-4vhgx                           1/1     Running   0          22m    192.168.76.101   k8s-test-101   <none>           <none>
kube-proxy-59vbr                           1/1     Running   0          22m    192.168.76.103   k8s-test-103   <none>           <none>
kube-proxy-f2kzr                           1/1     Running   0          22m    192.168.76.102   k8s-test-102   <none>           <none>
kube-proxy-jpcdn                           1/1     Running   0          2m8s   192.168.76.104   k8s-test-104   <none>           <none>
kube-scheduler-k8s-test-101                1/1     Running   0          30m    192.168.76.101   k8s-test-101   <none>           <none>
kube-scheduler-k8s-test-102                1/1     Running   0          29m    192.168.76.102   k8s-test-102   <none>           <none>
kube-scheduler-k8s-test-103                1/1     Running   0          29m    192.168.76.103   k8s-test-103   <none>           <none>
```