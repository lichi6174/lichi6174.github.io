---
layout: post
title: kubernets集群中部署Traefik
date:  2018-11-09 16:10:00 +0900  
description: kubernetes容器平台相关
img: post-4.jpg # Add image post (optional)
tags: [Blog, kubernetes]
author: Lichi # Add name author (optional)
kubernetes: true
---

# kubernets Traefik 的HTTP 和HTTPS

## Traefik 简介及部署

> 由于微服务架构以及 Docker 技术和 kubernetes 编排工具最近几年才开始逐渐流行，所以一开始的反向代理服务器比如 nginx、apache 并未提供其支持，毕竟他们也不是先知；所以才会出现 Ingress Controller 这种东西来做 kubernetes 和前端负载均衡器如 nginx 之间做衔接；即 Ingress Controller 的存在就是为了能跟 kubernetes 交互，又能写 nginx 配置，还能 reload 它，这是一种折中方案；而最近开始出现的 traefik 天生就是提供了对 kubernetes 的支持，也就是说 traefik 本身就能跟 kubernetes API 交互，感知后端变化，因此可以得知: 在使用 traefik 时，Ingress Controller 已经无卵用了。

## 部署Traefik
> 此部署是基于*https://github.com/gjmzj/kubeasz*{: style="color: red"}项目，使用Ansible脚本安装K8S集群的基础上实现的，朋友们最好参考下这个项目,同时也非常感谢这个项目成员的贡献。

### 创建secret保存HTTPS证书

- 若没有证书请先生成域名对应的证书

```bash
$openssl req -newkey rsa:2048 -nodes -keyout test.com.key  -x509 -days 365 -out test.com.crt
$kubectl create secret generic traefik-cert --from-file=test.com.crt --from-file=test.com.key --namespace=kube-system
```

### 创建configmap保存Traefik配置文件

> 我们这里只做HTTP请求转至HTTPS配置，关于Traefik详细配置请参考官方文档: *https://docs.traefik.io/configuration/commons/*{: style="color: red"}

```bash
$kubectl create configmap traefik-conf --from-file=traefik.toml --namespace=kube-system
```

- traefik.toml

```bash
defaultEntryPoints = ["http", "https"]

[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]
      [[entryPoints.https.tls.certificates]]
      certFile = "/etc/kubernetes/ssl/test.com.crt"
      keyFile = "/etc/kubernetes/ssl/test.com.key"
```

### 通过Deployment部署Traefik

```bash
kubectl create -f traefik-ingress.yaml
```

- traefik-ingress.yaml

```
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: Deployment
apiVersion: apps/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik:v1.6
        imagePullPolicy: IfNotPresent
        name: traefik-ingress-lb
        volumeMounts:
        - mountPath: "/etc/kubernetes/ssl"
          name: "ssl"
        - mountPath: "/config"
          name: "config"
        ports:
        - name: web
          containerPort: 80
          hostPort: 80
        - name: https
          containerPort: 443
          hostPort: 443
        - name: admin
          containerPort: 8080
          hostPort: 8080
        args:
        - --web
        - --kubernetes
        - --configfile=/config/traefik.toml
      volumes:
      - name: ssl
        secret:
          secretName: traefik-cert
      - name: config
        configMap:
          name: traefik-conf
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      # 该端口为 traefik ingress-controller的服务端口
      port: 80
      # 集群hosts文件中设置的 NODE_PORT_RANGE 作为 NodePort的可用范围
      # 从默认20000~40000之间选一个可用端口，让ingress-controller暴露给外部的访问
      nodePort: 23456
      name: web
    - protocol: TCP
      # 该端口为 traefik ingress-controller的服务https端口
      port: 443
      nodePort: 23443
      name: https
    - protocol: TCP
      # 该端口为 traefik 的管理WEB界面
      port: 8080
      name: admin
  type: NodePort
```

### 部署traefik-ui ingress

```bash
kubectl create -f traefik-ui.ing.yaml
```

- traefik-ui.ing.yaml

```
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: traefik-ui.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-ingress-service
          servicePort: 8080
```

### 创建一个测试ingress（以nginx为例）

- test-hello.ing.yaml

```bash
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-hello
  namespace: default
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: hello.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: test-hello
          servicePort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: test-hello
  namespace: default
spec:
  selector:
    app: test-hello
  ports:
  - name: http
    targetPort: 80
    port: 80

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-hello
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test-hello
  template:
    metadata:
      labels:
        app: test-hello
    spec:
      containers:
      - name: test-hello
        image: nginx
        ports:
        - name: http
          containerPort: 80
```

### 创建负载均衡规则后端绑定traefik端口，以及traefik域名解析到负载均衡前端公共IP

> 我们可以利用现有的haproxy来实现，请参看: https://github.com/gjmzj/kubeasz/blob/master/docs/guide/ingress.md。

> 修改源码中的模板文件/etc/ansible/roles/lb/templates/haproxy.cfg.j2，如下：

- haproxy.cfg.j2

> global
>         log /dev/log    local0
>         log /dev/log    local1 notice
>         chroot /var/lib/haproxy
>         stats socket /run/haproxy/admin.sock mode 660 level admin
>         stats timeout 30s
>         user haproxy
>         group haproxy
>         daemon
>         nbproc 1
> 
> defaults
>         log     global
>         timeout connect 5000
>         timeout client  10m
>         timeout server  10m
> 
> listen kube-master
>         bind 0.0.0.0:{{ KUBE_APISERVER.split(':')[2] }}
>         mode tcp
>         option tcplog
>         balance {{ BALANCE_ALG }} 
> {% for host in groups['kube-master'] %}
>         server {{ host }} {{ host }}:6443 check inter 2000 fall 2 rise 2 weight 1
> {% endfor %}
> {% for host in groups['new-master'] %}
>         server {{ host }} {{ host }}:6443 check inter 2000 fall 2 rise 2 weight 1
> {% endfor %}
> {% if INGRESS_NODEPORT_LB == "yes" %}
> 
> listen ingress-node
> 	bind 0.0.0.0:80
> 	mode tcp
>         option tcplog
>         balance {{ BALANCE_ALG }}
> {% for host in groups['kube-node'] %}
>         server {{ host }} {{ host }}:23456 check inter 2000 fall 2 rise 2 weight 1
> {% endfor %}
> {% for host in groups['new-node'] %}
>         server {{ host }} {{ host }}:23456 check inter 2000 fall 2 rise 2 weight 1
> {% endfor %}
> 
> listen ingress-node
> 	bind 0.0.0.0:443
> 	mode tcp
>         option tcplog
>         balance {{ BALANCE_ALG }}
> {% for host in groups['kube-node'] %}
>         server {{ host }} {{ host }}:23443 check inter 2000 fall 2 rise 2 weight 1
> {% endfor %}
> {% for host in groups['new-node'] %}
>         server {{ host }} {{ host }}:23443 check inter 2000 fall 2 rise 2 weight 1
> {% endfor %}
> {% endif %}
> 

*ansible-playbook应用修改后模板配置，并重启haproxy服务*{: style="color: red"}

```bash
$ansible-playbook /etc/ansible/01.prepare.yml -t restart_lb
```

### 验证效果

> 设置域名映射关系（其中ip为haproxy的vip,ansible里面设置好的）

- /etc/ansible/hosts

```bash
......
MASTER_IP="192.168.2.250"
......
```

- 添加主机的hosts映射关系

```bash
192.168.2.250 traefik-ui.test.com
192.168.2.250 hello.test.com
```

- http重定向到https解析逻辑

```bash
http://hello.test.com----redirect---> https://hello.test.com
http://traefik-ui.test.com----redirect---> https://traefik-ui.test.com
```

