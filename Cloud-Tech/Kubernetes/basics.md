# Kubernetes Basics

## K8s Cluster

k8s 集群有两部分组成：

- Master 节点
- Node 节点

Cluster Diagram

![cluster diagram](https://d33wubrfki0l68.cloudfront.net/99d9808dcbf2880a996ed50d308a186b5900cec9/40b94/docs/tutorials/kubernetes-basics/public/images/module_01_cluster.svg)

**Master 节点用于管理整个集群**。例如应用调度，管理应用的生命周期。扩展、滚动升级应用，等等。

**Node 节点用于执行真正的工作，它可以是一个 VM ，也可以是一个物理机**。每个 Node 会有一个 kubelet 的 agent 用于和 Master 通信。同时，每个 Node 也会有一些工具进行容器操作，比如 Docker 引擎。

## K8s Deployments

**Deployment** 配置用于配置如何创建或者更新你的应用。一旦你创建了 **Deployment** ，k8s Master 就会在指定的节点上调度相关的应用实例。

一旦应用实例创建完成，k8s deployment controller 会监控这些实例，一旦实例挂掉，deployment controller 将会启用新的实例替换它。

Deploying app on Kubernetes

![Deploying app on Kubernetes](https://d33wubrfki0l68.cloudfront.net/152c845f25df8e69dd24dd7b0836a289747e258a/4a1d2/docs/tutorials/kubernetes-basics/public/images/module_02_first_app.svg)

### 创建 Deployment

使用 `kubectl` 命令行工具创建 Deployment，创建 Deployment 有两个必须提供的：

- 容器镜像
- replica 数量

获取节点信息：

```bash
kubectl get nodes
```

创建新的 deployment：

```bash
kubectl run $deployment_name --image=$deployment_docker_image_full_url --port=$port
```

获取 deployments 信息：

```bash
kubectl get deployments
```

## K8s Pods

**Pod** 是 k8s 中一个抽象出来的概念，它用于表示一组（一个或多个）容器，以及这些容器共享的一些资源：

- 存储资源，例如共享存储，卷。
- 网络，使用同一个唯一的集群IP。
- 运行容器的一些元信息，例如容器镜像版本，可用的指定端口等。

**Pod** 通常用于将一组相关的容器组合起来进行建模。

每个 **Pods** 都拥有一个唯一的 IP 地址，即使这些 **Pods** 可能在同一个 **Node** 上。但是 **Pod** 的 IP 地址是无法对集群外暴露的。

Pods overview

![Pods overview](https://d33wubrfki0l68.cloudfront.net/fe03f68d8ede9815184852ca2a4fd30325e5d15a/98064/docs/tutorials/kubernetes-basics/public/images/module_03_pods.svg)

## K8s Nodes

**Node** 在 k8s 集群中是具体的工作节点，工作节点可能是虚拟机也可能是物理机，这取决于集群。每个 **Node** 由集群中的 **Master** 进行管理。一个 **Node** 用于多个 **Pods**，并且 k8s 会在所有的工作节点上对 **Pods** 进行调度。

每个 k8s **Node** 都至少运行了：

- `Kubectl`， 用于和 **Master** 进行通信；并且管理该工作节点（机器）上的 **Pods** 和容器。
- 一个容器运行环境（Docker 或者 rkt)，用于从仓库拉取指定的容器镜像，解包容器并运行应用。

Node overview

![Node overview](https://d33wubrfki0l68.cloudfront.net/5cb72d407cbe2755e581b6de757e0d81760d5b86/a9df9/docs/tutorials/kubernetes-basics/public/images/module_03_nodes.svg)

## `kubectl` 常用命令

- **kubectl get** 获取资源
- **kubectl describe** 打印某资源的详情
- **kubectl logs** 打印某个 pod 中的容器日志
- **kubectl exec** 在某个 pod 中的容器内执行命令

## K8s Service

**Service** 也是 k8s 集群中的一种抽象概念。它定义了一组 **Pods** 的逻辑集合， 以及如何访问这组 **Pods** 的一些策略。

**Service** 可以使用 YAML 或者 JSON 定义，但官方更推荐使用 YAML。

**Service** 所指定的 **Pods** 集合一般是由 `LabelSelector` 来决定的。

**Service** 可以使应用程序接收流量，它可以通过以下几种方式暴露到集群之外，通过在 `ServiceSpec` 中指定 `type`：

- `ClusterIP`， 这是默认选项，该选项使用集群内部IP暴露 **Service**。此种方式暴露的 **Service** 只能在集群内部访问。
- `NodePort`， 使用集群中选中 **Node** 的相同端口通过 NAT 技术将 **Service** 暴露出去。这样就可以在集群外部使用 `NodeIP:NodePort` 来访问指定的 **Service**。
- `LoadBalancer` 在云上创建一个外部的负载均衡，且拥有一个固定的外部 IP ，该 IP 指向了指定的 **Service** 。 这是 `NodePort` 的超集。
- `ExternalName` 使用任意的名称（使用 `kube-dns` 解析成 CNAME 记录）将 **Service** 暴露出去，（通过指定 `externalName`）。

Service and Labels

![Service and Labels1](https://d33wubrfki0l68.cloudfront.net/cc38b0f3c0fd94e66495e3a4198f2096cdecd3d5/ace10/docs/tutorials/kubernetes-basics/public/images/module_04_services.svg)

![Service and Labels2](https://d33wubrfki0l68.cloudfront.net/b964c59cdc1979dd4e1904c25f43745564ef6bee/f3351/docs/tutorials/kubernetes-basics/public/images/module_04_labels.svg)

## K8s Instance Scaling

K8s 中的 **scaling** 是通过更改 replicas 的数量完成的。

Scaling overview

![Scaling overview1](https://d33wubrfki0l68.cloudfront.net/043eb67914e9474e30a303553d5a4c6c7301f378/0d8f6/docs/tutorials/kubernetes-basics/public/images/module_05_scaling1.svg)

![Scaling overview2](https://d33wubrfki0l68.cloudfront.net/30f75140a581110443397192d70a4cdb37df7bfc/b5f56/docs/tutorials/kubernetes-basics/public/images/module_05_scaling2.svg)

## K8s Rolling Update

**Rolling Update** 可以理解成 zero downtime release。

**Rolling Update** 是通过 **Instance Scaling** 实现的。默认情况下，升级过程中最大不可用 **Pod** 数量为1。能够新建的 **Pod** 数量也为1。但这是可配置的。

在 k8s 中，每次升级都会有对应的内部版本号，可以轻松实现回滚。

Rolling update overview

![Rolling update overview1](https://d33wubrfki0l68.cloudfront.net/30f75140a581110443397192d70a4cdb37df7bfc/fa906/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates1.svg)

![Rolling update overview2](https://d33wubrfki0l68.cloudfront.net/678bcc3281bfcc588e87c73ffdc73c7a8380aca9/703a2/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates2.svg)

![Rolling update overview3](https://d33wubrfki0l68.cloudfront.net/9b57c000ea41aca21842da9e1d596cf22f1b9561/91786/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates3.svg)

![Rolling update overview4](https://d33wubrfki0l68.cloudfront.net/6d8bc1ebb4dc67051242bc828d3ae849dbeedb93/fbfa8/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates4.svg)

## Reference

[Kubernetes tutorials](https://kubernetes.io/docs/tutorials/hello-minikube/)
