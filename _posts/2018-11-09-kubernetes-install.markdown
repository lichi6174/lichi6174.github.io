---
layout: post
title: Kubeadm方式部署kubernetes
date:  2018-11-09 11:22:00 +0900  
description: kubernetes容器平台相关
img: post-7.jpg # Add image post (optional)
tags: [Blog,virtualization,kubernetes]
author: Lichi # Add name author (optional)
virtualization: true
---

# kubeadm部署kubernetes v1.11.2版本步骤说明：

## 第一步
### 准备系统环境

- Repo仓库准备

1. docker-ce.repo
2. kubernetes.repo

- 停用服务 

1. iptables
2. firewalld

- 设置时间同步

1. crontab

```bash
*/30 * * * * /usr/sbin/ntpdate -u ntp1.aliyun.com && hwclock -w --systohc >/dev/null 2>&1
```

- 设置host绑定

1. cat /etc/hosts

```bash
192.168.2.240 k8s-master01 k8s-master01.lichi.com
192.168.2.241 k8s-node01 k8s-node01.lichi.com
192.168.2.242 k8s-node02 k8s-node02.lichi.com
```

- 验证服务器网络情况

1. 内网访问正常
2. 外网访问正常

## 第二步 
### 开始在所有服务器上安装相关软件包   

- master节点

```bash
yum install docker-ce kubelet kubeadm kubectl
```

- node节点

```bash
yum install docker-ce kubelet kubeadm kubectl
```

## 第三步
### 配置所有节点的docker启动服务

- 调整配置，新增两个Environment变量

```bash
#vim /usr/lib/systemd/system/docker.service
Environment="HTTPS_PROXY=http://www.ik8s.io:10080"
Environment="NO_PROXY=192.168.2.0/24,127.0.0.0/8"
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
```

- 启动服务

```bash
$systemctl enable docker
$systemctl daemon-reload
$systemctl start docker
$docker info
```

- 优化内核iptables策略

```bash
$vim /etc/sysctl.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

- 查看kubelet安装后文件信息

```bash
$rpm -ql kubelet
/etc/kubernetes/manifests
/etc/sysconfig/kubelet
/etc/systemd/system/kubelet.service
/usr/bin/kubelet
```

- 设置kubelet开启自启动（但是现在不需要去启动它）

```bash
$systemctl enable kubelet
Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /etc/systemd/system/kubelet.service.

```

## 第四步
### 开始使用kubeadm部署kubernetes master节点

#### kubeadm初始化

- 忽略初始化时的swap报错设置

```bash
$vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
```

- 初始化

```bash
$kubeadm init --kubernetes-version=v1.11.2 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap

```

- 初始化完成后的信息：

```bash
[init] using Kubernetes version: v1.11.2
[preflight] running pre-flight checks
	[WARNING Swap]: running with swap on is not supported. Please disable swap
I0814 22:28:05.745158    3789 kernel_validator.go:81] Validating kernel version
I0814 22:28:05.745257    3789 kernel_validator.go:96] Validating kernel config
	[WARNING SystemVerification]: docker version is greater than the most recently validated version. Docker version: 18.06.0-ce. Max validated version: 17.03
[preflight/images] Pulling images required for setting up a Kubernetes cluster
[preflight/images] This might take a minute or two, depending on the speed of your internet connection
[preflight/images] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[preflight] Activating the kubelet service
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [k8s-master01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.2.240]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Generated etcd/ca certificate and key.
[certificates] Generated etcd/server certificate and key.
[certificates] etcd/server serving cert is signed for DNS names [k8s-master01 localhost] and IPs [127.0.0.1 ::1]
[certificates] Generated etcd/peer certificate and key.
[certificates] etcd/peer serving cert is signed for DNS names [k8s-master01 localhost] and IPs [192.168.2.240 127.0.0.1 ::1]
[certificates] Generated etcd/healthcheck-client certificate and key.
[certificates] Generated apiserver-etcd-client certificate and key.
[certificates] valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[controlplane] wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests" 
[init] this might take a minute or longer if the control plane images have to be pulled
[apiclient] All control plane components are healthy after 39.501584 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.11" in namespace kube-system with the configuration for the kubelets in the cluster
[markmaster] Marking the node k8s-master01 as master by adding the label "node-role.kubernetes.io/master=''"
[markmaster] Marking the node k8s-master01 as master by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "k8s-master01" as an annotation
[bootstraptoken] using token: myrc1h.a30u56580xols80m
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.2.240:6443 --token va1zo5.x017u25cly7ffivc --discovery-token-ca-cert-hash sha256:867118e705a8afa22f5f73d73ff3a95b0f6d555a444f32f1ad92b3ada5b45589

```

- 注意信息中两个附件的意义

```bash
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy
```


- 查看初始化k8s master节点拉取到的容器镜像

```bash
$docker images
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-apiserver-amd64            v1.11.2             821507941e9c        6 days ago          187MB
k8s.gcr.io/kube-controller-manager-amd64   v1.11.2             38521457c799        6 days ago          155MB
k8s.gcr.io/kube-proxy-amd64                v1.11.2             46a3cd725628        6 days ago          97.8MB
k8s.gcr.io/kube-scheduler-amd64            v1.11.2             37a1403e6c1a        6 days ago          56.8MB
k8s.gcr.io/coredns                         1.1.3               b3b94275d97c        2 months ago        45.6MB
k8s.gcr.io/etcd-amd64                      3.2.18              b8df3b177be2        4 months ago        219MB
k8s.gcr.io/pause                           3.1                 da86e6ba6ca1        7 months ago        742kB

```

- 根据初始化信息提示，操作余下步骤

```bash
$mkdir -p $HOME/.kube
$sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- 使用kubectl查看master节点中各个组件的状态

```bash
$kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"} 

$kubectl get node
NAME           STATUS     ROLES     AGE       VERSION
k8s-master01   NotReady   master    23m       v1.11.2
```

## 第五步
### 部署flannel

- [flannel官方GitHub地址](https://github.com/coreos/flannel)

```bash
$kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created
```

- 部署flannel之后，检查其运行状态以及镜像信息

*注意coredns的pod一直处于ContainerCreating状态，是因为发现我们系统禁用了IPV6导致的问题，开启ipv6就解决了，坑呀，找了好久的问题*{: style="color: red"}

```bash
$kubectl get pods -n kube-system
NAME                                   READY     STATUS              RESTARTS   AGE
coredns-78fcdf6894-dj8jf               0/1       Running             0          35m
coredns-78fcdf6894-zn8zx               0/1       Running             0          35m
etcd-k8s-master01                      1/1       Running             0          34m
kube-apiserver-k8s-master01            1/1       Running             0          34m
kube-controller-manager-k8s-master01   1/1       Running             0          34m
kube-flannel-ds-amd64-bl9q6            1/1       Running             0          3m
kube-proxy-h4w8x                       1/1       Running             0          35m
kube-scheduler-k8s-master01            1/1       Running             0          34m

$kubectl get nodes
NAME           STATUS    ROLES     AGE       VERSION
k8s-master01   Ready     master    36m       v1.11.2

$docker images
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-apiserver-amd64            v1.11.2             821507941e9c        6 days ago          187MB
k8s.gcr.io/kube-proxy-amd64                v1.11.2             46a3cd725628        6 days ago          97.8MB
k8s.gcr.io/kube-controller-manager-amd64   v1.11.2             38521457c799        6 days ago          155MB
k8s.gcr.io/kube-scheduler-amd64            v1.11.2             37a1403e6c1a        6 days ago          56.8MB
k8s.gcr.io/coredns                         1.1.3               b3b94275d97c        2 months ago        45.6MB
k8s.gcr.io/etcd-amd64                      3.2.18              b8df3b177be2        4 months ago        219MB
quay.io/coreos/flannel                     v0.10.0-amd64       f0fad859c909        6 months ago        44.6MB
k8s.gcr.io/pause                           3.1                 da86e6ba6ca1        7 months ago        742kB

$kubectl get ns
NAME          STATUS    AGE
default       Active    39m
kube-public   Active    39m
kube-system   Active    39m
```

## 第六步
### 开始使用kubeadm部署kubernetes node节点

- 参考master节点配置，修改相关配置文件

```bash
$vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--fail-swap-on=false"

$vim /usr/lib/systemd/system/docker.service
Environment="HTTPS_PROXY=http://www.ik8s.io:10080"
Environment="NO_PROXY=192.168.2.0/24,127.0.0.0/8"
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
```

- 或者直接scp master节点的配置到node节点

```bash
$scp /etc/sysconfig/kubelet k8s-node01:/etc/sysconfig/
$scp /etc/sysconfig/kubelet k8s-node02:/etc/sysconfig/
$scp /usr/lib/systemd/system/docker.service k8s-node01:/usr/lib/systemd/system/
$scp /usr/lib/systemd/system/docker.service k8s-node02:/usr/lib/systemd/system/
```

- 保证node节点的kubelet，docker自启动

```bash
$systemctl enable kubelet
$systemctl enable docker
```

- 启动node节点的docker服务

```bash
$systemctl start docker

```

- 查看docker状态信息

```bash
$docker info
```

- node节点加入kubernetes集群

```
$kubeadm join 192.168.2.240:6443 --token va1zo5.x017u25cly7ffivc --discovery-token-ca-cert-hash sha256:867118e705a8afa22f5f73d73ff3a95b0f6d555a444f32f1ad92b3ada5b45589 --ignore-preflight-errors=Swap

```

- node节点加入集群信息

```
[preflight] running pre-flight checks
	[WARNING RequiredIPVSKernelModulesAvailable]: the IPVS proxier will not be used, because the following required kernel modules are not loaded: [ip_vs_sh ip_vs ip_vs_rr ip_vs_wrr] or no builtin kernel ipvs support: map[ip_vs_wrr:{} ip_vs_sh:{} nf_conntrack_ipv4:{} ip_vs:{} ip_vs_rr:{}]
you can solve this problem with following methods:
 1. Run 'modprobe -- ' to load missing kernel modules;
 2. Provide the missing builtin kernel ipvs support

	[WARNING Swap]: running with swap on is not supported. Please disable swap
I0814 23:36:40.647760    3714 kernel_validator.go:81] Validating kernel version
I0814 23:36:40.647870    3714 kernel_validator.go:96] Validating kernel config
	[WARNING SystemVerification]: docker version is greater than the most recently validated version. Docker version: 18.06.0-ce. Max validated version: 17.03
[discovery] Trying to connect to API Server "192.168.2.240:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.2.240:6443"
[discovery] Requesting info from "https://192.168.2.240:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.2.240:6443"
[discovery] Successfully established connection with API Server "192.168.2.240:6443"
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.11" ConfigMap in the kube-system namespace
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[preflight] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "k8s-node02" as an annotation

This node has joined the cluster:
* Certificate signing request was sent to master and a response
  was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

- 检查node节点运行状态以及镜像拉取情况

```
$kubectl get node
sNAME           STATUS    ROLES     AGE       VERSION
k8s-master01   Ready     master    1h        v1.11.2
k8s-node01     Ready     <none>    3m        v1.11.2
k8s-node02     Ready     <none>    2m        v1.11.2

$kubectl get pod -n kube-system -o wide
NAME                                   READY     STATUS              RESTARTS   AGE       IP              NODE           NOMINATED NODE
coredns-78fcdf6894-dj8jf               0/1       Running             0          1h        <none>          k8s-master01   <none>
coredns-78fcdf6894-zn8zx               0/1       Running             0          1h        <none>          k8s-master01   <none>
etcd-k8s-master01                      1/1       Running             0          1h        192.168.2.240   k8s-master01   <none>
kube-apiserver-k8s-master01            1/1       Running             0          1h        192.168.2.240   k8s-master01   <none>
kube-controller-manager-k8s-master01   1/1       Running             0          1h        192.168.2.240   k8s-master01   <none>
kube-flannel-ds-amd64-bl9q6            1/1       Running             0          39m       192.168.2.240   k8s-master01   <none>
kube-flannel-ds-amd64-pjz88            1/1       Running             0          5m        192.168.2.242   k8s-node02     <none>
kube-flannel-ds-amd64-zfrbz            1/1       Running             0          6m        192.168.2.241   k8s-node01     <none>
kube-proxy-9tbdl                       1/1       Running             0          6m        192.168.2.241   k8s-node01     <none>
kube-proxy-h4w8x                       1/1       Running             0          1h        192.168.2.240   k8s-master01   <none>
kube-proxy-qkhrb                       1/1       Running             0          5m        192.168.2.242   k8s-node02     <none>
kube-scheduler-k8s-master01            1/1       Running             0          1h        192.168.2.240   k8s-master01   <none>

$docker images
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy-amd64   v1.11.2             46a3cd725628        6 days ago          97.8MB
quay.io/coreos/flannel        v0.10.0-amd64       f0fad859c909        6 months ago        44.6MB
k8s.gcr.io/pause              3.1                 da86e6ba6ca1        7 months ago        742kB

```

## 第七步
### 验证kubeadm搭建的kubernetes集群

- kubectl命令自动补全功能开启

```bash
echo "source <(kubectl completion zsh)" >> ~/.zshrc
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

- 常用命令集合

```bash
$kubectl run nginx-deploy --image=nginx:1.14-alpine --port=80 --replicas=1
$kubectl get deployments
$kubectl get pod -o wide
$kubectl delete pod nginx-deploy-5b595999-m4wd6
$kubectl expose deployment nginx-deploy --name=nginx --port=80 --target-port=80 --protocol=TCP
$kubectl delete svc nginx-deploy 
$kubectl run client --image=busybox -it --restart=Never
$kubectl edit svc nginx 
$kubectl get svc --show-labels
$kubectl get deployments --show-labels
$kubectl describe deployments nginx-deploy 
$kubectl run myapp --image=ikubernetes/myapp:v1 --replicas=2
$kubectl scale deployment myapp --replicas=5
$kubectl scale deployment myapp --replicas=3
$kubectl set image deployment myapp myapp=ikubernetes/myapp:v2
$kubectl rollout status deployment myapp
$kubectl rollout history deployment myapp
$kubectl rollout undo deployment myapp
$kubectl patch deployments myapp-deploy -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
$kubectl set image deployment myapp-deploy myapp=ikubernetes/myapp:v2 && kubectl rollout pause deployment myapp-deploy
$kubectl rollout resume deployment myapp-deploy
$kubectl rollout status deployment myapp-deploy
$kubectl rollout history deployment myapp-deploy
$kubectl rollout undo deployment myapp-deploy --to-revision=7
$kubect get rs -o wide
$kubectl expose deployment redis  --port=6379
$dig -t A nginx.default.svc.cluster.local @10.96.0.10
```
