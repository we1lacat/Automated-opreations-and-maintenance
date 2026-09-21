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
5. Service（ClusterIP / NodePort / LoadBalancer / ExternalName / kube-proxy 两模式）
6. Ingress（安装 / 使用 / 域名访问 / 路径重写 / 流量限制）
7. 存储抽象（环境准备 / 原生挂载 / PV&PVC / StorageClass / ConfigMap / Secret）
8. 集群调度（调度过程 / 指定节点 / 亲和性 / 污点容忍）
9. kubectl 排障工具箱（命令总表 / 三板斧 / 症状速查）
10. 资源治理（PDB / ResourceQuota / LimitRange）

**三、进阶专题**

1. HPA 水平扩展（控制循环 / 扩缩算法 / 稳定性机制）
2. Kubernetes 运行 SQL 数据库的可行性
3. StatefulSet（稳定身份 / Headless DNS / volumeClaimTemplates / 有序启停）
4. Job / DaemonSet / CronJob（节点守护 / 一次性任务 / 定时调度）
5. 安全：认证 / 鉴权 / 准入（RBAC 详解 / binding 拓扑 / VAP+CEL / 资源隔离实操）
6. Helm 与生态组件（Chart·Release·Repo / Helm3 无 Tiller / Dashboard / Prometheus / EFK）
7. 运维专题（kubeadm 证书续期 / etcd 备份）

---

# 一、Kubernetes 基础概念

## 1. 是什么

Kubernetes 这个词来自希腊语，意思是「舵手」或「领航员」，简称 K8s（K 和 s 之间 8 个字母）。它是 Google 2014 年开源的容器编排平台，前身是谷歌内部 Borg 系统

演进链条：

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
| **服务发现 & 负载均衡** | 用 Service 名称访问一组 Pod，自带负载均衡，不用自己 nginx分流 |
| **自动发布 / 回滚** | 滚动更新不停机，发布炸了轻松命令回滚 |
| **密钥和配置管理** | ConfigMap / Secret 热更新配置，不用重新打镜像 |
| **存储编排** | 本地盘、NFS、云盘统一抽象成 PV，随 Pod 挂载 |

### 和 Swarm 的关系（快速对齐）

| 项目 | Docker Swarm | Kubernetes |
|------|--------------|------------|
| 出身 | Docker 自带 | Google Borg 开源 |
| 门槛 | 低，会 docker 就行 | 高，概念繁杂 |
| 生态 | 几乎停滞 | CNCF 全家桶，事实标准 |
| 适合 | 小团队、内部工具 | 中大型、长期演进的项目 |

> **新项目学/用 K8s**。Swarm 的价值是帮你理解「编排」这个概念（跨机调度 + 自愈 + 滚动更新），概念完全通用。

---

## 2. 架构

### 2.1 工作方式

K8s 是典型的 **Master-Worker** 架构 + **声明式 API** + **控制循环**：

- **声明式**：不必告诉它「启动容器」，只是提交一份期望状态——「我要 3 个 nginx 副本」。至于怎么达成、挂了怎么补，K8s 自己想办法。
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

集群的大脑，管决策不管干活。**生产推荐奇数台（3/5/7），副本数 ≥3**——1 台单点，2 台脑裂，和 Swarm 的 manager 相似。

先给一份面试口诀版：

| 组件 | 一句话口诀 |
|------|-----------|
| **kube-apiserver** | 所有服务访问统一入口 |
| **kube-controller-manager** | 维持副本期望数目 |
| **kube-scheduler** | 负责调度任务，选择合适的节点分配任务 |
| **etcd** | 键值对数据库，储存 K8s 集群所有重要信息（持久化） |
| **cloud-controller-manager** | 和云厂商 API 交互（云 LB、节点、路由） |

具体为：

1. **kube-apiserver**：集群**唯一入口**，所有组件交互都要经过它；提供 REST API，负责认证、鉴权、准入控制；**唯一可以直接读写 etcd 的组件**。kubectl、Dashboard、各种 SDK，最终都是在调它。
2. **etcd**：分布式 KV 数据库，集群**所有元数据**（Pod、Service、ConfigMap……你 apply 的一切）都存在这。etcd 挂了且没备份 = 集群整个没了，**务必备份**。
3. **kube-scheduler**：调度器。只干一件事：监控新创建且未绑定节点的 Pod，跑「预选（过滤不满足条件的节点）→ 优选（打分挑最优）」两轮算法，把 Pod 绑定到某节点。**它不启动容器**，只写决策。
4. **kube-controller-manager**：控制器管理器，内部装了一堆控制器（副本、节点、端点、Job……），每个都在跑控制循环，对比期望 vs 实际，不断调谐。
5. **cloud-controller-manager**：把「节点、负载均衡、存储卷」这类操作委托给云厂商 API。自建裸机集群用不到。

#### 2.2.2 Node 组件

工作机上的组件：

1. **kubelet**：运行在每个节点上的节点代理。直接跟容器引擎（containerd/docker）交互实现**容器生命周期管理**——创建、启动、监控、销毁，同时把节点和 Pod 状态上报给 apiserver。Scheduler 只是指挥，**真正落地干活的永远是 kubelet**。
2. **kube-proxy**：每个节点上的网络代理。监听 Service 和 Endpoint 变化，把规则写入 **iptables / ipvs**，实现 Service 的转发和负载均衡——「访问 Service IP」可行。

3. **容器运行时**：containerd（新标配）或 docker，事实上跑容器的。

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

kubeadm 是官方提供的集群引导工具：把「装一套 K8s」从手工几十步压缩成 `init` + `join` 两条命令。**它只负责控制面组件拉起，网络组件、存储、监控都得自己安装**。

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

yum 系（推荐阿里云源，没有机场国内直连官方源基本拉不动）：

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

> 此时 kubelet 起不来是**正常的**（每几秒重启一次），要等 kubeadm init/join 生成配置后才能正常工作。

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

> 主节点需要全部镜像；node 节点实际只用到 `kube-proxy` 和 `pause`，但推荐全下省心，配置有限的可以忽略按需即可。

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

> **注意**：node 上不需要 `.kube/config`，那是管理集群用的配置文件。node 的 kubelet 配置由 join 自动生成。

### 3.4 验证集群

```bash
kubectl get nodes
# NAME         STATUS   ROLES    AGE   VERSION
# k8s-master   Ready    master   10m   v1.20.9
# k8s-node1    Ready    <none>   2m    v1.20.9
# k8s-node2    Ready    <none>   2m    v1.20.9

kubectl get pods -A     # -A = 所有命名空间，确认全部 Running
```

**全部 Ready + 系统组件全 Running = 集群搭建完成。** 卡在 NotReady 排查网络组件安装/镜像拉取情况。

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

> **值得注意的是**：1.24 起 ServiceAccount 不再自动创建带 token 的 Secret，上面这条命令会查不到东西。新版本用 `kubectl -n kubernetes-dashboard create token admin-user` 直接签发 token，或者自己建一个 `type: kubernetes.io/service-account-token` 的 Secret。

复制 token 贴到登录页 → 进入界面。

#### ⑤ 界面

左侧菜单：Overview（集群概览）、Nodes / Workloads（Pod、Deployment 等负载）、Config Maps / Secrets、Services。**可视化看资源、快速排障可以，日常操作还是 kubectl 为主**——生产环境 Dashboard 一般不开公网暴露，历史上出过未授权访问的大事故，请大家不要把家门对外开放。

### 3.6 小结

> **整体链路**：kubeadm 装集群 = 基础环境（关 swap/开桥接）→ 装三件套 → 拉取镜像 → master init + 装 CNI → node join → 验证全 Ready。
>
> 三个核心记住才不出大问题：
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
| **命令式** | `kubectl run nginx --image=nginx` | 适合临时验证；但复杂参数写不动，也没法版本管理 |
| **声明式（yaml）** | `kubectl apply -f xxx.yaml` | 一切皆资源，yaml 可以进 git、可以复用、可以 review；**生产唯一正道** |

命令式感受一下就够，不推荐大家常用：

```bash
kubectl run mynginx --image=nginx
kubectl delete pod mynginx
```

### yaml 基本语法（会写清单就够）

严格要求：

1. **缩进只允许空格**，tab 直接报错（编辑器先设成「tab → 空格」）
2. **缩进空格数不严格**（习惯 2 格），同层级左侧对齐即可
3. `#` 标识注释，到行尾都被解释器忽略
4. 键值对 `key: value`，**冒号后一个空格**不能省

三种数据结构（所有清单都是这三种的组合）：

| 结构 | 语法 | 在清单里长什么样 |
|------|------|----------------|
| **对象**（映射） | key: value，靠缩进嵌套 | apiVersion/kind/metadata/spec 本身都是对象 |
| **数组**（列表） | 每项一个 `- ` | `spec.containers` 下一个 `-` 一个容器 |
| **纯量** | 字符串/数字/布尔/null | `image: nginx`、`replicas: 3` |

```yaml
# 纯量
image: nginx              # 字符串，默认不需要引号
replicas: 3               # 数字
enabled: true             # 布尔
note: ~                   # Null 用 ~ 或 null

# 对象：值本身又是 key: value，缩进一层
metadata:
  name: mynginx
  labels:
    app: mynginx          # 嵌套再深一层

# 数组：一个 "- " 一项，"-" 后的字段也占一层缩进
containers:
- name: mynginx
  image: nginx
- name: sidecar
  image: busybox
```

两个补充点：

- **多资源同文件**：用 `---` 分隔，一个 yaml 管一组资源（本文 StatefulSet 示例里 Service + StatefulSet 就是这么写的）。
- **带引号的字符串**：值「长得像数字/布尔」时要加引号（`version: "1.20"` 不加会按数字解析），普通字符串加不加都行，但要知道为什么加。

> **值得一提的是**：新手报错 Top 2 就是 `key:value` 少了空格、tab 混入缩进。写完先本地干跑验证，不碰集群：`kubectl apply --dry-run=client -f xxx.yaml -o yaml`，格式错立刻报。

声明式的核心问题：**yaml 字段这么多，怎么知道写什么？** 

答案不是查文档，是善用explain命令（没英语功底的话还是乖乖问AI吧）：

```bash
kubectl explain pod            # pod 有哪些一级字段
kubectl explain pod.spec       # spec 下有什么
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.ports
```

> **版本差异**：写 yaml 时别直接抄网上旧文章——不同资源有 **apiVersion**（v1 / apps/v1 / batch/v1……），抄错了 `kubectl apply` 直接报错。不确定就用 explain 查当前版本的正确写法。

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

> **补充**：`kubectl get pods` 不带 `-n` 永远只看 default——「我 Pod 明明起来了怎么看不到」九成是没切 ns。资源被删排查思路第一条：先 `kubectl get ns` 确认 ns 本身还在不在。

## 3. Pod

### 3.1 概念

Pod 是 K8s 的**最小部署单元**（不是容器！）。一个 Pod 里装 1 个或多个容器，它们**共享网络（同 IP 同端口空间）、共享存储卷、共享 IPC/UTS**——像一个逻辑上的「超薄虚拟机」。

按管理方式分两类：

- **自主式 Pod**：直接创建的裸 Pod，`kubectl run` 出来的。**Pod 挂了没人管，不会自愈**，生产不建议使用。
- **控制器管理的 Pod**：由 Deployment/StatefulSet 等控制器创建，挂掉会自动重建。**生产一律用控制器**。

```bash
kubectl run mynginx --image=nginx          # 快速验证（自主式）
kubectl get pod -o wide                    # 看 IP、落在哪个节点
kubectl describe pod mynginx               # 排障第一步：事件看 Pull 镜像/调度失败原因
kubectl logs mynginx                       # 看日志
kubectl exec -it mynginx -- /bin/bash      # 进容器
```

### 3.2 Pod 的 yaml 模板（全字段带注释）

Pod 清单必备四要素——**apiVersion、kind、metadata、spec**，这是所有资源的通用骨架，缺一个 `apply` 直接报错：

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

> 三个探针的区别：
>
> **liveness 挂了→重启；**
>
> **readiness 挂了→不接流量但不重启；**
>
> **startup（慢启动应用用）没通过前，前两个探针不生效**。

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

> 结论：**生产 Pod 必定写 requests/limits**，否则就是节点一紧张第一个被牺牲的。

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
| **Pending** | 已创建，但容器没全跑起来 | 等调度、拉镜像、等 PV 绑定 |
| **Running** | 已绑定节点，容器已创建且至少一个在运行 | 正常服役中（含正在重启/重启中） |
| **Succeeded** | 所有容器成功终止且不会重启 | Job/CronJob 跑完（restartPolicy: Never/OnFailure） |
| **Failed** | 所有容器终止且至少一个失败 | 退出码非 0、被系统杀掉 |
| **Unknown** | 状态未知 | 节点失联（心跳超时） |

> **辨析**：`Running` ≠ 「能接流量」。Running 只表示容器进程在跑，能不能接流量由 readinessProbe 决定，健不健康由 livenessProbe 决定。`kubectl get pod` 看到一堆 Running 但服务不通，先看 `READY` 列是不是 `1/1`——`0/1` 就是没过就绪探针。

phase 太粗糙，排障时更常用**容器级三态**（`kubectl describe pod` 的 State 字段）：

| 容器状态 | 含义 | 常见原因 |
|---------|------|---------|
| Waiting | 等待启动 | 拉镜像中、CrashLoopBackOff、等 ConfigMap/Secret |
| Running | 正在运行 | 正常 |
| Terminated | 已终止 | 退出码、结束原因（OOMKilled / Error / Completed）都在这看 |

> **CrashLoopBackOff**：容器反复崩溃反复重启的退避状态，重启间隔指数拉长（10s → 20s → 40s … 上限 5min）。九成是应用拉起失败1（配置错、依赖连不上、探针配太激进把健康进程杀成循环崩溃），用 `kubectl logs --previous` 看上一次崩溃的日志。

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

> **值得注意的是**：liveness 探针指向一个「要查数据库」的接口是经典事故——数据库抖 30 秒，探针把所有 Pod 重启一遍，故障被放大成雪崩。什么人干什么事，**liveness 只测进程自身健康**；依赖健康度交给 readiness（摘流量就够了，重启解决不了依赖问题）。

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
2. **preStop 是同步的**：kubelet 等它执行完才发 SIGTERM。这是它最大的价值——**给「从 Service 摘除」留缓冲**：Endpoint 变更从 apiserver 传导到所有节点的 iptables 有延迟，摘除后立刻杀容器会漏掉在途请求。滚动更新偶发 502，优先检查是否没配 preStop sleep。

> preStop 的耗时计入 `terminationGracePeriodSeconds`（默认 30s）宽限期：preStop 10s + 应用退出 10s < 30s 才安全，超了直接 SIGKILL。需要更长优雅期就在 yaml 里调大 `spec.terminationGracePeriodSeconds`。

#### 3.4.6 小结

> **完整链路**：Pod 生命周期 = Pause 骨架 → Init 串行铺路 → 主容器 + postStart → 三种探针接管（startup 屏蔽期 → readiness 管流量 → liveness 管重启）→ preStop + 宽限期优雅退出。
>
> 四个核心记住不出大问题：
>
> 1. **Running ≠ READY**，接不接流量看 readiness；排障先看容器三态（Waiting/Running/Terminated）
> 2. **liveness 只测自身健康**，若把依赖检查塞进去，依赖一抖全量重启雪崩
> 3. **postStart 异步、preStop 同步**——必须先跑的逻辑用 Init 容器；滚动更新 502 先补 preStop sleep
> 4. **Init 容器串行且必须全成功**，失败看 restartPolicy，Never 直接整个 Pod Failed

## 4. Deployment

Deployment 是最常用的控制器，管理无状态应用（web、api 这类）。它下层有 ReplicaSet 管理副本：

```
Deployment（管版本：滚动更新/回滚）
    └── ReplicaSet（管副本数：保持 N 个）
          └── Pod × N（工作区）
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

> **注意**：`selector.matchLabels` 必须和 `template.labels` 匹配，不匹配直接拒绝创建。这不是格式问题——Deployment 就靠标签认领 Pod，标签对不上副本发生错误。

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

Deployment 的看家本领：

```bash
# 实验 1：手动删 Pod —— 秒级重建
kubectl delete pod my-dep-xxx
kubectl get pods      # 新 Pod 立刻补上（名字变了，哈希后缀不同）

# 实验 2：模拟节点宕机 —— Pod 漂移
# 把某台 node 直接关机，等几分钟后：
kubectl get pods -o wide
# 该节点上的 Pod 先变成 Terminating，然后在新节点重建
```

原理是控制循环：副本数 3 是期望状态，控制器发现实际是 2，就再创建。**节点 NotReady 超时后（默认约 5 分钟）其上的 Pod 才会被驱逐重建**，不是瞬间漂移。

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

> **纠偏**：很多人以为回滚是「重新拉旧镜像跑一遍」，实际是**把旧 ReplicaSet 的副本数从 0 调回 N、新 RS 调回 0**——所以旧 RS 建议永远不删。也因此 `kubectl delete rs` 千万别手贱，删了就没得副本回滚了。

### Deployment 小结

> **一句话**：Deployment = 副本管理（ReplicaSet）+ 版本管理（滚动更新/回滚）双层封装，无状态应用的默认选择。
> 有状态（数据库类 DBMS）别硬 Deployment，使用和性能有相当的困难，进阶专题的 StatefulSet 再聊。





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

### 5.3 LoadBalancer 与 ExternalName（四件套）

| type | 原理 | 适用 |
|------|------|------|
| ClusterIP | 集群内虚拟 IP | 默认，集群内部互访 |
| NodePort | 每节点开 30000-32767 真实端口 | 测试 / 没有负载均衡器的自建环境 |
| **LoadBalancer** | 云厂商 LB → 回源仍是 NodePort 那套 | **云上暴露服务标配** |
| **ExternalName** | 返回 DNS CNAME，不做任何转发 | 集群内引用外部服务 |

```yaml
# LoadBalancer：云环境一条 yaml 就有公网入口
spec:
  type: LoadBalancer
  # 云的 cloud-controller-manager（见 2.2.1）自动调云 API 创建 SLB/EIP
  # kubectl get svc 的 EXTERNAL-IP 从 <pending> 变成公网 IP 即就绪
```

> **注意**：裸机/自建集群没有 CCM，LoadBalancer 的 EXTERNAL-IP 永远 `<pending>`——不是坏了，是没人接活。自建环境要装 MetalLB 补位。

```yaml
# ExternalName：把外部地址包装成集群内 DNS 名
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com    # 访问 external-db = CNAME 到 db.example.com
```

场景：应用统一用 `external-db.default.svc.cluster.local` 访问外部数据库；哪天数据库换地址或搬进集群，只改这一条 Service就好，几十个应用的配置不用动。

### 5.4 底层实现：kube-proxy 的两种模式

Service 能通全靠每个节点的 kube-proxy 写转发规则（组件职责见 2.2.2），两种模式：

| 模式 | 原理 | 规模化表现 |
|------|------|-----------|
| **iptables**（默认） | 每条 Service 一串 iptables 规则，顺序匹配 | 规则上千后整表刷新变慢、转发有延迟 |
| **ipvs** | 内核哈希表 + 多种负载算法（rr/lc/sed…） | 大规模平滑，Service 多时首选 |

```bash
# 切 ipvs：改 kube-proxy 的 ConfigMap
kubectl -n kube-system edit cm kube-proxy
#   mode: "" → mode: "ipvs"
# 改完删 Pod，DaemonSet 重建后生效
kubectl -n kube-system delete pod -l k8s-app=kube-proxy

# 验证：能看到 Service 虚拟 IP 的哈希表就对了
ipvsadm -Ln
```

> 经验值：Service 几十条规模 iptables 随便用；上百上千（大集群、多租户）切 ipvs，否则规则同步会拖慢每次发布。

## 6. Ingress

Service 是**四层**（TCP/UDP）负载均衡；Ingress 是**七层**（HTTP/HTTPS）：按**域名、路径**转发，还能做 TLS 卸载、限流、重写。注意 Ingress 资源本身只是一份规则，真正干活的是 **Ingress Controller**（常用 ingress-nginx，但需要知道的是已经停止维护。推荐学习GetwayApi），得先装。

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

使用注解，一行限流：

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "1"    # 每秒 1 个请求
```

压测，超限的直接返回 **503**。

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

先补一张 **volume 卷类型速查表**（本文前后用到的全在这，选型依照具体需求）：

| 卷类型 | 数据生命周期 | 典型用途 |
|--------|-------------|---------|
| **emptyDir** | 随 Pod 销毁 | 临时空间、多容器共享目录（3.3 示例） |
| **hostPath** | 节点上的目录 | 节点级 agent（日志/监控 DaemonSet）；绑死节点，业务别用 |
| **nfs** | 节点之外 | 直接挂 NFS 共享目录（7.3 示例，写死存储细节） |
| **configMap / secret** | 随 ConfigMap/Secret | 配置文件 / 敏感数据注入容器（7.5 / 7.6） |
| **persistentVolumeClaim** | 跟 PV 走 | 走 PV/PVC 抽象层（本节，**生产正道**） |

> 选型：**临时用 emptyDir → 配置用 configMap/secret → 数据要活过 Pod 就走 PVC**；hostPath 只留给节点级守护进程。

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

#### 7.4.3 StorageClass：动态供给（自动造 PV）

手动建 PV 池是**静态供给**，有两大硬伤：容量错配（申请 5M 绑走 10M 的 PV，浪费一半）、规模上来后运维手工造不过来。**StorageClass（SC）= 自动化工单**：PVC 只管申请，PV 由 provisioner 按需创建。

```yaml
# ① 定义 StorageClass：说明「用什么后端、怎么造」
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: cluster.local/nfs-subdir-external-provisioner   # NFS 供给器
parameters:
  archiveOnDelete: "false"      # 删 PVC 连目录一起删（生产数据库慎用）
```

```yaml
# ② PVC 只写 storageClassName，不再挑具体 PV
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: auto-pvc
spec:
  accessModes: ["ReadWriteMany"]
  storageClassName: nfs-client     # 关键：从「挑 PV」变成「下工单」
  resources:
    requests:
      storage: 1Gi
```

apply 后 PVC 直接 Bound，PV 凭空出现（reclaimPolicy 跟着 SC 定义走，默认 Delete）。

| 对比 | 静态供给 | 动态供给（SC） |
|------|---------|---------------|
| 谁造 PV | 运维手动建池 | provisioner 自动 |
| 容量 | 错配浪费 | 按需精确 |
| 换后端（NFS→Ceph） | 重造整个 PV 池 | 换个 SC 名即可 |

> StatefulSet 的 `volumeClaimTemplates` + SC = 「每个 Pod 一块自动创建的盘」（进阶专题 3.2.2 那套机制的生产形态）。云上集群（ACK/TKE/EKS）更简单：SC 是现成的，PVC 一提交云盘自动挂上。

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

> **值得注意的是**：base64 是**编码不是加密**（`echo xxx | base64 -d` 秒解），Secret 的安全边界主要在「权限控制 + 不落盘 + 不进 yaml 明文」，不是保密强度，不要吧编码的重要数据乱传。

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

**Secret 的 `type` 字段**（决定 apiserver 怎么校验内容）：

| type | 用途 | data / stringData 里的 key |
|------|------|---------------------------|
| **Opaque** | 默认通用类型，任意键值对 | 自定义 |
| `kubernetes.io/service-account-token` | SA 的长期 token（1.24 前随 SA 自动生成） | `token`、`ca.crt`、`namespace` |
| `kubernetes.io/dockerconfigjson` | 私有镜像仓库凭据，配 Pod 的 `imagePullSecrets` | `.dockerconfigjson` |
| `kubernetes.io/tls` | TLS 证书，Ingress 配 HTTPS 时引用 | `tls.crt`、`tls.key` |
| `kubernetes.io/basic-auth` / `ssh-auth` | HTTP Basic 认证、SSH 私钥 | `username`+`password` / `ssh-privatekey` |

> `service-account-token` 这一行就是 RBAC 里 ServiceAccount 的身份载体（进阶专题 5.1）；`dockerconfigjson` 是「镜像拉不下来」第二常见的根因（9.3 症状表）。

### ConfigMap vs Secret

| 项目 | ConfigMap | Secret |
|------|-----------|--------|
| 存什么 | 普通配置（nginx.conf、yml） | 敏感数据（密码、token、证书） |
| 值的格式 | 明文 | base64 |
| 热更新 | 支持（subPath 除外） | 挂载方式支持，env 方式不支持 |
| 落盘 | etcd + 可挂载 | etcd（可配置加密）+ tmpfs 挂载 |

### 存储小结

> **总结**：K8s 把存储拆成「运维提供 PV / 用户申请 PVC / Pod 无感使用」三层，配置类数据走 ConfigMap（明文、热更新）、敏感数据走 Secret（base64、tmpfs）。
>
> 三个核心记住不出大问题：
>
> 1. **PVC Pending = 没有匹配的 PV**（storageClassName / accessModes / 容量三对照）
> 2. **ConfigMap 热更新只改文件，不 reload 进程**；subPath 单文件挂载连文件都不更新
> 3. **数据库类 PV 的回收策略用 Retain**，误删 PVC 也保得住数据

---

## 8. 集群调度（Pod 是怎么落到某个节点的）

Pod 落到哪个节点，是 **kube-scheduler** 说了算（组件职责见 2.2.1）——它只写决策，kubelet 按决策干活。调度不是随缘挑，而是「过滤 + 打分」两轮算法。

### 8.1 调度过程：预选 + 优选

```
新 Pod（未绑定节点）
    │
    ▼
预选（Predicates）：过滤掉不合格节点
    资源不够 / 端口冲突 / 不满足亲和性 / 污点不容忍 / 不满足卷拓扑……
    │
    ▼
优选（Priorities）：给剩下的节点打分
    资源余量越大分越高、镜像已在本机加分（少拉镜像）、副本尽量打散……
    │
    ▼
得分最高者 → 写回绑定关系（Pod.spec.nodeName）→ 该节点 kubelet 接手
```

> **排障唯一入口**：Pod 一直 Pending，先 `kubectl describe pod` 看 Events 里的 **FailedScheduling**——「0/3 nodes are available: 3 Insufficient cpu」「node(s) didn't match Pod's node selector」「had untolerated taint」——**后面这句就是具体原因**，三类：资源不够、约束不满足、污点不容忍。

### 8.2 四种调度约束（从粗到细）

| 方式 | 写在哪 | 一句话 |
|------|--------|--------|
| **nodeName** | Pod | 直接指名道姓，跳过 scheduler |
| **nodeSelector** | Pod | 按节点标签挑，最简单够用 |
| **亲和性**（node/pod Affinity） | Pod | nodeSelector 超集，软硬两档，能表达「优先」 |
| **污点 + 容忍**（taint / toleration） | 节点出题、Pod 答题 | 反向逻辑：节点拒绝大部分 Pod，带通行证才能上 |

#### ① 指定调度节点：nodeName / nodeSelector

```yaml
spec:
  nodeName: k8s-node1        # 写法一：直接指名（跳过 scheduler，测试用，生产别写死）
  nodeSelector:              # 写法二：按标签挑
    disktype: ssd
```

```bash
kubectl label node k8s-node1 disktype=ssd    # nodeSelector 的前提：先给节点打标签
```

> nodeSelector 是**硬性**的：没有带该标签的节点，Pod 直接 Pending。它表达不了「优先去、没得去也行」的软性语义——那是亲和性的事。

#### ② Pod 亲和性：硬亲和 vs 软亲和

亲和性分两个维度：**nodeAffinity**（和节点的关系，看节点标签）、**podAffinity/podAntiAffinity**（和其他 Pod 的关系，看 Pod 标签）。两个维度都分软硬两档：

| 档位 | 字段（长得吓人，拆开记） | 语义 |
|------|--------------------------|------|
| **硬亲和** | requiredDuringSchedulingIgnoredDuringExecution | 必须——不满足就 Pending，宁缺毋滥 |
| **软亲和** | preferredDuringSchedulingIgnoredDuringExecution | 优先——满足加分，不满足照样调度 |

> 那个长后缀 **IgnoredDuringExecution** 的含义：**约束只在调度那一刻生效**——Pod 跑起来后节点标签变了、邻居 Pod 挪走了，K8s 不会为维持亲和关系驱逐已运行的 Pod。

```yaml
affinity:
  nodeAffinity:                    # 节点亲和（软）：优先落 SSD 机器，没有也行
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 80                   # 软亲和带权重，参与优选打分
      preference:
        matchExpressions:
        - key: disktype
          operator: In             # In / NotIn / Exists / DoesNotExist / Gt / Lt
          values: ["ssd"]
  podAntiAffinity:                 # Pod 反亲和（硬）：同服务副本必须打散到不同节点
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: my-dep
      topologyKey: kubernetes.io/hostname
```

> **实用结论**：多副本 Deployment 上生产的标配是 **podAntiAffinity + topologyKey: kubernetes.io/hostname**——强制副本分散在不同节点，一台机器挂不至于全灭。反过来，前端应用想和它依赖的 Redis 同机省流量，用 podAffinity。

#### ③ 调度策略：污点与容忍

前面的约束都是 **Pod 挑节点**；污点（taint）是反过来的——**节点挑 Pod**：

- **污点**打在**节点**上：「我有特殊情况，别随便往我这调度」
- **容忍**（toleration）写在 **Pod** 上：「我知道你什么情况，我能接受」

三种 effect：

| effect | 行为 |
|--------|------|
| **NoSchedule** | 不容忍就不调度（已在跑的不动） |
| **PreferNoSchedule** | 尽量不调度，实在没得选也行（软性） |
| **NoExecute** | 不容忍不但不调度，**已运行的直接驱逐** |

```bash
# 运维排障：打 NoExecute 污点，把节点上需要操作的 Pod 全部逐出
kubectl taint node k8s-node1 maintenance=true:NoExecute
# 操作完删除污点（key 后面加减号）
kubectl taint node k8s-node1 maintenance=true:NoExecute-
```

> **运维污点的正确用法**（对应笔记里那条「污点使用—驱逐需要操作的 Pod」）：要清空某节点上的业务 Pod，打 `NoExecute` 污点——不容忍的 Pod 全部被驱逐并重新调度到别的节点，比手动一个个 delete 靠谱（`kubectl drain` 底层就是 cordon + 这套驱逐逻辑，见 2.3）。
>
> 反向记忆：**master 默认带 `node-role.kubernetes.io/master:NoSchedule` 污点**，所以业务 Pod 不落 master；DaemonSet 的系统组件是自带 toleration 才能做到每个节点都跑（见进阶专题 4.1）。

Pod 侧写容忍（能上被污点标记的节点）：

```yaml
tolerations:
- key: "maintenance"
  operator: "Equal"
  value: "true"
  effect: "NoExecute"
  tolerationSeconds: 3600        # 只忍 1 小时，之后照样被驱逐（NoExecute 专属参数）
```

### 调度小结

> **一句话**：调度 = scheduler 两轮算法（预选过滤 + 优选打分）；约束手段从粗到细 = nodeName（指名）→ nodeSelector（标签）→ 亲和性（软硬两档、节点/Pod 两维）→ 污点容忍（节点反向挑 Pod）。
>
> 三条核心记住不出大问题：
>
> 1. **Pod Pending 先 describe 看 FailedScheduling**，原因无非资源不够 / 约束不满足 / 污点不容忍三类
> 2. **软硬亲和的区别就一句话**：required 不满足就 Pending，preferred 不满足照样调度
> 3. **驱逐节点 Pod 用 NoExecute 污点或 drain**，别手动挨个 delete

## 9. kubectl 排障工具箱

前面各章的命令是散着讲的，这里收口成一张总表 + 一套固定套路——**排障时不用往前翻**。

### 9.1 高频命令总表

| 类别 | 命令 | 说明 |
|------|------|------|
| 看状态 | `kubectl get pods -o wide` | 带 IP / 落点节点；先看 READY 列 |
| 看详情 | `kubectl describe pod <name>` | **Events 是灵魂**：调度失败、镜像拉取、探针失败全在这 |
| 看日志 | `kubectl logs <name> [-f] [--previous]` | `--previous` 看上一次崩溃的日志（CrashLoop 必用） |
| 进容器 | `kubectl exec -it <name> -- /bin/sh` | 到现场排查 |
| 拷文件 | `kubectl cp <pod>:/path ./local` | 把日志/配置拉出来分析 |
| 临时端口 | `kubectl port-forward pod/<name> 8080:80` | 本机直连调试，不动 Service |
| 看资源占用 | `kubectl top pods / nodes` | 需先装 Metrics Server（HPA 同款依赖） |
| 看集群事件 | `kubectl get events --sort-by=.lastTimestamp` | 集群级时间线，比挨个 describe 快 |
| 临时工具 Pod | `kubectl run tmp --rm -it --image=busybox --restart=Never -- sh` | 测网络连通 / DNS 解析的利器，退出自动删 |

### 9.2 排障（固定顺序）

```
① kubectl get pod              → 先看 STATUS / READY，确定症状
② kubectl describe pod         → Events 定位阶段：调度？拉镜像？探针？
③ kubectl logs [--previous]    → 容器已跑起来，问题在应用内部
```

> 记忆点：**get 看症状、describe 看阶段、logs 看内因**。九成的 Pod 问题走完这三步就有结论；跨节点/服务不通再上第 9.1 表里的临时工具 Pod 测网络。

### 9.3 症状速查表

| STATUS | 含义 | 第一反应 |
|--------|------|---------|
| Pending | 没调度上 / 等 PV 绑定 | describe 看 FailedScheduling 三类原因（见二.8） |
| ImagePullBackOff / ErrImagePull | 镜像拉不下来 | 镜像名拼错 / 私有仓库没配 imagePullSecrets |
| CrashLoopBackOff | 反复崩溃反复重启 | `logs --previous`；查配置错 / 依赖连不上 / 探针过激 |
| OOMKilled（exit 137） | 内存超 limits 被内核杀 | 调大 limits.memory 或查内存泄漏 |
| Evicted | 被节点驱逐 | 节点资源紧张，对照 QoS（见 3.4）和节点压力 |
| Completed | 正常跑完退出 | Job 类的正常终态，不是故障 |
| Running 但 0/1 Ready | 没过就绪探针 | readiness 配置 / 应用启动慢（上 startupProbe） |

## 10. 资源治理：配额与驱逐保护

三个「兜底」资源，管的是**别人不守规矩时集群怎么办**——PDB 兜 drain、ResourceQuota 兜命名空间、LimitRange 兜没写资源的 Pod。前文多处引用过它们，这里补齐原理。

### 10.1 PDB：PodDisruptionBudget（驱逐保护）

2.3 讲 drain 时说过「驱逐遵守 PDB」——它约束的是**自愿中断**（drain、主动驱逐这类运维动作）时，一个服务最少必须保住几个副本：

```yaml
apiVersion: policy/v1beta1      # 1.21+ 改为 policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-dep-pdb
spec:
  minAvailable: 2               # 任何时刻至少 2 个可用（也可写百分比 "50%"）
  selector:
    matchLabels:
      app: my-dep
```

drain 时如果驱逐某个 Pod 会让可用数跌破 2，**drain 会卡住等待**（`--force` 可强推，但保护就没了）。

> 边界认清：PDB **只管自愿中断**。节点宕机这种**非自愿中断**它管不了——那要靠副本数 + podAntiAffinity 打散（见二.8）。

### 10.2 ResourceQuota：命名空间配额

多团队共用集群时，防一个 ns 吃光整个集群资源：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    pods: "20"                  # 最多 20 个 Pod
    requests.cpu: "4"           # ns 内所有 Pod 的 requests 总和上限
    requests.memory: 8Gi
    limits.cpu: "8"
    persistentvolumeclaims: "5"
```

超配额的创建请求直接被 apiserver 拒绝（报 `exceeded quota`）——**配额是硬门槛，不是软限制**。

> **注意**：一旦 quota 限制了 cpu/memory，该 ns 里**所有 Pod 都必须写 requests/limits**，不写的直接拒绝创建——「以前能跑的 yaml 突然 apply 不上了」，先查是不是有人新加了 quota。

### 10.3 LimitRange：给不守规矩的 Pod 兜底

Quota 要求人人写资源，但总有忘写的。LimitRange 给 ns 内的 Pod **注入默认值 + 卡上限**：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default:                    # 默认 limits（没写就自动补这组）
      cpu: 500m
      memory: 512Mi
    defaultRequest:             # 默认 requests
      cpu: 100m
      memory: 128Mi
    max:                        # 单容器上限，写超了直接拒绝创建
      cpu: "2"
      memory: 2Gi
```

三个字段分工：`default / defaultRequest` 补默认值——没写资源的 Pod 自动变 Burstable，不再是 BestEffort 被优先驱逐（QoS 见 3.4）；`max` 卡上限，防止有人写个 16 核把节点压垮。

### 治理小结

> **总的来说**：PDB 保服务在运维操作时的最小副本，ResourceQuota 卡命名空间总量，LimitRange 补默认值 + 卡上限——三个都是「防呆」设计，和 QoS（3.4）合起来构成资源治理全套。
>
> 上线顺序建议：**先 LimitRange（补默认）→ 再 ResourceQuota（卡总量）→ 多副本服务最后配 PDB（保 drain）**。

# 三、进阶专题

> 前面是「入门主线」，以下是七个专题：HPA（自动扩缩容）、SQL 数据库上 K8s 的可行性评估、StatefulSet（有状态应用）、Job/DaemonSet/CronJob（批处理与守护进程）、安全（认证/鉴权/准入 + RBAC 详解）、Helm 与生态组件（Dashboard/Prometheus/EFK）、运维（证书续期与 etcd 备份）。
>
> 配合主线食用：Deployment/Service 学完看 HPA，存储抽象学完看 StatefulSet，**存储和 Secret 学完看安全专题**（Secret 的权限、SA token 的身份链路都串在那），学完全文回头看运维专题。

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

## 4. Job / DaemonSet / CronJob 详解

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

### 4.2 Job：一次性任务

跑完就退出的批处理任务（数据转换、计算、初始化）。和 CronJob 的关系：**CronJob 是定时器，每到期造一个 Job——真正管 Pod 的是 Job**。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  completions: 5               # 总共要「成功」5 个 Pod（不写默认 1）
  parallelism: 2               # 最多同时跑 2 个（不写默认 1）
  backoffLimit: 4              # 失败重试上限，超了整个 Job 判 Failed
  activeDeadlineSeconds: 300   # 总超时兜底
  template:
    spec:
      restartPolicy: Never     # Job 只允许 Never / OnFailure
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
```

```bash
kubectl get jobs
# NAME   COMPLETIONS   DURATION   AGE
# pi     5/5           47s        50s       ← 5/5 即全部成功
```

三个语义：

1. **completions 数的是成功**：失败的 Pod 不算数，靠重试补到 5 个成功为止
2. **backoffLimit 是全局失败上限**：重试带退避（10s → 20s → 40s…），超限 Job 变 Failed，不再烧资源
3. **restartPolicy 只能 Never / OnFailure**：写 Always 会无限重启、Job 永远完不成——这是 Job 与长跑负载的本质区别

> **踩坑**：Job 完成后 Pod 和 Job 对象默认**一直留着**（方便看日志），跑得勤的集群会攒一堆 Completed Pod。加 `ttlSecondsAfterFinished: 600` 让它自动清理。

### 4.3 CronJob：定时任务

#### 4.3.1 核心定义

基于 Cron 表达式周期性创建 Job，每个 Job 创建一个或多个 Pod 执行一次性任务。

#### 4.3.2 典型使用场景

| 场景       | 示例                    |
| :--------- | :---------------------- |
| 数据库备份 | 每天凌晨 2 点备份 MySQL |
| 报表生成   | 每周一生成业务报表      |
| 数据清理   | 每小时清理过期日志      |
| 邮件推送   | 每天定时发送营销邮件    |
| 同步任务   | 每 5 分钟同步外部数据   |
| 证书续期   | 定期检查并续期 TLS 证书 |

#### 4.3.3 核心字段详解

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

#### 4.3.4 Cron 表达式格式

标准 5 字段：`分钟 小时 日 月 星期`

- 支持 `*`、`,`、`-`、`/`、`?`
- 宏：`@hourly`、`@daily`、`@weekly`、`@monthly`、`@yearly`
- 注意：日和星期同时指定时为 **OR** 关系。

#### 4.3.5 并发策略 `concurrencyPolicy`

| 策略            | 行为                        |
| :-------------- | :-------------------------- |
| `Allow`（默认） | 允许并发运行                |
| `Forbid`        | 跳过本次调度                |
| `Replace`       | 取消当前 Job，用新 Job 替换 |

#### 4.3.6 错过调度处理

- 未设置 `startingDeadlineSeconds`：尝试补跑，超过 100 次则不再调度。
- 设置 `startingDeadlineSeconds`：仅在截止时间内的错过才补跑。

#### 4.3.7 历史清理

- `successfulJobsHistoryLimit`：默认 3。
- `failedJobsHistoryLimit`：默认 1。
- `ttlSecondsAfterFinished`：完成后自动删除。

#### 4.3.8 重要限制与最佳实践

- 任务必须幂等。
- 设置超时 `activeDeadlineSeconds`。
- 设置资源请求/限制。
- 显式设置时区。
- `restartPolicy` 必须为 `Never` 或 `OnFailure`。
- 监控 Job 状态。
- 最小粒度是分钟，不支持秒。

### 4.4 五种工作负载控制器怎么选

| 控制器 | 目标 | Pod 身份 / 存储 | 典型用途 |
| :----- | :--- | :-------------- | :------- |
| **Deployment** | 固定副本数无状态服务 | 随机名 + 随机 IP | web、api |
| **StatefulSet** | 稳定身份有状态服务 | 固定名 + 独立 PVC + 稳定 DNS | 数据库、MQ、etcd |
| **DaemonSet** | 每个节点跑一个 | 随机，无稳定身份 | 日志、监控、网络插件 |
| **Job** | 一次性任务，跑完即退 | 随机 | 数据转换、批处理计算 |
| **CronJob** | 按 Cron 定时造 Job | 随机 | 备份、报表、清理 |

**选择原则**：

- 固定副本数无状态服务 → Deployment
- 稳定身份和存储的有状态服务 → StatefulSet
- 每个节点都跑常驻进程 → DaemonSet
- 一次性批处理任务 → Job
- 定时执行批处理任务 → CronJob

---

## 5. 安全：认证 / 鉴权 / 准入（RBAC 详解）

每个打到 apiserver 的请求都要过——这是 K8s 安全模型的主干：

```
kubectl / SDK / Dashboard / Pod 内进程
        │
        ▼
① 认证 Authentication —— 你是谁？
    （X509 客户端证书 / ServiceAccount token(JWT) / OIDC 账号……）
        │  没通过 → 401 Unauthorized
        ▼
② 鉴权 Authorization —— 你能干什么？
    （默认 RBAC：动词 × 资源 × 主体 三条一交，Rule 命中才放行）
        │  没通过 → 403 Forbidden（日常遇到的绝大多数报错在这）
        ▼
③ 准入控制 Admission —— 这个具体操作怎么才算合规？
    （ResourceQuota / PodSecurity / VAP+CEL / 自定义 Webhook，可改写可拦截）
        │  没通过 → 请求拒绝，或被改写后再落库
        ▼
    kube-apiserver ──► etcd
```

> **排障口诀**：**401 查认证**（证书过期、token 失效、kubeconfig 串了集群），**403 查鉴权**（没绑、绑错 ns、用了 RoleBinding 却想要集群权限）。绝大多数人一辈子遇的都是 403。
>
> 类比理解：**认证 = 门禁刷卡确认你是员工，鉴权 = 你的工牌能进哪些房间，准入 = 进房间要不要戴鞋套、登记**。三者独立，前门过了不代表后门过。

### 5.1 认证 Authentication：身份怎么来的

先记住一个反直觉的事实：**K8s 里没有「用户」这个 API 对象**。apiserver 不存用户表（`kubectl get users` 查不到），用户名就是认证插件从凭证里「提取出来的一串字符串」——所以**认证方式的本质 = 用什么凭证向 apiserver 声称一个身份**。

| 认证方式 | 凭证载体 | 谁在用 | 备注 |
|---------|---------|-------|------|
| **X509 客户端证书**（HTTPS 双向认证） | kubeconfig 里的 client-certificate-data / client-key-data | kubectl、集群组件（kubelet、controller-manager） | kubeadm 的 `/etc/kubernetes/admin.conf` 就是它；**CN = 用户名，O = 用户组** |
| **Bearer Token（JWT）** | ServiceAccount token | Pod 内进程、CI/CD、Dashboard 登录 | 你说的 jwt 就在这：**SA token 本身就是一张 JWT**，apiserver 用公钥验签 |
| Bootstrap Token | `kubeadm join` 用的临时 token | 节点加入集群 | 有时效，只够搬运一次性流程 |
| **OIDC** | 外部 IdP 签的 id_token | 企业人群登录（Keycloak / GitLab / 企业微信） | kubectl 侧靠 `kubelogin` 之类插件换 token |
| Webhook TokenReview | 甩给外部服务审查 token | 自研账号体系 | 每次请求都回调，注意可用性 |
| 静态密码 / 匿名 | 静态文件 / `--anonymous-auth` | — | 静态基本弃用；**生产建议关匿名** |

#### ① HTTPS 证书认证：kubeconfig 的三段结构

```yaml
clusters:                                  # ① 集群：地址 + 服务端 CA
- name: kubernetes
  cluster:
    server: https://172.31.0.4:6443
    certificate-authority-data: LS0t...    # 用来验证「对面真的是 apiserver」
users:                                     # ② 用户：我的凭证（证书或 token）
- name: kubernetes-admin
  user:
    client-certificate-data: LS0t...       # 客户端证书：证明我是 kubernetes-admin
    client-key-data: LS0t...
contexts:                                  # ③ 上下文：把 ① 和 ② 绑一起，还带默认 ns
- name: kubernetes-admin@kubernetes
  context: {cluster: kubernetes, user: kubernetes-admin, namespace: default}
current-context: kubernetes-admin@kubernetes
```

```bash
kubectl config view --raw          # 看当前 kubeconfig
kubectl config current-context     # 我现在是谁
kubectl config get-contexts        # 所有环境切换列表
kubectl config use-context xxx     # 切环境（多集群运维日常）
```

> **双向 TLS 到底验什么**：客户端拿 CA 验服务端证书（防中间人），服务端拿 CA 验客户端证书，**并从证书里抽 `CN=用户名` / `O=用户组`**。所以 kubeadm 出来的 admin.conf 是 `CN=kubernetes-admin, O=system:masters`——**「超管」这个身份是证书里的 O 字段声明出来的**，不是数据库里配的。这也是为什么泄露 admin.conf = 直接交出集群。

#### ② ServiceAccount 与 JWT：Pod 的身份

Pod 里的进程没法刷卡，它的身份是 ServiceAccount（SA）。SA 是**命名空间级**对象，属于"集群内的机器用户"。

```bash
kubectl create sa app-viewer -n dev
kubectl -n dev create token app-viewer                     # 签发一张有时效的 JWT（1.24+）
kubectl -n dev create token app-viewer --duration=8h       # 指定有效期
```

Pod 内默认挂载位置（kubelet 投射进去的）：

```text
/var/run/secrets/kubernetes.io/serviceaccount/
├── token        # JWT —— 调 apiserver 时塞在 Header: Authorization: Bearer <这一坨>
├── ca.crt       # 集群 CA
└── namespace    # 当前 ns
```

```yaml
spec:
  automountServiceAccountToken: false   # 不需要调 apiserver 的 Pod 关掉自动挂载（安全加固常用）
```

> **版本分水岭（1.24），背下来**：
>
> 1. **1.24 之前**：创建 SA 会自动生成一个同名的 `kubernetes.io/service-account-token` Secret，token **永不过期**——泄露 = 永久后门。
> 2. **1.24 起**：不再自动创建 Secret。`kubectl get secret | grep xxx` 查不到 token 是正常的，改用 `kubectl create token` 现签。Dashboard 章节第 ④ 步（1.3.5）踩的就是这个坑。
> 3. **Bound ServiceAccount Token**（1.22 GA）：token 有时效（默认 1h）、**绑定 Pod UID**，Pod 删了 token 立刻失效。现在取的都是从这来的。
>
> 需要长期凭证的特殊场景（CI、备份脚本）只能手工建 Secret，`-n <ns> create token` 出来的临时 token 别往配置文件里贴。

### 5.2 RBAC 四件套与 binding 拓扑

RBAC（Role-Based Access Control）的思路就三件事：**谁（subject）→ 被授予哪份权限（role）→ 绑起来（binding）**。所以对象只有四个两个层级：

| 对象 | 级别 | 作用 |
|------|------|------|
| **Role** | 命名空间 | 定义权限（哪些资源、哪些动词） |
| **ClusterRole** | 集群 | 同上，且能管集群级资源（Node / PV / ns / 非资源端点） |
| **RoleBinding** | 命名空间 | 把角色绑给主体，只在**该 ns** 生效 |
| **ClusterRoleBinding** | 集群 | 把角色绑给主体，**全集群 + 所有 ns** 生效 |

**binding 拓扑**（这 2×2 矩阵是重点，四个格子里有三个坑）：

| 角色 ↓ ＼ 绑定 → | **RoleBinding**（ns 级） | **ClusterRoleBinding**（集群级） |
|-----------------|------------------------|--------------------------------|
| **Role**（ns 级） | ✅ 本 ns 生效（最标准的用法） | ⚠️ 合法但几乎不用：把 ns 级 Role 放大到全集群，权限直接溢出 |
| **ClusterRole**（集群级） | ✅ **高频混搭**：权限定义是集群级的，但**降级到单个 ns 生效** | ✅ 全集群 + 所有 ns 生效 |

> 中间那格「ClusterRole + RoleBinding」是实操最常用的省钱组合：官方现成的 ClusterRole（`view` / `edit` / `admin`）直接复用，用 RoleBinding 限制在某一个 ns——**不用为每个 ns 抄一份 Role**。
>
> 反过来「Role + ClusterRoleBinding」是个坑：Role 里压根定义不了集群级资源，绑出去只会让同名 ns 的权限跨 ns 生效，语义混乱，别用。

**ClusterRole 的三类典型用途**（对应你草稿里那三条）：

| 用途 | 说明 | 例子 |
|------|------|------|
| **集群级资源** | Node / Namespace / PV / StorageClass / ClusterRole 这类不属于任何 ns 的资源 | 给监控只读 Node |
| **非资源端点** | `/healthz`、`/metrics`、`/version` 这类 URL（走 `nonResourceURLs`，**只能写在 ClusterRole，Role 不支持**） | 给探活组件放行 `/healthz` |
| **所有命名空间的控制资源** | Pod / Deployment / Service……一次性给全 ns 授权 | `cluster-admin`、集群级只读账号 |

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metrics-viewer
rules:
- nonResourceURLs: ["/healthz", "/readyz", "/metrics"]    # 非资源端点
  verbs: ["get", "head"]
- apiGroups: [""]                                          # 集群级资源：Node
  resources: ["nodes", "nodes/metrics"]                    # nodes/metrics 是子资源
  verbs: ["get", "list"]
- apiGroups: ["", "apps"]                                  # 跨所有 ns 的资源
  resources: ["pods", "deployments"]
  verbs: ["get", "list", "watch"]
```

### 5.3 权限怎么描述：Resources × Verbs × Subjects

一条 `rules` 就是「**对哪些资源（resources）能动哪些动词（verbs）**」，`binding` 回答「**授给谁（subjects）**」。

**rule 的四要素**：

| 字段 | 说明 | 常见踩点 |
|------|------|---------|
| `apiGroups` | core 组写成 `""`；其余 `apps` / `batch` / `networking.k8s.io` | **组写错 = 权限静默不生效**（不报错，只是没权限） |
| `resources` | 一律**复数**：`pods`、`deployments`；子资源写 `pods/log`、`pods/exec`、`deployments/scale` | `kubectl logs`/`exec` 要**单独**给 `pods/log`、`pods/exec` |
| `verbs` | `get` `list` `watch` `create` `update` `patch` `delete` `deletecollection` | 只读三件套 = get+list+watch（漏 list 会出现「能 get 单个对象但列表 403」的怪象） |
| `resourceNames` | 精确到某个对象名 | 只让看某一个 Secret 时使用 |

**subjects：角色绑给谁**（三种，写错 ns 是高频事故）：

| kind | 值 | 场景 |
|------|-----|------|
| **User** | 字符串，如 `"dev-user"`（= 证书 CN） | 人类账号 |
| **Group** | 如 `"dev-team"`（= 证书 O） | 人类组 |
| **ServiceAccount** | 必须带 ns：`<ns>/<sa名>` | Pod 内程序、CI/CD |

```yaml
subjects:
- kind: ServiceAccount
  name: ci-bot
  namespace: dev          # 不写 ns 默认取当前 ns —— 绑了个空气，403 查半天
```

**system: 保留关键字**（这是一类内置身份，别自己建同名对象）：

| 内置组 / 用户 | 含义 | 危险度 |
|-------------|------|-------|
| `system:masters` | 超管组（admin.conf 的 `O=system:masters`），**直接绕过所有 RBAC** | ⚠️ 最高 |
| `system:authenticated` | 所有通过认证的请求 | 给它授权 = 给全人类授权 |
| `system:unauthenticated` | 匿名请求 | 生产禁止授予任何权限 |
| `system:nodes` / `system:node:<hostname>` | kubelet 专用 | 由 Node authorizer 管 |
| `system:serviceaccounts:<ns>` | 该 ns 下所有 SA | 批量授权会连新 SA 一起放行 |

顺手记四个**内置 ClusterRole**（拿来即用，省得自己写 rule）：

| ClusterRole | 权限 |
|-------------|------|
| `cluster-admin` | 集群一切（所有 ns + 所有资源 + 所有动词） |
| `admin` | 某 ns 内一切（配 RoleBinding 用） |
| `edit` | 能改能删工作负载，不能动 RBAC 和配额 |
| `view` | 只读，不能改 Secret |

完整示例（Role + RoleBinding，Pod 只读）：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role                          # 权限：dev 空间里能读 Pod
metadata:
  namespace: dev
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding                   # 绑定：SA ci-bot 获得这份权限
metadata:
  namespace: dev
  name: read-pods
subjects:
- kind: ServiceAccount
  name: ci-bot
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

### 5.4 准入控制 Admission：合法—>过审

过了 RBAC 只说明「有资格做这个动作」，准入管的是「这个动作做得标不标准 / 要不要顺手改一下」。分两类：**改写型（Mutating）先跑，校验型（Validating）后跑**。

**常用内置准入插件**（默认随 apiserver 启用）：

| 准入插件 | 干的事 |
|---------|-------|
| NamespaceLifecycle | ns 不存在 / 正在删除时禁止建资源（防资源孤儿） |
| **LimitRanger / ResourceQuota** | 落实 LimitRange 的默认值与 ns 配额（见 10.2 / 10.3） |
| ServiceAccount | 给 Pod 补 SA、投射 token 卷 |
| DefaultStorageClass | PVC 没写 SC 时补默认 SC |
| **PodSecurity**（1.25+） | 执行 `privileged` / `baseline` / `restricted` 三档安全基线，接替已移除的 PodSecurityPolicy(PSP) |
| MutatingAdmissionWebhook / ValidatingAdmissionWebhook | 动态准入：把判决甩给集群内的一个 Webhook 服务 |

> 动态准入是 Service Mesh 这类东西的实现底座（istio 的 sidecar 自动注入就是 **MutatingWebhook** 干的）。但代价是**每次请求都多一次网络回调**：webhook 挂了 + `failurePolicy: Fail` = **整个集群无法创建 Pod**（除非设 Ignore，但那样策略被绕过）。所以生产要给 webhook 本身做高可用，`timeoutSeconds` 别设太大。

#### VAP + CEL：不用起服务的校验策略

**ValidatingAdmissionPolicy（VAP）** = 用 CRD 在集群内直接声明校验规则，规则用 **CEL 表达式**写，不需要维护一个 webhook 服务。版本线：1.26 alpha → 1.29 beta → **1.30 GA**。

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-owner-label
spec:
  failurePolicy: Fail                    # 策略自身出错时 Fail=拒绝（安全优先）/ Ignore=放行
  matchConstraints:                      # 对哪些请求生效
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]  # 别写 DELETE：删除时 object 为 null，表达式会崩
      resources:   ["deployments"]
  validations:                           # CEL 表达式，全部为真才放行
  - expression: "has(object.metadata.labels) && 'owner' in object.metadata.labels"
    message: "每个 Deployment 必须带 owner 标签，出事要能找到人"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-owner-label-binding
spec:
  policyName: require-owner-label
  validationActions: [Deny]              # Deny=直接拦截 / Audit=只记审计日志（灰度期神器）
  matchResources:
    namespaceSelector:
      matchLabels: {kubernetes.io/metadata.name: dev}
```

常用 CEL 规则（照抄改改就能用）：

| 想拦什么 | CEL 表达式 |
|---------|-----------|
| 禁止 `:latest` 标签 | `object.spec.template.spec.containers.all(c, !c.image.endsWith(':latest'))` |
| 必须写资源限制 | `object.spec.template.spec.containers.all(c, has(c.resources.limits))` |
| 副本数至少 2 | `object.spec.replicas >= 2` |
| 禁止特权容器 | `object.spec.template.spec.containers.all(c, !has(c.securityContext.privileged) \|\| c.securityContext.privileged == false)` |
| 必须指定 imagePullPolicy | `object.spec.template.spec.containers.all(c, has(c.imagePullPolicy))` |

> **注意边界**：VAP 只能**校验**，不能改写对象；要改对象（自动补默认值、注入 sidecar）得用 MutatingAdmissionWebhook（MutatingAdmissionPolicy 是更晚才出现的能力，别混着记）。
>
> 上线姿势：**先 `validationActions: [Audit]` 跑一周，看审计日志里有多少会误伤，再切 `[Deny]`**——一步到位切拒绝容易在半夜拦掉一堆 CI 部署。

### 5.5 实操：nginx 与 sql 的资源分隔与鉴权准入（#9-4）

目标：两个业务分别在 `app-nginx` / `app-sql` 两个 ns，各自的 SA **只能动自己 ns 的资源**，跨 ns 一律 403。

**① 建命名空间与 SA**

```bash
kubectl create ns app-nginx
kubectl create ns app-sql
kubectl -n app-nginx create sa nginx-runner
kubectl -n app-sql    create sa sql-runner
```

**② 各给一份 ns 内权限**（sql 额外放开读 Secret，因为连接串在那；nginx 不需要）

```yaml
# app-nginx：管理自己 ns 的 Pod/Service，不许碰 Secret
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {name: nginx-role, namespace: app-nginx}
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "services", "endpoints"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {name: nginx-bind, namespace: app-nginx}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-role
subjects:
- kind: ServiceAccount
  name: nginx-runner
  namespace: app-nginx
---
# app-sql：能读 ConfigMap 和 Secret，但不许删 PVC（防删库）
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {name: sql-role, namespace: app-sql}
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch"]          # 只读，删是不给的
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {name: sql-bind, namespace: app-sql}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: sql-role
subjects:
- kind: ServiceAccount
  name: sql-runner
  namespace: app-sql
```

**③ 用 SA 的 JWT 造一份专用 kubeconfig**（给人用 / 给 CI 用的都这么干）

```bash
TOKEN=$(kubectl -n app-nginx create token nginx-runner)

kubectl config set-cluster prod --server=https://172.31.0.4:6443 \
  --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true \
  --kubeconfig=nginx.kubeconfig
kubectl config set-credentials nginx-runner --token=$TOKEN --kubeconfig=nginx.kubeconfig
kubectl config set-context nginx@prod --cluster=prod --user=nginx-runner \
  --namespace=app-nginx --kubeconfig=nginx.kubeconfig
kubectl config use-context nginx@prod --kubeconfig=nginx.kubeconfig
```

**④ 验证：该通的通，不该通的一律 403**

```bash
export K="kubectl --kubeconfig=nginx.kubeconfig"
$K get pods -n app-nginx          # ✅ 自己的 ns
$K get pods -n app-sql            # ❌ 403 forbidden
$K get secrets -n app-nginx       # ❌ 403 forbidden（role 里压根没 secrets）
$K get nodes                      # ❌ 403 forbidden（ns 级 Role 管不了集群级资源）

# 或者不用 kubeconfig，用 --as 直接模拟身份：
kubectl auth can-i get secrets -n app-sql --as=system:serviceaccount:app-nginx:nginx-runner  # no
kubectl auth can-i get secrets -n app-sql --as=system:serviceaccount:app-sql:sql-runner      # yes
```

> **小结（三个最常犯）**：
>
> 1. **ClusterRole 建了但没 ClusterRoleBinding**（或误配成 RoleBinding）→ 权限只在一个 ns 生效，还以为失效了。
> 2. **namespace 写错位置**：ClusterRoleBinding **没有 namespace 字段**（它天生集群级），RoleBinding 必须写在目标 ns 里。
> 3. **subjects 里的 SA 没带 namespace** → 默认取当前 ns，绑到一个不存在的 SA 上，表现为一直 403 却查不出错在哪。
>
> 另外 RBAC 变更是**即时生效**的，不用重启任何组件；但**已经投射进 Pod 的 token 不会因为改了 Role 就重签**，要等它自然过期或重建 Pod。

### 5.6 最小权限与排障

> **Dashboard 章节那个 `cluster-admin` 绑定只是实验图省事**（1.3.5 第 ③ 步）——生产千万别这么干，「Dashboard 未授权暴露 + cluster-admin」的组合历史上出过整集群被端的大事故。正确姿势：Role 精确到资源、verbs 精确到动作，跨 ns 需求用 ClusterRole + RoleBinding 混搭。
>
> verbs 别偷懒写 `["*"]`：审 RBAC 时重点盯 **create pods / exec / escalate / bind / impersonate** 这几个高危动词——`create pods` + `exec` 组合约等于把节点 root 送出去（起个挂载宿主机 `/` 的特权 Pod 即可逃逸，再配 PodSecurity restricted 基线从准入反向兜住）。

```bash
kubectl auth can-i delete pods --as=system:serviceaccount:dev:ci-bot   # 我能干这个吗
kubectl auth can-i --as=system:serviceaccount:dev:ci-bot --list        # 列出这个身份的全部权限
kubectl auth reconcile -f role.yaml                                    # 按文件对齐 RBAC，避免漂移
```

### 5.7 安全小结

> **安全全流程**：apiserver 的三道门是 **认证（身份从凭证里提取）→ 鉴权（RBAC 三元组：资源 × 动词 × 主体）→ 准入（内置插件 + Webhook + VAP/CEL）**，kubectl 报 401 查认证、403 查鉴权。
>
> 1. **K8s 没有 User 对象**，证书里 CN=用户名 / O=组名，SA token 就是一张 JWT
>2. **1.24 起 SA 不再自动建 Secret token**，用 `kubectl create token` 现签；旧版永久 token 泄露就是永久后门
> 3. **ClusterRole + RoleBinding = 集群权限降级到单 ns**，这是复用官方 `view/edit/admin` 的标准姿势
> 4. **apiGroups / resources / verbs 用复数、别写 `*`**，`pods/log`、`pods/exec` 是独立子资源，得单独授
> 5. **VAP+CEL 先 Audit 后 Deny**；动态 Webhook 的 `failurePolicy: Fail` 会把 webhook 故障放大成全集群不可部署

---

## 6. Helm 与生态组件（Dashboard / Prometheus / EFK）

### 6.1 Helm：K8s 的包管理器

类比 yum/apt（学习路线里就是按这个记的）。

概念：**Chart** = 一套带参数的 yaml 模板包；

**Release** = 一次安装实例；

**Repository** = 仓库。

解决的问题是「部署一个应用 = 手工 apply 十几个 yaml + 逐个手改镜像/域名/副本数」的复制粘贴地狱。

| 概念 | 含义 | yum 类比 |
|------|------|---------|
| **Chart** | 模板包（内含 Deployment / Service / Ingress / values） | `.rpm` 包本身 |
| **Repository** | 存放分发 Chart 的仓库 | yum 源 |
| **Release** | Chart 的一次安装实例（同一 chart 可装多份） | 同一包装出来的多个实例 |

#### Helm 2 vs Helm 3：Tiller 为什么没了

| | Helm 2（已废弃） | Helm 3（现役） |
|---|---|---|
| 架构 | 客户端 + **Tiller**（集群内的服务端 Pod） | 纯客户端二进制，直接读 kubeconfig 调 apiserver |
| 权限来源 | Tiller 自己的 SA，通常是 **cluster-admin** | 沿用操作者的 kubeconfig 权限（RBAC 天然约束） |
| Release 存储 | ConfigMap（谁都能看） | Secret |
| 三方插件 | helm-diff 要靠插件 | 官方 `plugin` 机制 + 大量社区插件 |

> **Tiller 被砍的原因就是个权限放大器**：它是集群里一个手握 cluster-admin 的常驻 Pod，**任何能访问它的人都能间接获得集群最高权限**——等于给 RBAC 开了后门。Helm3 直接删掉 Tiller，现在 Helm 干的事本质就是「渲染 templates → 用你的身份 kubectl apply → 记录 Release 历史」。**看到任何讲 Tiller / `helm init` 的教程直接跳过，那是 Helm2。**

### 6.2 安装与命令速查

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm repo list
helm search repo nginx
```

| 动作 | 命令 |
|------|------|
| 安装 | `helm install <release> <chart>` |
| 指定 ns | `helm install my-redis bitnami/redis -n db --create-namespace` |
| 升级 | `helm upgrade <release> <chart> --set replicaCount=3 -f my-values.yaml` |
| **安装或升级**（CI 幂等写法） | `helm upgrade --install <release> <chart> -f prod-values.yaml` |
| 回滚 | `helm history <release>` → `helm rollback <release> 1`（每次 upgrade 版本 +1） |
| 查生效配置 | `helm get values <release>` / `helm get manifest <release>` |
| 看状态 | `helm status <release>` / `helm list -A` |
| 拉包研究 | `helm pull <chart> --untar` |
| 卸载 | `helm uninstall <release>`（保留历史加 `--keep-history`） |
| 本地渲染 | `helm template <release> <chart>` —— **不连集群，先看最终渲染成啥样** |
| 干跑 | `helm install ... --dry-run --debug` |

> 对比记忆：Helm 的 `install / upgrade / rollback` ≈ 应用级的 `apply / set image / rollout undo`，但管的是**一整套资源**（Deployment + Service + ConfigMap + Ingress 一把梭），版本历史随 Release 走。

### 6.3 自定义 Chart（模板语法速览）

```text
mychart/
├── Chart.yaml          # 名字、版本、appVersion、dependencies
├── values.yaml         # 默认参数（唯一该改的东西）
├── charts/             # 子 chart（依赖）
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── _helpers.tpl    # 公共命名片段，被 include 复用
    └── NOTES.txt       # install 完打印的使用说明
```

| 写法 | 作用 |
|------|------|
| `{{ .Values.replicaCount }}` | 取 values.yaml 的值 |
| `{{ .Release.Name }}` / `{{ .Release.Namespace }}` | Release 名 / 命名空间 |
| `{{ .Chart.Name }}-{{ .Chart.Version }}` | 做标签、做命名 |
| `{{ .Values.image.tag \| quote }}` | 强制加引号（防 `tag: 1.20` 被当数字） |
| `{{ .Values.port \| default 80 }}` | 默认值兜底 |
| `{{- if .Values.ingress.enabled }} ... {{- end }}` | 条件开关（最常用：要不要建 Ingress） |
| `{{- range .Values.hosts }} ... {{- end }}` | 遍历数组 |
| `{{ include "mychart.fullname" . }}` | 复用 `_helpers.tpl` 里的命名片段 |

```yaml
# templates/deployment.yaml 里长这样：参数从 values 取
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repo }}:{{ .Values.image.tag }}"
```

```bash
helm install myapp ./mychart -f prod-values.yaml   # values 覆盖部署
helm lint ./mychart                                 # 语法体检，CI 里值得加一步
helm template myapp ./mychart                       # 本地渲染，看最终 yaml
```

> **踩坑**：
>
> 1. **同一 chart 部署多环境 = 不同 values 文件**（dev-values / prod-values），**别改 templates 里的硬编码**——环境差异全部收敛到 values，这是 Helm 的使用纪律。
> 2. `--set` 里的 `.` 要转义（`--set ingress\.enabled=true`），数组用 `{a,b,c}`；复杂值一律 `-f` 文件，别在命令行堆一串 --set（不可追溯）。
> 3. **upgrade 不会更新 CRD**：Chart 里的 CRD 只在首次安装时创建，chart 升版带了新 CRD 要手工 apply。
> 4. **rollback 不回滚数据**：它会回滚工作负载与配置，但 PV 里的业务数据、已经应用的数据库迁移不会回退——和 `rollout undo` 一个道理。

### 6.4 Dashboard：官方可视化

1.3.5 讲的部署流程，这里补「组件视角 + 安全边界」：

| 项 | 结论 |
|------|------|
| 定位 | 官方 Web UI：看资源、看日志、进终端，排障与演示用，**日常操作仍以 kubectl 为主** |
| 访问方式 | `kubectl proxy`（最安全，只监听 127.0.0.1）> NodePort（内网）/ Ingress（**必须前置鉴权**） |
| 身份 | SA token（`kubectl create token`，短时效）或 kubeconfig |
| 权限纪律 | 常备一个**只读 SA**（`view` ClusterRole + RoleBinding 到目标 ns）；**严禁 cluster-admin + 公网暴露** |

```bash
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard 8443:443
# 浏览器 https://127.0.0.1:8443 —— 只在本地能开，天然免疫公网扫描
```

### 6.5 Prometheus：监控栈

先分清两个常被混淆的东西：

| | **Metrics Server** | **Prometheus** |
|---|---|---|
| 数据留存 | 只存最近一个采样点（内存，重启即丢） | 时序数据库，长期保留 |
| 服务谁 | `kubectl top`、HPA 的 CPU/内存决策（专题 1.3） | Grafana 面板、历史回溯、告警规则 |
| 结论 | 装它只为了让 `top` / HPA 能工作 | 要做可观测必须上 Prometheus，**两者不是替代关系** |

组件流水线：

```
cadvisor（容器指标，kubelet 自带）
node-exporter（节点 CPU/内存/磁盘/网络，DaemonSet 每节点一个）
kube-state-metrics（Deployment/Pod/Node 的对象状态：期望副本 vs 就绪）
应用自有 /metrics（业务埋点）
        │  Prometheus Server 主动拉取（pull）+ 存入 TSDB
        ├─► Grafana        面板可视化（官方 dashboard 直接 import id 用）
        └─► Alertmanager   告警去重 / 分组 / 静默 / 路由（→ 钉钉 / 企微 / webhook）
```

| 组件 | 角色 | 备注 |
|------|------|------|
| Prometheus Server | 拉取 + 存储时序 | Operator 模式下用 **ServiceMonitor** CRD 声明抓取目标 |
| node-exporter | 机器指标 | DaemonSet + hostNetwork（4.1.4 的示例） |
| kube-state-metrics | **资源对象状态**（想要/就绪副本数） | 不是性能指标，专门看对象是否健康 |
| Alertmanager | 告警路由 | Prometheus 只算出告警，通知交给它 |

常用 PromQL：

| 需求 | PromQL |
|------|--------|
| Pod CPU 使用率 | `sum(rate(container_cpu_usage_seconds_total{container!="",pod!=""}[5m])) by (pod)` |
| Pod 内存（工作集） | `sum(container_memory_working_set_bytes{container!="",pod!=""}) by (pod)` |
| 节点 Ready 异常 | `kube_node_status_condition{condition="Ready",status="true"} == 0` |
| Pod 不在 Running | `kube_pod_status_phase{phase!="Running"} == 1` |
| 频繁重启 | `increase(kube_pod_container_status_restarts_total[15m]) > 3` |
| 磁盘将满（线性预测 4h） | `predict_linear(node_filesystem_avail_bytes[6h], 4*3600) < 0` |

```bash
# 一条命令上全家桶（Prometheus Operator + Grafana + Alertmanager + exporters）
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace -f monitoring-values.yaml
```

> **四个黄金指标**是面板设计的起点：**延迟（Latency）、流量（Traffic）、错误（Errors）、饱和度（Saturation）**。告警别盯着 CPU 这种症状指标尖叫，优先告警用户能感知到的：错误率、延迟 P99。
>
> 另一个原则：**告警必须有处置手册**，没人看、没人能处理的告警等于噪音，会被习惯性忽略，真出事时那一条也被一起忽略掉。

### 6.6 EFK：日志栈

| 组件 | 角色 |
|------|------|
| **E**lasticsearch | 存储 + 倒排索引检索 |
| **F**luent Bit / **F**luentd | 采集、过滤、转发（**Fluent Bit 更轻，K8s 首选**；Fluentd 插件生态更全） |
| **K**ibana | 检索界面与可视化 |

两种采集拓扑：

| 模式 | 做法 | 适用 |
|------|------|------|
| **DaemonSet（推荐）** | 每节点一个采集器，tail `/var/log/containers/*.log` | 全集群统一收集，资源省，绝大多数场景 |
| **Sidecar** | 每个 Pod 塞一个采集容器，读共享卷 | 多租户强隔离、应用私有日志格式（业务自己写文件） |

```text
容器 stdout/stderr
    │  kubelet 落盘
    ▼
/var/log/containers/<pod>_<ns>_<容器名>-<id>.log
    │  Fluent Bit（DaemonSet：tail 输入 + kubernetes 过滤，自动补 ns/pod/标签元数据）
    ▼
Elasticsearch（索引按 k8s-<ns>-YYYY.MM.DD 滚动）
    ▼
Kibana（按 ns / pod / 关键字检索）
```

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm upgrade --install fluent-bit fluent/fluent-bit -n logging --create-namespace
```

> **日志第一原则：应用日志必须打到 stdout/stderr**，别往容器内的文件里写——写进文件的话 `kubectl logs` 看不到、EFK 采不到、Pod 删了也没了。这条做不到，后面整套 EFK 都白搭。
>
> **踩坑**：
>
> 1. **ES 吃内存且不支持 swap**：`resources.requests.memory` 给足（实验环境 ≥2Gi），配 `discovery.type: single-node`；**JVM heap 别超过容器 limit 的 50%**，否则照样 OOMKilled。
> 2. **日志暴涨撑爆磁盘**：Fluent Bit 配 `Mem_Buf_Limit` 做背压，ES 配 ILM 生命周期策略（例如 7 天滚动删除）。
> 3. **时区字段配错**：`time_key` / TZ 不对，Kibana 里查"刚刚"的日志一无所获——日志看着丢了其实是时间错位。

### 6.7 组件小结

> Helm 是**打包分发层**（Chart 模板 + Release 版本管理，Helm3 已无 Tiller），Dashboard / Prometheus / EFK 是**可观测与运维层**（看得见 → 指标告警 → 日志定位），和 2.2.3 那张生态组件表对照着看。
>
> 四条核心：
>
> 1. **环境差异全部收敛到 values 文件**，别改 templates；一行 `upgrade --install` 搞定幂等部署
> 2. **Metrics Server ≠ Prometheus**：前者服务 `top`/HPA，后者做长期存储与告警，都要装
> 3. **日志必须走 stdout**，采集优先 DaemonSet 拓扑（Sidecar 留给多租户隔离场景）
> 4. **Dashboard 永远别绑 cluster-admin 放公网**，用 port-forward 或只读 SA

---

## 7. 运维专题：证书续期与 etcd 备份

### 7.1 kubeadm 证书续期（默认 1 年的坑）

kubeadm 签发的 apiserver / controller-manager / scheduler / etcd 证书**默认有效期 1 年**。过期 = 控制面起不来（业务 Pod 还活着，但你失去整个集群的管理能力）。

```bash
# 先会查：证书还剩多少天
kubeadm certs check-expiration
# CERTIFICATE                EXPIRES
# apiserver                  Oct 15, 2026 09:12   ← 30 天内就该动手
```

**续法一（官方命令，续完仍是一年）**：

```bash
kubeadm certs renew all
# 证书换了，静态 Pod 不会自动重载，手动滚一遍控制面：
cd /etc/kubernetes/manifests && mkdir -p /tmp/k8s-bak
mv *.yaml /tmp/k8s-bak/ && sleep 20 && mv /tmp/k8s-bak/*.yaml .
# manifests 里的 Pod 被挪走再放回 = 重启（kubelet 监听该目录）
```

**续法二（改到 10 年）**：改 kubeadm 源码重编译，替换二进制后再 renew：

```bash
# 1. 拉对应版本源码，改 cluster.go：
#    CertificateValidity = time.Hour * 24 * 365 * 10
# 2. go build -o kubeadm cmd/kubeadm，替换 /usr/bin/kubeadm
# 3. kubeadm certs renew all + 重启静态 Pod（同上）
```

> 运维正道不是「十年一劳永逸」而是**监控 + 定期续**：对证书文件做到期告警（<30 天报警，Prometheus 或 cron 跑 check-expiration），配自动 renew。改 10 年适合测试/学习集群省心——生产别指望一次管十年。

### 7.2 etcd 备份（集群唯一状态源）

呼应 2.2.1：**etcd 挂了且没备份 = 集群整个没了**。apiserver 无状态、可重建，etcd 里的数据才是集群本体。

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-$(date +%F).db

# 校验快照
etcdctl snapshot status /backup/etcd-2026-09-15.db
```

> 实操三条：
>
> 1. **异机存放**：快照留在集群同机等于没备份（cron 每日 + 推到对象存储/备份机）
> 2. **定期演练**：没恢复过的备份 = 没有备份，`snapshot restore` 流程要在测试集群走通
> 3. **边界认清**：etcd 备份只救「集群定义」，**PV 里的业务数据不在这**——数据库数据另走 PV 快照 / 应用级备份

---

> 总结：K8s = 「声明式 API + 控制循环」的集群操作系统——kubeadm 装好集群，Deployment 管无状态负载，Service/Ingress 管流量入口，PV/PVC/ConfigMap/Secret 管数据与配置，五大控制器（Deployment/StatefulSet/DaemonSet/Job/CronJob）覆盖所有工作负载形态，RBAC + 准入控制管权限边界，Helm 管打包分发，Prometheus / EFK 管可观测，证书与 etcd 备份保命。
