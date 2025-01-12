## 1、权限控制

**Context**

为部署流水线创建一个新的 `ClusterRole` 并将其绑定到范围为特定的 namespace 的特定 `ServiceAccount`。

**Task**

创建一个名为 `deployment-clusterrole` 且仅允许创建以下资源类型的新 `ClusterRole`：

- `Deployment`
- `StatefulSet`
- `DaemonSet`

在现有的 namespace `app-team1` 中创建一个名为 `cicd-token` 的新 `ServiceAccount`。

限于 namespace `app-team1` 中，将新的 ClusterRole `deployment-clusterrole` 绑定到新的 ServiceAccount `cicd-token`。

涉及的对象如下图所示：

![image-20241229173340603](https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20241229173340603.png)

1. 创建ClusterRole

```bash
# 创建语句
candidate@node01:~$ kubectl create clusterrole deployment-clusterrole --verb create --resource deployment,statefulset,daemonset 
clusterrole.rbac.authorization.k8s.io/deployment-clusterrole created
candidate@node01:~$ kubectl get clusterrole deployment-clusterrole 
NAME                     CREATED AT
deployment-clusterrole   2024-12-29T09:29:51Z
# 检查clusterrole
candidate@node01:~$ kubectl describe clusterrole deployment-clusterrole 
Name:         deployment-clusterrole
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources          Non-Resource URLs  Resource Names  Verbs
  ---------          -----------------  --------------  -----
  daemonsets.apps    []                 []              [create]
  deployments.apps   []                 []              [create]
  statefulsets.apps  []                 []              [create]
```

2. 创建ServiceAccount

```bash
candidate@node01:~$ kubectl create serviceaccount -n app-team1 cicd-token
serviceaccount/cicd-token created
candidate@node01:~$ kubectl get serviceaccounts -n app-team1 cicd-token 
NAME         SECRETS   AGE
cicd-token   0         18s
```

3. 创建ClusterRoleBinding

```bash
# 创建
# kubectl -n app-team1 create rolebinding <rolebinding-name> --clusterrole <clusterrole-name> --serviceaccount <namespace>:<serviceaccount-name>
candidate@node01:~$ kubectl -n app-team1 create rolebinding cicd-token-rolebinding --clusterrole deployment-clusterrole --serviceaccount app-team1:cicd-token
# 检查
rolebinding.rbac.authorization.k8s.io/cicd-token-rolebinding created
candidate@node01:~$ kubectl -n app-team1 describe rolebindings.rbac.authorization.k8s.io
 cicd-token-rolebinding 
Name:         cicd-token-rolebinding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  deployment-clusterrole
Subjects:
  Kind            Name        Namespace
  ----            ----        ---------
  ServiceAccount  cicd-token  app-team1
```

4. 验证是否正常

```bash
candidate@node01:~$ kubectl auth can-i create deployment --as system:serviceaccount:app-team1:cicd-token
no
candidate@node01:~$ kubectl -n app-team1 auth can-i create deployment --as system:serviceaccount:app-team1:cicd-token
yes
```

## 2、扩容 deployment 副本数量

**Task**

将 deployment `presentation` 扩展至 `4` 个 pods

```bash
# 使用scale命令扩容
candidate@node01:~$ kubectl scale deployment presentation --replicas 4
deployment.apps/presentation scaled
# 检查pod数量
candidate@node01:~$ kubectl get pod |grep presentation
presentation-7d4c9c8446-975tz   1/1     Running   4 (55m ago)   34d
presentation-7d4c9c8446-fsxvm   1/1     Running   0             9s
presentation-7d4c9c8446-mkpcp   1/1     Running   0             9s
presentation-7d4c9c8446-p9c2m   1/1     Running   0             9s
```

![image-20241229175213989](https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20241229175213989.png)

如果忘记命令可以使用 `kubectl scale -h` 查看

也可以使用 `kubectl edit` 命令操作

## 3、 配置网络策略 NetworkPolicy

**Task**

在现有的 namespace `my-app` 中创建一个名为 `allow-port-from-namespace` 的新 `NetworkPolicy`。

确保新的 `NetworkPolicy` 允许 namespace `echo` 中的 Pods 连接到 namespace `my-app` 中的 Pods 的 `9000` 端口。

进一步确保新的 NetworkPolicy：

**不允许**对没有在监听 端口 `9000` 的 Pods 的访问

**不允许**非来自 namespace `echo` 中的 Pods 的访问

英文索引：Concepts → Services, Load Balancing, and Networking → Network Policies

中文索引：概念 → 服务、负载均衡和联网 → 网络策略

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20250105124332271.png" alt="image-20250105124332271" style="zoom: 67%;" />

1. 给访问者命名空间打标签

```bash
candidate@node01:~$ kubectl label ns echo project=echo
namespace/echo labeled
candidate@node01:~$ kubectl get ns echo --show-labels 
NAME   STATUS   AGE   LABELS
echo   Active   41d   kubernetes.io/metadata.name=echo,project=echo
```

2. 创建 networkpolicy

```bash
candidate@node01:~$ vim networkpolicy.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-from-namespace
  namespace: my-app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          project: echo #访问者的命名空间的标签 label（echo 访问 my-app），也就是上面我们手动打的那个标签。
    ports:
    - protocol: TCP
      port: 9000 #被访问者公开的端口

:wq
candidate@node01:~$ kubectl apply -f networkpolicy.yaml 
networkpolicy.networking.k8s.io/allow-port-from-namespace created
# 检查
candidate@node01:~$ kubectl get networkpolicies -n my-app 
NAME                        POD-SELECTOR   AGE
allow-port-from-namespace   <none>         12s

candidate@node01:~$ kubectl describe networkpolicies.networking.k8s.io -n my-app 
Name:         allow-port-from-namespace
Namespace:    my-app
Created on:   2025-01-05 12:50:42 +0800 CST
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Allowing ingress traffic:
    To Port: 9000/TCP
    From:
      NamespaceSelector: project=echo
  Not affecting egress traffic
  Policy Types: Ingress
```

## 4、暴露服务 service

**Task**

请重新配置现有的 deployment `front-end` 以及添加名为 `http` 的端口规范来公开现有容器 `nginx` 的端口 `80/tcp`。

创建一个名为 `front-end-svc` 的新 service，以公开容器端口 http。

配置此 service，以通过各个 Pod 所在的节点上的 `NodePort` 来公开他们。

英文索引：Concepts → Workloads → Workload Resources → Deployments

中文索引：概念 → 工作负载 → 工作负载管理 → Deployments

1. 现有Deployment front-end 添加端口规范

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20241229175857891.png" alt="image-20241229175857891" style="zoom:50%;" />

```bash
candidate@node01:~$ kubectl edit deployments.apps front-end
```

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20241229180105337.png" alt="image-20241229180105337" style="zoom: 67%;" />

验证：`kubectl describe deployments.apps front-end`

```bash
candidate@node01:~$ kubectl describe deployments.apps front-end | grep Port
    Port:          80/TCP
    Host Port:     0/TCP
```

2. 创建 NodePort 类型 Service 暴露服务
   1. 通过 kubectl expose 创建 service 暴露：
    ```bash
   # 查询使用方式
   candidate@node01:~$ kubectl expose -h
    ...
   Usage:
     kubectl expose (-f FILENAME | TYPE NAME) [--port=port]
   [--protocol=TCP|UDP|SCTP] [--target-port=number-or-name] [--name=name]
   [--external-ip=external-ip-of-service] [--type=type] [options]
   # 填写参数
   candidate@node01:~$ kubectl expose deployment front-end --port 80 --target-port 80 --protocol TCP --name front-end-svc --type NodePort
   service/front-end-svc expose
   # 检查和验证
   candidate@node01:~$ kubectl get service front-end-svc 
   NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
   front-end-svc   NodePort   10.102.148.228   <none>        80:32129/TCP   2m45s
   # clusterip:80
   candidate@node01:~$ curl http://10.102.148.228:80
   Hello World ^_^
   # <nodeIP>:<nodeport>
   candidate@node01:~$ curl http://11.0.1.112:32129
   Hello World ^_^
    ```
   2. 直接创建 service 暴露
   
   

## 5、创建 Ingress

**Task**

如下创建一个新的 nginx Ingress 资源：

名称: `ping`

Namespace: `ing-internal`

使用服务端口 `5678` 在路径 `/hello` 上公开服务 `hello`

可以使用以下命令检查服务 `hello` 的可用性，该命令应返回 `hello`：

```bash
curl -kL <INTERNAL_IP>/hello
```

英文索引：Concepts → Services, Load Balancing, and Networking → Ingress

中文索引：概念 → 服务、负载均衡和联网 → Ingress

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20241229181710262.png" alt="image-20241229181710262" style="zoom:67%;" />

根据官方案例进行修改

1. 检查环境的 ingress-class：

```bash
candidate@node01:~$ kubectl get ingressclasses.networking.k8s.io 
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       34d
```

2. 查看对应的 service

```bash
candidate@node01:~$ kubectl get svc -n ing-internal 
NAME    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
hello   ClusterIP   10.104.77.13   <none>        5678/TCP   34d
```

3. 编辑 ingress.yaml

```yaml
candidate@node01:~$ vim ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ping
  namespace: ing-internal
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /hello
        pathType: Prefix
        backend:
          service:
            name: hello
            port:
              number: 5678

:wq
# 应用
candidate@node01:~$ kubectl apply -f ingress.yaml 
ingress.networking.k8s.io/ping created
# 检查
candidate@node01:~$ kubectl get ingress -n ing-internal 
NAME   CLASS   HOSTS   ADDRESS   PORTS   AGE
ping   nginx   *                 80      10s
# 等待大概3分钟...
candidate@node01:~$ kubectl get ingress -n ing-internal 
NAME   CLASS   HOSTS   ADDRESS          PORTS   AGE
ping   nginx   *       10.104.181.152   80      61s
# 验证
candidate@node01:~$ curl http://10.104.181.152/hello
Hello World ^_^
```

## 6、查看 pod 的 CPU

**Task**

通过 pod label `name=cpu-loader`，找到运行时占用大量 CPU 的 pod，

并将占用 CPU 最高的 pod 名称写入文件 `/opt/KUTR000401/KUTR00401.txt`（已存在）。

```bash
candidate@node01:~$ kubectl -A top pod -l name=cpu-loader --sort-by cpu
NAMESPACE   NAME                          CPU(cores)   MEMORY(bytes)   
cpu-top     redis-test-78fd799bd8-bl7wn   1m           4Mi             
cpu-top     nginx-host-564bffdb9-xn847    0m           3Mi             
cpu-top     test0-5b59ff84c7-97jxn        0m           4Mi
# 写入文件
candidate@node01:~$ echo redis-test-78fd799bd8-bl7wn > /opt/KUTR000401/KUTR00401.txt
candidate@node01:~$ cat /opt/KUTR000401/KUTR00401.txt
redis-test-78fd799bd8-bl7wn
```

## 7、调度 pod 到指定节点

**Task**

按如下要求调度一个 pod：

名称：`nginx-kusc00401`

Image：`nginx:1.16`

Node selector：`disk=ssd`

英文索引：Tasks → Configure Pods and Containers → Assign Pods to Nodes

中文索引：任务 → 配置 Pods 和容器 → 将 Pod 分配给节点

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20241229183028935.png" alt="image-20241229183028935" style="zoom: 50%;" />

```bash
candidate@node01:~$ vim pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: nginx-kusc00401
spec:
  containers:
  - name: nginx
    image: nginx:1.16
    ports:
    - containerPort: 80
  nodeSelector:
    disk: ssd

:wq
# 应用
candidate@node01:~$ kubectl apply -f pod.yaml 
pod/nginx-kusc00401 created
# 检查
candidate@node01:~$ kubectl get nodes -l disk=ssd
NAME     STATUS   ROLES    AGE   VERSION
node01   Ready    <none>   35d   v1.31.0
candidate@node01:~$ kubectl get pod nginx-kusc00401 -o wide 
NAME              READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
nginx-kusc00401   1/1     Running   0          21s   10.244.196.168   node01   <none>           <none>
```

## 8、查看可用节点数量

**Task**

检查有多少 nodes 已准备就绪（不包括被打上 Taint：`NoSchedule` 的节点），

并将数量写入 `/opt/KUSC00402/kusc00402.txt`

1. 查看总节点数量

```bash
candidate@node01:~$ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
master01   Ready    control-plane   62d   v1.31.0
node01     Ready    <none>          35d   v1.31.0
node02     Ready    <none>          62d   v1.31.0
```

2. 查看打上 Taint：`NoSchedule` 的节点

```bash
candidate@node01:~$ kubectl describe nodes | grep Taints
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
Taints:             <none>
Taints:             <none>
```

3. 写入结果

```bash
candidate@node01:~$ echo 2 > /opt/KUSC00402/kusc00402.txt
candidate@node01:~$ cat /opt/KUSC00402/kusc00402.txt
2
```

## 9、创建多容器的 pod

**Task**

按如下要求调度一个 Pod：

名称：`kucc8`

app containers: `2`

container name/images：（name 去掉版本号）

- `nginx:1.16`
- `memcached:2.1`

英文索引：Concepts → Workloads → Pods

中文索引：概念 → 工作负载 → Pods

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20241229184115699.png" alt="image-20241229184115699" style="zoom:50%;" />

```yaml
candidate@node01:~$ vim pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: kucc8
spec:
  containers:
  - name: nginx
    image: nginx:1.16
  - name: memcached
    image: memcached:2.1

:wq
# 应用
candidate@node01:~$ kubectl apply -f pod.yaml 
pod/kucc8 created
# 检查
candidate@node01:~$ kubectl get pod kucc8 
NAME    READY   STATUS    RESTARTS   AGE
kucc8   2/2     Running   0          36s
```

## 10、创建 PV

创建名为 `app-config` 的 persistent volume，容量为 `1Gi`，访问模式为 `ReadWriteMany`。volume 类型为 `hostPath`，位于 `/srv/app-config`

英文索引：Tasks → Configure Pods and Containers → Configure a Pod to Use a PersistentVolume for Storage

中文索引：任务 → 配置 Pod 和容器 → 配置 Pod 以使用 PersistentVolume 作为存储

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20241229184716838.png" alt="image-20241229184716838" style="zoom:50%;" />

```yaml
candidate@node01:~$ vim pv.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-config
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /srv/app-config
    
:wq
# 应用
candidate@node01:~$ kubectl apply -f pv.yaml 
persistentvolume/app-config created
# 检查
candidate@node01:~$ kubectl get pv app-config 
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
app-config   1Gi        RWX            Retain           Available                          <unset>                          8s
```

## 11、创建 PVC

创建一个新的 `PersistentVolumeClaim`：

名称: `pv-volume`

Class: `csi-hostpath-sc`

容量: `10Mi`

创建一个新的 Pod，来将 `PersistentVolumeClaim` 作为 volume 进行挂载：

名称：`web-server`

Image：`nginx:1.16`

挂载路径：`/usr/share/nginx/html`

配置新的 Pod，以对 volume 具有 `ReadWriteOnce` 权限。

最后，使用 `kubectl edit` 或 `kubectl patch` 将 `PersistentVolumeClaim` 的容量扩展为 `70Mi`，并记录此更改。

英文索引：Tasks → Configure Pods and Containers → Configure a Pod to Use a PersistentVolume for Storage

中文索引：任务 → 配置 Pod 和容器 → 配置 Pod 以使用 PersistentVolume 作为存储

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20241230093914040.png" alt="image-20241230093914040" style="zoom: 50%;" />

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20241230093955062.png" alt="image-20241230093955062" style="zoom:50%;" />

1. 创建 pvc

```yaml
candidate@node01:~$ vim pvc.yaml 

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-volume
spec:
  storageClassName: csi-hostpath-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi

:wq
candidate@node01:~$ kubectl apply -f pvc.yaml 
persistentvolumeclaim/csi-hostpath-sc created
```

2. 创建pod

```yaml
candidate@node01:~$ vim pod.yaml 

apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: pv-volume
  containers:
    - name: task-pv-container
      image: nginx:1.16
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
          
:wq
candidate@node01:~$ kubectl apply -f pod.yaml 
pod/web-server created
```

3. 修改容量为70Mi，并记录：

```bash
kubectl edit pvc pv-volume --record
```

## 12、查看 pod 日志

监控 pod `foo` 的日志并：

提取与错误 `RLIMIT_NOFILE` 相对应的日志行

将这些日志行写入 `/opt/KUTR00101/foo`

```bash
candidate@node01:~$ kubectl logs foo | grep RLIMIT_NOFILE > /opt/KUTR00101/foo
candidate@node01:~$ cat /opt/KUTR00101/foo
2025/01/02 01:16:37 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
```

## 13、使用 sidecar 代理容器日志

**Context**

将一个现有的 Pod 集成到 Kubernetes 的内置日志记录体系结构中（例如 kubectl logs）。

添加 streaming sidecar 容器是实现此要求的一种好方法。

**Task**

使用 `busybox:2.3 Image` 来将名为 `sidecar` 的 sidecar 容器添加到现有的 Pod `11-factor-app` 中。

新的 sidecar 容器必须运行以下命令：

`/bin/sh -c tail -n+1 -f /var/log/11-factor-app.log`

使用挂载在 `/var/log` 的 Volume，使日志文件 `11-factor-app.log` 可用于 sidecar 容器。

除了添加所需要的 volume mount 以外，请勿更改现有容器的规格。

英文索引：Concepts → Cluster Administration → Logging Architecture

中文索引：概念 → 集群管理 → 日志架构

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20250102093202455.png" alt="image-20250102093202455" style="zoom:50%;" />

1. 导出原来的pod
去掉metadata的annotation等信息，只保留name和namespace，去掉status字段：

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20250102093422616.png" alt="image-20250102093422616" style="zoom:50%;" />

2. 添加sidecar容器导出日志

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20250102093927834.png" alt="image-20250102093927834" style="zoom:50%;" />

## 14、升级集群

**Task**

现有的 Kubernetes 集群正在运行版本 `1.31.0`。**仅将 master 节点**上的所有 Kubernetes 控制平面和节点组件升级到版本 `1.31.1`。

确保在升级之前 drain master 节点，并在升级后 uncordon master 节点。

可以使用以下命令，通过 ssh 连接到 master 节点：

ssh master01

可以使用以下命令，在该 master 节点上获取更高权限：

sudo -i

另外，在主节点上升级 `kubelet` 和 `kubectl`。

请不要升级工作节点，etcd，container 管理器，CNI 插件， DNS 服务或任何其他插件。

英文索引：Tasks → Administer a Cluster → Administration with kubeadm → Upgrading kubeadm clusters

中文索引：任务 → 管理集群 → 用 kubeadm 进行管理 → 升级 kubeadm 集群

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20250103004521692.png" style="zoom:50%;" />

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20250103093714971.png" alt="image-20250103093714971" style="zoom:50%;" />

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20250103093325886.png" alt="image-20250103093325886" style="zoom:50%;" />

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20250103093340879.png" alt="image-20250103093340879" style="zoom:50%;" />

1、腾空 master 节点

```bash
# cordon 停止调度，将 node 调为 SchedulingDisabled。新 pod 不会被调度到该 node，但在该 node 的旧 pod 不受影响。
# drain 驱逐节点。首先，驱逐该 node 上的 pod，并在其他节点重新创建。接着，将节点调为 SchedulingDisabled。
# 所以 kubectl cordon master01 这条，可写可不写。
candidate@node01:~$ kubectl cordon master01 
node/master01 cordoned
candidate@node01:~$ kubectl drain master01 --ignore-daemonsets 
node/master01 already cordoned
Warning: ignoring DaemonSet-managed Pods: calico-system/calico-node-zz67m, calico-system/csi-node-driver-2q7v5, kube-system/kube-proxy-xzcsl
evicting pod kube-system/coredns-6ddff5bd6d-rnrl2
evicting pod calico-system/calico-kube-controllers-6cc79b54b4-nwtf8
evicting pod calico-apiserver/calico-apiserver-6b9cd8cf69-9r2mn
evicting pod kube-system/coredns-6ddff5bd6d-gqw29
pod/calico-kube-controllers-6cc79b54b4-nwtf8 evicted
pod/calico-apiserver-6b9cd8cf69-9r2mn evicted
pod/coredns-6ddff5bd6d-gqw29 evicted
pod/coredns-6ddff5bd6d-rnrl2 evicted
node/master01 drained
# 验证
candidate@node01:~$ kubectl get nodes
NAME       STATUS                     ROLES           AGE   VERSION
master01   Ready,SchedulingDisabled   control-plane   67d   v1.31.0
node01     Ready                      <none>          39d   v1.31.0
node02     Ready                      <none>          67d   v1.31.0
```

2、升级 kubeadm （下载新版本的 kubeadm）

```bash
# 切换到master01
candidate@node01:~$ ssh master01 
# 切换root
candidate@master01:~$ sudo -i
# 更新软件源
root@master01:~# apt-get update
...
# 查询版本
root@master01:~# apt-cache show kubeadm | grep 1.31.1
Version: 1.31.1-1.1
Filename: amd64/kubeadm_1.31.1-1.1_amd64.deb
# 取查询到的完整版本号
root@master01:~# apt-get install kubeadm=1.31.1-1.1
...
# 验证版本
root@master01:~# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"31", GitVersion:"v1.31.1", GitCommit:"948afe5ca072329a73c8e79ed5938717a5cb3d21", GitTreeState:"clean", BuildDate:"2024-09-11T21:26:49Z", GoVersion:"go1.22.6", Compiler:"gc", Platform:"linux/amd64"}
```

3、使用新版本 kubeadm 升级控制面

```bash
# 查看升级计划
root@master01:~# kubeadm upgrade plan
...
Upgrade to the latest version in the v1.31 series:

COMPONENT                 NODE       CURRENT    TARGET
kube-apiserver            master01   v1.31.0    v1.31.4
kube-controller-manager   master01   v1.31.0    v1.31.4
kube-scheduler            master01   v1.31.0    v1.31.4
kube-proxy                           1.31.0     v1.31.4
CoreDNS                              v1.11.1    v1.11.3
etcd                      master01   3.5.15-0   3.5.15-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.31.4

Note: Before you can perform this upgrade, you have to update kubeadm to v1.31.4.

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________

# 执行升级命令 这里的版本号是三位版本号
root@master01:~# kubeadm upgrade apply v1.31.1 --etcd-upgrade=false
...
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.31.1". Enjoy!
```

4、升级 kubelet

```bash
# 获取新版本
root@master01:~# apt-get install kubelet=1.31.1-1.1
Reading package lists... Done
Building dependency tree
...
Unpacking kubelet (1.31.1-1.1) over (1.31.0-1.1) ...
Setting up kubelet (1.31.1-1.1) ...
# 重启kubelet
root@master01:~# systemctl daemon-reload 
root@master01:~# systemctl restart kubelet.service
# 查看版本
root@master01:~# kubelet --version
Kubernetes v1.31.1
```

5、升级 kubectl

```bash
# 获取新版本
root@master01:~# apt-get install kubectl=1.31.1-1.1
Reading package lists... Done
Building dependency tree       
...
Unpacking kubectl (1.31.1-1.1) over (1.31.0-1.1) ...
Setting up kubectl (1.31.1-1.1) ...
# 检查
root@master01:~# kubectl version
Client Version: v1.31.1
Kustomize Version: v5.4.2
Server Version: v1.31.1
```

6、返回初始节点，恢复节点调度

```bash
root@master01:~# exit
logout
candidate@master01:~$ exit
logout
Connection to master01 closed.
# 恢复节点
candidate@node01:~$ kubectl uncordon master01 
node/master01 uncordoned
# 检查节点状态
candidate@node01:~$ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
master01   Ready    control-plane   68d   v1.31.1
node01     Ready    <none>          41d   v1.31.0
node02     Ready    <none>          68d   v1.31.0
```

## 15、备份还原 etcd

**Task**

您必须从 master01 主机执行所需的 etcdctl 命令。

首先，为运行在 https://127.0.0.1:2379 上的现有 etcd 实例创建快照并将快照保存到 /var/lib/backup/etcd-snapshot.db

提供了以下 TLS 证书和密钥，以通过 etcdctl 连接到服务器。

CA 证书: `/opt/KUIN00601/ca.crt`

客户端证书: `/opt/KUIN00601/etcd-client.crt`

客户端密钥: `/opt/KUIN00601/etcd-client.key`

为给定实例创建快照预计能在几秒钟内完成。 如果该操作似乎挂起，则命令可能有问题。用 CTRL + C 来取消操作，然后重试。

然后通过位于`/data/backup/etcd-snapshot-previous.db` 的先前备份的快照进行还原。

英文索引：Tasks → Administer a Cluster → Operating etcd clusters for Kubernetes

中文索引：任务 → 管理集群 → 操作 Kubernetes 中的 etcd 集群

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20250104173840601.png" alt="image-20250104173840601" style="zoom: 67%;" />

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20250104173929440.png" alt="image-20250104173929440" style="zoom:67%;" />

1. 切换到主节点

```bash
candidate@node01:~$ ssh master01
candidate@master01:~$ sudo -i
root@master01:~# 
```

2. 备份 etcd

```bash
root@master01:~# ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/opt/KUIN00601/ca.crt --cert=/opt/KUIN00601/etcd-client.crt --key=/opt/KUIN00601/etcd-client.key \
  snapshot save /var/lib/backup/etcd-snapshot.db
{"level":"info","ts":"2025-01-04T17:44:00.441423+0800","caller":"snapshot/v3_snapshot.go:65","msg":"created temporary db file","path":"/var/lib/backup/etcd-snapshot.db.part"}
{"level":"info","ts":"2025-01-04T17:44:00.448500+0800","logger":"client","caller":"v3@v3.5.15/maintenance.go:212","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":"2025-01-04T17:44:00.448539+0800","caller":"snapshot/v3_snapshot.go:73","msg":"fetching snapshot","endpoint":"https://127.0.0.1:2379"}
{"level":"info","ts":"2025-01-04T17:44:00.518689+0800","logger":"client","caller":"v3@v3.5.15/maintenance.go:220","msg":"completed snapshot read; closing"}
{"level":"info","ts":"2025-01-04T17:44:00.528989+0800","caller":"snapshot/v3_snapshot.go:88","msg":"fetched snapshot","endpoint":"https://127.0.0.1:2379","size":"11 MB","took":"now"}
{"level":"info","ts":"2025-01-04T17:44:00.529061+0800","caller":"snapshot/v3_snapshot.go:97","msg":"saved","path":"/var/lib/backup/etcd-snapshot.db"}
Snapshot saved at /var/lib/backup/etcd-snapshot.d
# 检查备份情况
root@master01:~# ETCDCTL_API=3 etcdctl snapshot status /var/lib/backup/etcd-snapshot.db -w table
Deprecated: Use `etcdutl snapshot status` instead.

+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 69048169 |    70133 |       2920 |      11 MB |
+----------+----------+------------+------------+
```

3. 使用文件还原 etcd

```yaml
# 创建临时目录
root@master01:~# mkdir -p /opt/bak/
# 将/etc/kubernetes/manifests/kube-*文件移动到临时目录里
root@master01:~# mv /etc/kubernetes/manifests/kube-* /opt/bak/
root@master01:~# ls /opt/bak/
kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
# 使用文件还原
# # 需要指定 etcd 还原后的新数据目录，随便写一个系统里没有的目录，比如/var/lib/etcd-restore
root@master01:~# ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --data-dir /var/lib/etcd-restore snapshot restore /data/backup/etcd-snapshot-previous.db
Deprecated: Use `etcdutl snapshot restore` instead.

2025-01-04T17:48:38+08:00       info    snapshot/v3_snapshot.go:265     restoring snapshot      {"path": "/data/backup/etcd-snapshot-previous.db", "wal-dir": "/var/lib/etcd-restore/member/wal", "data-dir": "/var/lib/etcd-restore", "snap-dir": "/var/lib/etcd-restore/member/snap", "initial-memory-map-size": 0}
2025-01-04T17:48:38+08:00       info    membership/store.go:141 Trimming membership information from the backend...
2025-01-04T17:48:38+08:00       info    membership/cluster.go:421       added member    {"cluster-id": "cdf818194e3a8c32", "local-member-id": "0", "added-peer-id": "8e9e05c52164694d", "added-peer-peer-urls": ["http://localhost:2380"]}
2025-01-04T17:48:38+08:00       info    snapshot/v3_snapshot.go:293     restored snapshot       {"path": "/data/backup/etcd-snapshot-previous.db", "wal-dir": "/var/lib/etcd-restore/member/wal", "data-dir": "/var/lib/etcd-restore", "snap-dir": "/var/lib/etcd-restore/member/snap", "initial-memory-map-size": 0}

# 检查数据
root@master01:~# ls /var/lib/etcd-restore/
member
# 修改 etcd 配置文件
root@master01:~# vim /etc/kubernetes/manifests/etcd.yaml
# 将临时目录的文件，移动回/etc/kubernetes/manifests/目录里
root@master01:~# mv /opt/bak/kube-* /etc/kubernetes/manifests/
# 重启kubelet
root@master01:~# systemctl daemon-reload 
root@master01:~# systemctl restart kubelet.service 
```

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image-20250105012952545.png" alt="image-20250105012952545" style="zoom:67%;" />

## 16、排查集群中故障节点

**Task**

名为 node02 的 Kubernetes worker node 处于 `NotReady` 状态。

调查发生这种情况的原因，并采取相应的措施将 node 恢复为 `Ready` 状态，确保所做的任何更改永久生效。

可以使用以下命令，通过 ssh 连接到 node02 节点：

ssh node02

可以使用以下命令，在该节点上获取更高权限：

sudo -i

**没必要参考官网，记住先 restart 再 enable 就行。**

```bash

candidate@node01:~$ kubectl get nodes
NAME       STATUS     ROLES           AGE   VERSION
master01   Ready      control-plane   69d   v1.31.0
node01     Ready      <none>          41d   v1.31.0
node02     NotReady   <none>          69d   v1.31.0
# node02异常，登录到node02检查kubelet
candidate@node01:~$ ssh node02
candidate@node02:~$ sudo -i
root@node02:~# systemctl status kubelet.service 
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; disabled; vendor preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: inactive (dead) since Sun 2025-01-05 00:05:46 CST; 1min 39s ago
       Docs: https://kubernetes.io/docs/
   Main PID: 1415 (code=exited, status=0/SUCCESS)
...
# 重启kubelet
root@node02:~# systemctl restart kubelet.service
# enable kubelet
root@node02:~# systemctl enable kubelet.service 
Created symlink /etc/systemd/system/multi-user.target.wants/kubelet.service → /lib/systemd/system/kubelet.service.
# 检查
root@node02:~# systemctl status kubelet.service 
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Sun 2025-01-05 00:09:19 CST; 1min 5s ago
       Docs: https://kubernetes.io/docs/
   Main PID: 34625 (kubelet)
      Tasks: 12 (limit: 2530)
     Memory: 96.0M
...
# 退回初始节点
root@node02:~# exit
logout
candidate@node02:~$ exit
logout
Connection to node02 closed.
# 检查节点是否正常
candidate@node01:~$ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
master01   Ready    control-plane   69d   v1.31.0
node01     Ready    <none>          41d   v1.31.0
node02     Ready    <none>          69d   v1.31.0
```

## 17、节点维护

**Task**

将名为 `node02` 的 node 设置为不可用，并重新调度该 node 上所有运行的 pods

考点：cordon 和 drain 命令的使用

1. 暂停节点2的调度

```bash
candidate@node01:~$ kubectl cordon node02 
node/node02 cordoned
candidate@node01:~$ kubectl get nodes
NAME       STATUS                     ROLES           AGE   VERSION
master01   Ready                      control-plane   69d   v1.31.0
node01     Ready                      <none>          41d   v1.31.0
node02     Ready,SchedulingDisabled   <none>          69d   v1.31.0
```

2. 驱逐节点 node02 的负载

```bash
candidate@node01:~$ kubectl drain node02 --ignore-daemonsets --delete-emptydir-data
node/node02 already cordoned
Warning: ignoring DaemonSet-managed Pods: calico-system/calico-node-qf2t6, calico-system/csi-node-driver-7zxmr, ingress-nginx/ingress-nginx-controller-8nx8w, kube-system/kube-proxy-dstv7
evicting pod cpu-top/redis-test-78fd799bd8-bl7wn
evicting pod cpu-top/nginx-host-564bffdb9-xn847
evicting pod ing-internal/hello-557f75f95d-wtfhv
evicting pod kube-system/metrics-server-8586b6676c-jhkml
pod/metrics-server-8586b6676c-jhkml evicted
pod/redis-test-78fd799bd8-bl7wn evicted
pod/hello-557f75f95d-wtfhv evicted
pod/nginx-host-564bffdb9-xn847 evicted
node/node02 drained
```
