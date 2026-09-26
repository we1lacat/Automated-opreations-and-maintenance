# Automated Operations and Maintenance

> Personal notes on the path from a single Linux box to container orchestration: **Linux → Shell → Docker → Kubernetes**.
> Written while learning and practising; every command here was run on my own machines first.

> 运维方向的学习笔记仓库，一条从单机到编排的完整链路：
> **Linux 基础 → Shell 编程 → Docker 容器化 → Kubernetes 编排**。

## 目录

- [仓库内容](#仓库内容)
- [建议阅读顺序](#建议阅读顺序)
- [后续计划](#后续计划)
- [笔记版式约定](#笔记版式约定)
- [版本基线](#版本基线)
- [怎么用这份笔记](#怎么用这份笔记)
- [写在最后](#写在最后)

---

## 仓库内容

```text
Linux/           Linux 基础与 Shell 编程
container/       容器化与容器编排
common server/   常用服务
```

| 文件 | 规模 | 覆盖内容 |
|------|------|---------|
| [`Linux-note.md`](Linux/Linux-note.md) | 1109 行 | 一切皆文件、目录结构、文件与目录操作（含 `ls` 长格式逐列详解）、Vim、开关机与登录注销、用户与用户组、**权限管理**（chmod / chown、属主属组、数字与符号模式、setuid·setgid·sticky、批量修复目录权限）、**文本三剑客**（grep / sed / awk、正则 BRE 与 ERE、管道符与重定向）、运行级别、帮助命令；进阶专题：退出登录后任务中断的原因与解法（nohup / setsid / tmux / screen / systemd user） |
| [`shell-note-zh.md`](Linux/shell-note-zh.md) | 870 行 | Shell 编程中文版：shebang、Bash 特性、父 shell 与子 shell 执行环境、变量、字符串操作、命令、脚本开发（函数 / 运算 / 条件 / 循环） |
| [`shell.md`](Linux/shell.md) | 842 行 | 英文原稿，章节一一对应；保留英文是想练原版术语的可以直接读 |
| [`docker-note.md`](container/docker-note.md) | 2462 行 | 安装、Docker 结构与镜像原理、常用命令、容器卷、Dockerfile（含分层缓存与构建实战）、Docker 网络（docker0 / veth pair / 自定义网络）、Compose、Swarm 集群与 stack/secret |
| [`k8s-note.md`](container/k8s-note.md) | 3550 行 | 基础概念与架构 → kubeadm 部署集群（含 Calico CNI、Dashboard、k9s）→ 资源清单 / Namespace / Pod 生命周期 / Deployment（含生产级 web 部署实战）/ Service / Ingress / 存储（PV·PVC·SC·ConfigMap·Secret）/ 调度 / kubectl 排障工具箱 / 资源治理 → 进阶专题（HPA、SQL 上 K8s 可行性、StatefulSet、Job·DaemonSet·CronJob、安全认证鉴权准入与 RBAC（含 CI/CD 凭证实战）、Helm 与生态组件、证书续期与 etcd 备份） |
| [`NFS.md`](common%20server/NFS.md) | 1018 行 | NFS 共享存储：NFS 与 RPC/rpcbind 的端口注册原理、版本演进（v2 → v4.2）、安装部署与防火墙 / SELinux、`/etc/exports` 语法与参数全表、客户端挂载选项、fstab 与 autofs 自动挂载、root_squash 与 UID 映射的权限模型、14 条排障速查、与 K8s PV / StorageClass 的联动、性能与安全加固 |
| `LICENSE` | — | MIT License，Copyright (c) 2026 we1l |

---

## 建议阅读顺序

```text
① Linux 基础        先能在一台机器上干活（目录、权限、用户、进程与会话）
    │
② Shell 编程        把重复操作变成脚本，是所有自动化的入口
    │
③ Docker            单机容器化：镜像怎么分层、数据怎么存、网络怎么通
    │
④ Kubernetes        多机编排：声明式 API + 控制循环
```

> 跳过前两步直接上 K8s 也能学，但**排障时会卡在 Linux / Shell 层面**——比如 Pod 里 `curl` 不通，真正的原因可能是节点 iptables 桥接没开、或者脚本里变量没加引号。前两块是地基。

`shell.md` 与 `shell-note-zh.md` 内容同源，读一份即可；中文版在开头统一了术语（父 shell / 子 shell、内置命令、环境变量等首次出现标注英文）。

## 后续计划

现有笔记覆盖到「容器编排」这一层，在补方向：

| 方向 | 待补内容 |
|------|---------|
| 常用服务 | 堡垒机、DNS、邮件服务器（NFS 已整理，见上表） |
| Web 服务 | Nginx、Tomcat、LVS + Keepalived + HAProxy |
| 数据库 | MySQL、Redis 等主流数据库的部署、备份与故障处理 |
| 自动化 | Ansible、Jenkins |
| 可观测 | Prometheus + Grafana、EFK 日志栈 |

> 补充无法确保及时，整理进度有限，所以更新不定期——目录文件就是当前进度。

## 笔记版式约定

所有笔记遵循同一套写法：

| 约定 | 说明 |
|------|------|
| **表格优先** | 概念对比、命令速查、参数说明一律用表格，方便横向对比与检索 |
| **示例可抄** | 命令与 YAML 都带注释，标出「字段要改、坑」 |
| **`>` 引用块 = 踩坑 / 纠偏** | 版本差异、常见误用、生产事故都写在引用块里 |
| **每章小结** | 收尾复习时只看小结也能回忆起主干 |

版本相关的结论会显式标注，例如 **1.24 起 ServiceAccount 不再自动生成 Secret token**、**1.25 起 PodSecurityPolicy 被 PodSecurity 取代**、**ValidatingAdmissionPolicy 1.30 GA**、**CronJob `timeZone` 1.27+ 稳定**。

## 版本基线

笔记里的实操步骤基于这套环境，版本不同可能有差异：

| 组件 | 版本 / 说明 |
|------|------------|
| Kubernetes | 安装与引导示例（§3.1–3.4）用 kubeadm v1.20.9；CLI 部分的老教程地址也保留其原貌。集群实战与排障（§3.5 Dashboard、CNI、§5.x 安全与 CI/CD、§6 组件）已在 **v1.33.x** 集群上跑过，1.27+ 的 Breaking Change 都在正文里单独标注。v1.33 现已 EOL（当前稳定线 v1.37，2026-08），作为学习环境的实操记录保留 |
| CNI / Ingress | Calico v3.31.6（对应 k8s 1.33）、ingress-nginx v0.46.0（**2026-03 已退役**，正文 §6.1 标注，新集群走 Gateway API） |
| Dashboard | Helm 安装 v3.x（k8s 1.27 起官方只维护 Helm 路线，单文件 `recommended.yaml` 已废弃）；**项目 2026-01 已归档**，SIG UI 继任者为 Headlamp（正文 §3.5 / 进阶 6.4 有说明） |
| 终端 TUI | k9s v0.51.0 |
| Docker | Engine 27.3.1、Compose v2.29.1 |
| 镜像源 | 示例中给出国内镜像（阿里云等），境外拉不动时替换即可 |

## 怎么用这份笔记

1. **系统学习**：按上面的顺序从头读，每章结尾有小结。
2. **排障速查**：直接搜症状关键字——`ImagePullBackOff`、`CrashLoopBackOff`、`Pending` / `FailedScheduling`、`OOMKilled`、`403 forbidden` 等，都有对应的排查路径。

> 说明：这是个人学习笔记，不是官方文档的替代品。笔记里的命令**先在测试环境验证再上生产**，尤其是涉及删除、证书、etcd 恢复的部分。

## 写在最后

> AI 让查资料和整理的速度快了很多，也让人不敢松懈。笔记里的每一条命令、每一份 yaml 都在自己的机器上跑过——实战这块没有捷径，这行做起来比学起来并不会轻松。

> 作为爱好者，天赋不算耀眼。就算想着笨鸟先飞，进度也常被各种事情打断。前辈留下的足迹和时代、技术的变迁走进去才有切肤之感，不要忧虑自己无能无用。
>
> 纵使天空无穷远，但还是祝看到这段话的人保持热爱：能站在今天已经足够幸运，惟愿你能分享我对世界的爱。

## License

[MIT](LICENSE) © 2026 we1l
