# FAQ

## minikube Docker Registry
配置 Docker Registry 为 minikube 提供的本地镜像仓库已保证可以将镜像推送至 k8s 集群中
> 注意: 仅在当前终端有效

```sh
eval $(minikube docker-env)
```

## 本地预览 Ingress
```sh
$ minikube tunnel

Status:
        machine: minikube
        pid: 234235
        route: 10.96.0.0/12 -> 192.168.59.101  # <- 访问这个 192.168.x.x IP, 默认端口 80
        minikube: Running
        services: []
    errors: 
                minikube: no errors
                router: no errors
                loadbalancer emulator: no errors
```

## minikube tunnel 后访问 Ingress IP 失败
启用 Ingress
```sh
minikube addons enable ingress
```

## Ingress X-Forwarded-For
配置 Ingress 即可获取客户端 IP (会自动附加 Header)

详情参阅 [Ingress | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress)

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: your-ingress-name
spec:
  rules:
  - http:  # 匹配 root
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: your-service
            port:
              number: your-service-port
```
