---
layout: post
title: 部署glusterfs-kubernetes和heketi
date:  2018-11-09 15:15:00 +0900  
description: kubernetes容器平台相关
img: post-5.jpg # Add image post (optional)
tags: [Blog, kubernetes]
author: Lichi # Add name author (optional)
kubernetes: true
---

# 安装glusterfs-kubernetes和heketi

## 前言

## Heketi
- Heketi提供了一个RESTful管理界面，可以用来管理GlusterFS卷的生命周期。 通过Heketi，就可以像使用OpenStack Manila，Kubernetes和OpenShift一样申请可以动态配置GlusterFS卷。Heketi会动态在集群内选择bricks构建所需的volumes，这样以确保数据的副本会分散到集群不同的故障域内。同时Heketi还支持任意数量的ClusterFS集群，以保证接入的云服务器不局限于单个GlusterFS集群。

## Gluster-Kubernetes
- Gluster-Kubernetes是一个可以将GluserFS和Hekiti轻松部署到Kubernetes集群的开源项目。另外也提供在Kubernetes中可以采用StorageClass来动态管理GlusterFS卷。


## 部署环境

## 服务器分配信息

Hostname | 服务器IP | 存储IP | 硬盘信息 | 容量G
---|---|---|---|---
k8s-master | 192.168.2.240 | 192.168.2.240 | /sdb | 500G
k8s-node01 | 192.168.2.241 | 192.168.2.241 | /sdb | 500G
k8s-node02 | 192.168.2.242 | 192.168.2.242 | /sdb | 500G


## 部署步骤
### 安装glusterfs依赖组件

```bash
$yum install -y glusterfs glusterfs-server glusterfs-fuse glusterfs-rdma glusterfs-geo-replication glusterfs-devel
```

### 加载内核模块

```bash
modprobe dm_snapshot
modprobe dm_thin_pool
modprobe dm_mirror
```

### 确认模块已加载

```bash
$lsmod |egrep "dm_snapshot|dm_mirror|dm_thin_pool"
dm_thin_pool           66298  0 
dm_persistent_data     75269  1 dm_thin_pool
dm_bio_prison          18209  1 dm_thin_pool
dm_snapshot            39104  0 
dm_bufio               28014  2 dm_persistent_data,dm_snapshot
dm_mirror              22289  0 
dm_region_hash         20813  1 dm_mirror
dm_log                 18411  2 dm_region_hash,dm_mirror
dm_mod                123941  14 dm_log,dm_mirror,dm_bufio,dm_thin_pool,dm_snapshot
```

4. 获取gluster-kubernetes源码文件

```bash
$cd  ~/mainfests/
$git clone https://github.com/gluster/gluster-kubernetes.git

```

5. 用于3个glusterfs节点的存储的磁盘空间分区和格式化

*官网说每个用于glusterfs存储空间的磁盘节点都需要分区和格式化实际上，经过反复测试，实际上不需要这步操作，可以忽略这一步*{: style="color: red"}

```bash
$mkfs.xfs /dev/sdb
```

*磁盘添加报错，需要如下操作（这一步在重新部署的时候用得上）*{: style="color: red"}

```
$dd if=/dev/zero of=/dev/sdb bs=1k count=1
blockdev --rereadpt /dev/sdb
```

6. 准备配置文件

```bash
$cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.2.240 k8s-master01 k8s-master01.lichi.com
192.168.2.241 k8s-node01 k8s-node01.lichi.com
192.168.2.242 k8s-node02 k8s-node02.lichi.com
192.168.2.192 k8s-store01 k8s-store01.lichi.com

$cd  ~/mainfests/gluster-kubernetes/deploy

$cp topology.json.sample topology.json
```

7. 修改topology.json文件中的存储节点IP地址和设备名称，如下示:

*注意hostnames/manage下对应的节点名称，应该与kubect get nodes中的NAME一致*{: style="color: red"}

- topology.json

```json
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "k8s-master01"
              ],
              "storage": [
                "192.168.2.240"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "k8s-node01"
              ],
              "storage": [
                "192.168.2.241"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "k8s-node02"
              ],
              "storage": [
                "192.168.2.242"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        }
      ]
    }
  ]
}
```

8. 如果要在master节点部署glusterfs存储节点的话需要修改相关的yaml文件，允许容忍污点,修改后如下:

```bash
$cat glusterfs-daemonset.yaml
---
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: glusterfs
  labels:
    glusterfs: daemonset
  annotations:
    description: GlusterFS DaemonSet
    tags: glusterfs
spec:
  template:
    metadata:
      name: glusterfs
      labels:
        glusterfs: pod
        glusterfs-node: pod
    spec:
      nodeSelector:
        storagenode: glusterfs
      hostNetwork: true
      tolerations:
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
      containers:
      - image: gluster/gluster-centos:latest
        imagePullPolicy: IfNotPresent
        name: glusterfs
        env:
        - name: GB_GLFS_LRU_COUNT
          value: "15"
        - name: TCMU_LOGDIR
          value: "/var/log/glusterfs/gluster-block"
        resources:
          requests:
            memory: 100Mi
            cpu: 100m
        volumeMounts:
        - name: glusterfs-heketi
          mountPath: "/var/lib/heketi"
        - name: glusterfs-run
          mountPath: "/run"
        - name: glusterfs-lvm
          mountPath: "/run/lvm"
        - name: glusterfs-etc
          mountPath: "/etc/glusterfs"
        - name: glusterfs-logs
          mountPath: "/var/log/glusterfs"
        - name: glusterfs-config
          mountPath: "/var/lib/glusterd"
        - name: glusterfs-dev
          mountPath: "/dev"
        - name: glusterfs-misc
          mountPath: "/var/lib/misc/glusterfsd"
        - name: glusterfs-cgroup
          mountPath: "/sys/fs/cgroup"
          readOnly: true
        - name: glusterfs-ssl
          mountPath: "/etc/ssl"
          readOnly: true
        - name: kernel-modules
          mountPath: "/usr/lib/modules"
          readOnly: true
        securityContext:
          capabilities: {}
          privileged: true
        readinessProbe:
          timeoutSeconds: 3
          initialDelaySeconds: 40
          exec:
            command:
            - "/bin/bash"
            - "-c"
            - systemctl status glusterd.service
          periodSeconds: 25
          successThreshold: 1
          failureThreshold: 50
        livenessProbe:
          timeoutSeconds: 3
          initialDelaySeconds: 40
          exec:
            command:
            - "/bin/bash"
            - "-c"
            - systemctl status glusterd.service
          periodSeconds: 25
          successThreshold: 1
          failureThreshold: 50
      volumes:
      - name: glusterfs-heketi
        hostPath:
          path: "/var/lib/heketi"
      - name: glusterfs-run
      - name: glusterfs-lvm
        hostPath:
          path: "/run/lvm"
      - name: glusterfs-etc
        hostPath:
          path: "/etc/glusterfs"
      - name: glusterfs-logs
        hostPath:
          path: "/var/log/glusterfs"
      - name: glusterfs-config
        hostPath:
          path: "/var/lib/glusterd"
      - name: glusterfs-dev
        hostPath:
          path: "/dev"
      - name: glusterfs-misc
        hostPath:
          path: "/var/lib/misc/glusterfsd"
      - name: glusterfs-cgroup
        hostPath:
          path: "/sys/fs/cgroup"
      - name: glusterfs-ssl
        hostPath:
          path: "/etc/ssl"
      - name: kernel-modules
        hostPath:
          path: "/usr/lib/modules"
```

9. 如果想暴露heketi的服务到节点的话记得启用type: NodePort，截取部分内如如下

```bash
$cat heketi-deployment.yaml
---
kind: Service
apiVersion: v1
metadata:
  name: heketi
  labels:
    glusterfs: heketi-service
    heketi: service
  annotations:
    description: Exposes Heketi Service
spec:
  selector:
    glusterfs: heketi-pod
  ports:
  - name: heketi
    port: 8080
    targetPort: 8080
  type: NodePort
......

```

10. 执行部署命令

```bash
$./gk-deploy -g -n default -c kubectl -v
Welcome to the deployment tool for GlusterFS on Kubernetes and OpenShift.

Before getting started, this script has some requirements of the execution
environment and of the container platform that you should verify.

The client machine that will run this script must have:
 * Administrative access to an existing Kubernetes or OpenShift cluster
 * Access to a python interpreter 'python'

Each of the nodes that will host GlusterFS must also have appropriate firewall
rules for the required GlusterFS ports:
 * 2222  - sshd (if running GlusterFS in a pod)
 * 24007 - GlusterFS Management
 * 24008 - GlusterFS RDMA
 * 49152 to 49251 - Each brick for every volume on the host requires its own
   port. For every new brick, one new port will be used starting at 49152. We
   recommend a default range of 49152-49251 on each host, though you can adjust
   this to fit your needs.

The following kernel modules must be loaded:
 * dm_snapshot
 * dm_mirror
 * dm_thin_pool

For systems with SELinux, the following settings need to be considered:
 * virt_sandbox_use_fusefs should be enabled on each node to allow writing to
   remote GlusterFS volumes

In addition, for an OpenShift deployment you must:
 * Have 'cluster_admin' role on the administrative account doing the deployment
 * Add the 'default' and 'router' Service Accounts to the 'privileged' SCC
 * Have a router deployed that is configured to allow apps to access services
   running in the cluster

Do you wish to proceed with deployment?

[Y]es, [N]o? [Default: Y]: y
Using Kubernetes CLI.

Checking status of namespace matching 'default':
default   Active    17d
Using namespace "default".
Checking for pre-existing resources...
  GlusterFS pods ... 
Checking status of pods matching '--selector=glusterfs=pod':

Timed out waiting for pods matching '--selector=glusterfs=pod'.
not found.
  deploy-heketi pod ... 
Checking status of pods matching '--selector=deploy-heketi=pod':

Timed out waiting for pods matching '--selector=deploy-heketi=pod'.
not found.
  heketi pod ... 
Checking status of pods matching '--selector=heketi=pod':

Timed out waiting for pods matching '--selector=heketi=pod'.
not found.
  gluster-s3 pod ... 
Checking status of pods matching '--selector=glusterfs=s3-pod':

Timed out waiting for pods matching '--selector=glusterfs=s3-pod'.
not found.
Creating initial resources ... kubectl -n default create -f /root/mainfests/gluster-kubernetes/deploy/kube-templates/heketi-service-account.yaml 2>&1
serviceaccount/heketi-service-account created
kubectl -n default create clusterrolebinding heketi-sa-view --clusterrole=edit --serviceaccount=default:heketi-service-account 2>&1
clusterrolebinding.rbac.authorization.k8s.io/heketi-sa-view created
kubectl -n default label --overwrite clusterrolebinding heketi-sa-view glusterfs=heketi-sa-view heketi=sa-view
clusterrolebinding.rbac.authorization.k8s.io/heketi-sa-view labeled
OK
Marking 'k8s-master01' as a GlusterFS node.
kubectl -n default label nodes k8s-master01 storagenode=glusterfs --overwrite 2>&1
node/k8s-master01 labeled
Marking 'k8s-node01' as a GlusterFS node.
kubectl -n default label nodes k8s-node01 storagenode=glusterfs --overwrite 2>&1
node/k8s-node01 labeled
Marking 'k8s-node02' as a GlusterFS node.
kubectl -n default label nodes k8s-node02 storagenode=glusterfs --overwrite 2>&1
node/k8s-node02 labeled
Deploying GlusterFS pods.
sed -e 's/storagenode\: glusterfs/storagenode\: 'glusterfs'/g' /root/mainfests/gluster-kubernetes/deploy/kube-templates/glusterfs-daemonset.yaml | kubectl -n default create -f - 2>&1
daemonset.extensions/glusterfs created
Waiting for GlusterFS pods to start ... 
Checking status of pods matching '--selector=glusterfs=pod':
glusterfs-4bh8q   1/1       Running   0         1m
glusterfs-4p9mz   1/1       Running   0         1m
glusterfs-wwxhd   1/1       Running   0         1m
OK
kubectl -n default create secret generic heketi-config-secret --from-file=private_key=/dev/null --from-file=./heketi.json --from-file=topology.json=topology.json
secret/heketi-config-secret created
kubectl -n default label --overwrite secret heketi-config-secret glusterfs=heketi-config-secret heketi=config-secret
secret/heketi-config-secret labeled
sed -e 's/\${HEKETI_EXECUTOR}/kubernetes/' -e 's#\${HEKETI_FSTAB}#/var/lib/heketi/fstab#' -e 's/\${HEKETI_ADMIN_KEY}//' -e 's/\${HEKETI_USER_KEY}//' /root/mainfests/gluster-kubernetes/deploy/kube-templates/deploy-heketi-deployment.yaml | kubectl -n default create -f - 2>&1
service/deploy-heketi created
deployment.extensions/deploy-heketi created
Waiting for deploy-heketi pod to start ... 
Checking status of pods matching '--selector=deploy-heketi=pod':
deploy-heketi-559446b649-22hrh   1/1       Running   0         11s
OK
Determining heketi service URL ... OK
kubectl -n default exec -i deploy-heketi-559446b649-22hrh -- heketi-cli -s http://localhost:8080 --user admin --secret '' topology load --json=/etc/heketi/topology.json 2>&1
Creating cluster ... ID: 338132d846622007777716cece92d64c
Allowing file volumes on cluster.
Allowing block volumes on cluster.
Creating node k8s-master01 ... ID: ff05103befdafe43ab89e6043002d8dd
Adding device /dev/sdb ... OK
Creating node k8s-node01 ... ID: f6f67cf79c9cdd48210e62d5b16887a8
Adding device /dev/sdb ... OK
Creating node k8s-node02 ... ID: 87481f1c57e7db8ce668fe92fce30bc4
Adding device /dev/sdb ... OK
heketi topology loaded.
kubectl -n default exec -i deploy-heketi-559446b649-22hrh -- heketi-cli -s http://localhost:8080 --user admin --secret '' setup-openshift-heketi-storage --listfile=/tmp/heketi-storage.json  2>&1
Saving /tmp/heketi-storage.json
kubectl -n default exec -i deploy-heketi-559446b649-22hrh -- cat /tmp/heketi-storage.json | kubectl -n default create -f - 2>&1
secret/heketi-storage-secret created
endpoints/heketi-storage-endpoints created
service/heketi-storage-endpoints created
job.batch/heketi-storage-copy-job created

Checking status of pods matching '--selector=job-name=heketi-storage-copy-job':
heketi-storage-copy-job-jf7q5   0/1       Completed   0         2s
kubectl -n default label --overwrite svc heketi-storage-endpoints glusterfs=heketi-storage-endpoints heketi=storage-endpoints
service/heketi-storage-endpoints labeled
kubectl -n default delete all,service,jobs,deployment,secret --selector="deploy-heketi" 2>&1
pod "deploy-heketi-559446b649-22hrh" deleted
service "deploy-heketi" deleted
deployment.apps "deploy-heketi" deleted
job.batch "heketi-storage-copy-job" deleted
secret "heketi-storage-secret" deleted
sed -e 's/\${HEKETI_EXECUTOR}/kubernetes/' -e 's#\${HEKETI_FSTAB}#/var/lib/heketi/fstab#' -e 's/\${HEKETI_ADMIN_KEY}//' -e 's/\${HEKETI_USER_KEY}//' /root/mainfests/gluster-kubernetes/deploy/kube-templates/heketi-deployment.yaml | kubectl -n default create -f - 2>&1
service/heketi created
deployment.extensions/heketi created
Waiting for heketi pod to start ... 
Checking status of pods matching '--selector=heketi=pod':
heketi-86f98754c-xb55c   1/1       Running   0         13s
OK
Determining heketi service URL ... Flag --show-all has been deprecated, will be removed in an upcoming release
OK

heketi is now running and accessible via http://10.244.2.83:8080 . To run
administrative commands you can install 'heketi-cli' and use it as follows:

  # heketi-cli -s http://10.244.2.83:8080 --user admin --secret '<ADMIN_KEY>' cluster list

You can find it at https://github.com/heketi/heketi/releases . Alternatively,
use it from within the heketi pod:

  # kubectl -n default exec -i heketi-86f98754c-xb55c -- heketi-cli -s http://localhost:8080 --user admin --secret '<ADMIN_KEY>' cluster list

For dynamic provisioning, create a StorageClass similar to this:

---
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: glusterfs-storage
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://10.244.2.83:8080"


Deployment complete!
```

11. 查看节点信息

- *方法一：在heketi的pod容器内查看信息*{: style="color: red"}

```bash
$kubectl exec -i heketi-86f98754c-xb55c -- heketi-cli -s http://localhost:8080 --user admin --secret '' topology info

Cluster Id: 338132d846622007777716cece92d64c

    File:  true
    Block: true

    Volumes:

	Name: heketidbstorage
	Size: 2
	Id: ec749b80bc957c03d4c67037190044c3
	Cluster Id: 338132d846622007777716cece92d64c
	Mount: 192.168.2.242:heketidbstorage
	Mount Options: backup-volfile-servers=192.168.2.241,192.168.2.240
	Durability Type: replicate
	Replica: 3
	Snapshot: Disabled

		Bricks:
			Id: 55a0519a822d0a25f2a8446731c3f63d
			Path: /var/lib/heketi/mounts/vg_f3a6e7962e5801144d94ed8e32d4c322/brick_55a0519a822d0a25f2a8446731c3f63d/brick
			Size (GiB): 2
			Node: 87481f1c57e7db8ce668fe92fce30bc4
			Device: f3a6e7962e5801144d94ed8e32d4c322

			Id: ab2abbdc20cd1d014e974646edef09d6
			Path: /var/lib/heketi/mounts/vg_5337eece8164d7d3fbc67672ef18bbed/brick_ab2abbdc20cd1d014e974646edef09d6/brick
			Size (GiB): 2
			Node: f6f67cf79c9cdd48210e62d5b16887a8
			Device: 5337eece8164d7d3fbc67672ef18bbed

			Id: b8ac1182bc114be945804dc4a5088697
			Path: /var/lib/heketi/mounts/vg_6cacfa3ce1a7f0099574ce5263c2e473/brick_b8ac1182bc114be945804dc4a5088697/brick
			Size (GiB): 2
			Node: ff05103befdafe43ab89e6043002d8dd
			Device: 6cacfa3ce1a7f0099574ce5263c2e473


    Nodes:

	Node Id: 87481f1c57e7db8ce668fe92fce30bc4
	State: online
	Cluster Id: 338132d846622007777716cece92d64c
	Zone: 1
	Management Hostnames: k8s-node02
	Storage Hostnames: 192.168.2.242
	Devices:
		Id:f3a6e7962e5801144d94ed8e32d4c322   Name:/dev/sdb            State:online    Size (GiB):499     Used (GiB):2       Free (GiB):497     
			Bricks:
				Id:55a0519a822d0a25f2a8446731c3f63d   Size (GiB):2       Path: /var/lib/heketi/mounts/vg_f3a6e7962e5801144d94ed8e32d4c322/brick_55a0519a822d0a25f2a8446731c3f63d/brick

	Node Id: f6f67cf79c9cdd48210e62d5b16887a8
	State: online
	Cluster Id: 338132d846622007777716cece92d64c
	Zone: 1
	Management Hostnames: k8s-node01
	Storage Hostnames: 192.168.2.241
	Devices:
		Id:5337eece8164d7d3fbc67672ef18bbed   Name:/dev/sdb            State:online    Size (GiB):499     Used (GiB):2       Free (GiB):497     
			Bricks:
				Id:ab2abbdc20cd1d014e974646edef09d6   Size (GiB):2       Path: /var/lib/heketi/mounts/vg_5337eece8164d7d3fbc67672ef18bbed/brick_ab2abbdc20cd1d014e974646edef09d6/brick

	Node Id: ff05103befdafe43ab89e6043002d8dd
	State: online
	Cluster Id: 338132d846622007777716cece92d64c
	Zone: 1
	Management Hostnames: k8s-master01
	Storage Hostnames: 192.168.2.240
	Devices:
		Id:6cacfa3ce1a7f0099574ce5263c2e473   Name:/dev/sdb            State:online    Size (GiB):499     Used (GiB):2       Free (GiB):497     
			Bricks:
				Id:b8ac1182bc114be945804dc4a5088697   Size (GiB):2       Path: /var/lib/heketi/mounts/vg_6cacfa3ce1a7f0099574ce5263c2e473/brick_b8ac1182bc114be945804dc4a5088697/brick


$kubectl exec -i heketi-86f98754c-xb55c -- heketi-cli -s http://localhost:8080 --user admin --secret '' cluster list
Clusters:
Id:338132d846622007777716cece92d64c [file][block]
```

- *方法二：在本地安装heketi-cli工具，它来查看信息*{: style="color: red"}

```
获取工具路径：https://github.com/heketi/heketi/releases
$heketi-cli -s http://192.168.2.242:32020 --user admin --secret '' cluster list
Clusters:
Id:cf056a60eef3a878f72b4a9da7682dc3 [file][block]
```

12. 查看相关资源情况

```bash
$kubectl get all -n default|grep -E "glusterfs|heketi|NAME"

NAME                              READY     STATUS    RESTARTS   AGE
pod/glusterfs-6kcps               1/1       Running   0          32m
pod/glusterfs-vbktr               1/1       Running   0          32m
pod/glusterfs-x59gn               1/1       Running   0          32m
pod/heketi-86f98754c-xv4xs        1/1       Running   0          30m
NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/heketi                     NodePort    10.109.180.162   <none>        8080:32020/TCP   30m
service/heketi-storage-endpoints   ClusterIP   10.111.4.148     <none>        1/TCP            30m

NAME                       DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR           AGE

daemonset.apps/glusterfs   3         3         3         3            3           storagenode=glusterfs   32m
NAME                           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/heketi         1         1         1            1           30m

NAME                                    DESIRED   CURRENT   READY     AGE
replicaset.apps/heketi-86f98754c        1         1         1         30m
```


## 重新部署步骤

1. 删除pod,svc等；

```bash
$./gk-deploy -g --abort
```

2. 再将节点的目录/var/lib/glusterd清空

```bash
rm -rf /var/lib/glusterd
```

3. 删除磁盘的vg和pv

```bash
$dmsetup ls
vg_3117b9de40d6d6e4fe4a6e591a9cada6-tp_6df170a861d466829fc248937721b75c-tpool	(253:5)
vg_3117b9de40d6d6e4fe4a6e591a9cada6-tp_6df170a861d466829fc248937721b75c_tdata	(253:4)
vg_3117b9de40d6d6e4fe4a6e591a9cada6-tp_6df170a861d466829fc248937721b75c_tmeta	(253:3)
vg_3117b9de40d6d6e4fe4a6e591a9cada6-brick_6df170a861d466829fc248937721b75c	(253:7)
vg_3117b9de40d6d6e4fe4a6e591a9cada6-tp_6df170a861d466829fc248937721b75c	(253:6)
centos-home	(253:2)
centos-swap	(253:1)
centos-root	(253:0)

$dmsetup remove vg_3117b9de40d6d6e4fe4a6e591a9cada6-brick_6df170a861d466829fc248937721b75c

$dmsetup remove vg_3117b9de40d6d6e4fe4a6e591a9cada6-tp_6df170a861d466829fc248937721b75c

$dmsetup remove vg_3117b9de40d6d6e4fe4a6e591a9cada6-tp_6df170a861d466829fc248937721b75c-tpool

$dmsetup remove vg_3117b9de40d6d6e4fe4a6e591a9cada6-tp_6df170a861d466829fc248937721b75c_tdata

$dmsetup remove vg_3117b9de40d6d6e4fe4a6e591a9cada6-tp_6df170a861d466829fc248937721b75c_tmeta
```

4. 重新执行

```bash
$./gk-deploy -g -n default -c kubectl -v
```


## 验证glusterfs和heketi

1. 常用的方法是设置一个StorageClass，让Kubernetes为提交的PersistentVolumeClaim自动配置存储

> 回收策略
由 storage class 动态创建的 Persistent Volume 会在的 reclaimPolicy 字段中指定回收策略，可以是 Delete 或者 Retain。如果 StorageClass 对象被创建时没有指定 reclaimPolicy ，它将默认为 Delete。
通过 storage class 手动创建并管理的 Persistent Volume 会使用它们被创建时指定的回收政策。 

- storage-class-heketi.yaml

```bash
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: glusterfs-storage
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://10.244.2.92:8080"
```

- 创建storageclass

```bash
#kubectl create -f storage-class-heketi.yaml
storageclass.storage.k8s.io/glusterfs-storage created
```


2. 创建pvc

- heketi-pvc.yaml

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: glusterfs-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: glusterfs-storage  #storage-class的名字
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

- 创建pvc

```bash
$kubectl create -f heketi-pvc.yaml
persistentvolumeclaim/glusterfs-claim created
```

3. 查看storageclasses,pvc,pv

```bash
$kubectl get storageclasses,pvc,pv
NAME                                            PROVISIONER               AGE
storageclass.storage.k8s.io/glusterfs-storage   kubernetes.io/glusterfs   15m

NAME                                      STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
persistentvolumeclaim/glusterfs-claim     Bound     pvc-11c9fb90-af1a-11e8-ac01-0050569c01a4   5Gi        RWX            glusterfs-storage   11m
persistentvolumeclaim/myappdata-myapp-0   Bound     pv002                                      5Gi        RWO                                11d
persistentvolumeclaim/myappdata-myapp-1   Bound     pv001                                      5Gi        RWO,RWX                            11d
persistentvolumeclaim/myappdata-myapp-2   Bound     pv003                                      5Gi        RWO,RWX                            11d
persistentvolumeclaim/myappdata-myapp-3   Bound     pv005                                      10Gi       RWO,RWX                            11d
persistentvolumeclaim/myappdata-myapp-4   Bound     pv004                                      20Gi       RWO,RWX                            11d

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                       STORAGECLASS        REASON    AGE
persistentvolume/pv001                                      5Gi        RWO,RWX        Retain           Bound     default/myappdata-myapp-1                                 11d
persistentvolume/pv002                                      5Gi        RWO            Retain           Bound     default/myappdata-myapp-0                                 11d
persistentvolume/pv003                                      5Gi        RWO,RWX        Retain           Bound     default/myappdata-myapp-2                                 11d
persistentvolume/pv004                                      20Gi       RWO,RWX        Retain           Bound     default/myappdata-myapp-4                                 11d
persistentvolume/pv005                                      10Gi       RWO,RWX        Retain           Bound     default/myappdata-myapp-3                                 11d
persistentvolume/pvc-11c9fb90-af1a-11e8-ac01-0050569c01a4   5Gi        RWX            Delete           Bound     default/glusterfs-claim     glusterfs-storage             11m
```

4. 通过nginx验证存储使用

- 创建nginx容器 heketi-test.yaml

```bash
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod1
  labels:
    name: nginx-pod1
spec:
  containers:
  - name: nginx-pod1
    image: nginx
    ports:
    - name: web
      containerPort: 80
    volumeMounts:
    - name: gluster-vol1
      mountPath: /usr/share/nginx/html
  volumes:
    - name: gluster-vol1
      persistentVolumeClaim:
        claimName: glusterfs-claim
```

- 创建此pod

```bash
$kubectl create -f heketi-test.yaml
pod/nginx-pod1 created
```

- 进入pod，手动生成测试页面文件

```bash
$kubectl exec -ti nginx-pod1 /bin/sh
$cd /usr/share/nginx/html
$echo 'Hello World from GlusterFS!!!' > index.html
$ls
index.html
$exit
```

- 验证访问存储资源和nginx的pod容器存储资源挂载情况

- [x] 验证方法一：

```bash
$kubectl port-forward nginx-pod1 1888:80
Forwarding from 127.0.0.1:1888 -> 80

$curl http://localhost:1888
Hello World from GlusterFS!!!
```

- [x] 验证方法二：

```bash
$kubectl get pods -o wide|grep nginx-pod1
nginx-pod1               1/1       Running   0          20m       10.244.2.104    k8s-node02     <none>
$kubectl run cirror-$RANDOM --rm -it --image=cirros -- /bin/sh
curl http://10.244.2.104
Hello World from GlusterFS!!!
```

- nginx pod挂载存储信息

```bash
$kubectl exec nginx-pod1 -- mount | grep gluster
192.168.2.240:vol_d4a4d18e90cfb07977ce2868c1900f4f on /usr/share/nginx/html type fuse.glusterfs (rw,relatime,user_id=0,group_id=0,default_permissions,allow_other,max_read=131072)
```
