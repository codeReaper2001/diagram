## 5、创建 Ingress

**设置配置环境：**

```bash
[candidate@node-1] $ kubectl config use-context k8s
```

**Task**

如下创建一个新的 nginx Ingress 资源：

名称: `ping`

Namespace: `ing-internal`

使用服务端口 `5678` 在路径 `/hello` 上公开服务 `hello`

可以使用以下命令检查服务 `hello` 的可用性，该命令应返回 `hello`：

```bash
curl -kL <INTERNAL_IP>/hello
```

考点：Ingress 的创建

```yaml
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
```

## 7、调度 pod 到指定节点

**设置配置环境：**

```bash
[candidate@node-1] $ kubectl config use-context k8s
```

**Task**

按如下要求调度一个 pod：

名称：`nginx-kusc00401`

Image：`nginx:1.16`

Node selector：`disk=ssd`

```yaml
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
```

## 9、创建多容器的 pod

**设置配置环境：**

```bash
[candidate@node-1] $ kubectl config use-context k8s
```

**Task**

按如下要求调度一个 Pod：

名称：`kucc8`

app containers: `2`

container name/images：（name 去掉版本号）

- `nginx:1.16`
- `memcached:2.1`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kucc8
spec:
  containers:
  - name: nginx
    image: nginx:1.16
    imagePullPolicy: IfNotPresent
  - name: memcached
    image: memcached:2.1
    imagePullPolicy: IfNotPresent
```

## 10、创建 PV

创建名为 `app-config` 的 persistent volume，容量为 `1Gi`，访问模式为 `ReadWriteMany`。volume 类型为 `hostPath`，位于 `/srv/app-config`

```yaml
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

pvc.yaml

```yaml
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
      storage: 10M
```

pod.yaml

```yaml
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
```

## 12、查看 pod 日志

监控 pod `foo` 的日志并：

提取与错误 `RLIMIT_NOFILE` 相对应的日志行

将这些日志行写入 `/opt/KUTR00101/foo`

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
