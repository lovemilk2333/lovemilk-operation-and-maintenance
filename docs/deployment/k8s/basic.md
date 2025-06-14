# k8s 基本使用

## 概念

|             名称              | 描述                                                                                                                                                                    |
| :---------------------------: | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|             Node              | 服务器/物理机/虚拟机                                                                                                                                                    |
|              Pod              | 最小调度单元, 一个/多个容器的组合 (一般情况下只运行一个容器), 运行于 Node, (多个)容器可以共享一些资源 (包括但不限于 配置, 存储, 网络) (IP 可能变化, 可轻易被创建与销毁) |
|            Service            | 将一组 Pod 封装, 统一管理和入口 (IP 不变)                                                                                                                               |
|         内部 Service          | 仅需要在 Node 内部被访问的 Service                                                                                                                                      |
|         外部 Service          | 需要暴露给外部的 Service (如 API 接口)                                                                                                                                  |
| NodePort (一般用于 Dev 环境) | 在 Node 上开放一个端口, 用于访问 Service                                                                                                                                |
| Ingress (一般用于 Prod 环境)  | 用于管理 Node 内外部访问入口和方式, 可配置转发规则与域名, 配置 TLS, 负载均衡等                                                                                          |
|       ConfigMap (明文)        | 封装配置, 解耦配置与 Pad, 以免配置改变时造成 Pad 重新编译和部署 (仅需重载)                                                                                              |
|        Secret (Base64)        | 类似于 ConfigMap, 一般用于存储敏感信息, 但是需要其他方式保证安全                                                                                                        |
|            Volume             | 挂载 Pod 内的内容至 Node 本地磁盘或其他 Node 远程存储, 以持久化数据 (类似于 Docker Volume)                                                                              |
|          Deployment           | 定义管理 Pod 副本数量和更新策略, 以便冗余                                                                                                                               |
|          StatefulSet          | 类似于 Deployment, 但 StatefulSet 适用于有持久化数据或状态的 Pod, 例如数据库 (或将 数据库 置于 宿主机, 而不是 k8s 内)                                                   |

## k8s 是如何工作的

k8s 使用 M/W (Master/Worker) 架构, 由 Master-Node 管理所有 Worker-Node

每个 Worker-Node 带有 3 个组件
| 组件 | 描述 |
| :-: | :- |
| kubelet | 管理和维护 Pod, 接受 Pod 的修改, 监控/上报 Pod 的状态 |
| kube-proxy | 为 Pod 提供网络代理和负载均衡, 以高效的方式通讯 Service 和 Pod |
| container-runtime | 实际运行容器的进程 (拉取/创建/启动/停止容器等) |

Master-Node 管理/调度/增删节点/监控状态 Worker-Node, 其组件有
| 组件 | 描述 |
| :-: | :- |
| kube-apiserver | 为 k8s 提供 API 接口, 所有 Node 或 用户均须提供该 API 操作 k8s; 操作认证功能; 后将请求转发至 Scheduler |
| Scheduler | 监控 Node 资源使用情况, 将 Pod 调度到合适的 Node |
| Controller Manager | 对 Pod 状态进行监控和处理, 如自动重启或重建 Pod, 切换到其他 Pod |
| etcd | 键-值存储系统, 记录 Pod 状态等, 用于 Controller Manager 的状态获取与管理 (数据存储中心) |
| Cloud Controller Manager (云服务提供商提供) | 与云平台 API 交互 |

## 使用 minikube 与 kubectl 进行本地 k8s 集群的搭建与管理

minikube 是一个单节点的轻量级 k8s 实现, 用于在本地开发环境模拟 k8s 集群的运行环境进行调试

kubectl 是与 k8s 交互的命令行工具, 可提供向 kube-apiserver 发送请求进行交互

### 安装 minikube (ArchLinux)

> 详细内容请参阅 [Minikube - ArchWiki](https://wiki.archlinux.org/title/Minikube)

安装 minikube (会自动安装 kubectl)

```sh
sudo pacman -S minikube
```

验证安装

```sh
minikube version
```

常用命令

```
minikube 提供并管理针对开发工作流程优化的本地 Kubernetes 集群。

基本命令：
  start            启动本地 Kubernetes 集群
  status           获取本地 Kubernetes 集群状态
  stop             停止正在运行的本地 Kubernetes 集群
  delete           删除本地的 Kubernetes 集群
  dashboard        访问在 minikube 集群中运行的 kubernetes dashboard
  pause            暂停 Kubernetes
  unpause          恢复 Kubernetes
```

要进入 minikube 的 master 节点 (实际上是虚拟机), 请使用
```sh
minikube ssh
```

启动集群

> CN 用户可使用 `minikube start --image-mirror-country=cn` 使用国内镜像源

```sh
minikube start
```

查看集群状态

```sh
minikube status
```

### 使用 kubectl 交互集群

列出资源对象

> `kubectl get` 后跟 `pod` 和 `pods` 等的作用是一样的  
> `all` 表示获取查看所有资源  
> `-o wide` 表示获取更详细的信息, 包括 Pod 的 IP 地址

```sh
kubectl get all/nodes/pods/svc/...
```

例如

```sh
$ kubectl get nodes

NAME       STATUS   ROLES           AGE    VERSION
minikube   Ready    control-plane   1d     v1.31.0
```

获取详细信息

```sh
kubectl describe <type> service/deployment/...
```

创建并运行 Pod

> `<image>` 为 Docker Hub 上的镜像, 仅限 `create` 命令可用

```sh
kubectl run <name> --image=<image>
```

创建/删除资源对象 (除 Pod 外)

```sh
kubectl create/delete [-n=<namespace||'default'>] service/deployment/... [--image=<image>] [...options]
```

例如

```sh
kubectl create/delete deployment nginx-deployment --image=nginx
```

会创建一个 `nginx-deployment` 的 Deployment 对象, 一个 `nginx-deployment-<rep_hash>` 的 ReplicaSet 对象 (用于管理副本), 一个 `nginx-deployment-<rep_hash>-<pod_hash>` 的 Pod 对象

查看 Pod 日志 (类似于 `docker logs`)

```sh
kubectl logs <pod_name>
```

进入容器 (类似于 `docker exec`)

> `--` 表示后续内容均为在容器内执行的命令, 而非 kubectl 参数

```sh
kubectl exec -it <pod_name> -- /bin/bash
```

将 Deployment 对象对外公开为服务
```sh
kubectl expose deployment <deployment_name>
```
