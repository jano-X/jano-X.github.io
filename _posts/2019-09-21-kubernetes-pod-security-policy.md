---  
layout: post                          # 表明是博文  
title: "kubernetes容器安全策略"           # 博文的标题  
subtitle: "[PodSecurityPolicy]的测试及使用"    # 博文的小标题  
date: 2019-09-20                      # 博文的发表日期，此日期决定主页上博文的先后顺序  
author: "jano X"                       # 博文的作者  
header-img: "img/post-bg-cka.jpg"     # 博文页面上端的背景图片  
header-mask: "0.1"                    # 博文页面上端的背景图片的亮度，数值越大越黑暗  
catalog: True                         # 开启catalog，将在博文侧边展示博文的结构  
tags:                                 # 本篇博文的标签，该标签将在archive页面中对博文进行分类  
  - kubernetes  
  - docker  
  - psp  
  - PodSecurityPolicy
---

# kubernetes Pod安全策略
Pod安全策略支持对pod创建和更新进行细粒度授权

## 什么是容器组安全策略[PodSecurityPolicy]


_容器组安全策略_ 是集群级别的用于控制pod安全相关选项的一种资源.
`PodSecurityPolicy`定义了一系列pod相要进行在系统中必须满足的约束条件,以及一些默认的约束值.它允许管理员控制以下方面内容:

| Control Aspect                                      | Field Names                                 |
| ----------------------------------------------------| ------------------------------------------- |
| Running of privileged containers                    | [`privileged`](#privileged)                                |
| Usage of host namespaces                            | [`hostPID`, `hostIPC`](#host-namespaces)    |
| Usage of host networking and ports                  | [`hostNetwork`, `hostPorts`](#host-namespaces) |
| Usage of volume types                               | [`volumes`](#volumes-and-file-systems)      |
| Usage of the host filesystem                        | [`allowedHostPaths`](#volumes-and-file-systems) |
| White list of Flexvolume drivers                    | [`allowedFlexVolumes`](#flexvolume-drivers) |
| Allocating an FSGroup that owns the pod's volumes   | [`fsGroup`](#volumes-and-file-systems)      |
| Requiring the use of a read only root file system   | [`readOnlyRootFilesystem`](#volumes-and-file-systems) |
| The user and group IDs of the container             | [`runAsUser`, `runAsGroup`, `supplementalGroups`](#users-and-groups) |
| Restricting escalation to root privileges           | [`allowPrivilegeEscalation`, `defaultAllowPrivilegeEscalation`](#privilege-escalation) |
| Linux capabilities                                  | [`defaultAddCapabilities`, `requiredDropCapabilities`, `allowedCapabilities`](#capabilities) |
| The SELinux context of the container                | [`seLinux`](#selinux)                       |
| The Allowed Proc Mount types for the container      | [`allowedProcMountTypes`](#allowedprocmounttypes) |
| The AppArmor profile used by containers             | [annotations](#apparmor)                    |
| The seccomp profile used by containers              | [annotations](#seccomp)                     |
| The sysctl profile used by containers               | [`forbiddenSysctls`,`allowedUnsafeSysctls`](#sysctl)                      |


## 启用容器组安全策略

Pod security policy control is implemented as an optional (but recommended)
[admission
controller](/docs/reference/access-authn-authz/admission-controllers/#podsecuritypolicy). PodSecurityPolicies
are enforced by [enabling the admission
controller](/docs/reference/access-authn-authz/admission-controllers/#how-do-i-turn-on-an-admission-control-plug-in),
but doing so without authorizing any policies **will prevent any pods from being
created** in the cluster.

Since the pod security policy API (`policy/v1beta1/podsecuritypolicy`) is
enabled independently of the admission controller, for existing clusters it is
recommended that policies are added and authorized before enabling the admission
controller.

- 开启方式

通过准入控制加入`PodSecurityPolicy`
```bash
cat /opt/kubernetes/conf/manifests/kube-apiserver.yaml |grep 'enable-admission'
    - --enable-admission-plugins=NodeRestriction,PodSecurityPolicy
```
注意开启后,会禁止任何容器组的创建
```bash
#开启后创建一个测试容器
kubectl create -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name:      pause
spec:
  containers:
    - name:  pause
      image: 192.168.76.104/k8s.gcr.io/pause:3.1
EOF
Error from server (Forbidden): error when creating "STDIN": pods "pause" is forbidden: no providers available to validate pod request
```


## 创建一个容器组安全策略
以下创建限制最少的策略，相当于不使用pod安全策略[psp-for-admin.yaml]
```yml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp-for-admin
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
spec:
  privileged: true
  allowPrivilegeEscalation: true
  allowedCapabilities:
  - '*'
  volumes:
  - '*'
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  hostIPC: true
  hostPID: true
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```
使用集群管理员账号创建它
```bash
[root@k8s-test-101 PodSecurityPolicy]# kubectl create -f psp-for-admin.yaml
podsecuritypolicy.policy/psp-for-admin created
[root@k8s-test-101 PodSecurityPolicy]# 
[root@k8s-test-101 PodSecurityPolicy]# kubectl get psp
NAME            PRIV   CAPS   SELINUX    RUNASUSER   FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
psp-for-admin   true   *      RunAsAny   RunAsAny    RunAsAny   RunAsAny   false    
```

创建好策略后，再次尝试创建pod
```bash
kubectl create -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name:      pause
spec:
  containers:
    - name:  pause
      image: 192.168.76.104/k8s.gcr.io/pause:3.1
EOF
pod/pause created
```
创建成功。
> 因为使用的是管理员账号，因此不需要授权就能创建了

## 通过RBAC来授权普通用户使用策略
#### 创建一个测试服务账号
设置命名空间和服务帐户来测试。我们将使用此服务帐户模拟非管理员用户。

```bash
kubectl create namespace psp-example
kubectl create serviceaccount -n psp-example fake-user
kubectl create rolebinding -n psp-example fake-editor --clusterrole=edit --serviceaccount=psp-example:fake-user
```
> 这里绑定了一个`clusterrole=edit`的角色,让该账户能拥有对命名空间`psp-exampl`的编辑权限

创建2个别名:

```bash
#管理员账号
alias kubectl-admin='kubectl -n psp-example'
#模拟服务帐户
alias kubectl-user='kubectl --as=system:serviceaccount:psp-example:fake-user -n psp-example'
```
模拟服务账号`psp-example:fake-user`来创建pod
```bash
kubectl-user create -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name:      pause
spec:
  containers:
    - name:  pause
      image: 192.168.76.104/k8s.gcr.io/pause:3.1
EOF
Error from server (Forbidden): error when creating "STDIN": pods "pause" is forbidden: unable to validate against any pod security policy: []
```
虽然该账户有命名空间`psp-example`的编辑权限,但是因为没有匹配到任何容器组安全策略,被禁止创建pod.
使用以下命令验证：
```bash
[root@k8s-test-101 PodSecurityPolicy]# kubectl-user  auth can-i use podsecuritypolicy/psp-for-admin
Warning: resource 'podsecuritypolicies' is not namespace scoped in group 'extensions'
no
```

#### 创建集群角色及进行角色绑定
RBAC是标准的Kubernetes授权模式，可以轻松用于授权使用策略。
首先，创建`Role`或`ClusterRole`并授予`use`对所需策略的访问权限[clusterrole-psp-for-admin.yaml]。授予访问权限的规则如下所示：
```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: psp-for-admin
rules:
- apiGroups: 
  - extensions
  resources: 
  - podsecuritypolicies
  verbs:
  - use
  resourceNames:
  - psp-for-admin
```
这里创建了一个集群角色,为了让服务帐户`psp-example:fake-user`能够使用这个角色,需要创建角色绑定或者集群角色绑定
创建角色绑定的配置如下[rolebinding-psp-for-admin.yaml]：
```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: psp-for-admin
roleRef:
  kind: ClusterRole
  name: psp-for-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fake-user
  namespace: psp-example
```
使用管理员`kubectl-admin` 创建
```bash
[root@k8s-test-101 PodSecurityPolicy]# kubectl-admin  create -f clusterrole-psp-for-admin.yaml 
clusterrole.rbac.authorization.k8s.io/psp-for-admin created
[root@k8s-test-101 PodSecurityPolicy]# kubectl-admin  create -f rolebinding-psp-for-admin.yaml 
rolebinding.rbac.authorization.k8s.io/psp-for-admin created
```
验证账户是否能使用安全策略：
```bash
[root@k8s-test-101 PodSecurityPolicy]# kubectl-user  auth can-i use podsecuritypolicy/psp-for-admin
Warning: resource 'podsecuritypolicies' is not namespace scoped in group 'extensions'
yes
```
验证账户能使用,尝试创建pod
```bash
kubectl-user create -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name:      pause
spec:
  containers:
    - name:  pause
      image: 192.168.76.104/k8s.gcr.io/pause:3.1
EOF
pod/pause created
```
pod创建成功


## 创建严格限制的安全策略
以下是限制性策略的示例，要求用户以非特权用户身份运行，阻止可能的升级到root，并需要使用多种安全机制[psp-for-public.yaml]
```yml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp-for-public
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default,runtime/default'
    #seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
    #apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'runtime/default'
    #apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
spec:
  privileged: false
  # 阻止升级权限到root
  allowPrivilegeEscalation: false
  # 禁止权限升级,配合runAsNonRoot多重限制提权
  requiredDropCapabilities:
    - ALL
  #linux中的某些功能可用于权限升级和容器突破,这里指定需要从容器中删除的功能。
  # 允许使用的卷类型.
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'hostPath'
  hostNetwork: false 
  #是否允许容器可以使用节点的网络命名空间
  hostIPC: false
  #控制pod容器是否可以共享主机IPC命名空间。
  hostPID: false
  #控制pod容器是否可以共享主机进程ID命名空间
  runAsUser:
    # 容器禁止使用root运行
    rule: 'MustRunAsNonRoot'
  seLinux:
    # This policy assumes the nodes are using AppArmor rather than SELinux.
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      #禁止设置为0，即root组
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      # 禁止设置为0，即root组
      - min: 1
        max: 65535
  readOnlyRootFilesystem: false
  #是否将容器根文件系统设置为只读
  allowedHostPaths:
  #允许挂载到容器中的主机目录
  # This allows "/log", "/log/", "/log/bar" etc., but
  # disallows "/logl", "/etc/log" etc.
  # "/log/../" is never valid.
  - pathPrefix: "/log"
    readOnly: false 
  - pathPrefix: "/data"
    readOnly: false
```
创建新的安全策略
```bash
#删除原安全策略
[root@k8s-test-101 PodSecurityPolicy]# kubectl delete psp psp-for-admin
podsecuritypolicy.extensions "psp-for-admin" deleted

[root@k8s-test-101 PodSecurityPolicy]# kubectl create -f psp-for-public.yaml 
podsecuritypolicy.policy/psp-for-public created
[root@k8s-test-101 PodSecurityPolicy]# 
[root@k8s-test-101 PodSecurityPolicy]# kubectl get psp
NAME             PRIV    CAPS   SELINUX    RUNASUSER          FSGROUP     SUPGROUP    READONLYROOTFS   VOLUMES
psp-for-admin    true    *      RunAsAny   RunAsAny           RunAsAny    RunAsAny    false            *
psp-for-public   false          RunAsAny   MustRunAsNonRoot   MustRunAs   MustRunAs   false            configMap,emptyDir,projected,secret,downwardAPI,persistentVolumeClaim
```
使用新的安全策略
```bash
#删除原clusterrole及rolebinding
[root@k8s-test-101 PodSecurityPolicy]# kubectl-admin  delete clusterrole psp-for-admin
warning: deleting cluster-scoped resources, not scoped to the provided namespace
clusterrole.rbac.authorization.k8s.io "psp-for-admin" deleted
[root@k8s-test-101 PodSecurityPolicy]# kubectl-admin  delete rolebinding psp-for-admin
rolebinding.rbac.authorization.k8s.io "psp-for-admin" deleted

#修改clusterrole及rolebinding
[root@k8s-test-101 PodSecurityPolicy]# cat clusterrole-psp-for-public.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: psp-for-public
rules:
- apiGroups: 
  - extensions
  resources: 
  - podsecuritypolicies
  verbs:
  - use
  resourceNames:
  - psp-for-public
[root@k8s-test-101 PodSecurityPolicy]# cat rolebinding-psp-for-public.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: psp-for-public
roleRef:
  kind: ClusterRole
  name: psp-for-public
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fake-user
  namespace: psp-example
  
[root@k8s-test-101 PodSecurityPolicy]# kubectl-admin  create -f clusterrole-psp-for-public.yaml 
clusterrole.rbac.authorization.k8s.io/psp-for-public created
[root@k8s-test-101 PodSecurityPolicy]# kubectl-admin  create -f rolebinding-psp-for-public.yaml 
rolebinding.rbac.authorization.k8s.io/psp-for-public created
```
- 尝试创建一个满足该限制的容器[test-pod.yaml]
```yml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  labels:
    app: test-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: test-container
    image: busybox
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 9999
    command: ["/bin/sh","-c"]
    args:
    - while true;do
        printenv POD_NAME  POD_IP;
        nc -lp 9999;
        sleep 3600s;
      done;
    volumeMounts:
    - name: log-dir
      mountPath: /log
      readOnly: false
  restartPolicy: Never
  volumes:
  - name: log-dir
    hostPath:
      path: /log/
      type: Directory
```
> 主要包括：1.指定容器运行的用户及组；2.只挂载满足限制规则的卷组目录

最终能够成功创建

## 授权政策分析

大多数Kubernetes容器不是由用户直接创建的。而是通常通过各种控制管理器将它们作为Deployment， ReplicaSet或其他模板化控制器的一部分间接创建 。向控制器授予对策略的访问权限将为该控制器创建的所有 Pod 授予访问权限，因此，授权策略的首选方法是向Pod的服务帐户[serviceAccount]授予访问权限

1. 针对容器的serviceAccount进行授权
2. 静态容器是由kubelet创建,因此要对Group: "system:nodes"授权
3. 网络组件及proxy也需要给对应的serviceAccount进行授权

以下给出建议的针对管理员使用的角色绑定配置:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: psp-for-admin
roleRef:
  kind: ClusterRole
  name: psp-for-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
# 授予kubelet创建静态容器权限，如apiserver,controlManager,scheduler等
- kind: Group
  name: system:nodes
  namespace: kube-system
#授予kube-system命名空间所有服务账号创建容器权限
- kind: Group
  name: system:serviceaccounts
  namespace: kube-system
```
