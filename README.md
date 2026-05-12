# kgale-web-gitops

这是一个用于演示 **GitHub Actions + Harbor + Argo CD + Kubernetes** 的 GitOps 项目。

项目目标是：开发者只需要修改代码并推送到 GitHub，后续的镜像构建、镜像推送、Kubernetes YAML 更新、Argo CD 同步发布，都可以自动完成。

## 项目架构

```text
开发者提交代码
      |
      v
GitHub 仓库
      |
      v
GitHub Actions 自动构建镜像
      |
      v
Harbor 私有镜像仓库
      |
      v
更新 Kubernetes YAML 镜像版本
      |
      v
Argo CD 监听 Git 仓库变化
      |
      v
Kubernetes 集群自动部署
      |
      v
Service / Ingress 对外提供访问
```

## 当前环境

| 组件 | 地址 / 说明 |
| --- | --- |
| Kubernetes Master | `172.16.11.142` |
| Kubernetes Worker1 | `172.16.11.145` |
| Kubernetes Worker2 | `172.16.11.146` |
| Harbor 镜像仓库 | `172.16.11.142` |
| 应用 NodePort | `http://172.16.11.142:30084` |
| HTTPS 入口域名 | `https://web.xoho.top` |
| Argo CD | `https://172.16.11.142:30083` |
| Kubernetes Dashboard | `https://172.16.11.142:30443` |
| Grafana 监控面板 | `http://172.16.11.142:30030` |

## 仓库文件说明

| 文件 | 作用 |
| --- | --- |
| `index.html` | Web 演示页面 |
| `Dockerfile` | 构建 Nginx 静态网页镜像 |
| `kgale-web-gitops.yaml` | Kubernetes Deployment 和 Service 清单 |
| `kgale-web-ingress.yaml` | Kubernetes Ingress 和 HTTPS 入口清单 |
| `.github/workflows/build-and-deploy.yaml` | GitHub Actions 自动构建和发布流程 |

## CI/CD 流程说明

当 `main` 分支中的以下文件发生变化时，会触发 GitHub Actions：

```text
index.html
Dockerfile
.github/workflows/build-and-deploy.yaml
```

触发后会执行以下流程：

1. 拉取 GitHub 仓库代码。
2. 根据 GitHub Actions 运行编号生成镜像版本，例如 `v7`。
3. 使用 `Dockerfile` 构建镜像。
4. 将镜像推送到 Harbor：`172.16.11.142/library/kgale-web:v版本号`。
5. 自动修改 `kgale-web-gitops.yaml` 中的镜像版本。
6. 自动提交一次 `[skip ci]` 提交，避免重复触发流水线。
7. Argo CD 检测到 Git 仓库变化后，同步到 Kubernetes 集群。

## Kubernetes 部署说明

当前应用由一个 Deployment 和一个 Service 组成。

Deployment 负责运行 Web Pod：

```yaml
kind: Deployment
metadata:
  name: kgale-web-gitops
```

Service 负责通过 NodePort 暴露服务：

```yaml
kind: Service
metadata:
  name: kgale-web-gitops-svc
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30084
```

Ingress 负责通过域名访问服务：

```yaml
host: web.xoho.top
service:
  name: kgale-web-gitops-svc
```

## 常用操作命令

查看应用 Pod：

```bash
kubectl get pods -n default -l app=kgale-web-gitops -o wide
```

查看 Service：

```bash
kubectl get svc -n default kgale-web-gitops-svc
```

查看 Ingress：

```bash
kubectl get ingress -n default kgale-web-ingress
```

查看 Argo CD 应用状态：

```bash
kubectl -n argocd get application kgale-web-gitops
```

手动同步 Argo CD：

```bash
argocd app sync kgale-web-gitops
```

查看当前部署使用的镜像：

```bash
kubectl get deploy kgale-web-gitops -n default -o jsonpath='{.spec.template.spec.containers[0].image}'
```

## 镜像仓库说明

当前镜像使用 Harbor 私有仓库：

```text
172.16.11.142/library/kgale-web
```

镜像标签由 GitHub Actions 自动生成：

```text
v1
v2
v3
...
```

示例镜像：

```text
172.16.11.142/library/kgale-web:v7
```

## 代理说明

当前实验环境中，三台 Kubernetes 节点已配置代理，用于访问 GitHub、Docker Hub 等外部资源：

```text
http://172.16.11.60:7898
```

该地址转发到 Windows 本机代理：

```text
127.0.0.1:7897
```

已配置到以下位置：

```text
/etc/environment
/etc/profile.d/proxy.sh
/etc/apt/apt.conf.d/95proxies
/etc/systemd/system/containerd.service.d/http-proxy.conf
/etc/systemd/system/kubelet.service.d/20-proxy.conf
```

## 项目特点

| 特点 | 说明 |
| --- | --- |
| 自动构建 | GitHub Actions 自动构建 Docker 镜像 |
| 私有镜像仓库 | 镜像推送到 Harbor |
| GitOps 发布 | Argo CD 根据 Git 仓库自动同步 |
| Kubernetes 部署 | 应用运行在 Kubernetes 集群中 |
| HTTPS 入口 | 通过 Ingress 提供域名访问 |
| 可观测性 | 已接入 Prometheus、Alertmanager、Grafana |

## 一句话总结

这个仓库不是单纯的静态页面项目，而是一个完整的 Kubernetes GitOps CI/CD 实验项目。

它演示了从代码提交到容器构建、镜像推送、GitOps 同步、Kubernetes 发布、域名访问的完整 DevOps 流程。
