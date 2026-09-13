# Kubernetes 实战入门

> **定位一句话**：Kubernetes（K8s）= 生产级别的容器编排平台，负责自动部署、扩缩容、自愈、负载均衡一堆容器。
> 和 Docker Swarm 一个赛道（Swarm 那篇在 `docker-note.md` 里写过），但 K8s 是 Google 拿 10 年 Borg 经验开源出来的，生态碾压，事实标准。
>
> **学习路线（进阶目标）**：
> 资源清单（yaml 语法、Pod 生命周期）→ Pod 控制器（各控制器特点）→ 服务发现（SVC 原理）→ 存储（多种存储类型选型）→ 安全（认证/鉴权/访问控制）→ HELM（类似 Linux yum，模板自定义、部署常用插件）→ 运维（如改 kubeadm 证书有效期到 10 年）

---

## 目录

**一、Kubernetes 基础概念**

1. 是什么
2. 架构（工作方式 / 组件架构 / Node 深入）
3. kubeadm 创建集群（安装 / 引导 / 加入节点 / 验证 / Dashboard）

**二、Kubernetes 核心实战**

1. 资源创建方式
2. Namespace
3. Pod
4. Deployment（多副本 / 扩缩容 / 自愈 / 滚动更新 / 版本回退）
5. Service（ClusterIP / NodePort）
6. Ingress（安装 / 使用 / 域名访问 / 路径重写 / 流量限制）
7. 存储抽象（环境准备 / 原生挂载 / PV&PVC / ConfigMap / Secret）

**三、进阶专题**（HPA / SQL 数据库上 K8s / StatefulSet / DaemonSet 与 CronJob）

---

# 一、Kubernetes 基础概念

## 1. 是什么

Kubernetes 这个词来自希腊语，意思是「舵手」或「领航员」，简称 K8s（K 和 s 之间 8 个字母）。它是 Google 2014 年开源的容器编排平台，前身是内部跑了十几年的 Borg 系统——**不是实验室产品，是从大规模生产环境里长出来的**。

一句话理解演进链条：

```
单机容器：docker run            （一台机器跑几个容器）
    │
多机编排：docker swarm / k8s     （一堆机器跑一堆容器，谁来调度？谁来重启挂掉的？）
    │
生产落地：kubernetes            （自愈 + 扩缩容 + 服务发现 + 滚动发布 + 存储卷，全包）
```

### K8s 能干什么

| 能力 | 说明 |
|------|------|
| **自动装箱** | 按 CPU/内存请求自动把容器调度到合适的节点 |
| **自愈** | 容器挂了自动重启、节点挂了自动把 Pod 挪到别的节点重建 |
| **水平扩展** | 按 CPU 等指标自动扩缩副本数（HPA，见进阶专题） |
| **服务发现 & 负载均衡** | 用 Service 名称访问一组 Pod，自带负载均衡，不用自己改 nginx |
| **自动发布 / 回滚** | 滚动更新不停机，发布炸了一条命令回滚 |
| **密钥和配置管理** | ConfigMap / Secret 热更新配置，不用重新打镜像 |
| **存储编排** | 本地盘、NFS、云盘统一抽象成 PV，随 Pod 挂载 |

### 和 Swarm 的关系（快速对齐）

| 项目 | Docker Swarm | Kubernetes |
|------|--------------|------------|
| 出身 | Docker 自带 | Google Borg 开源 |
| 门槛 | 低，会 docker 就行 | 高，概念一堆（本文就是干这个的） |
| 生态 | 几乎停滞 | CNCF 全家桶，事实标准 |
| 适合 | 小团队、内部工具 | 中大型、长期演进的项目 |

> 结论很直接：**新项目学/用 K8s**。Swarm 的价值是帮你 10 分钟理解「编排」这个概念（跨机调度 + 自愈 + 滚动更新），概念完全通用。

---

## 2. 架构

### 2.1 工作方式

K8s 是典型的 **Master-Worker** 架构 + **声明式 API** + **控制循环**：

- **声明式**：你不告诉它「去启动一个容器」，而是提交一份期望状态——「我要 3 个 nginx 副本」。至于怎么达成、挂了怎么补，K8s 自己想办法。
- **控制循环（Reconcile）**：所有控制器都在无限循环干一件事：**对比期望状态和实际状态，有偏差就调谐**。这是 K8s 一切自愈能力的根源。

```
kubectl apply（提交期望状态：我要 3 个 nginx）
        │
        ▼
   kube-apiserver ──写入──► etcd（集群唯一数据源）
        │
        │ watch（监听变化）
        ▼
kube-controller-manager ──发现只有 0 个副本，创建 3 个 Pod 对象（还没落地）
        │
kube-scheduler ──挑出没绑定节点的 Pod，选好节点，写回绑定关系
        │
        ▼
目标节点 kubelet ──看到「分给我的 Pod」，调 containerd 真正把容器跑起来
```

### 2.2 组件架构

#### 2.2.1 控制平面组件（Control Plane Components）

集群的大脑，管决策不管干活。**生产推荐奇数台（3/5/7），副本数 ≥3**——1 台单点，2 台脑裂，和 Swarm 的 manager 一个道理。

先给一份面试口诀版：

| 组件 | 一句话口诀 |
|------|-----------|
| **kube-apiserver** | 所有服务访问统一入口 |
| **kube-controller-manager** | 维持副本期望数目 |
| **kube-scheduler** | 负责调度任务，选择合适的节点分配任务 |
| **etcd** | 键值对数据库，储存 K8s 集群所有重要信息（持久化） |
| **cloud-controller-manager** | 和云厂商 API 打交道（云 LB、节点、路由） |

再展开说：

1. **kube-apiserver**：集群**唯一入口**，所有组件交互都要经过它；提供 REST API，负责认证、鉴权、准入控制；**唯一可以直接读写 etcd 的组件**。kubectl、Dashboard、各种 SDK，最终都是在调它。
2. **etcd**：分布式 KV 数据库，集群**所有元数据**（Pod、Service、ConfigMap……你 apply 的一切）都存在这。etcd 挂了且没备份 = 集群整个没了，**务必备份**。
3. **kube-scheduler**：调度器。只干一件事：监控新创建且未绑定节点的 Pod，跑「预选（过滤不满足条件的节点）→ 优选（打分挑最优）」两轮算法，把 Pod 绑定到某节点。**它不启动容器**，只写决策。
4. **kube-controller-manager**：控制器管理器，内部装了一堆控制器（副本、节点、端点、Job……），每个都在跑控制循环，对比期望 vs 实际，不断调谐。
5. **cloud-controller-manager**：把「节点、负载均衡、存储卷」这类操作委托给云厂商 API。自建裸机集群用不到。

#### 2.2.2 Node 组件

干活的机器上的组件：

1. **kubelet**：运行在每个节点上的节点代理。直接跟容器引擎（containerd/docker）交互实现**容器生命周期管理**——创建、启动、监控、销毁，同时把节点和 Pod 状态上报给 apiserver。Scheduler 只是指挥，**真正落地干活的永远是 kubelet**。
2. **kube-proxy**：每个节点上的网络代理。监听 Service 和 Endpoint 变化，把规则写入 **iptables / ipvs**，实现 Service 的转发和负载均衡——让「访问 Service IP」这件事真的能通。

3. **容器运行时**：containerd（新标配）或 docker，真正跑容器的。

#### 2.2.3 生态组件（第三方，了解即可）

| 组件 | 作用 |
|------|------|
| **CoreDNS** | 为集群中的 Service 创建域名 → IP 解析，`svc名.命名空间.svc.cluster.local` 直接访问 |
| **Dashboard** | 给 K8s 集群提供 B/S 结构的可视化访问体系 |
| **Ingress Controller** | 官方 Service 只能四层代理，Ingress 实现**七层**（HTTP 域名/路径）代理 |
| **Federation** | 跨多个 K8s 集群统一管理，项目基本废弃，知道概念即可 |
| **Prometheus** | 集群监控（指标采集 + 告警 + Grafana 可视化），云原生监控标准 |
| **ELK / EFK** | 集群日志统一收集、检索、分析平台 |

### 2.3 Node 深入：状态、心跳与节点管理

Node 是集群中的工作机器（物理机或虚拟机），是 Pod 实际运行的地方。

**节点条件（Conditions）**：

| 条件 | 含义 |
|------|------|
| Ready | 节点是否健康 |
| MemoryPressure | 内存是否不足 |
| DiskPressure | 磁盘空间是否不足 |
| PIDPressure | 进程 ID 是否不足 |
| NetworkUnavailable | 网络是否不可用 |

**心跳机制**：kubelet 每 10 秒更新状态 → Node Controller 每 5 秒检查 → **超 40 秒没心跳 → NotReady → 超 5 分钟 → 驱逐该节点上的 Pod**（重新调度到其他节点）。

**节点管理命令**：

| 操作 | 命令 |
|------|------|
| 查看节点 | `kubectl get nodes` |
| 查看详情 | `kubectl describe node <name>` |
| 标记不可调度 | `kubectl cordon <node>` |
| 驱逐 Pod（维护前） | `kubectl drain <node>` |
| 恢复调度 | `kubectl uncordon <node>` |
| 打标签 | `kubectl label node <node> key=value` |
| 打污点 | `kubectl taint node <node> key=value:NoSchedule` |

> **drain 流程**：cordon（先禁调度）→ 驱逐 Pod（遵守 PDB）→ 等优雅终止 → 人工确认后 uncordon。**维护节点前必做**，不然 Pod 强杀导致业务抖动。

资源压力时 kubelet 按 QoS 等级驱逐 Pod：BestEffort（无 requests/limits）最先被赶，Guaranteed（requests = limits）最后。

---

## 3. kubeadm 创建集群

kubeadm 是官方提供的集群引导工具：把「装一套 K8s」从手工几十步压缩成 `init` + `join` 两条命令。**它只负责控制面组件的拉起，网络组件、存储、监控都得自己装**。

环境约定（三台机器示例）：

| 机器 | 角色 | 假设 IP |
|------|------|---------|
| k8s-master | 主节点 | 172.31.0.4 |
| k8s-node1 | 工作节点 | 172.31.0.5 |
| k8s-node2 | 工作节点 | 172.31.0.6 |

> 以下以 CentOS/Alibaba Cloud Linux（yum 系）为主线，**Debian/Ubuntu（apt 系）的差异单独标注**——正好对应 A/B 两台机器的情况。

### 3.1 安装 kubeadm

#### 3.1.1 基础环境（所有机器都要做）

```bash
# 1. 每台机器设置各自 hostname
hostnamectl set-hostname k8s-master    # node 机器改成 k8s-node1 / k8s-node2

# 2. 关闭 swap（K8s 强制要求，不关 kubelet 起不来）
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab

# 3. 关闭 SELinux / 防火墙（测试环境图省事；生产按安全组/策略放行端口）
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
systemctl disable --now firewalld

# 4. 允许 iptables 检查桥接流量（CNI 网络的前提）
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

> **踩坑**：swap 忘了关是新手第一大坑——kubelet 报错 `running with swap on is not supported`，一堆人卡半天。第二条坑是桥接流量没开，表现为跨节点 Pod 不通、CoreDNS 起不来。

#### 3.1.2 安装 kubelet、kubeadm、kubectl（所有机器）

yum 系（阿里云源，国内直连官方源基本拉不动）：

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
EOF

# 三个组件版本必须一致；固定小版本，别装 latest（跨小版本可能有 API 变化）
sudo yum install -y kubelet-1.20.9 kubeadm-1.20.9 kubectl-1.20.9 --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

apt 系（Debian/Ubuntu）：

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet=1.20.9-00 kubeadm=1.20.9-00 kubectl=1.20.9-00
sudo systemctl enable --now kubelet
```

> 注意此时 kubelet 起不来是**正常的**（每几秒重启一次），它要等 kubeadm init/join 生成配置后才能正常工作。

### 3.2 使用 kubeadm 引导集群

#### 3.2.1 下载各个机器需要的镜像（所有机器）

镜像在官方 registry 拉不动，统一走阿里云镜像仓库：

```bash
sudo tee ./images.sh <<-'EOF'
#!/bin/bash
images=(
kube-apiserver:v1.20.9
kube-proxy:v1.20.9
kube-controller-manager:v1.20.9
kube-scheduler:v1.20.9
coredns:1.7.0
etcd:3.4.13-0
pause:3.2
)
for imageName in ${images[@]} ; do
  docker pull registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/$imageName
  docker tag  registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/$imageName k8s.gcr.io/$imageName
  docker rmi  registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/$imageName
done
EOF
chmod +x ./images.sh && ./images.sh
```

> 主节点需要全部镜像；node 节点实际只用到 `kube-proxy` 和 `pause`，但全下省心。

#### 3.2.2 初始化主节点（只在 master 执行）

```bash
# 所有机器：添加 master 域名映射（join 时要用域名，IP 变了也不用重发 token）
echo "172.31.0.4 k8s-master" >> /etc/hosts

# kubeadm 配置文件（指定国内镜像仓库 + Pod 网段）
cat > kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.20.9
imageRepository: registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images
podSubnet: 10.244.0.0/16
EOF

# 开始初始化（tee 存日志，失败了好排查）
kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs | tee kubeadm-init.log
```

成功后输出长这样，**把 join 命令复制存好**：

```text
Your Kubernetes control-plane has initialized successfully!

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s-master:6443 --token 3v6pmo.m2b7kpqz1gmhxk9f \
    --discovery-token-ca-cert-hash sha256:xxx...
```

> 高可用（多 master）的话：副本 ≥3 的**奇数**（3/5/7），后面的 join 加 `--control-plane --certificate-key` 参数。测试环境单 master 就够。

#### 3.2.3 根据提示继续

**① 设置 .kube/config**（master 上，让 kubectl 有权限）：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

此时 `kubectl get nodes` 会看到 master 是 **NotReady**——因为还没装网络组件。

**② 安装网络组件（Calico）**：

```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml

# 等 Calico 的 Pod 全部 Running（装好后节点自动变 Ready）
kubectl get pods -n kube-system -w
```

> **注意**：Calico 默认 Pod 网段和 kubeadm-config.yaml 里的 `podSubnet` 必须一致（calico.yaml 里的 `CALICO_IPV4POOL_CIDR`），不一致跨节点不通——这是第二常见的「装完集群 Pod 全 Pending」根因。用 Flannel 的话对应 `10.244.0.0/16`。

### 3.3 加入 node 节点（每个 node 执行）

把 init 输出的 join 命令原样贴到两台 node 上：

```bash
kubeadm join k8s-master:6443 --token 3v6pmo.m2b7kpqz1gmhxk9f \
    --discovery-token-ca-cert-hash sha256:xxx...
# This node has joined the cluster:
```

忘了存 join 命令？master 上随时重新生成：

```bash
kubeadm token create --print-join-command
```

> **注意**：node 上不需要 `.kube/config`，那是管理集群用的。node 的 kubelet 配置由 join 自动生成。

### 3.4 验证集群

```bash
kubectl get nodes
# NAME         STATUS   ROLES    AGE   VERSION
# k8s-master   Ready    master   10m   v1.20.9
# k8s-node1    Ready    <none>   2m    v1.20.9
# k8s-node2    Ready    <none>   2m    v1.20.9

kubectl get pods -A     # -A = 所有命名空间，确认全部 Running
```

**全部 Ready + 系统组件全 Running = 集群搭建完成。** 卡在 NotReady 就是网络组件没装好/镜像没拉到。

### 3.5 部署 Dashboard

#### ① 部署

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/recommended.yaml
# 国内拉不动就 wget 下来，把镜像改成 aliyun 的再 apply
```

#### ② 设置访问端口

默认 Service 是 ClusterIP，集群外访问不到，改成 NodePort：

```bash
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
#   type: ClusterIP  →  type: NodePort

kubectl get svc -n kubernetes-dashboard
# NAME                   TYPE       CLUSTER-IP     PORT(S)         AGE
# kubernetes-dashboard   NodePort   10.96.54.199   443:31245/TCP   1m
```

浏览器访问 `https://任意节点IP:31245`（自签证书，浏览器告警点继续即可）。

#### ③ 创建访问账号

Dashboard 默认没有可登录的 admin 账号，创建一个绑定最高权限的 ServiceAccount：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin        # 直接绑集群管理员角色
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

```bash
kubectl apply -f dash-account.yaml
```

#### ④ 令牌访问

```bash
# 1.24 之前的版本：直接从 Secret 里解出 token
kubectl -n kubernetes-dashboard get secret \
  $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}') \
  -o go-template="{{.data.token | base64decode}}"
```

> **纠偏**：1.24 起 ServiceAccount 不再自动创建带 token 的 Secret，上面这条命令会查不到东西。新版本用 `kubectl -n kubernetes-dashboard create token admin-user` 直接签发 token，或者自己建一个 `type: kubernetes.io/service-account-token` 的 Secret。

复制 token 贴到登录页 → 进入界面。

#### ⑤ 界面

左侧菜单：Overview（集群概览）、Nodes / Workloads（Pod、Deployment 等负载）、Config Maps / Secrets、Services。**可视化看资源、快速排障可以，日常操作还是 kubectl 为主**——生产环境 Dashboard 一般不开公网暴露，历史上出过未授权访问的大事故。

### 3.6 小结

> **一句话**：kubeadm 装集群 = 基础环境（关 swap/开桥接）→ 装三件套 → 拉镜像 → master init + 装 CNI → node join → 验证全 Ready。
>
> 三个核心记住不出大问题：
>
> 1. **kubelet 装完起不来是正常的**，init/join 之后才会正常
> 2. **网络组件（Calico/Flannel）不装，节点永远 NotReady**，且 CNI 网段要和 podSubnet 一致
> 3. **token 会过期（24h）**，过期后 `kubeadm token create --print-join-command` 重新生成

---

# 二、Kubernetes 核心实战

## 1. 资源创建方式

两种路子，对应两种哲学：

| 方式 | 命令 | 特点 |
|------|------|------|
| **命令式** | `kubectl run nginx --image=nginx` | 快，适合临时验证；但复杂参数写不动，也没法版本管理 |
| **声明式（yaml）** | `kubectl apply -f xxx.yaml` | 一切皆资源，yaml 可以进 git、可以复用、可以 review；**生产唯一正道** |

命令式感受一下就够：

```bash
kubectl run mynginx --image=nginx
kubectl delete pod mynginx
```

yaml基本语法

缩进不允许tab,只有空格

缩进空格数目不严格，同层级左侧对齐即可

#标识注释，到行尾都会被解释器忽略

数据结构

对象

数组

纯量 ：Null用~

字符串：默认格式

声明式的核心问题：**yaml 字段这么多，怎么知道写什么？** 答案不是查文档，是问集群自己：

```bash
kubectl explain pod            # pod 有哪些一级字段
kubectl explain pod.spec       # spec 下有什么
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.ports
```

> **注意**：写 yaml 时别抄网上旧文章——不同资源有 **apiVersion**（v1 / apps/v1 / batch/v1……），抄错了 `kubectl apply` 直接报错。不确定就用 explain 查当前版本的正确写法。

## 2. Namespace

命名空间 = 集群内的**逻辑分区**，用来隔离资源、划分环境（dev/test/prod）或团队。

集群自带 4 个：

| Namespace | 存放 |
|-----------|------|
| default | 不指定 ns 时资源默认落这 |
| kube-system | 系统组件（Calico、CoreDNS……） |
| kube-public | 公开资源，几乎不用 |
| kube-node-lease | 节点心跳租约 |

```bash
kubectl get ns                  # 列出所有命名空间
kubectl get pods -A             # 所有 ns 的 Pod（-A = --all-namespaces）
kubectl get pods -n kube-system # 指定 ns

# 命令式创建/删除
kubectl create ns hello
kubectl delete ns hello         # 删 ns 会连带删里面所有资源，慎用
```

yaml 方式：



```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hello
```

> **踩坑**：`kubectl get pods` 不带 `-n` 永远只看 default——「我 Pod 明明起来了怎么看不到」九成是没切 ns。资源被删排查思路第一条：先 `kubectl get ns` 确认 ns 本身还在不在。

## 3. Pod

### 3.1 概念

Pod 是 K8s 的**最小部署单元**（不是容器！）。一个 Pod 里装 1 个或多个容器，它们**共享网络（同 IP 同端口空间）、共享存储卷、共享 IPC/UTS**——像一个逻辑上的「超薄虚拟机」。

按管理方式分两类：

- **自主式 Pod**：直接创建的裸 Pod，`kubectl run` 出来的。**Pod 挂了没人管，不会自愈**，生产别用。
- **控制器管理的 Pod**：由 Deployment/StatefulSet 等控制器创建，挂了自动重建。**生产一律用控制器**，从不裸奔。

```bash
kubectl run mynginx --image=nginx          # 快速验证（自主式）
kubectl get pod -o wide                    # 看 IP、落在哪个节点
kubectl describe pod mynginx               # 排障第一步：事件看 Pull 镜像/调度失败原因
kubectl logs mynginx                       # 看日志
kubectl exec -it mynginx -- /bin/bash      # 进容器
```

### 3.2 Pod 的 yaml 模板（全字段带注释）

yaml必须携带一部分属性

```yaml
apiVersion: v1            # 版本，kubectl explain 可查
kind: Pod                 # 资源类型
metadata:                 # 元数据
  name: mynginx           # Pod 名
  labels:                 # 标签——给 Selector 用的，非常关键
    app: mynginx
spec:                     # 期望状态
  containers:             # 容器列表，一个 Pod 可多个容器
  - name: mynginx
    image: nginx          # 镜像，默认 :latest（生产必须固定 tag）
    ports:
    - containerPort: 80   # 声明端口（信息性质，不写也能访问）
    env:                  # 环境变量
    - name: ENV_TEST
      value: "hello"
    resources:            # 资源请求与上限
      requests:
        cpu: 100m         # 100m = 0.1 核
        memory: 100Mi
      limits:
        cpu: 200m
        memory: 200Mi
    livenessProbe:        # 存活探针：失败就重启容器
      httpGet:
        path: /
        port: 80
    readinessProbe:       # 就绪探针：失败就从 Service 摘除
      httpGet:
        path: /
        port: 80
  restartPolicy: Always   # Always（默认）/ OnFailure / Never
```

> 三个探针的区别记死：**liveness 挂了→重启；readiness 挂了→不接流量但不重启；startup（慢启动应用用）没通过前，前两个探针不生效**。

### 3.3 多容器 Pod：共享卷示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: sidecar            # 边车容器：和主容器同生共死、共享一切
    image: busybox
    command: ["sh", "-c", "while true; do echo hello > /data/index.html; sleep 5; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}             # 临时共享目录，Pod 删了就没
```

sidecar 往 `/data` 写文件，nginx 的 `/usr/share/nginx/html` 立刻可见——**同一个卷挂两个容器**，这就是「Pod 内共享」。

### 3.4 Pod 生命周期与 QoS

**五个阶段**：Pending（等调度/拉镜像）→ Running → Succeeded / Failed（退出）；Unknown（节点失联）。

**终止流程**（滚动更新不丢流量的关键）：标记 Terminating → **先从 Service 端点摘除** → 执行 PreStop 钩子 → 发 SIGTERM → 等 `terminationGracePeriodSeconds`（默认 30s）→ 超时 SIGKILL。

**QoS 三档**（决定节点资源紧张时谁先被驱逐）：

| QoS | 条件 | 被驱逐优先级 |
|-----|------|--------------|
| Guaranteed | 所有容器 limits = requests | 最低（最安全） |
| Burstable | 部分设置 requests/limits | 中 |
| BestEffort | 全没设置 | **最高（先赶走）** |

> 结论：**生产 Pod 必须写 requests/limits**，否则就是节点一紧张第一个被牺牲的。

以上是速览，下面按你占位笔记的四块逐个展开：**Pod Phase → Init 容器 → 探针 → 启动/退出钩子**。

#### 3.4.1 完整生命周期全景图

```
kubectl apply
    │
    ▼
Pending（写 etcd，等调度；镜像拉取也在这阶段）
    │  kube-scheduler 选好节点，kubelet 接手
    ▼
启动 Pause 容器（先立住网络/IPC 命名空间，Pod 的"骨架"）
    │
    ▼
Init 容器（多个则严格串行：C1 → C2 → ...，任何一个失败都进不了下一步）
    │  全部成功
    ▼
主容器启动 + postStart 钩子（异步触发，不保证先于 ENTRYPOINT）
    │
    ▼
探针接管：
    startupProbe（可选）没通过前，下面两个不生效
    → readinessProbe 通过，才挂到 Service 端点接流量
    → livenessProbe 持续体检，失败则重启容器
    │  收到删除指令（delete / 滚动更新 / 驱逐）
    ▼
Terminating → 先从 Service 端点摘除 → preStop 钩子（同步阻塞）
    → 发 SIGTERM → 等 terminationGracePeriodSeconds（默认 30s）→ 超时 SIGKILL
    │
    ▼
Succeeded / Failed（终态）；Unknown = 节点失联，状态未知
```

#### 3.4.2 Pod Phase（相位）

Phase 是 Pod 级别的粗粒度状态，只有 5 种：

| Phase | 含义 | 典型场景 |
|-------|------|---------|
| **Pending** | 已创建但容器没全跑起来 | 等调度、拉镜像、等 PV 绑定 |
| **Running** | 已绑定节点，容器已创建且至少一个在运行 | 正常服役中（含正在重启/重启中） |
| **Succeeded** | 所有容器成功终止且不会重启 | Job/CronJob 跑完（restartPolicy: Never/OnFailure） |
| **Failed** | 所有容器终止且至少一个失败 | 退出码非 0、被系统杀掉 |
| **Unknown** | 状态未知 | 节点失联（心跳超时） |

> **辨析**：`Running` ≠ 「能接流量」。Running 只表示容器进程在跑，能不能接流量由 readinessProbe 决定，健不健康由 livenessProbe 决定。`kubectl get pod` 看到一堆 Running 但服务不通，先看 `READY` 列是不是 `1/1`——`0/1` 就是没过就绪探针。

phase 太粗，排障时更常用**容器级三态**（`kubectl describe pod` 的 State 字段）：

| 容器状态 | 含义 | 常见原因 |
|---------|------|---------|
| Waiting | 等待启动 | 拉镜像中、CrashLoopBackOff、等 ConfigMap/Secret |
| Running | 正在运行 | 正常 |
| Terminated | 已终止 | 退出码、结束原因（OOMKilled / Error / Completed）都在这看 |

> **CrashLoopBackOff**：容器反复崩溃反复重启的退避状态，重启间隔指数拉长（10s → 20s → 40s … 上限 5min）。九成是应用起不来（配置错、依赖连不上、探针配太激进把健康进程杀成循环崩溃），用 `kubectl logs --previous` 看上一次崩溃的日志。

#### 3.4.3 Init 容器

主容器启动**前**跑完就退出的容器，做一次性初始化。三条铁律：

1. **串行执行**：多个 init 容器按声明顺序逐个跑，前一个成功才轮到下一个（和主容器并行启动完全相反）。
2. **必须全部成功**主容器才会启动；失败的 init 容器按 Pod 的 `restartPolicy` 处理——Always/OnFailure 会重试，Never 则整个 Pod 直接 Failed。
3. **不支持的特性**：没有 lifecycle 钩子、没有 readiness/liveness/startup 探针（它本来就是跑到完成即退出）。

**使用场景**：等依赖就绪（数据库能连了再启动应用）、生成配置/预置数据、把含敏感逻辑的初始化脚本和主镜像分离（主镜像不用装 curl/jq 这类工具）。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  initContainers:
  - name: wait-for-db          # 第一个：死等 mysql 能连通
    image: busybox
    command: ['sh', '-c', 'until nc -z mysql-svc 3306; do echo waiting; sleep 2; done']
  - name: init-config          # 第二个：往共享卷写初始化文件
    image: busybox
    command: ['sh', '-c', 'echo "init done" > /work/config/ready']
    volumeMounts:
    - name: workdir
      mountPath: /work/config
  containers:
  - name: myapp
    image: busybox
    command: ['sh', '-c', 'cat /work/config/ready && sleep 3600']
    volumeMounts:
    - name: workdir
      mountPath: /work/config
  volumes:
  - name: workdir
    emptyDir: {}
```

> init 容器和主容器共享整个 Pod 的网络与卷（volumes 在 Pod 级定义）——init 容器先往共享卷里放好的东西，主容器直接用。

#### 3.4.4 探针：三种探针 × 三种检测方式

**三种探针**（管的事完全不同，记死）：

| 探针 | 检测失败后的动作 | 本质问题 |
|------|----------------|---------|
| **livenessProbe** 存活检测 | **重启容器** | 进程还活着吗？（死锁、卡死但进程还在） |
| **readinessProbe** 就绪检测 | **从 Service 端点摘除**，不重启 | 能接流量了吗？（启动中、依赖断了、过载降级） |
| **startupProbe** 启动检测 | 通过前**屏蔽 liveness/readiness**，超时则重启容器 | 慢启动应用要多久才就绪？（大 JVM、老系统） |

> startupProbe 是给慢启动应用兜底的：以前只能把 liveness 的 initialDelaySeconds 调得很大，但那样真崩溃也要等很久才发现；有了 startupProbe，`failureThreshold: 30 × periodSeconds: 10` = 允许慢跑 5 分钟，一旦判定启动完成，liveness 立刻用短周期接管。

**三种检测方式**（每种探针任选一种）：

| 方式 | 原理 | 适用 |
|------|------|------|
| **httpGet** | 对容器 IP 发 HTTP GET，2xx/3xx 算成功 | 有 HTTP 服务的（最常用） |
| **tcpSocket** | 对指定端口建 TCP 连接，通了算成功 | 没有健康接口、只开端口的（数据库类） |
| **exec** | 在容器里执行命令，退出码 0 算成功 | 需要脚本判断的复杂逻辑 |

三合一完整示例：

```yaml
containers:
- name: myapp
  image: myapp:1.0
  startupProbe:               # ① 先等它启动完成（最多 30×10s = 5 分钟）
    httpGet:
      path: /healthz
      port: 8080
    failureThreshold: 30
    periodSeconds: 10
  livenessProbe:              # ② 存活体检：连续 3 次失败（共 ~45s）就重启
    httpGet:
      path: /healthz
      port: 8080
    periodSeconds: 15
    timeoutSeconds: 3
    failureThreshold: 3
  readinessProbe:             # ③ 就绪检查：连续 2 次失败就摘流量
    exec:                     # 检测方式换成 exec：跑命令看退出码
      command:
      - cat
      - /tmp/ready            # 上游把文件放好才算就绪（简易信号量玩法）
    periodSeconds: 5
    failureThreshold: 2
```

**通用调优参数**（三种探针都支持）：

| 参数 | 默认 | 说明 |
|------|------|------|
| initialDelaySeconds | 0 | 容器启动后等多久开始第一次探测 |
| periodSeconds | 10 | 探测周期 |
| timeoutSeconds | 1 | 单次探测超时（应用响应慢要调大，否则误判失败） |
| successThreshold | 1 | 连续成功几次算通过（readiness 摘除后恢复流量用它防抖） |
| failureThreshold | 3 | 连续失败几次判定失败 |

> **踩坑**：liveness 探针指向一个「要查数据库」的接口是经典事故——数据库抖 30 秒，探针把所有 Pod 重启一遍，故障被放大成雪崩。**liveness 只测进程自身健康**；依赖健康度交给 readiness（摘流量就够了，重启解决不了依赖问题）。

#### 3.4.5 启动/退出钩子：postStart / preStop

`lifecycle` 字段里的两个钩子，在容器启动/终止时插入自定义动作：

| 钩子 | 时机 | 同步性 |
|------|------|--------|
| **postStart** | 容器创建后立刻触发 | **异步**——和 ENTRYPOINT 并发执行，不保证先后顺序 |
| **preStop** | 收到终止信号**前**触发 | **同步阻塞**——执行完才发 SIGTERM |

```yaml
containers:
- name: nginx
  image: nginx
  lifecycle:
    postStart:                # 启动后：改个首页（不保证 nginx 已就绪，别干重活）
      exec:
        command: ["/bin/sh", "-c", "echo launched > /usr/share/nginx/html/index.html"]
    preStop:                  # 终止前：先优雅退出，再睡 10s 等端点摘除传播
      exec:
        command: ["/bin/sh", "-c", "nginx -s quit; sleep 10"]
```

两个关键差异必须记住：

1. **postStart 是异步的**：kubelet 发出事件后不等它执行完，钩子和主进程谁先跑赢不确定。要做「必须先于应用执行」的初始化，用 Init 容器，别用 postStart。
2. **preStop 是同步的**：kubelet 等它执行完才发 SIGTERM。这是它最大的价值——**给「从 Service 摘除」留缓冲**：Endpoint 变更从 apiserver 传导到所有节点的 iptables 有延迟，摘除后立刻杀容器会漏掉在途请求。滚动更新偶发 502，九成是没配 preStop sleep。

> preStop 的耗时计入 `terminationGracePeriodSeconds`（默认 30s）宽限期：preStop 10s + 应用退出 10s < 30s 才安全，超了直接 SIGKILL。需要更长优雅期就在 yaml 里调大 `spec.terminationGracePeriodSeconds`。

#### 3.4.6 小结

> **一句话**：Pod 生命周期 = Pause 骨架 → Init 串行铺路 → 主容器 + postStart → 三种探针接管（startup 屏蔽期 → readiness 管流量 → liveness 管重启）→ preStop + 宽限期优雅退出。
>
> 四个核心记住不出大问题：
>
> 1. **Running ≠ READY**，接不接流量看 readiness；排障先看容器三态（Waiting/Running/Terminated）
> 2. **liveness 只测自身健康**，别把依赖检查塞进去，否则依赖一抖全量重启
> 3. **postStart 异步、preStop 同步**——必须先跑的逻辑用 Init 容器；滚动更新 502 先补 preStop sleep
> 4. **Init 容器串行且必须全成功**，失败看 restartPolicy，Never 直接整个 Pod Failed

## 4. Deployment

Deployment 是最常用的控制器，管理无状态应用（web、api 这类）。它下面还有一层 ReplicaSet 帮它管副本：

```
Deployment（管版本：滚动更新/回滚）
    └── ReplicaSet（管副本数：保持 N 个）
          └── Pod × N（真正干活的）
```

### 4.1 多副本

```bash
kubectl create deployment my-dep --image=nginx --replicas=3
```

yaml 方式（生产写法）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-dep
spec:
  replicas: 3                     # 期望副本数
  selector:                       # 选择器：认领带这些标签的 Pod
    matchLabels:
      app: my-dep
  template:                       # Pod 模板：按这个造 Pod
    metadata:
      labels:
        app: my-dep
    spec:
      containers:
      - name: nginx
        image: nginx
```

```bash
kubectl apply -f deployment.yaml
kubectl get deploy,pod -o wide     # deploy 3/3 READY 即符合期望
```

> **踩坑**：`selector.matchLabels` 必须和 `template.labels` 匹配，不匹配直接拒绝创建。这不是格式问题——Deployment 就靠标签认领 Pod，标签对不上副本就失控。

### 4.2 扩缩容

```bash
# 方式一：scale 命令
kubectl scale deployment/my-dep --replicas=5
kubectl scale deployment/my-dep --replicas=2

# 方式二：改 yaml 再 apply（推荐，可追溯）
#   replicas: 5
kubectl apply -f deployment.yaml

# 自动扩缩容（HPA）→ 见进阶专题
```

### 4.3 自愈 & 故障转移

Deployment 的看家本领，两个实验直接感受：

```bash
# 实验 1：手动删 Pod —— 秒级重建
kubectl delete pod my-dep-xxx
kubectl get pods      # 新 Pod 立刻补上（名字变了，哈希后缀不同）

# 实验 2：模拟节点宕机 —— Pod 漂移
# 把某台 node 直接关机，等几分钟后：
kubectl get pods -o wide
# 该节点上的 Pod 先变成 Terminating，然后在新节点重建
```

原理就是控制循环：副本数 3 是期望状态，控制器发现实际是 2，就再造一个。**节点 NotReady 超时后（默认约 5 分钟）其上的 Pod 才会被驱逐重建**，不是瞬间漂移。

### 4.4 滚动更新

```bash
kubectl set image deployment my-dep nginx=nginx:1.16.1 --record
# --record 在 1.21+ 已废弃，新版本给资源加 annotation：
#   kubernetes.io/change-cause: "update nginx to 1.16.1"

# 观察过程
kubectl rollout status deployment/my-dep
# Waiting for deployment "my-dep" rollout to finish: 1 out of 3 new replicas...

kubectl get rs      # 关键：新旧两个 ReplicaSet 并存
# my-dep-6b5f9c8d4b   3     （新版 RS，副本 3）
# my-dep-77d8f6f9cb   0     （旧版 RS，副本 0，但没删！这就是能回滚的原因）
```

滚动策略（yaml 里控制）：

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # 最多多起多少（先加新的）
      maxUnavailable: 25%  # 最多少几个（再删旧的）
```

### 4.5 版本回退

```bash
kubectl rollout history deployment/my-dep    # 查历史版本
kubectl rollout undo deployment/my-dep       # 回滚到上一版
kubectl rollout undo deployment/my-dep --to-revision=2   # 回滚到指定版本
kubectl rollout pause deployment/my-dep      # 暂停滚动（多次改配置后再 resume 一次生效）
kubectl rollout resume deployment/my-dep
```

> **纠偏**：很多人以为回滚是「重新拉旧镜像跑一遍」，实际是**把旧 ReplicaSet 的副本数从 0 调回 N、新 RS 调回 0**——所以旧 RS 永远不删。也因此 `kubectl delete rs` 千万别手贱，删了就没得回滚了。

### Deployment 小结

> **一句话**：Deployment = 副本管理（ReplicaSet）+ 版本管理（滚动更新/回滚）双层封装，无状态应用的默认选择。
> 有状态（数据库类 DBMS）别硬上 Deployment，进阶专题的 StatefulSet 再聊。

## 5. Service

**为什么需要 Service**：Pod 的 IP 一旦重建就变，而且副本有多个——总不能让调用方记一串随时失效的 IP。Service 提供一个**稳定的访问入口 + 自带负载均衡**，靠标签选择器自动追踪下面的 Pod。

```
调用方 ──► Service（固定 IP / 固定 DNS 名）
              │  selector: app=my-dep   ←──┐
              ▼                             │ 谁带这个标签就转发给谁
         Pod1   Pod2   Pod3（IP 随便变，换了也无所谓）──┘
```

Service 的四连招：

```bash
kubectl expose deployment my-dep --port=8000 --target-port=80   # 命令式快速版
kubectl get svc
kubectl describe svc my-dep        # 看 Endpoints 是否绑上 Pod
kubectl delete svc my-dep
```

### 5.1 ClusterIP（默认类型）

集群内部的虚拟 IP，**只有集群内能访问**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP            # 不写 type 默认就是它
  selector:
    app: my-dep
  ports:
  - port: 8000               # Service 暴露的端口
    targetPort: 80           # 转发到 Pod 的哪个端口
    protocol: TCP
    name: http
```

集群内两种访问方式：

```bash
# 1. Service IP（重启不变，但新建 Service 会变）
curl 10.96.54.199:8000

# 2. DNS 名（推荐，CoreDNS 提供）
#   格式：服务名.命名空间.svc.cluster.local
curl my-service.default.svc.cluster.local:8000
#   同命名空间内直接用服务名：curl my-service:8000
```

特殊变体 **Headless Service**（`clusterIP: None`）：不分配虚拟 IP，DNS 直接解析到每个 Pod 的 IP——客户端自己挑 Pod，StatefulSet 专属配置，进阶专题见。

### 5.2 NodePort

在每个节点上开一个真实端口（**30000-32767**），外部通过 `节点IP:端口` 访问：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: my-dep
  ports:
  - port: 8000
    targetPort: 80
    nodePort: 31000      # 不写就随机分配 30000-32767
```

```bash
curl 172.31.0.4:31000    # master
curl 172.31.0.5:31000    # node1 —— 任意节点都行，kube-proxy 把流量导向真实 Pod
```

> 任何节点都能访问是因为 **kube-proxy 在每个节点都写了 iptables/ipvs 规则**——请求到了 node2，即使 Pod 跑在 node1，也会被转发过去。
>
> **NodePort 的痛点**：端口范围受限（30000+，没法用 80/443）、访问入口是 IP:Port 没法按域名/路径分流、没有 HTTPS 卸载——这些正是 Ingress 要解决的。

## 6. Ingress

Service 是**四层**（TCP/UDP）负载均衡；Ingress 是**七层**（HTTP/HTTPS）：按**域名、路径**转发，还能做 TLS 卸载、限流、重写。注意 Ingress 资源本身只是一份规则，真正干活的是 **Ingress Controller**（常用 ingress-nginx），得先装。

```
浏览器 ──► Ingress Controller（Pod，NodePort 暴露）
              │  按 host / path 分流
              ├── hello.atguigu.com/  ──► hello-service:8000
              └── demo.atguigu.com/nginx ──► nginx-service:8000
```

### 6.1 安装

```bash
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.47.0/deploy/static/provider/baremetal/deploy.yaml

# 1. 把镜像改成国内源（yaml 里搜 image:）
#    registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/ingress-nginx-controller:v0.46.0
# 2. apply
kubectl apply -f deploy.yaml
# 3. 老版本集群有 admission webhook 的 bug，直接删掉：
kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission
# 4. Service 改 NodePort，方便外部访问
kubectl edit svc ingress-nginx-controller -n ingress-nginx
#   type: NodePort（并固定端口 nodePort: 30080 / 30443）
```

### 6.2 使用

准备两个后端服务（hello-server、nginx），然后写 Ingress 规则：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-host-bar
spec:
  ingressClassName: nginx
  rules:
  - host: "hello.atguigu.com"          # 按域名分流
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: hello-server
            port:
              number: 8000
  - host: "demo.atguigu.com"
    http:
      paths:
      - pathType: Prefix
        path: "/nginx"                  # 域名 + 路径双重分流
        backend:
          service:
            name: nginx-service
            port:
              number: 8000
```

### 6.3 测试环境

#### ① 域名访问

本机没有 DNS，直接改 hosts 伪造域名（Windows：`C:\Windows\System32\drivers\etc\hosts`）：

```text
172.31.0.4 hello.atguigu.com
172.31.0.4 demo.atguigu.com
```

```bash
curl hello.atguigu.com:30080    # 命中 hello-server
curl demo.atguigu.com:30080     # 命中 nginx-service
```

#### ② 路径重写

问题：`demo.atguigu.com/nginx` 转给 nginx-service 时，路径原样带着 `/nginx` 前缀，nginx 容器里没有这个路径 → 404。用注解重写：

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: "demo.atguigu.com"
    http:
      paths:
      - pathType: Prefix
        path: "/nginx(/|$)(.*)"       # 正则捕获：/nginx 后面的部分
        backend: ...
```

效果：`/nginx` → `/`，`/nginx/a.html` → `/a.html`。

#### ③ 流量限制

还是注解，一行限流：

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "1"    # 每秒 1 个请求
```

压测一下，超限的直接返回 **503**。

> Ingress 注解（annotations）是个宝库：限流、重写、超时、跨域、TLS 都是一行注解的事，需要什么查 ingress-nginx 官方注解文档。

## 7. 存储抽象

### 7.1 为什么需要存储抽象

容器天然「用完即弃」：Pod 删了、漂移到别的节点了，写在容器里的数据全没了。裸挂载（hostPath）又会绑死节点——**Pod 换机器，数据留在原机器**。

K8s 的解法是把「存储的提供」和「存储的使用」拆开：

```
运维：造好一批 PV（存储池，比如 NFS 的三个目录）
                          ▲
                          │ 绑定（按容量/accessMode 自动匹配）
使用方：写 PVC（申请单：我要 5M 空间）
                          ▲
                          │ 引用
Pod：像用普通 volume 一样用 PVC，不关心背后是什么存储
```

| 概念 | 角色 | 类比 |
|------|------|------|
| **PV**（PersistentVolume） | 存储本身，集群级资源，运维创建 | 机房里的硬盘 |
| **PVC**（PersistentVolumeClaim） | 存储申请单，用户创建 | 「给我一块 5M 的盘」工单 |
| **StorageClass** | 动态供给，PVC 申请时自动造 PV | 自动化工单系统 |

### 7.2 环境准备（NFS 服务器）

实验用 NFS 做共享存储（生产换 Ceph/云盘，思路完全一样）。

**① 所有节点**：安装 NFS 客户端

```bash
yum install -y nfs-utils     # Debian: apt-get install -y nfs-common
```

**② 主节点**：作为 NFS Server，导出目录

```bash
mkdir -p /nfs/data
# insecure：允许非特权端口；no_root_squash：root 保留 root 权限
echo "/nfs/data/ *(insecure,rw,sync,no_root_squash)" > /etc/exports
systemctl enable rpcbind --now
exportfs -r
```

**③ 从节点**：验证并挂载（挂载只是为了手动测试，Pod 用 Pod 内挂载，见下）

```bash
showmount -e 172.31.0.4        # 应输出 /nfs/data *
mkdir -p /nfs/data
mount -t nfs 172.31.0.4:/nfs/data /nfs/data
```

### 7.3 原生方式数据挂载

不搞 PV/PVC，直接在 Pod 里指定 NFS：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis-nfs
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - name: data
      mountPath: /data          # redis 的落盘目录
  volumes:
  - name: data
    nfs:
      server: 172.31.0.4        # 写死了存储细节
      path: /nfs/data/redis
```

这样能跑，但问题明显：**每个用存储的 Pod 都得写 NFS 地址和路径**，存储换了（NFS → Ceph）所有 yaml 全得改。所以生产用下面的 PV/PVC 抽象层。

### 7.4 PV & PVC

#### 7.4.1 创建 PV 池（主节点：先建目录，再声明 PV）

```bash
mkdir -p /nfs/data/01 /nfs/data/02 /nfs/data/03
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv01-10m
spec:
  capacity:
    storage: 10M                 # 这个 PV 有 10M
  accessModes:
    - ReadWriteMany              # 多节点可读写（NFS 支持；块存储一般只有 ReadWriteOnce）
  storageClassName: standard     # 分组名，PVC 按这个名字匹配
  nfs:
    server: 172.31.0.4
    path: /nfs/data/01
---
# pv02-20m → /nfs/data/02（20M）、pv03-30m → /nfs/data/03（30M），同构再来两份
```

```bash
kubectl apply -f pv.yaml
kubectl get pv
# NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
# pv01-10m   10M        RWX            Retain           Available
```

#### 7.4.2 PVC 创建与绑定

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: standard
  resources:
    requests:
      storage: 5M                # 申请 5M
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc
# NAME        STATUS   VOLUME     CAPACITY
# nginx-pvc   Bound    pv01-10m   10M       # 绑上了：5M 的申请 → 10M 的 pv01
```

> **绑定规则**：storageClassName 一致 + accessModes 兼容 + **容量 ≥ 申请值**，挑满足条件里最小的。没有满足的 PV → PVC 一直 `Pending`。
>
> **RECLAIM POLICY 注意**：`Retain`（默认）删 PVC 后 PV 变 Released 不释放给别人；`Delete` 删 PVC 连 PV 一起删。测试环境用 Delete 爽，**生产数据库类一律 Retain**，防止误删数据。

Pod 里用 PVC（和普通 volume 长得几乎一样）：

```yaml
volumes:
- name: html
  persistentVolumeClaim:
    claimName: nginx-pvc
```

### 7.5 ConfigMap

配置和镜像解耦：改配置不用重新打镜像，且**支持热更新**。

#### ① 把配置文件创建为配置集

```bash
echo "appendonly yes" > redis.conf
kubectl create configmap redis-conf --from-file=redis.conf

kubectl get cm redis-conf -o yaml
# apiVersion: v1
# data:
#   redis.conf: |
#     appendonly yes
# kind: ConfigMap
```

#### ② 创建 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    command: ["redis-server"]
    args: ["/etc/redis/redis.conf"]     # 指定配置文件启动
    volumeMounts:
    - name: config
      mountPath: /etc/redis
  volumes:
  - name: config
    configMap:
      name: redis-conf
      items:
      - key: redis.conf        # configmap 里的 key
        path: redis.conf       # 挂出来的文件名
```

#### ③ 检查默认配置

```bash
kubectl exec -it redis -- cat /etc/redis/redis.conf
# appendonly yes

kubectl exec -it redis -- redis-cli
127.0.0.1:6379> CONFIG GET appendonly        # 运行态确认
# 1) "appendonly"
# 2) "yes"
```

#### ④ 修改 ConfigMap

```bash
kubectl edit cm redis-conf
# 把 appendonly yes 改成 appendonly no
```

#### ⑤ 检查配置是否更新

```bash
# 文件确实变了（ConfigMap 热更新特性，约 1 分钟内同步）
kubectl exec -it redis -- cat /etc/redis/redis.conf
# appendonly no

# 但运行态的 redis 没变！
kubectl exec -it redis -- redis-cli
127.0.0.1:6379> CONFIG GET appendonly
# 2) "yes"        ← redis 并不会自己 reload 配置文件
```

> **两个大坑**，都在上面这个实验里：
>
> 1. **热更新只更新文件，不会让应用 reload**——文件是新的，进程还在跑旧配置。要么应用支持 watch（如 nginx 需要配 nginx -s reload 的 sidecar），要么滚动重启 `kubectl rollout restart`。
> 2. **subPath 挂载不热更新**——`mountPath: /etc/redis/redis.conf` + `subPath: redis.conf` 这种「挂单个文件」写法，文件内容永远不更新。要热更新就挂目录。

### 7.6 Secret

和 ConfigMap 结构、用法几乎一样，区别是：**存敏感数据**（密码、token、证书），值是 **base64 编码**存储，且 Pod 里挂载成 tmpfs（内存）。

> **纠偏**：base64 是**编码不是加密**（`echo xxx | base64 -d` 秒解），Secret 的安全边界主要在「权限控制 + 不落盘 + 不进 yaml 明文」，不是保密强度。

```bash
# 命令行创建（--from-literal）
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=admin123

# 查看（data 值是 base64）
kubectl get secret db-secret -o yaml
# data:
#   password: YWRtaW4xMjM=
# 解码验证：
echo -n 'YWRtaW4xMjM=' | base64 -d    # admin123
```

yaml 方式（`stringData` 写明文，提交时自动转 base64，比手算省事）：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:                # 直接写明文
  username: admin
  password: "123456"
```

**用法一：环境变量注入**

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

**用法二：文件挂载**

```yaml
volumes:
- name: secret-vol
  secret:
    secretName: db-secret
# 容器里 /etc/secret/db-secret/password 文件内容就是明文密码，容器删除即消失
```

### ConfigMap vs Secret

| 项目 | ConfigMap | Secret |
|------|-----------|--------|
| 存什么 | 普通配置（nginx.conf、yml） | 敏感数据（密码、token、证书） |
| 值的格式 | 明文 | base64 |
| 热更新 | 支持（subPath 除外） | 挂载方式支持，env 方式不支持 |
| 落盘 | etcd + 可挂载 | etcd（可配置加密）+ tmpfs 挂载 |

### 存储小结

> **一句话**：K8s 把存储拆成「运维提供 PV / 用户申请 PVC / Pod 无感使用」三层，配置类数据走 ConfigMap（明文、热更新）、敏感数据走 Secret（base64、tmpfs）。
>
> 三个核心记住不出大问题：
>
> 1. **PVC Pending = 没有匹配的 PV**（storageClassName / accessModes / 容量三对照）
> 2. **ConfigMap 热更新只改文件，不 reload 进程**；subPath 单文件挂载连文件都不更新
> 3. **数据库类 PV 的回收策略用 Retain**，误删 PVC 也保得住数据

---

# 三、进阶专题

> 前面是「入门主线」，以下四个专题是原笔记保留内容：HPA（自动扩缩容）、SQL 数据库上 K8s 的可行性评估、StatefulSet（有状态应用，对应主线的存储抽象进阶）、DaemonSet 与 CronJob。配合主线食用：Deployment/Service 学完看 HPA，存储抽象学完看 StatefulSet。

## 1. HPA 水平扩展的实现

HPA（Horizontal Pod Autoscaler）的核心是一个基于控制循环的“指标-计算-调整”闭环系统，由控制器、指标 API 和稳定性机制共同构成。

### 1.1 核心控制循环

HPA 控制器运行在 `kube-controller-manager` 中，遵循标准控制循环：

1. **循环周期**：默认每 **30 秒**轮询一次。
2. **获取指标**：通过 Metrics API 从聚合层拉取目标工作负载的当前指标值。
3. **计算期望副本数**：根据当前指标值和目标值计算理想 Pod 数量。
4. **执行调整**：与当前实际副本数比较，若超出容忍范围，则修改 ReplicaSet 副本数。

### 1.2 扩缩容算法

**期望副本数 = ceil [ 当前副本数 × ( 当前指标值 / 期望指标值 ) ]**

**示例**：目标 CPU 利用率 50%，当前 4 个 Pod，平均利用率 80%。  
计算：`ceil[4 × (80% / 50%)] = ceil(6.4) = 7`，HPA 将副本数从 4 调整到 7。

### 1.3 指标来源

| 类型       | API 来源                  | 提供者                | 用途                         |
| :--------- | :------------------------ | :-------------------- | :--------------------------- |
| 资源指标   | `metrics.k8s.io`          | Metrics Server        | CPU、内存使用率              |
| 自定义指标 | `custom.metrics.k8s.io`   | Prometheus Adapter 等 | QPS、延迟、连接数等业务指标  |
| 外部指标   | `external.metrics.k8s.io` | 云服务商适配器        | 消息队列长度、数据库连接数等 |

### 1.4 稳定性机制（防止抖动）

- **容忍度（Tolerance）**：默认 `0.1`（10%）。仅当 `|当前值/目标值 - 1.0| > 容忍度` 时才触发扩缩容。
- **稳定窗口（Stabilization Window）**：记录一段时间内的扩缩容建议，选择最保守（副本数最少）的建议执行。默认缩容 5 分钟，扩容 3 分钟。
- **扩缩容策略（Scaling Policies）**：自定义扩缩容速率，如“每 60 秒最多扩容 4 个 Pod”。

### 1.5 扩展与增强实现

- **KEDA**：将 HPA 作为执行层，自身作为指标适配器，支持缩容到零。
- **Prometheus Adapter**：将 Prometheus 查询结果转换为自定义指标。
- **云厂商增强**：
  - GKE：Performance HPA profile，扩缩容速度提升 2 倍，支持 1000 个 HPA 对象。
  - TKE/CCE：提供更丰富指标（硬盘、网络、GPU）和 CronHPA 定时策略。

### 1.6 总结

HPA 由控制循环、比例算法、多源指标和稳定性机制构成。通过 KEDA、Prometheus Adapter 及云厂商增强，可适应从简单资源监控到复杂事件驱动的各类场景。

---

## 2. Kubernetes 运行 SQL 数据库的可行性

**结论**：可以稳定运行，但必须采用正确姿势——依赖 StatefulSet、成熟 Operator 以及精心设计的存储与网络方案。

### 2.1 稳定性验证：混沌工程

- 对 KubeDB 管理的 MySQL Group Replication 集群执行 **80 多项混沌实验**（Pod 驱逐、节点宕机、网络分区、OOMKill 等），**全部通过，零数据丢失、零脑裂**。
- 学术评估表明，PostgreSQL Operator 在多区域 K8s 集群中能保持一致性地水平扩展、自主故障转移，并支撑生产级部署。

### 2.2 稳定运行的核心支柱

1. **StatefulSet**：提供稳定的 Pod 身份、固定名称、有序启动/停止、一一绑定的 PVC。
2. **Operator**：封装 DBA 运维知识，自动处理备份、扩缩容、故障转移。常见：KubeDB、Percona Operator、CloudNativePG。
3. **存储与网络**：
   - PVC 设置 `reclaimPolicy: Retain`，避免误删数据。
   - 选择高性能 StorageClass（NVMe 本地卷或高性能云盘）。
   - 使用 Headless Service 和稳定 DNS 策略。

### 2.3 性能代价与适用边界

- **性能损耗**：MySQL 在 K8s 中 TPS 下降约 5.6%，平均延迟增加 6.3%，P99 增加 15.6%。PostgreSQL 写吞吐量可能下降高达 72%，而 MySQL PXC 反而提升 37.3%，取决于存储配置。
- **不适合场景**：
  - 超低延迟场景（高频交易、实时竞价）。
  - 严格合规要求物理硬件隔离。
  - 核心交易库初期建议先用虚拟机跑稳。

### 2.4 行动建议

1. 选择成熟 Operator。
2. 从非核心业务起步（开发、测试、读副本）。
3. 投资高性能存储与完整监控。
4. 上生产前执行混沌测试（Chaos Mesh 等）。

---

## 3. StatefulSet 场景与详解

StatefulSet 专为有状态应用设计，为每个 Pod 提供稳定、唯一的网络标识和持久化存储，并保证有序部署、伸缩和更新。

> 服务分类复习：**无状态服务**（LVS、Apache）→ Deployment；**有状态服务**（DBMS、MQ）→ StatefulSet。

### 3.1 核心对比：StatefulSet vs Deployment

| 维度     | Deployment       | StatefulSet                            |
| :------- | :--------------- | :------------------------------------- |
| 适用场景 | 无状态应用       | 有状态应用（MySQL、Kafka、etcd）       |
| Pod 身份 | 随机名称和 IP    | 固定名称（`web-0`、`web-1`）和稳定 DNS |
| 存储     | 共享 PVC 模板    | 每个 Pod 独立 PVC，Pod 重建后存储跟随  |
| 扩缩容   | 并行创建/删除    | 按序号串行（0→1→2 或逆序）             |
| 服务发现 | Service 负载均衡 | Headless Service，每个 Pod 独立 DNS    |
| 更新策略 | 并行滚动更新     | 支持分区更新，控制更新节奏             |

**选择原则**：能用 Deployment 解决的，不要用 StatefulSet。

### 3.2 三大核心机制

#### 3.2.1 稳定的网络标识：Headless Service

- 需要关联 `clusterIP: None` 的 Service。
- Pod 名称：`<StatefulSet名称>-<序号>`，如 `mysql-0`。
- DNS 格式：`<Pod名称>.<Headless Service名称>.<命名空间>.svc.cluster.local`。
- Pod 重建后继承原有名称，DNS 自动解析到新 IP。

#### 3.2.2 稳定的持久化存储：volumeClaimTemplates

- 为每个 Pod 自动生成独立 PVC，命名 `<模板名称>-<Pod名称>`。
- Pod 删除时 PVC 保留，并自动关联到相同序号的新 Pod。
- 删除 StatefulSet 默认不删除关联存储卷。

#### 3.2.3 有序部署与伸缩

- **创建/扩容**：按 `0 → 1 → 2 → ... → N-1` 串行创建，前一个 Ready 后才创建下一个。
- **删除/缩容**：按 `N-1 → ... → 1 → 0` 逆序终止。
- **更新**：默认 `OrderedReady`，从最高序号逐个更新。

### 3.3 典型应用场景

- 数据库集群：MySQL、PostgreSQL、MongoDB。
- 分布式存储：Elasticsearch、Cassandra、Ceph。
- 消息队列：Kafka、RabbitMQ。
- 协调服务：ZooKeeper、etcd。
- 模拟虚拟机：需要持久状态的类虚拟机工作负载。

### 3.4 实践示例：MySQL 主从集群

```yaml
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
    - port: 3306
      name: mysql
  clusterIP: None
  selector:
    app: mysql
---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

**部署后结构**：

| Pod 名称  | DNS 域名                                  | PVC 名称             |
| :-------- | :---------------------------------------- | :------------------- |
| `mysql-0` | `mysql-0.mysql.default.svc.cluster.local` | `mysql-data-mysql-0` |
| `mysql-1` | `mysql-1.mysql.default.svc.cluster.local` | `mysql-data-mysql-1` |
| `mysql-2` | `mysql-2.mysql.default.svc.cluster.local` | `mysql-data-mysql-2` |

### 3.5 重要限制与注意事项

- Headless Service 必须手动创建。
- 删除/缩容时 PVC 和 PV 保留，需手动管理回收。
- 删除 StatefulSet 不保证有序终止，应先缩容到 0。
- 滚动更新可能因某个 Pod 无法就绪而卡住。
- `terminationGracePeriodSeconds` 不应设为 0。

---

## 4. DaemonSet 与 CronJob 详解

### 4.1 DaemonSet：节点级守护进程

#### 4.1.1 核心定义

确保所有（或部分）节点上运行一个 Pod 副本。新节点加入时自动创建，节点移除时自动回收。

#### 4.1.2 典型使用场景

| 场景         | 示例                                    |
| :----------- | :-------------------------------------- |
| 日志收集     | Fluentd、Filebeat、Logstash             |
| 节点监控     | Prometheus Node Exporter、Datadog Agent |
| 网络插件     | Calico、Flannel、Cilium、kube-proxy     |
| 存储守护进程 | Ceph OSD、GlusterFS                     |
| 安全代理     | Falco、Aqua Security                    |
| 节点配置管理 | 自动调整内核参数、挂载设备              |

#### 4.1.3 核心工作机制

- **Pod 创建与节点绑定**：控制器监听 Node 和 Pod 事件，为每个符合条件的节点创建 Pod，通过 `nodeAffinity` 绑定。
- **目标节点选择**：`nodeSelector`、`affinity`、`tolerations`。控制器自动添加常见污点容忍（not-ready、unreachable、disk-pressure 等），但自定义污点需显式配置。
- **更新策略**：
  - `OnDelete`：手动删除旧 Pod 后创建新 Pod。
  - `RollingUpdate`（默认）：自动滚动更新，可配置 `maxUnavailable`、`maxSurge`。

#### 4.1.4 YAML 示例：Node Exporter

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          ports:
            - containerPort: 9100
              hostPort: 9100
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
            limits:
              cpu: 200m
              memory: 200Mi
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
```

#### 4.1.5 重要限制与最佳实践

- Pod 无稳定身份，不适合有状态应用。
- 必须设置资源请求/限制。
- 避免运行重型应用。
- 注意控制平面节点默认有 `NoSchedule` 污点。
- 删除 DaemonSet 会删除所有 Pod，但不会删除节点数据。

### 4.2 CronJob：定时任务

#### 4.2.1 核心定义

基于 Cron 表达式周期性创建 Job，每个 Job 创建一个或多个 Pod 执行一次性任务。

#### 4.2.2 典型使用场景

| 场景       | 示例                    |
| :--------- | :---------------------- |
| 数据库备份 | 每天凌晨 2 点备份 MySQL |
| 报表生成   | 每周一生成业务报表      |
| 数据清理   | 每小时清理过期日志      |
| 邮件推送   | 每天定时发送营销邮件    |
| 同步任务   | 每 5 分钟同步外部数据   |
| 证书续期   | 定期检查并续期 TLS 证书 |

#### 4.2.3 核心字段详解

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Shanghai"       # 1.27+ 稳定
  concurrencyPolicy: Forbid
  suspend: false
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 60
  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 600
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: mysql:8.0
              command:
                - /bin/sh
                - -c
                - "mysqldump -h $DB_HOST -u $DB_USER -p$DB_PASS mydb > /backup/mydb-$(date +%Y%m%d).sql"
              env:
                - name: DB_HOST
                  value: "mysql.default.svc.cluster.local"
                - name: DB_USER
                  value: "root"
                - name: DB_PASS
                  valueFrom:
                    secretKeyRef:
                      name: mysql-secret
                      key: password
```

#### 4.2.4 Cron 表达式格式

标准 5 字段：`分钟 小时 日 月 星期`

- 支持 `*`、`,`、`-`、`/`、`?`
- 宏：`@hourly`、`@daily`、`@weekly`、`@monthly`、`@yearly`
- 注意：日和星期同时指定时为 **OR** 关系。

#### 4.2.5 并发策略 `concurrencyPolicy`

| 策略            | 行为                        |
| :-------------- | :-------------------------- |
| `Allow`（默认） | 允许并发运行                |
| `Forbid`        | 跳过本次调度                |
| `Replace`       | 取消当前 Job，用新 Job 替换 |

#### 4.2.6 错过调度处理

- 未设置 `startingDeadlineSeconds`：尝试补跑，超过 100 次则不再调度。
- 设置 `startingDeadlineSeconds`：仅在截止时间内的错过才补跑。

#### 4.2.7 历史清理

- `successfulJobsHistoryLimit`：默认 3。
- `failedJobsHistoryLimit`：默认 1。
- `ttlSecondsAfterFinished`：完成后自动删除。

#### 4.2.8 重要限制与最佳实践

- 任务必须幂等。
- 设置超时 `activeDeadlineSeconds`。
- 设置资源请求/限制。
- 显式设置时区。
- `restartPolicy` 必须为 `Never` 或 `OnFailure`。
- 监控 Job 状态。
- 最小粒度是分钟，不支持秒。

### 4.3 四种工作负载控制器怎么选

| 维度     | DaemonSet                                 | CronJob                                  |
| :------- | :---------------------------------------- | :--------------------------------------- |
| 目标     | 每个节点一个 Pod，持续运行                | 按时间周期创建一次性 Job                 |
| 生命周期 | 长期运行，随节点存在                      | 任务完成即退出                           |
| 副本控制 | 由节点数量决定                            | 由 Cron 表达式决定                       |
| Pod 身份 | 随机名称，无稳定身份                      | 每次创建新 Job，Pod 随机                 |
| 典型用途 | 日志、监控、网络、存储守护进程            | 备份、报表、清理、同步                   |
| 更新策略 | OnDelete / RollingUpdate                  | 更新 CronJob 对象本身                    |
| 资源影响 | 随节点数线性增长                          | 按调度周期间歇性消耗                     |
| 关键配置 | nodeSelector、tolerations、updateStrategy | schedule、concurrencyPolicy、jobTemplate |

**选择原则**：

- 每个节点都跑常驻进程 → DaemonSet
- 定时执行批处理任务 → CronJob
- 固定副本数无状态服务 → Deployment
- 稳定身份和存储的有状态服务 → StatefulSet

---

> **全文一句话总结**：K8s = 「声明式 API + 控制循环」的集群操作系统——kubeadm 装好集群，Deployment 管无状态负载，Service/Ingress 管流量入口，PV/PVC/ConfigMap/Secret 管数据与配置，四大控制器（Deployment/StatefulSet/DaemonSet/CronJob）覆盖所有工作负载形态。
