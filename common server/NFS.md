# NFS 服务

> **定位**：NFS（Network File System）= 让一台 Linux 的目录，像本地磁盘一样被另一台 Linux 挂载读写，是内网里最省事的共享存储方案。
>
> **适用边界**：NFS 认机器不认人（按 IP/网段授权）、明文传输、无加密校验，所以**只在可信内网用**；跨公网、要认证、要 Windows 混合访问的场景另有解决方案（见 1.1 选型表）。
>
> **相关阅读**：Kubernetes 里拿 NFS 当共享存储做 PV/StorageClass 的实操在 [`k8s-note.md` 第 7 章](../container/k8s-note.md)，本文打的是 NFS 自身的地基。

---

## 目录

**一、文件传输工具选型**

1. 几种文件共享方案对比
2. NFS 的定位：为什么选它

**二、NFS 特点与企业使用场景**

1. 什么是 NFS
2. 企业为什么需要 NFS
3. 什么情况不该用 NFS

**三、共享存储的几种形式**

1. DAS / NAS / SAN
2. 从单机盘到分布式存储

**四、NFS 与 RPC 原理**

1. 为什么 NFS 离不开 RPC
2. 服务关系拓扑图
3. RPC 怎么知道 NFS 的端口
4. 一次完整挂载的调用时序
5. NFSv3 与 NFSv4 的端口差异

**五、版本演进**

**六、安装与部署**

1. 包安装
2. 环境准备与目录规划
3. 防火墙与 SELinux
4. 启动顺序与 rpcbind.socket 的坑

**七、NFS 配置文件 /etc/exports**

1. 文件模板
2. 语法格式
3. 客户端地址形式
4. 参数详解
5. exportfs：不重启让配置生效

**八、客户端远程挂载的使用**

1. 查看服务端导出了什么
2. 挂载与挂载选项
3. 卸载
4. 挂载后的一致性验证

**九、自动挂载**

1. /etc/fstab 开机挂载
2. autofs 自动挂载服务
3. 使用场景
4. 检查本地文件系统情况
5. fstab 与 autofs 怎么选

**十、权限模型：root_squash 与 UID 映射**

1. 为什么要把 root 压成普通用户
2. squash 相关参数
3. 权限是三层叠加的结果
4. 实战：统一 UID

**十一、排障速查**

**十二、NFS 与 Kubernetes 联动**

**十三、性能与安全加固**

---

# 一、文件传输工具选型

## 1.1 几种文件共享方案对比

| 方案 | 协议 / 端口 | 主战场 | 是否可挂载为本地目录 | 认证方式 | 适合谁 |
|------|------------|--------|---------------------|---------|--------|
| **NFS** | NFSv3/4，2049 + rpcbind 111 | **Linux ↔ Linux** | ✅ `mount -t nfs` | 仅主机 IP / 网段（AUTH_SYS），可叠加 Kerberos | 集群内部共享存储、K8s PV |
| **Samba / CIFS** | SMB，445 / 139 | **Linux ↔ Windows 混合** | ✅ `mount -t cifs` | 用户名密码 / AD 域 | 办公网文件共享、跨平台目录 |
| **vsftpd / FTP** | FTP，21 + 被动随机端口 | 公网 / 跨组织传文件 | ❌ 只能用客户端上传下载 | 用户名密码 / 虚拟用户 | 对外提供下载、发布包投放 |
| **scp / sftp / rsync** | 走 SSH 22 | 运维日常传文件、备份同步 | ❌ | SSH 密钥 | 临时拷文件、定时备份同步 |
| **对象存储（MinIO / OSS）** | HTTP(S) 80/443 | 海量非结构化数据 | ❌ 走 API/SDK | AK/SK 签名 | 图片视频、日志归档、云原生应用 |

> **`ftp` 和 `nfs` 不是一类东西**：FTP 是「传文件的操作」，传完就结束，两边是各自的副本；NFS 是「挂载远程文件系统」，读写的是同一份数据，没有「传」这个动作。选型时先问自己是「要拷一份」还是「要共用一份」。

> 这里提一嘴，SAMBA在跨操作系统和云服务也有优异表现，不妨学一下

## 1.2 NFS 的定位：为什么选它

NFS 的三个不可替代点：

| 特性 | 说明 |
|------|------|
| **Linux 原生** | 内核态支持，挂载点就是一个普通目录，`cd`、`ls`、`vim`、`dd` 全都能直接用，应用侧零改造 |
| **Linux ↔ Linux 最优解** | 在同平台上，权限语义（UID/GID/POSIX 权限）天然一致，不像 Samba 要做用户映射 |
| **对容器友好** | K8s 的 `volume.nfs`、PV/PVC、nfs-subdir-external-provisioner 全链条成熟，是自建集群的默认共享存储 |

只要有下面任意一条，就弃选 NFS：**客户端是 Windows**、**要跨公网**、**要求传输加密**、**要求按用户（而不是按机器）鉴权**。

---

# 二、NFS 特点与企业使用场景

## 2.1 什么是 NFS

NFS（Network File System，网络文件系统）由 Sun 在 1984 年提出，核心思想是**把远端主机的一段目录空间，通过 RPC 协议映射到本地目录树里**，让远程文件访问和本地文件访问在 API 层面完全一致。

「NFS = remote share file」，不是传文件，是在把别机的**文件系统**接入。

## 2.2 企业为什么需要 NFS

设想 3 台 Nginx 跑同一个站点，每台都放一份 2GB 的用户上传图片：

```text
不用共享存储：              用了 NFS：

Web1 [2GB 图片]             Web1  ┐
Web2 [2GB 图片]   =====>    Web2  ┼──→  NFS Server [2GB 图片 × 1 份]
Web3 [2GB 图片]             Web3  ┘

磁盘：6GB 且三份可能不一致     磁盘：2GB，一份为准，天然一致
```

带来的收益：

| 收益 | 具体表现 |
|------|---------|
| **减少重复存储** | 同一份静态资源只存一份，省磁盘、省备份空间 |
| **减少重复 I/O 请求** | 资源集中在一处，缓存命中集中，不再三台机器各自回源三次 |
| **保证数据一致** | 用户在 Web1 上传的图，Web2/Web3 立刻可见，不需要 rsync 同步脚本 |
| **集中运维** | 备份、扩容、清理只在一台机器上做 |
| **支撑容器编排** | 多副本 Pod 飘到任意节点都能读到同一份数据（ReadWriteMany） |

## 2.3 弃用 NFS

| 场景 | 原因 | 替代 |
|------|------|------|
| 数据库主存储（MySQL data dir） | NFS 的锁与 fsync 语义弱，容易丢数据或死锁 | 本地盘 / 云盘 / 专用数据库存储 |
| 高并发小文件写入 | 每次写都要跨网络一次 RPC，延迟被网络放大 | 对象存储、本地 SSD + 异步同步 |
| 跨公网 / 不可信网络 | NFS 明文、仅靠 IP 授权，端口暴露即裸奔 | SFTP、VPN 内网穿透、对象存储 |
| 要求 HA 的单点敏感业务 | 传统 NFS Server 是单点 | NFS 集群（DRBD + Pacemaker）、CephFS、GlusterFS |

> **最常见的问题**：把 MySQL 的数据目录挂到 NFS 上跑，压测时不是慢，是直接崩。`mysqld` 对 fsync 的依赖很强，而 NFS 客户端的写缓存语义会让数据库以为数据已落盘——断电就是损坏。数据库请用本地盘，要么用支持的一致性更强的分布式块存储。

---

# 三、共享存储的几种形式

## 3.1 DAS / NAS / SAN

| 形式 | 全称 | 连接方式 | 呈现给主机 | 典型产品 | 成本 |
|------|------|---------|-----------|---------|------|
| **DAS** | Direct Attached Storage 直连存储 | SATA / SAS 线直插主板 | 一块本地硬盘 `/dev/sdb` | 服务器内置盘 | 低 |
| **NAS** | Network Attached Storage 网络附加存储 | 以太网（TCP/IP） | **文件系统级**：一个共享目录 `server:/data` | NFS、Samba、群晖 | 中 |
| **SAN** | Storage Area Network 存储区域网络 | 光纤 FC / iSCSI 专用网络 | **块设备级**：一块「网络硬盘」 `/dev/sdc` | FC-SAN、iSCSI、云盘 | 高 |

> **NAS 给你「文件」，SAN 给你「盘」**。NFS 是典型的 NAS。

## 3.2 从单机盘到分布式存储

```text
① DAS 本地盘        快，但数据钉死在一台机器上，机器挂了数据在其在
     │
② NFS / NAS         多台机器共用一份，解决共享与一致性；但 Server 本身是单点
     │
③ NFS 集群          双机 + DRBD 实时复制 + Pacemaker/Corosync 做 VIP 漂移
     │              （原笔记说的「大学/大厂往往采用」就是这一层）
     │
④ 分布式存储        Ceph / GlusterFS / MinIO，多副本、自愈、无单点，运维复杂度显著上升
```

> **中小型公司为什么往往没有专职存储**：
>
> ③ 和 ④ 的成本不止是钱，更是维护能力。一套 Ceph 至少要知道 OSD、MON、PG、CRUSH 这一堆概念才敢上生产；
>
> 而 NFS 只需要 `yum install nfs-utils` + 写一行 `/etc/exports`。
>
> **起步阶段用 NFS，规模到了再迁 Ceph，是被验证过无数次的路线**——K8s 里也是同样的路径：先 NFS PV，再上 StorageClass / CSI。

---

# 四、NFS 与 RPC 原理

## 4.1 为什么 NFS 离不开 RPC

NFS 本身不是一个单一服务，而是一组协作程序的集合（`nfsd`、`mountd`、`statd`、`lockd`、`idmapd` 等）。除了新版 NFS 主协议固定用 **2049**，其余辅助程序启动时向内核申请**随机空闲端口**，每次重启都可能不一样，后续深入了解再进行补充。

这很好引出一个问题：**客户端怎么知道该连哪个端口？**

答案是 RPC（Remote Procedure Call，远程过程调用）。 **111 端口** 的 `rpcbind`（旧称 portmapper）充当「总机接线员」：

| 角色 | 职责 |
|------|------|
| NFS 各服务 | 启动后主动向 111 端口**注册**：「我是 nfs，我在 2049」「我是 mountd，我在 20048」 |
| **rpcbind**（portmapper） | 维护一张「程序号 → 端口号」的注册表，对外只守 111 端口 |
| NFS 客户端 | 先连服务端的 111，问「我要 mount 服务，端口号是多少？」拿到端口后再直连 |

> **这就是必须先启动 rpcbind 再启动 NFS 的原因**（见第六章第 4 节）。rpcbind 没起来，NFS 服务无处注册，客户端连 111 自然无法问询。

## 4.2 服务关系拓扑图

```text
                        ┌──────────────────────── NFS Server ─────────────────────┐
                        │                                                          │
   客户端先问路 ────────>│   rpcbind (portmapper)  监听 111/TCP+UDP                 │
   ① "我要 mount"       │   ┌────────────┬──────────────┬──────────────┐           │
                        │   │ nfs  2049  │ mountd  随机 │ statd  随机  │           │
   拿到端口后直连 ─────>│   │ (固定)     │ (如 20048)  │ (如 32803)   │           │
   ② "连 2049 读写"     │   └────────────┴──────────────┴──────────────┘           │
                        │        ▲            ▲              ▲                    │
                        │        └────────────┴──────────────┘                     │
                        │              启动时全部向 rpcbind 注册自己的端口          │
                        │                                                          │
                        │   /etc/exports  ──>  mountd 读取它判断「这个客户端能否挂载」│
                        │   本地 POSIX 权限 ──> nfsd 最终决定是否放行读写            │
                        └──────────────────────────────────────────────────────────┘
```

## 4.3 RPC 怎么知道 NFS 的端口

注册方向从**服务端内部自下而上**的，不依靠 RPC 去扫描：

```text
rpcbind 先起（占 111）
    │
NFS 各服务启动 → 向本机 111 发起 RPC 注册请求，报上（程序号, 版本, 协议, 端口）
    │
rpcbind 把条目写进注册表
    │
客户端 showmount / mount → 连 111 查询 → 拿到端口 → 直连目标端口
```

用下面命令可以直接看到注册表长什么样：

```bash
rpcinfo -p 172.31.0.4
#  program vers proto   port  service
#   100000    4   tcp    111  portmapper   ← rpcbind 自己
#   100000    3   tcp    111  portmapper
#   100003    3   tcp   2049  nfs          ← NFS 主协议，端口固定
#   100003    4   tcp   2049  nfs          ← NFSv4 同样是 2049
#   100005    1   udp  20048  mountd       ← 挂载服务，端口随机
#   100021    1   udp  32803  nlockmgr     ← 文件锁（v3 才有）
#   100024    1   udp  41776  status       ← statd 状态监视（v3 才有）
```

> **`rpcinfo -p` 是排障好手**。挂不上时先在服务端跑它：看不到 `nfs` 100003 说明 nfs-server 没起来，看不到 `mountd` 说明导出或 mountd 有问题，看到但客户端访问不到 —— 那就是防火墙的问题。

## 4.4 一次完整挂载的调用时序

```text
客户端                                           服务端
  │                                                 │
  │── ① showmount -e server ──连 111 查 mountd────> [rpcbind]
  │<── 应答：mountd 在 20048 ────────────────────────┤
  │                                                 │
  │── ② 问 mountd：/nfs/data 我能挂吗？────────────> [mountd]
  │                              核对 /etc/exports   │
  │<── 放行并返回文件句柄 file handle ───────────────┤
  │                                                 │
  │── ③ 挂载：mount -t nfs server:/nfs/data /mnt ──> [nfsd 2049]
  │                                                 │
  │── ④ 之后所有读写请求直连 2049 ─────────────────> [nfsd]
  │                              核对本地 POSIX 权限  │
```

> **注意 ③ 之后就不再需要 mountd 了**：挂载握手结束，数据通道一直在 2049（或 v3 的对应端口）上。「能挂载但读写卡住」和「压根挂载不上」是两个不同的故障层面——前者权限/网络，后者 exports/服务。

## 4.5 NFSv3 与 NFSv4 的端口差异

| 对比项 | NFSv3 | NFSv4 |
|--------|-------|-------|
| 主端口 | 2049（TCP/UDP） | **2049，固定且只用 TCP** |
| 依赖 rpcbind 注册 | **必须**，客户端要问 111 找 mountd | 挂载也需要，但**协议本身不需要额外查询** |
| 需要 mountd | 是 | 否（挂载能力并入协议） |
| 需要 statd / nlockmgr | 是（锁是外部服务） | 否（协议自带**有状态锁**） |
| 防火墙放行 | 111 + 2049 + 若干随机端口 | 基本只需 **2049**（+111 用于挂载协商） |

> 这就是为什么生产配 NFSv4 能省一堆防火墙开通的口水——只需要一个端口的事，别让安全同学random 一堆端口范围。

---

# 五、版本演进

| 版本 | 年份 | 关键变化 | 现状 |
|------|------|---------|------|
| **v2** | 1989 | 原始 UDP 实现，32 位偏移，单文件上限 2GB | 已淘汰 |
| **v3** | 1995 | 支持大文件、异步写、`READDIRPLUS`；**无状态**设计，服务器重启后客户端能自愈 | 仍在大量使用，兼容性最好 |
| **v4.0** | 2000 | **有状态**、单一 2049 端口、复合操作（compound RPC）、ACL、Kerberos 安全、伪文件系统（pseudo root） | 主流默认 |
| **v4.1** | 2010 | 会话（session）、并行 pNFS、目录委托 | IDC 内并发场景常用 |
| **v4.2** | 2016 | 服务端拷贝 `copy_file_range`、稀疏文件、应用 I/O 提示 | 新内核默认协商到这一档 |

```bash
nfsstat -m           # 客户端：看当前挂载实际协商出的版本
nfsstat -s           # 服务端：统计 v3/v4 各协议调用次数
cat /proc/fs/nfsfs/volumes   # 客户端另一种查看挂载详情的方式
```

> **版本协商的坑**：客户端和服务端会自动协商最高共同版本，但**你可以也应该在挂载时显式指定**。曾经遇到过「服务端配好了 v4，客户端默认协商到 v3，结果多放一堆随机端口」的情况，写 `-o vers=4.2` 一劳永逸。另外老内核（比如 CentOS 6）对 v4.1/4.2 支持不全，会报 `requested NFS version or transport protocol is not supported`，加 `-o vers=4.0` 即可。

---

# 六、安装与部署

> 全文实操基于：服务端 **CentOS 7/8 / Rocky/Alma**（同时给出 Debian/Ubuntu 差异），服务端 IP `172.31.0.4`，客户端 `172.31.0.5`，导出目录 `/nfs/data`。

## 6.1 包安装

| 发行版 | 服务端包 | 客户端包 |
|--------|---------|---------|
| CentOS / RHEL / Rocky / Alma | `nfs-utils` | `nfs-utils` |
| Debian / Ubuntu | `nfs-kernel-server` | `nfs-common` |

```bash
# ---------- 服务端（CentOS 系） ----------
yum install -y nfs-utils rpcbind

# ---------- 服务端（Debian 系） ----------
apt-get update && apt-get install -y nfs-kernel-server

# ---------- 客户端（CentOS 系） ----------
yum install -y nfs-utils

# ---------- 客户端（Debian 系） ----------
apt-get install -y nfs-common     # 注意：不是 nfs-kernel-server，客户端别装服务端的包
```

包里到底装了什么（以 `nfs-utils` 为例）：

| 组件 | 作用 |
|------|------|
| `rpc.nfsd` | NFS 主服务进程，处理读写请求 |
| `rpc.mountd` | 处理挂载请求，读 `/etc/exports` 做授权 |
| `rpc.statd` | 状态监视，配合锁在服务器重启后恢复（v3） |
| `rpc.idmapd` | NFSv4 的 用户名 ↔ UID 映射 |
| `exportfs` | 管理导出表的命令行工具 |
| `showmount` | 查询远端导出列表 |

## 6.2 环境准备与目录规划

```bash
# ① 创建共享目录
mkdir -p /nfs/data

#RHEL 8+ 已经废弃 nfsnobody，系统里只有 nobody（uid 65534）。所以 chown nfsnobody 直接报 "no such user"
# ② 权限先给足，第 10 章会讲为什么这一步经常是隐患根源
chown -R nfsnobody:nfsnobody /nfs/data    # CentOS 默认匿名用户是 nfsnobody (uid 65534)
# Debian 系匿名用户叫 nobody:nogroup
chmod 755 /nfs/data

# ③ 确认匿名用户 id，后面配 anonuid 要用到同一个值
id nfsnobody
# uid=65534(nfsnobody) gid=65534(nfsnobody) groups=65534(nfsnobody)
```

> **目录规划建议**：别直接把 `/` 或 `/home` 这类目录导出。统一用 `/nfs/<业务名>` 前缀，一个业务一个子目录。好处是
>
> ① 导出粒度可控，`/nfs/data/app1` 出问题不影响 `app2`；
>
> ②  K8s 的 nfs-subdir-external-provisioner 场景下，每个 PVC 自动在共享目录下再切一层子目录，相性相当不错。

## 6.3 防火墙与 SELinux

*如果使用的是云服务器，未必有防火墙。可以学习下配置安全组，请提前确认，且不要随意放行*

```bash
# ---------- firewalld（CentOS 系） ----------
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
firewall-cmd --reload
firewall-cmd --list-services          # 验证：应能看到 nfs mountd rpc-bind

# ---------- ufw（Debian 系） ----------
ufw allow from 172.31.0.0/24 to any port nfs

# ---------- SELinux（仅 CentOS 系需要） ----------
getenforce                                    # Enforcing 才需要处理
setsebool -P nfs_export_all_rw on             # 允许 NFS 导出目录读写，-P 表示永久
# 或者给目录打正确的上下文：
# semanage fcontext -a -t public_content_rw_t "/nfs/data(/.*)?"
# restorecon -Rv /nfs/data
```

> **防火墙是「本机 showmount 正常、远端访问不了」的头号内鬼**。排查顺序：
>
> **服务端本地** `showmount -e localhost` → 能通说明服务 OK；
>
> **客户端** `rpcinfo -p <server>` → 连不上 111 就是防火墙/网络；
>
> `mount` 报 `No route to host` → 99% 是防火墙。
>
> 另外注意 `firewall-cmd --reload` 会让**已有挂载短暂中断**，生产环境放在低峰做。

## 6.4 启动顺序与 rpcbind.socket 

```bash
# ① 必须先起 rpcbind（NFS 要向它注册端口）
systemctl enable rpcbind --now

# ② 再起 nfs-server
systemctl enable nfs-server --now

# ③ 检查注册情况
rpcinfo -p | grep -E 'nfs|mountd'
systemctl status nfs-server rpcbind
```

服务名在不同发行版上的差异：

| 发行版 | 服务名 |
|--------|--------|
| CentOS 7+ / RHEL 7+ | `nfs-server`（旧写 `nfs` 也能用，会被别名过去） |
| Debian / Ubuntu | `nfs-kernel-server` |
| CentOS 6 | `nfs`、`rpcbind`（6 之前叫 `portmap`） |

> **`RPCbind 停用也不会停止 NFS，因为 rpcbind.socket`**
> 这不是 bug，是 systemd **socket 激活**机制的必然结果。
>
> `nfs-server.service` 单元里声明的依赖是 `rpcbind.socket`（而不是 `rpcbind.service`）。
>
> socket 激活的语义是：**只要有进程往 111 端口发请求，systemd 就会自动把 rpcbind 拉起来**，而不需要 rpcbind 常驻。因此：
>
> - `systemctl stop rpcbind` 只是停了当前进程，**不会连带停掉 nfs-server**（依赖方向是 nfs → socket，不是 nfs → service）；
> - 反过来，`systemctl start nfs-server` 时 systemd 会自动把 `rpcbind.socket` 点亮；
> - 想验证的话：`systemctl stop rpcbind` 后执行 `rpcinfo -p`，会看到 111 依然响应 —— rpcbind 被 socket 唤醒了。
>
> **结论：管理 NFS 时不要手动动 rpcbind，交给 systemd 处理即可。** 
>
> 反过来推还得到一个排障视角：如果 `systemctl stop rpcbind` 之后 NFS 真的挂了，说明依赖被改过或者版本不支持 socket 激活（老版本 CentOS 6 就没有这套）。

---

# 七、NFS 配置文件 /etc/exports

## 7.1 文件模板

因为版本不同，需要注意写法的差异

```bash
cat /etc/exports
```

```text
# <共享目录>        <客户端地址>(选项1,选项2)    <客户端地址2>(选项)
# ↓ 下面是一个典型模板：给内网网段读写，给备份机只读
/nfs/data           172.31.0.0/24(rw,sync,no_root_squash)  172.31.0.9(ro,sync)
/nfs/public         *(ro,sync,all_squash)
/nfs/upload         172.31.0.5(rw,sync,all_squash,anonuid=1000,anongid=1000)
```

模板解读：

| 行 | 含义 |
|----|------|
| 第 1 行 | `/nfs/data` 给整个 `172.31.0.0/24` 网段读写；另单独给备份机 `172.31.0.9` 只读（**同一目录对不同客户端可以给不同权限**） |
| 第 2 行 | `/nfs/public` 对所有人只读，且所有访问者都压成匿名用户 |
| 第 3 行 | `/nfs/upload` 只允许一台机器写，所有人压成 uid=1000 的用户 |

> **格式极其严格的问题**：客户端地址与紧跟其后的 `(选项)` 之间**不能有空格**。写成 `/nfs/data 172.31.0.0/24 (rw,sync)`（括号前有空格）会被解析成「给 `172.31.0.0/24` 默认权限」+「给名为 `(rw,sync)` 的主机」。这是新手挂出来发现没权限的最经典原因。

## 7.2 语法格式

```text
<共享目录>  <客户端1>(选项列表)  <客户端2>(选项列表)  ...
<共享目录>  @<netgroup>(选项列表)
# 行内 # 之后为注释；可以用 \ 续行
```

规则速记：

| 规则 | 说明 |
|------|------|
| 一个目录一行 | 同一目录要配多个客户端，就写在一行的多个地址里 |
| 默认继承 | 未指定的选项走默认值（默认即 `ro,sync,root_squash,wdelay,subtree_check` 等） |
| 后写的同选项覆盖 | 同一行内冲突选项以最后出现的为准 |
| 修改后不用重启 | `exportfs -arv` 即可（见 7.5） |
| 目录必须真实存在 | 导出一个不存在的目录，`exportfs -r` 会报错 |

## 7.3 客户端地址形式

| 写法 | 含义 | 示例 |
|------|------|------|
| 单个 IP | 指定一台主机 | `172.31.0.5` |
| CIDR 网段 | 一段 IP（**最常用**） | `172.31.0.0/24` |
| 子网掩码写法 | 老式掩码 | `172.31.0.0/255.255.255.0` |
| 主机名 | 需能正向/反向解析 | `node5.internal` |
| 通配符域名 | 一类主机 | `*.example.com` |
| NIS netgroup | `@组名`，集中管理一堆机器 | `@webservers` |
| `*` | 所有主机（**谨慎**） | `*` |
| IPv6 | 需方括号括起 CIDR | `[2001:db8::/32]` |

> **`*` 是高危配置**：等于把共享目录开给整个能路由到你服务器的网络，而 NFS 唯一的鉴权就是 IP 白名单。生产环境写死网段；测试环境图方便写 `*`，上线前务必记得改掉。
>
> 另外主机名形式依赖 DNS，**一旦 DNS 挂了挂载就失败**。生产用 IP 或网段，别用主机名，少一个依赖。

## 7.4 参数详解

### 访问权限

| 选项 | 作用 | 备注 |
|------|------|------|
| `ro` | 只读 | 默认值 |
| `rw` | 读写 | 真正能写还需要本地 POSIX 权限配合（见第 10 章） |

### 落盘策略

| 选项 | 作用 | 取舍 |
|------|------|------|
| `sync` | 服务端写请求落盘后才回 ACK | **默认+安全**，断电最多丢当前请求 |
| `async` | 数据先进缓存就回 ACK | 性能好，服务器宕机**可能丢数据** |

> **性能调优的常见误区**：卡了就把 `sync` 改成 `async`。这是拿数据安全性换 IOPS，跑着日志可能无所谓，跑着数据库/虚拟机镜像会出事。正确顺序是先调 `rsize/wsize`、服务端磁盘、网络，**把 `async` 当最后手段**。

### squash（权限压缩）

| 选项 | 作用 | 风险 |
|------|------|------|
| `root_squash` | 客户端 root（uid 0）压成匿名用户 `nfsnobody` | **默认值，安全** |
| `no_root_squash` | 客户端 root 保持服务端 root 权限 | ⚠️ **高危**，可随意改服务端文件 |
| `all_squash` | 所有用户（含普通用户）一律压成匿名用户 | 公共只读/上传目录常用 |
| `anonuid=<UID>` | 指定匿名用户映射到哪个 uid | 配合 `all_squash` 使用 |
| `anongid=<GID>` | 指定匿名用户映射到哪个 gid | 配合 `all_squash` 使用 |

> **`no_root_squash` 为什么危险**：客户端只要是 root，`/etc`、`/root/.ssh` 随便改。它唯一正当的使用场景是 **K8s 的 nfs-subdir-external-provisioner** —— provisioner 容器要以 root 在共享目录里 `mkdir` 子目录给 PVC 用。即便是这个场景，也应该把导出目录限定在 `/nfs/data` 这种专用路径下，**绝对不要对 `/` 或业务根分区配 `no_root_squash`**。

### 网络与杂项

| 选项 | 作用 |
|------|------|
| `secure` | 要求客户端源端口 < 1024（默认值） |
| `insecure` | 允许源端口 ≥ 1024。**K8s / 容器场景必须加**，因为容器里是普通用户进程在高端口发起请求 |
| `subtree_check` | 检查请求的文件是否在导出子目录内（默认） |
| `no_subtree_check` | 关闭子树检查，**提高可靠性**，现代版本推荐 |
| `wdelay` | 合并多个写请求（默认） |
| `no_wdelay` | 不合并，配合 `sync` 用 |
| `hide` / `nohide` | 是否隐藏嵌套挂载的其他文件系统 |
| `crossmnt` | 允许客户端穿越到导出目录下的其他挂载点 |
| `fsid=0` | 把该导出标记为 **NFSv4 根（pseudo root）** |

## 7.5 exportfs：不重启让配置生效

改完 `/etc/exports` 后**不需要重启 nfs-server**，用 `exportfs` 刷新即可：

| 命令 | 作用 | 何时用 |
|------|------|--------|
| `exportfs -r` | 重新读取 `/etc/exports`，重导所有目录 | **改完配置必执行** |
| `exportfs -a` | 导出 `/etc/exports` 中所有目录 | 初次批量导出 |
| `exportfs -v` | 显示详细导出表（含生效的选项） | **核对配置是否真的生效** |
| `exportfs -arv` | 上面几个的常用组合 | 改完后一条搞定 |
| `exportfs -u <客户端>:<目录>` | 取消某个导出 | 临时下线一个客户端 |
| `exportfs -uav` | 取消全部导出 | 整机关共享 |

```bash
# 改完配置的标准动作
vi /etc/exports
exportfs -arv
# exporting 172.31.0.0/24:/nfs/data

exportfs -v        # 再核一遍，重点看选项是不是自己想要的
```

> **`exportfs -v` 的输出才是真相**。遇到过「改了 exports 但权限没变」的情况，最后一查发现改的是另一个文件（`vi /etc/exports` 手滑）、或者文件里有多条同质覆盖。先 `exportfs -v` 确认服务端视角的权限，再去客户端排权限问题，能省一半时间。

---

# 八、客户端远程挂载的使用

## 8.1 查看服务端导出了什么

```bash
# 列出远端导出了哪些目录、对谁开放
showmount -e 172.31.0.4
# Export list for 172.31.0.4:
# /nfs/data  172.31.0.0/24

showmount -a 172.31.0.4    # 列出哪些客户端正在挂载
showmount -d 172.31.0.4    # 只列出被挂载的目录
```

> **`showmount` 依赖 MOUNT 协议（mountd）**。纯 NFSv4 的服务端如果没跑 mountd，`showmount -e` 会报 `RPC: Program not registered`，但 `mount -t nfs -o vers=4` 照样能挂成功。所以**别用 showmount 判断 NFSv4 服务是否可用**，它是 v3 时代的工具。

## 8.2 挂载与挂载选项

```bash
mkdir -p /nfs/data

# 基础挂载
mount -t nfs 172.31.0.4:/nfs/data /nfs/data

# 推荐写法：显式指定版本与传输协议
mount -t nfs -o vers=4.2,proto=tcp,_netdev 172.31.0.4:/nfs/data /nfs/data

# NFSv3 场景（服务端锁服务正常时）
mount -t nfs -o vers=3,proto=tcp 172.31.0.4:/nfs/data /nfs/data
```

常用挂载选项：

| 选项 | 作用 | 建议 |
|------|------|------|
| `vers=4.2` / `nfsvers=4` | 指定协议版本 | 显式写，别协商 |
| `proto=tcp` | 强制 TCP（UDP 丢包重传表现很差） | 默认建议 TCP |
| `hard` | I/O 失败后**无限重试**，直到服务端恢复 | **默认，生产推荐**（保证数据一致性） |
| `soft` | 超时后返回错误给应用 | 只用于纯读、可容忍失败的旁路数据 |
| `intr` / `nointr` | 允许/禁止中断挂起的 I/O | 新版内核默认允许 |
| `timeo=<n>` | 超时等待时间（单位 0.1 秒） | 默认 600（60s），一般不用改 |
| `retrans=<n>` | 重试次数 | 配合 `timeo` 调 |
| `rsize/wsize=<n>` | 单次读写最大字节数 | v3 上限 32K（或 1M），v4 可到 1M；内网可适度调大 |
| `_netdev` | 标记为网络设备，**网络就绪后再挂载** | **写进 fstab 必加** |
| `nofail` | 挂载失败不阻塞开机 | 推荐与 `_netdev` 一起加 |
| `noatime` | 不更新访问时间 | 减少小文件读的写放大 |

```bash
# 看当前挂载真实生效的选项（协商结果往往和命令行写的不完全一样）
nfsstat -m
cat /proc/mounts | grep nfs
```

> **`hard` 的危险面**：服务端宕机时，`hard` 挂载会让所有访问该目录的进程卡在不可中断睡眠（D 状态），`kill -9` 都杀不掉，`df` 也会卡住。这是它保证一致性的代价。缓解办法是服务端做 HA；客户端侧可以把 `timeo` 调小让失败更快暴露。**不要因此改用 `soft` 挂数据库类数据**——`soft` 超时后返回 EIO，应用会以为写失败了，数据照样错乱。

## 8.3 卸载

```bash
umount /nfs/data              # 正常卸载
umount -f /nfs/data           # 强制卸载（服务端不可达但未繁忙时）
umount -l /nfs/data           # lazy 懒卸载：先摘掉挂载点，等引用结束再真正断开
umount -fl /nfs/data          # 组合拳，服务端失联时的常规手段
lsof +D /nfs/data             # 找出是谁占着
fuser -mv /nfs/data           # 或者用 fuser 定位占用进程
```

> **卸载不掉的标准处理顺序**：① `fuser -mv` 找出占用进程 → ② 让业务停止写入或直接杀进程 → ③ 还不行再 `umount -l`。**直接 `-f` 可能导致数据未刷**，先看清有没有进程在写。

## 8.4 挂载后的一致性验证

四步确认共享生效：

```bash
# ① 看文件系统类型是不是 nfs
df -hT | grep nfs
# 172.31.0.4:/nfs/data  nfs4   40G   8.2G   32G  21% /nfs/data

# ② 服务端写，客户端读
# [服务端]
echo "hello from server" > /nfs/data/probe.txt
# [客户端]
cat /nfs/data/probe.txt            # 能读到 → 共享方向 OK

# ③ 客户端写，服务端读（这一步最容易暴露权限问题）
# [客户端]
touch /nfs/data/from-client.txt
# [服务端]
ls -l /nfs/data/from-client.txt    # 能看到且属主符合预期 → 读写双向 OK

# ④ 看属主映射是否符合预期（第 10 章的验证手段）
# [客户端]
id && touch /nfs/data/whoami.txt
# [服务端]
ls -ln /nfs/data/whoami.txt        # -n 显示数字 uid/gid，用来比对映射关系
```

---

# 九、自动挂载

## 9.1 /etc/fstab 开机挂载

```bash
vi /etc/fstab
```

```text
# <设备>                    <挂载点>     <类型>  <选项>                          <dump> <pass>
172.31.0.4:/nfs/data        /nfs/data    nfs     defaults,_netdev,nofail,vers=4.2  0     0
172.31.0.4:/nfs/backup      /backup      nfs     ro,_netdev,nofail,vers=4.2       0     0
```

| 字段 | 要点 |
|------|------|
| 选项必须含 `_netdev` | 告诉 systemd 这是网络文件系统，**等 network-online 之后再挂**，否则开机时网络没起来就注定失败 |
| 建议加 `nofail` | 挂载失败也允许开机完成，否则会掉进 emergency mode，救援还得抱着显示器去机房 |
| vers 显式写 | 避免内核协商到预期之外的版本 |
| dump / pass 恒为 `0 0` | 网络文件系统不做 dump，也不该在本机 fsck |

```bash
mount -a            # 不改重启，直接按 fstab 重新挂载一遍验证（必须先确保能 mount）
systemctl daemon-reload
```

> **`mount -a` 之前一定要先看 fstab 写得对不对**。fstab 语法错误 + 重启 = 开不了机；`mount -a` 会立刻把错误报出来。这是唯一值得顶礼膜拜的检查习惯。

## 9.2 autofs 自动挂载服务

autofs 的思路和 fstab 相反：**不是开机就挂，而是「用到了才挂、一段时间不用自动断」**。

```bash
# ① 安装
yum install -y autofs            # Debian: apt-get install -y autofs

# ② 配置主映射
vi /etc/auto.master
```

```text
# <挂载点父目录>   <映射文件>              <选项>
/misc              /etc/auto.misc          --timeout=60
/-                 /etc/auto.direct        --timeout=300      # 直接映射写法
```

```bash
# ③ 配置子映射
vi /etc/auto.misc
```

```text
# <key（触发目录名）>   <挂载选项>                        <远端路径>
data                     -fstype=nfs,rw,vers=4.2,soft      172.31.0.4:/nfs/data
backup                   -fstype=nfs,ro,vers=4.2           172.31.0.4:/nfs/backup
```

```bash
# ④ 启动并检查
systemctl enable autofs --now
systemctl status autofs

# ⑤ 触发：只要访问这个路径，autofs 自动挂载
ls /misc/data            # 此刻才真正发起 mount
df -hT | grep /misc/data
```

> **autofs 的「魔法」在于 key 目录在执行 `ls /misc` 时可能是空的**（没挂载时看不到 `data` 目录），但 `cd /misc/data` 却能进去。不要以为配错了——这是 autofs 的正常行为，目录是它按需「变」出来的。

## 9.3 使用场景

| 场景 | 用 fstab | 用 autofs |
|------|---------|----------|
| 常年要用的共享目录（业务数据） | ✅ 简单直接 | ⭕ 可以但不必要 |
| 偶尔访问（备份、软件源、跳板） | ❌ 常年占连接 | ✅ **按需挂载** |
| 客户端数量远多于服务端、且同时在线少 | ❌ 服务端常驻大量连接 | ✅ **用完自动断开，省资源** |
| 服务端可能不在，客户端必须能正常开机 | ❌ 有卡住风险 | ✅ 不访问就完全没影响 |
| 挂载点很多（几十上百） | ❌ 全挂起来拖慢启动 | ✅ 只有真正访问的那几个会挂 |
| 需要在超时后自动清理 | ❌ 挂着就在 | ✅ `--timeout` |

总结：**fstab 是「开机就建立」，autofs 是「用到才建立」**。挂载点多、服务端弱、客户端中有一部分常年不用共享目录时，autofs 是明显更优的选择。

## 9.4 检查本地文件系统情况

```bash
df -hT                          # 看所有已挂载文件系统及类型（T 显示类型）
mount | grep nfs                # 只筛 NFS 挂载
cat /proc/mounts | grep nfs     # 内核视角的真实挂载表，比 mount 输出更可靠
cat /proc/self/mountinfo        # 含挂载传播、超级块选项等更细的信息
nfsstat -m                      # 每个 NFS 挂载协商后的完整选项
findmnt -t nfs4,nfs             # 树形展示挂载关系

# autofs 专项
automount --dumpmaps            # 打印 autofs 解析后的所有映射表（查配置是否生效）
systemctl status autofs         # 服务状态
ls /misc                        # 列出当前已激活的挂载点（未挂载的 key 不显示）
```

> **`df` 卡住时不要慌**：八成是某个 `hard` 挂载的服务端失联了。用 `df -hT -x nfs -x nfs4`（排除 NFS 类型）或者 `df -l`（只列本地）绕开它，然后去处理那个失联的挂载。

## 9.5 fstab 与 autofs 怎么选

```text
这个挂载点是不是每次开机都必须可用？
    ├── 是 → fstab（加 _netdev + nofail）
    └── 否 → 挂载点超过 3 个？ 或者服务端随时可能不在线？
                ├── 是 → autofs
                └── 否 → fstab 就够了，别引入不必要的复杂度
```

---

# 十、权限模型：root_squash 与 UID 映射

## 10.1 为什么要把 root 压成普通用户

NFS 的认证模型极其原始：**服务端相信客户端报上来的 UID/GID**（这就是 `AUTH_SYS`，也叫 AUTH_UNIX）。如果客户端 root（uid 0）发起写请求，服务端会认为「这是 uid 0 写的」——而 uid 0 在服务端就是真正的 root，可以删 `etc`、改 `shadow`。

`root_squash` 就是堵这个漏洞的：**看到客户端说自己是 uid 0，服务端暗自把它换成匿名用户（uid 65534，nfsnobody）**，于是 root 在挂载点上失去特权，只剩普通用户的权限。

```text
客户端 root (uid=0)
    │
    ├─ 服务端配了 root_squash  →  实际以 nfsnobody(65534) 操作
    │                              写不了 root 属主的目录 ❌  （这是保护）
    │
    └─ 服务端配了 no_root_squash → 保持 uid=0，等同服务端 root ⚠️
```

## 10.2 squash 相关参数

| 参数 | 客户端来的身份 | 服务端实际身份 |
|------|--------------|--------------|
| `root_squash`（默认） | uid 0 | 匿名用户 65534 |
| `no_root_squash` | uid 0 | **仍是 uid 0** ⚠️ |
| `all_squash` | 任意 uid | 匿名用户 65534（或 `anonuid` 指定的） |
| `all_squash,anonuid=1000` | 任意 uid | uid 1000 |
| 不加 squash 类参数 | uid 1000 | 仍是 uid 1000（**按数字 UID 认人**） |

> **NFS 认的是数字 UID，不是用户名**。客户端有用户 `web(uid=1001)`，服务端也有同名用户 `web` 但 uid=1002，两边同名不同号 —— 结果就是互相看不到对方文件的属主，权限完全对不上。**跨机器 NFS 共享必须保证 UID/GID 一致**，这是分布式共享存储的通用铁律（LDAP 之类集中账号体系解决的就是这个问题）。

## 10.3 权限是三层叠加的结果

能不能写一个文件，要同时过三关，**任何一关拒绝都写不了**：

```text
① /etc/exports 选项        ro 就是只读，rw 才有可能写
        ↓ 通过了
② root_squash / UID 映射   客户端 root 会被压成 nfsnobody(65534)
        ↓ 通过了
③ 服务端本地 POSIX 权限     nfsnobody 必须对目标目录有 w 权限
        ↓ 通过了
④ SELinux（仅 CentOS 系）    Enforcing 下还要被 SELinux 规则放行
```

排查顺序必须**从上到下**，反向排查会白白浪费时间：

```bash
# ① 服务端：确认导出选项
exportfs -v | grep /nfs/data

# ② 服务端：确认目录本身的属主与权限
ls -ld /nfs/data
ls -ln /nfs/data        # -n 看数字 UID，对比匿名用户 id

# ③ 服务端：确认匿名用户 id
id nfsnobody            # CentOS → 65534；Debian → nobody/nogroup

# ④ SELinux
getenforce
setsebool -P nfs_export_all_rw on      # 确认不是 SELinux 挡的临时验证手段
```

> **经典故障「能挂载但写不了 」** 的根因分布，按我踩过的频率排：**① 服务端本地目录属主是 `root:root` 而客户端 root 被 squash 成 65534（约占一半）→ ② exports 里忘了写 `rw`（默认 ro）→ ③ UID 不一致导致两边看到的属主不同 → ④ SELinux。**
>
> 快速验证手段：临时 `chmod 777 /nfs/data` 若能写，就证明问题在③本地权限层，回头再把权限收紧成正确的属主，别让 777 留在生产里。

## 10.4 实战：统一 UID

假设要给 web 集群提供上传目录，希望客户端上 `www` 用户（uid 1000）写的文件，服务端也认成同一身份：

```bash
# ---------- 服务端 ----------
# ① 所有节点创建一致的用户（关键：显式指定 uid/gid）
groupadd -g 1000 www
useradd -u 1000 -g 1000 -s /sbin/nologin www

# ② 目录属主给到该用户
mkdir -p /nfs/data
chown -R 1000:1000 /nfs/data
chmod 775 /nfs/data

# ③ 导出：不 squash，直接按数字 uid 认人
vi /etc/exports
```

```text
/nfs/data  172.31.0.0/24(rw,sync,no_subtree_check)
```

```bash
exportfs -arv
```

```bash
# ---------- 客户端（每台都执行） ----------
groupadd -g 1000 www
useradd -u 1000 -g 1000 -s /sbin/nologin www

mount -t nfs -o vers=4.2 172.31.0.4:/nfs/data /nfs/data

# 验证：以 www 身份写文件，服务端看到的属主应同样是 1000
sudo -u www touch /nfs/data/test
ls -ln /nfs/data/test
# -rw-r--r-- 1 1000 1000 0 ... test     ← uid/gid 两侧一致，成功
```

> **生产上的更优解**：上百台机器手动同步 uid 是不现实的，LDAP/SSSD 做统一身份之后 NFS 的权限模型才是真正可用的。自己搭建小集群用上面的脚本写 uid 也能顶住，但要把「新增用户必须指定 uid」这条写进运维规范里，否则半年后就会出现权限错乱。

---

# 十一、排障速查

| 症状 | 可能根因 | 处理 |
|------|---------|------|
| `mount: RPC: Remote system error - Connection refused` | 服务端 nfs-server 未启动，或防火墙挡了 | `systemctl start nfs-server`；`firewall-cmd --add-service=nfs` |
| `mount: RPC: Unable to receive; errno = No route to host` | **防火墙**（最常见），或网络不通 | 放行 `nfs`/`rpc-bind`/`mountd`；`telnet <ip> 2049` 验证 |
| `mount: RPC: Program not registered` | 只起了 nfs 没起 rpcbind / mountd 挂了 | `systemctl restart rpcbind nfs-server`；`rpcinfo -p` 核对 |
| `mount.nfs: access denied by server while mounting` | exports 里客户端地址不匹配 / 写了错误网段 | `exportfs -v` 核对；注意括号前不能有空格 |
| `mount.nfs: requested NFS version or transport protocol is not supported` | 版本协商失败（老服务端不支持高版本） | 挂载加 `-o vers=3` 或 `vers=4.0` 显式降级 |
| 能挂载，写文件报 `Permission denied` | 见第 10.3 三层权限模型 | 依次查 exports 选项 → 本地 POSIX 权限 → UID 映射 → SELinux |
| `touch: cannot touch 'x': Read-only file system` | exports 默认 `ro`（忘写 `rw`） | 改 exports 加 `rw`，`exportfs -arv` |
| `Stale file handle`（陈旧文件句柄） | 服务端目录被删除重建、inode 变了，客户端句柄失效 | 客户端 `umount -fl` 后重新 `mount`（**这是唯一解法**） |
| `nfs: server X not responding, still trying` | 服务端宕机 / 网络抖动，`hard` 挂载在重试 | 先恢复服务端；客户端侧检查网络与 `timeo` |
| `df` / `ls` 卡住不动 | 某 `hard` 挂载的服务端失联 | `df -l` 绕开；`umount -fl` 摘掉；服务端做 HA |
| `umount: device is busy` | 有进程占用挂载点 | `fuser -mv` / `lsof +D` 定位 → 停进程 → `umount -l` |
| 开机卡在 emergency mode | fstab 里 NFS 挂载失败 | 加 `nofail,_netdev`；确认 `network-online.target` 已生效 |
| `showmount -e` 无输出但挂载正常 | 纯 NFSv4 未启用 mountd | 正常现象，别用它判断 NFSv4 可用性 |
| 属主显示成 `nobody` / 数字 | UID/GID 两端不一致，或 idmapd 未跑 | 统一 uid（10.4）；`systemctl start nfs-idmapd` |

排障三板斧（按此顺序，能定位九成问题）：

```bash
# ① 服务端视角：服务注册了吗
rpcinfo -p | grep -E 'nfs|mountd'

# ② 服务端视角：导出表对不对
exportfs -v

# ③ 客户端视角：端口通不通、版本协商到哪了
telnet <server> 2049     # 或 nc -vz <server> 2049
nfsstat -m
```

---

# 十二、NFS 与 Kubernetes 联动

NFS 在 K8s 里是最常见的自建共享存储后端，三种用法：

| 用法 | 位置 | 适用场景 |
|------|------|---------|
| Pod 里直接写 `volumes.nfs` | [`k8s-note.md` 7.3](../container/k8s-note.md) | 临时验证，**写死地址路径，不推荐生产** |
| PV + PVC | [`k8s-note.md` 7.4](../container/k8s-note.md) | 静态供给，人工造 PV 池 |
| nfs-subdir-external-provisioner + StorageClass | [`k8s-note.md` 7.6](../container/k8s-note.md) | **生产推荐**，PVC 一申请自动在共享目录切子目录 |

```bash
# 为了让 provisioner 能在共享目录里建子目录，服务端要这样导出
echo "/nfs/data *(insecure,rw,sync,no_root_squash)" > /etc/exports
exportfs -arv
```

参数含义对照本文第 7.4 节：

| 参数 | 为什么 K8s 场景必须 |
|------|-------------------|
| `insecure` | 容器里是普通端口（≥1024）发起请求，不加会被 `secure` 拒绝 |
| `rw` | provisioner 要写子目录 |
| `no_root_squash` | provisioner 以容器内 root 身份 `mkdir`，不 squash 才有权限 |
| `*` | 所有节点都要能挂；生产建议收敛成节点网段 |

> **这里的 `no_root_squash` 是有意为之，但代价要看清楚**：任何能进入共享目录的人，如果拥有集群 root 权限，就能反向改服务端文件。缩小爆炸半径的办法是**导出专用目录（`/nfs/data` 而非 `/`）、并限制 `*` 为节点网段**。另外 K8s 侧要注意：`ReadWriteOnce` 的块存储一个 PV 只能一个节点挂，而 NFS 支持 **ReadWriteMany**，这是它做多副本共享的核心优势。

---

# 十三、性能与安全加固

## 13.1 性能

| 手段 | 做法 | 效果 |
|------|------|------|
| 调大读写块 | 挂载加 `-o rsize=1048576,wsize=1048576` | 减少 RPC 往返次数，大文件明显 |
| 用 TCP | `-o proto=tcp` | 避免 UDP 在大块传输时丢包重传 |
| 关子树检查 | exports 加 `no_subtree_check` | 减少每次请求的路径校验 |
| 合并写 | 保持默认 `wdelay`（配合 `sync`） | 服务端合并相邻写 |
| 服务端磁盘 | SSD / 独立磁盘，别和数据盘抢 IO | 最有效的往往是这一条 |
| 看瓶颈在哪 | `nfsstat -c`（客户端）/ `nfsstat -s`（服务端） | 先看 retrans 是否偏高，高就是网络问题 |

```bash
nfsstat -c        # 客户端侧统计，重点看 retrans（重传率）
nfsstat -s        # 服务端侧统计
nfsstat -o net    # 网络层统计
```

> **`nfsstat` 里若 `retrans` 持续增长，说明是网络丢包而非磁盘慢**。这时候调任何 NFS 参数都是隔靴搔痒，该去查交换机、网卡、MTU（NFS 大块传输对 MTU 敏感，巨型帧能显著降低分片开销）。

## 13.2 安全

| 加固项 | 做法 |
|--------|------|
| **限定网段** | exports 里写 `10.0.0.0/8` 这样具体的网段，**不要 `*`** |
| **慎用 `no_root_squash`** | 只对专用导出目录开，绝不作用于 `/` 或业务根 |
| **只读导出** | 备份、发布包这类只读需求单独一行 `ro` |
| **不得暴露公网** | NFS 端口（111 / 2049 / mountd）**永远不应对公网开放**，这是扫描器的头号目标 |
| **传输加密** | 需要加密时用 `-o sec=krb5p`（NFSv4 + Kerberos），或走 VPN / 专用 VLAN |
| **最小权限原则** | 按业务拆多个导出目录，而不是一个大目录给所有人 rw |
| **NFSv4** | 优先 v4（单端口 + 内建更强的模型），防火墙更好管 |
| **日志监控** | `/var/log/messages` 里关注 `rpc.mountd` 的拒绝记录 |

> **别把 NFS 端口映射到公网**，这条值得单独说三遍。历史上的 NFS 漏洞（rpcbind 信息泄露、mountd 栈溢出、未授权挂载）几乎都建立在「攻击者能连通 111/2049」这个前提上。内网 + 安全组 + 网段白名单三重限制，才是它唯一正确的部署姿势。

---

## 小结

> **总的说**：NFS 靠 RPC 的 111 端口做「服务目录」、2049 端口做「数据通道」，把远端目录接进本地目录树；配好 `/etc/exports` 的四要素（目录 / 客户端 / 选项 / 地址紧跟括号），再用 `mount` 或 autofs 挂上即可。

**十一条**：

1. **选型边界**：Linux↔Linux 用 NFS，跨 Windows 用 Samba，对外传文件用 FTP/SFTP，海量对象用对象存储。
2. **NFS 的价值**：一份数据多处挂载，省重复存储与重复 I/O，天然保证多副本一致。
3. **RPC 是地基**：NFS 辅助服务端口随机，靠 rpcbind(111) 注册查询，所以**必须先起 rpcbind 再起 NFS**。
4. **`rpcbind.socket` 的坑**：socket 激活让 rpcbind 被请求唤醒，停 `rpcbind` 不停 NFS，这是 systemd 的正常行为。
5. **exports 格式严格**：地址与 `(选项)` 之间**不能留空格**，多了空格就变成「给名为 `(rw)` 的主机授权」。
6. **改完不用重启**：`exportfs -arv` 重新导出，`exportfs -v` 核对服务端真实生效的选项。
7. **版本差异**：v3 依赖 rpcbind + mountd + 多随机端口；**v4 基本只要 2049**，生产优先 v4。
8. **自动挂载两条路**：fstab 开机就挂（加 `_netdev,nofail`），autofs 用才挂不用自动断（挂载点多、或服务端可能离线时更优）。
9. **权限三层**：exports 选项 → squash/UID 映射 → 服务端本地 POSIX 权限 → SELinux，**从上到下排查**。
10. **UID 必须一致**：NFS 认数字 UID 不认用户名，两端同名不同号就是权限错乱的根源。
11. **只在可信内网**：NFS 明文、按 IP 授权，111/2049 一旦暴露公网就是等着被扫。

> 踩坑优先级最高的是这三条：**防火墙**、**exports 括号前的空格**、**UID 不一致导致写不了**。
