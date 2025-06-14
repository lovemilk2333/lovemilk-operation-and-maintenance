# 使用 YAWL 配置文件配置 k8s

## 配置文件

> 其他资源对象类型的配置参见
> [配置 | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/configuration/)
> 创建任意名称的 YAML 格式配置文件 (类似于 Docker Compose), 格式类似于

```yaml
apiVersion: apps/v1 # API 的 group/version; group 包括 apps (应用), batch (批处理), autoscaling (自动扩/缩容) 等; 也可以只填写 version
kind: Deployment # 资源对象类型
metadata: # 资源对象元数据, 名称和标签等
  name: nginx-deployment
spec: # 定义资源对象的配置信息 (Deployment 的配置信息)
  # ReplicaSet 的配置
  selector: # 选择器, 选择 app==nginx 的 Pod
    matchLabels:
      app: nginx
  replicas: 3 # 副本数量
  template:
    # 指定 Pod 的配置模板
    metadata:
      labels:
        app: nginx
    spec: # Pod 的配置信息
      containers:
        - name: nginx # 容器名称
          image: nginx # 使用的镜像
          resources: # 资源限制 (可选)
            requests:
              cpu: 100m
              memory: 100Mi
          ports:
            - containerPort: 80 # 暴露的端口
```

## 使用配置文件创建/更新/删除资源对象

`-f` 参数以使用指定的配置文件创建/更新/删除资源对象

> `create` 与 `apply` 的区别: 前者仅创建, 后者会创建 (不存在时) 或更新资源对象

```sh
kubectl create/apply/delete -f <config_file>
```

## 最佳实践: Nginx Service

这将创建一个内部的 Nginx 服务, 可通过在 集群内部的设备 (如 Master Ndoe)
`curl <Server_IP>:8080` 访问 Pod 内部的 Nginx 的 80 端口, 但是外部无法访问\
[下载配置](nginx-example.cluster-ip.yml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1 # 创建 1 个 nginx 副本
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx # Pod 的标签, Service 选择器会选择
    spec:
      containers:
        - name: nginx
          image: nginx:latest # 使用官方的 nginx 镜像
          ports:
            - containerPort: 80 # 容器内部暴露端口
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector: # 选择 app==nginx 的 Pod
    app: nginx
  ports:
    - protocol: TCP # 协议
      port: 8080 # 外部暴露的端口
      targetPort: 80 # Pod 内的端口
  type: ClusterIP # 默认为 ClusterIP (可选)
```

将 `type: ClusterIP` 设置为 `NodePort`, 并在 `ports` 的 `nodePort` 字段指定 Node
的端口 (必须在 $[30000, 32767]$) 后, 可以通过 `curl <Node_IP>:<nodePort>` 访问

> 注意: 是 Node 的 IP 而非 Server 的 IP

[下载修改后的配置](nginx-example.node-port.yml)

可选的 Service `type` 如下

|     类型     | 说明                                     |
| :----------: | :--------------------------------------- |
|  ClusterIP   | 默认类型, 集群内部的服务                 |
|   NodePort   | 节点端口类型, 将服务公开到集群节点上     |
| LoadBalancer | 负载均衡类型, 将服务公开到外部负载均衡器 |
| ExternalName | 外部名称类型, 将服务映射到一个外部域名上 |
|   Headless   | 无头类型, 主要用于 DNS 解析和服务发现    |
