# Prometheus + Grafana

> **定位**：Prometheus 是**指标监控**的事实标准——用 Pull 模式周期性抓取各服务暴露的 `/metrics`，把指标存成时序库，用 PromQL 查询，再交给 Grafana 画图、交给 Alertmanager 发告警。它解决的是「**现在有没有事**」，不解决「日志里发生了什么」。
>
> **适用边界**：Prometheus 只吃**数值型时序数据**（CPU、内存、QPS、耗时、队列长度）。日志、链路追踪、事件报表分别属于 Loki / Jaeger / ELK 的活儿，四套数据拼起来才是完整的可观测性三支柱（Metrics + Logs + Traces）。硬拿 Prometheus 存日志，TSDB 会被字符串标签拖垮。
>
> **相关阅读**：K8s 里的部署实操（Ingress、存储、监控自身）在 [`k8s-note.md`](../container/k8s-note.md)；告警邮件的纯配置化流程与规则模板在本文第十二、十三章；Docker 侧采集器的坑在 [`docker-note.md`](../container/docker-note.md)。

---

## 目录

**一、监控体系与 Prometheus 定位**

1. 四类数据与选型边界
2. Prometheus 的四个核心特征

**二、数据模型：一切的地基**

1. 指标名 + 标签 + 值
2. 四种指标类型
3. 高基数：最贵的坑

**三、Pull 机制：target / endpoint / exporter**

1. 三个概念必须分清
2. 服务暴露的四种方式
3. 为什么 Prometheus 坚持 Pull
4. node_exporter
5. port 的解决策略

**四、配置 Prometheus**

1. prometheus.yml 骨架
2. scrape_configs 与 relabel
3. 定义规则：recording vs alerting

**五、Prometheus 数据储存**

1. TSDB 的结构
2. 内存怎么算
3. 磁盘会涨到多大

**六、查询和数据结构**

1. 两种查询
2. 必须先背下来的算子
3. 聚合与分组
4. 常用排障表达式
5. PromQL 的陷阱

**七、Prometheus 角色与集群**

1. 单机够用吗
2. 联邦与远端存储
3. 集群不等于高可用

**八、K8s 部署与监控（helm operator）**

1. kube-prometheus-stack 组件清单
2. Operator 与四个 CRD
3. instance / job 的区分
4. endpoint 角色
5. 相同目标的 collection
6. 开放可视化 UI

**九、Grafana**

1. 数据源
2. 仪表盘
3. 告警与权限
4. dashboard 迁移

**十、第三方应用程序监控和部署**

1. exporter 是什么
2. client 库：不必在意如何暴露服务
3. 定义衡量标准：RED 与 USE
4. 在逻辑关系追踪值
5. nginx 为例
6. Redis 为例
7. k8s charts 部署 exporter
8. endpoint 发现
9. 数据源如何引入 Grafana dashboard

**十一、实战：双机 + 隧道监控架构**

**十二、告警规则与 Ruler**

**十三、Alertmanager**

**十四、排障速查**

**十五、常见误区**

---

# 一、监控体系与 Prometheus 定位

## 1.1 四类数据与选型边界

| 数据类型 | 长什么样 | 典型问题 | 归属组件 |
|---------|---------|---------|---------|
| **指标 Metrics** | `93.2`、`1043 req/s`、`0.87` 这样的**数值** | 现在有没有事？CPU 够不够？ | Prometheus |
| **日志 Logs** | 带时间戳的**文本行** | 出错了吗？具体为什么？ | ELK / Loki / 文件 |
| **链路 Traces** | 一次请求跨服务的**调用链** | 慢在哪一段？ | Jaeger / SkyWalking |
| **事件 Events** | 离散的**状态变更** | 谁在什么时候改了配置 | 审计日志 / Event |

四者关系用一句话串起来：

```text
Metrics  告诉你「有事发生」      → 立刻报警
Traces   告诉你「卡在哪一步」    → 定位到具体服务/接口
Logs     告诉你「到底报什么错」  → 拿到根因
Events   告诉你「谁改的」        → 定责
```

> **先想清楚「要回答什么问题」再决定采什么指标**。很多团队上监控失败不是工具选错，而是采了一堆没人看的指标（`go_gc_duration_seconds` 没人查、`process_resident_memory_bytes` 没人关心），真出事时依然定位不了。**指标的落点是「能触发一个动作」**——触发扩容、触发告警、触发回滚；否则就是负债。

## 1.2 Prometheus 的四个核心特征

| 特征 | 说明 | 带来的后果 |
|------|------|-----------|
| **多维数据模型** | 每条时间序列由 `指标名{标签=值,...}` 唯一标识 | 标签是维度语言，但用不好会炸内存（见 2.3） |
| **Pull 为主** | Prometheus 主动去各个 target 拉 `/metrics` | 天然知道「谁还活着」——`up` 指标；同时要求目标可达 |
| **PromQL** | 专门为时序数据设计的查询语言 | 能直接做 `rate()`、分位数、预测，SQL 干不了这些 |
| **本地时序库** | 单二进制，TSDB 自己管自己 | 部署极简，但需要自己面对高可用和长期存储 |

> **「Pull 还是 Push」不是信仰之争，是场景之争**。Prometheus 官方只在一个场景推荐 Push：**批处理任务**（Job 跑完就退出，Prometheus 根本轮询不到它），这时用 Pushgateway 临时存一下结果。日常微服务一律 Pull——因为 `up == 0` 这个指标本身就有 enormous（巨大的）价值：Prometheus 一秒钟就能告诉你「B 机上的 exporter 挂了」，而 Push 架构里服务挂了是**静默的**，没人通知你。

---

# 二、数据模型：一切的地基

## 2.1 指标名 + 标签 + 值

一条时间序列由三部分构成：

```text
指标名（metric name）  标签集合（labels）              值（value）
node_cpu_seconds_total{instance="10.0.0.1:9100",   1234.5
                        cpu="0",                       ↑ 随时间不断变化的数值
                        mode="idle"}
                       ↑ 维度：描述「这条序列代表什么」，可以任意增删
```

**指标名只能是这一套**：`^[a-zA-Z_:][a-zA-Z0-9_:]*$`。不能用 `-`、`.`、空格、斜杠，中文也不行（`exporter_name="中文"` 会被静默丢弃，不报错——这是个大坑）。

唯一性规则：

```text
序列 ID = 指标名 + 排序后的全部标签
          ↓
同一条指标在同一台机器上，标签不同 = 两条独立序列
同一条指标标签完全一样、时间不同 = 同一序列上的一串点
```

由此推出两条铁律：

| 铁律 | 反例 | 正确做法 |
|------|------|---------|
| **值永远不能进标签** | `user_id="10086"` | 指标名固定，用 `count` / `histogram` 聚合掉 |
| **标签值集合要有限** | `url="/api/user/12345"` | 路由归一化成模板，或干脆不采 |

## 2.2 四种指标类型

| 类型 | 语义 | 只能用的算子 | 命名要求 | 典型用途 |
|------|------|------------|---------|---------|
| **Counter** | 只增不减的累计量 | `rate()` `increase()` **不能用 `sum()` 直接取值** | 必须以 `_total` 结尾 | 请求数、错误数、字节数 |
| **Gauge** | 瞬时值，可升可降 | 直接取值、`avg()` `max()` | — | 内存占用、队列长度、温度 |
| **Histogram** | 分桶计数 | `histogram_quantile()` `rate(_bucket)` | 必须带 `_bucket` `_sum` `_count` | 请求耗时、响应大小 |
| **Summary** | 分位数（客户端算） | `quantile()` | 名字带 `quantile` 参数 | 少量场景，**Prometheus 官方不推荐** |

```yaml
# 一个规范的 exporter 暴露（节选）
node_cpu_seconds_total{cpu="0",mode="idle"}      1234.5    # Counter
node_memory_MemAvailable_bytes                  987654321  # Gauge
http_request_duration_seconds_bucket{le="0.05"}  982       # Histogram
http_request_duration_seconds_sum{...}           12.34
http_request_duration_seconds_count{...}         1000
```

> **Counter 为什么不能直接 `sum`**：`http_requests_total` 是「从进程启动到现在累计请求数」，直接 `sum` 得到的是一个无意义的巨大数字。必须用 `rate(x[5m])` 求「每秒多少」。同理，**用 `increase(x[1h])` 求一小时内的量**，而不是 `delta()`——后者会被 Counter 重启清零打断。
>
> **Histogram 和 Summary 的取舍**：Summary 的分位数在客户端算，精度高但**不可跨实例聚合**（3 个 Pod 各自的 99 分位没法合并成一个全局 99 分位）；Histogram 分位数在查询时用 `histogram_quantile` 聚合，可跨实例、可调桶。所以 Prometheus 官方推荐 **Histogram**，你看到的 p99 永远应该是 `histogram_quantile(0.99, sum by (le) (rate(..._bucket[5m])))` 这个形状，不是 exporter 直接给的。

## 2.3 高基数：最贵的坑

**高基数 = 序列数量爆炸 = 内存和磁盘爆炸。**

| 场景 | 标签 | 序列数 | 后果 |
|------|------|--------|------|
| 正常 | `instance` × 200 台 | 200 | 正常 |
| 加 `pod` | × 每台 30 个 Pod | 6000 | 开始吃力 |
| 再加 `url` 全路径 | × 每 Pod 200 条路由 | 1,200,000 | **内存打满，Prometheus 崩溃** |
| 再加 `user_id` | × 百万用户 | 天文数字 | 直接 OOM |

内存的粗算公式（记这个就够日常用了）：

```text
内存占用 ≈ 序列数 × (1~2 KB) + 活跃 chunk 大小 + 查询开销

估算示例：
  4,000 条序列 × 1.5 KB ≈ 6 MB      → 小规模，几十 MB 足够
  200,000 条序列 × 1.5 KB ≈ 300 MB   → 中等规模，1G 内存起步
  2,000,000 条序列 × 1.5 KB ≈ 3 GB   → 上限区，head 块直接吃掉几 G
```

排障时第一时间查序列总数：

```bash
curl -s http://localhost:9090/api/v1/status/tsdb | python -m json.tool
# 看 headStats.series（当前活跃序列）和 seriesCountByMetricName（按指标名分布）

# 按指标名找元凶
curl -s http://localhost:9090/api/v1/status/tsdb \
  | python -c "import json,sys; d=json.load(sys.stdin)['data']['seriesCountByMetricName']; [print(v,k) for k,v in sorted(d.items(), key=lambda x:-x[1])[:15]]"
```

> **踩过的坑**：某次 `topk(5, http_requests_total)` 出来的 label 列表里赫然出现着一整串 `path="/api/v2/tenants/acme-1234567/orders/..."`。结论是**把原始 URL 直接塞进了标签**。修法是 exporter 侧加路由归一化（只保留 `/api/v2/tenants/{id}/...` 模板），或者干脆在采集层用 `metric_relabel_configs` 丢掉。**不要指望 Prometheus 自己长记性**——它的设计就是「你给什么标签它就存什么」。
>
> 另一个隐蔽来源：**`target` 标签里带了 Pod IP + 随机端口**。K8s 里 Pod 重建换 IP，历史序列全部成为孤儿序列留在磁盘上，直到 retention 过期。TSDB 的 `Isolated` 指标如果长期在涨，就是这个原因。

---

# 三、Pull 机制：target / endpoint / exporter

## 3.1 三个概念必须分清

速记里这三个词经常混着用，但它们在 Prometheus 里是严格分开的：

| 概念 | 定义 | 在哪体现 | 例子 |
|------|------|---------|------|
| **target** | 一个被监控的**对象**，通常是「主机 + 端口」 | scrape_configs 里的 `static_configs` | `10.0.0.1:9100` |
| **endpoint** | target 的**抓取地址 + 标签集**，可以多个 endpoint 归到一个 target 下 | K8s SD 生成的 `__meta_*` 标签 | 同上，另带 `__meta_kubernetes_pod_node_name` |
| **exporter** | 真正**产生并暴露** `/metrics` 的那个程序 | 目标端口上的 HTTP 服务 | node_exporter、mysqld_exporter |

```text
   [Prometheus]  ──HTTP GET /metrics──>  [exporter]  ──采集──>  [被监控的中间件/应用]
      target:                              node_exporter
   10.0.0.1:9100                           :9100/metrics
      endpoint:                             →
   instance=10.0.0.1:9100                [内核 /proc、/sys、statfs]
```

`/metrics` 返回的原始文本长这样（这是 exporter 的"数据源"，Prometheus 负责解析成序列）：

```text
# HELP node_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 3.128271182
node_memory_MemAvailable_bytes 1.254983424e+08
```

三行约定：注释行 `# HELP`（说明）、`# TYPE`（类型）、`# 指标名{标签} 值`（数据）。**`# HELP` 和 `# TYPE` 会被 Prometheus 原样存进元数据**，所以它们也算存储成本之一。

## 3.2 服务暴露的四种方式

| 方式 | 谁发起 | 适用 | 代价 | 评价 |
|------|-------|------|------|------|
| **原生暴露** | 应用自己实现 `/metrics` | Go/Java 写的自研服务 | 要改代码 | 最干净，Prometheus 官方推荐 |
| **exporter 代理** | exporter 拉取中间件数据再暴露 | MySQL、Redis、Nginx、主机 | 装一个进程 | **现实主力**，覆盖 90% 需求 |
| **Pushgateway** | 任务自己 push | 短生命周期批处理 Job | 标签会串、需自己清理 | 只用于 Job，别乱用 |
| **服务发现** | Prometheus 主动发现 target | K8s、Consul、Etcd | 配置略复杂 | 动态环境必需 |

```bash
# 方式②：exporter 代理的标准三件套
mysqld_exporter --config.my-cnf=/etc/my.cnf          # 监听 9104
redis_exporter --redis.addr=127.0.0.1:6379            # 监听 9121
node_exporter --web.listen-address=:9100              # 监听 9100
# 然后 Prometheus 侧：
#   - targets: ['10.0.0.1:9104']   labels: { service: mysql }
```

> **Pushgateway 的两个经典错误**（官方文档专门开了一节警告）：
>
> **① 标签污染**：一个 Batch Job 实例 push 了 100 万条序列，Job 结束后这些序列**不会自动消失**，会一直留在存储里累积到 retention 过期。解法是 Job 结束时显式 `curl -X DELETE http://pushgateway:9091/metrics/job/xxx/instance/yyy`。
>
> **② 忘记分组**：`push_time` 这类每次都变的标签，等于每次 push 都创建一批全新序列。官方建议只 push 业务指标，把时间戳放在 value 里。
>
> **判断要不要用 Pushgateway 的唯一标准：这个任务 Prometheus 能不能主动拉到？** 能拉（哪怕是常驻服务）就用 Pull；只有「跑完就没了」的任务才 Push。

## 3.3 为什么 Prometheus 坚持 Pull

| 角度 | Pull 的优势 |
|------|------------|
| **可观测性** | `up == 0` 立刻告诉你 target 挂了；Push 模式下服务静默死亡你永远不知道 |
| **安全性** | Prometheus 主动出网，被动暴露面小；不需要给每个服务开入站端口 |
| **服务解耦** | 服务不需要知道谁在监控它，加/减监控不影响业务代码 |
| **一致性** | 所有 target 在同一时刻被抓取，跨主机对比不会出现时间错位 |
| **问题** | 需要网络可达（跨网络/跨防火墙时就得靠隧道）；短生命周期 Job 抓不到 |

## 3.4 node_exporter

主机级监控的事实标准，读的是内核 `/proc` 和 `/sys`，不装 agent 之外的东西。

```bash
# ---------- Debian / Ubuntu ----------
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar xzf node_exporter-1.8.2.linux-amd64.tar.gz
mv node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/
cat > /etc/systemd/system/node_exporter.service <<'EOF'
[Unit]
Description=node_exporter
After=network.target
[Service]
ExecStart=/usr/local/bin/node_exporter --web.listen-address=:9100
User=nobody
Restart=always
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload && systemctl enable --now node_exporter

# ---------- 关键端点 ----------
# ① 只看 node_ 前缀（其余如 go_* 是 exporter 自身运行时指标）
curl -s http://localhost:9100/metrics | grep '^node_' | head
# ② 确认某个指标存在（排障第一步）
curl -s http://localhost:9100/metrics | grep 'node_memory_MemAvailable_bytes'
# ③ textfile collector 落地目录
systemctl cat node_exporter | grep textfile
```

**textfile collector 是 node_exporter 最有用的扩展**：任何脚本只要把指标写成 `.prom` 文件丢进指定目录，node_exporter 下次抓取就会把它一起吐出来。用于采集「exporter 覆盖不到」的自定义数据。

```bash
# /var/lib/node_exporter/textfile_collector/ 下写 .prom 文件
cat > /var/lib/node_exporter/textfile_collector/nfs_mount.prom <<'EOF'
# HELP nfs_mount_ok Whether the NFS share is mounted.
# TYPE nfs_mount_ok gauge
nfs_mount_ok{mountpoint="/nfs/data"} 1
# HELP nfs_mount_stale_age_seconds Age of the mount since last successful stat.
# TYPE nfs_mount_stale_age_seconds gauge
nfs_mount_stale_age_seconds{mountpoint="/nfs/data"} 0
EOF
# 验证：指标应该已经出现在 /metrics 里
curl -s http://localhost:9100/metrics | grep nfs_mount_ok
```

> **`textfile` 的两个注意点**：① 文件必须以 `.prom` 结尾，命名规则用下划线而不是点（`foo.bar.prom` 不行，`foo_bar.prom` 可以）；② 写文件要**先写临时文件再 `mv`**（`mv` 在同一文件系统内是原子的），否则 Prometheus 会读到写了一半的半截文件，解析报错刷满日志。
>
> 配合 `systemd.timer` 定时刷新即可，比如每 30 秒探测一次 NFS、每 5 分钟探测一次证书有效期——见本文第十六章的踩坑。

## 3.5 port 的解决策略

速记里这条说的是监控部署时端口怎么规划。核心矛盾是：**监控组件要占端口，端口不够就会冲突**。

| 场景 | 端口冲突原因 | 解决策略 |
|------|------------|---------|
| 一台机跑多个 exporter | 9100 被占 | 用 `--web.listen-address=:9110` 换端口；**生产建议按服务分配固定段**（9100 node、9104 mysql、9121 redis、9273 postgres） |
| K8s 里 exporter 要被集群访问 | 只能监听 127.0.0.1 不够 | 改成 `0.0.0.0`，但**只监听在 Pod IP 上不监听公网** |
| 多个 Prometheus 实例 | 都想要 9090 | 局域网直连用不同 IP；K8s 里改用 Pod IP 直连，**不映射到宿主机** |
| UI 只想本机访问 | 暴露公网有风险 | `127.0.0.1:3000` + SSH 隧道（推荐）或 Ingress + 认证 |
| 容器 hostPort 抢宿主端口 | 节点间冲突 | 优先 Service；非要 hostPort 就每节点调度约束 |

```text
【错误示范】为了省事把监控全塞 hostPort
  node01:host31090 → Prometheus  占用 node01 的 9090
  node02:host39090 → Prometheus  占用 node02 的 9090   ← 三个节点三套地址
  结果：Service 的 selector 只能选中一个，其他 target 全 down
        而且 Prometheus 自己也发现不了「同集群有三个自己」

【正确示范】集群内直连，不碰宿主机端口
  Prometheus（StatefulSet，headless Service prometheus-operated）
      ↓ 用 Pod IP + 9100 抓 node-exporter
  node-exporter 在每个节点上监听 9100（DaemonSet 天然每节点一个）
      ↓ ServiceMonitor 按 label 选中
  所有 target 都在集群内网，宿主机端口完全干净
```

> **端口规划的黄金法则**：**监控系统自己的端口不要和被监控的服务端口混在同一段**。混淆会导致「9090 是 Grafana 还是 Prometheus」的经典问题，也让防火墙策略没法按段写。给自己的监控栈划一段专用高位端口（9100~9199、9090~9099），一眼就知道是谁。
>
> 另外 Grafana 官方就是默认 `127.0.0.1:3000` 只监听本机，这不是疏忽而是**有意的安全默认值**——需要远程访问时用 SSH 隧道 `-L 3000:127.0.0.1:3000`，比开公网端口 + 设密码安全得多。

---

# 四、配置 Prometheus

## 4.1 prometheus.yml 骨架

```yaml
global:
  scrape_interval: 15s            # 抓取间隔
  scrape_timeout: 10s             # 单次抓取超时，必须 < scrape_interval
  evaluation_interval: 15s        # 规则评估间隔
  external_labels:                # 打到远端存储/告警时会附加的标签
    monitor: "a-machine"
    env: "prod"

rule_files:
  - /etc/prometheus/rules/*.yml   # 规则目录（不是配置文件目录）

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["127.0.0.1:9093"]   # Alertmanager 地址

scrape_configs:
  - job_name: 'node'
    scrape_interval: 15s
    static_configs:
      - targets: ['127.0.0.1:9100']
        labels:
          env: prod
```

```bash
promtool check config /etc/prometheus/prometheus.yml   # 语法自检，改完必跑
systemctl restart prometheus
curl -s http://localhost:9090/api/v1/status/config     # 看生效的最终配置
```

> **`scrape_timeout` 必须小于 `scrape_interval`**，否则 Prometheus 会因为「上一个抓取还没结束就发下一个」而堆积任务，最终整个采集线程卡死。这个约束很容易在从 15s 改到 60s 时被忘掉。改完记得同步检查超时值。
>
> **改配置前先 `promtool check config`**：Prometheus 启动时如果配置有语法错误会直接退出，配错一个 YAML 缩进就是监控全盲，比没有监控更危险——因为你以为有人在看。

## 4.2 scrape_configs 与 relabel

```yaml
  - job_name: 'multi-target'
    metrics_path: /metrics        # 暴露路径，非 /metrics 时要改
    scrape_interval: 15s
    scrape_timeout: 10s
    honor_labels: true            # 目标自带的 label 优先于我加的（很少用）
    params:
      module: [http_2xx]          # 带查询参数抓取（部分 exporter 支持）
    static_configs:
      - targets:
          - 10.0.0.1:9100
          - 10.0.0.2:9100
        labels:
          group: 'cn-north'
    relabel_configs:              # 抓取前改写 target 标签，可丢弃 target
      - source_labels: [__address__]
        regex: '10\.0\.0\.2:.*'
        target_label: '__address__'
        replacement: '127.0.0.1:9100'   # 抓远端改抓本地隧道
      - source_labels: [__meta_kubernetes_pod_node_name]
        target_label: 'node'
```

**relabel 的两个时机，作用完全不同**（初学必混）：

| 配置项 | 时机 | 作用 | 典型用途 |
|--------|------|------|---------|
| `relabel_configs` | **抓取前** | 改写/丢弃 **target**（连接层面） | 过滤掉不该抓的实例、重写 `__address__` |
| `metric_relabel_configs` | **抓到之后** | 改写/丢弃**样本**（数据层面） | 删掉高基数标签、丢弃无用指标 |

```yaml
    # 抓取后丢弃特定指标：能显著降低存储压力
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'go_memstats_.*|go_gc_.*|process_.*'
        action: drop
```

> **`action: drop` 是控制高基数的第一道防线**。很多 exporter 会暴露一堆 Go 运行时指标（`go_*`）和进程指标（`process_*`），单条无害，但 200 台机器 × 每条几十个指标就是几千条没用的序列。加一条 `drop` 直接省掉。
>
> relabel 是**正则驱动的**，写错了不会报错只会静默不生效。调试方法：`promtool` 没有 relabel 的 dry-run 命令，最快的办法是看 `http://localhost:9090/targets` 页面里每个 target 的 `Discovered labels`，那里展示的就是 relabel 处理**之后**的最终标签，一眼能看出规则是否命中。

## 4.3 定义规则：recording vs alerting

速记里「定义规则」是这两件事的总称，它们作用完全不同：

| 类型 | 作用 | 产出 | 是否发告警 | 典型用途 |
|------|------|------|-----------|---------|
| **recording rule**（记录规则） | 把高频复杂查询**预先算好**存成新时间序列 | 新指标 | 否 |  dashboards 上把 10 秒查询变成 0 秒 |
| **alerting rule**（告警规则） | 表达式超阈值时进入告警状态 | 告警事件 | 是 | 见第十二章 |

```yaml
groups:
  - name: node_recording
    interval: 30s                 # 本组规则的评估间隔，可覆盖全局
    rules:
      # ① 记录规则：把「按实例分组的 CPU 使用率」预先算好
      - record: node:node_cpu_utilisation:rate5m
        expr: |
          1 - avg without (cpu, mode) (
            sum without (cpu, mode) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
          ) by (instance)

      # ② 告警规则：基于上面的记录规则再判断
      - alert: NodeHighCpuLoad
        expr: node:node_cpu_utilisation:rate5m > 0.85
        for: 10m                 # 持续 10 分钟才真报警
        labels:
          severity: warning
        annotations:
          summary: "CPU 高负载 {{ $labels.instance }}"
```

> **为什么要用 recording rule**：Grafana 面板上写 `1 - avg without(cpu,mode)(sum without(cpu,mode)(rate(node_cpu_seconds_total{mode="idle"}[5m]))) by (instance)` 这种表达式，刷新一次就是几十秒的全量计算，多个面板同时刷会直接把 Prometheus 的 CPU 打满。改成记录规则预先算成 `node:node_cpu_utilisation:rate5m`，面板查询就只剩一个简单取值。
>
> **命名规范**：官方推荐 `level:metric:operations`，如 `node:node_cpu_utilisation:rate5m`。看到冒号分层就知道这是记录规则不是原始指标——排查时能立刻判断「这个值是采集来的还是算出来的」。`job:xxx` 也是同一套。
>
> 记录规则的**第一个规则必须写成能覆盖所有实例的宽查询**，后面的规则才能引用它。否则新加的机器因为没有历史记录规则，告警永远不触发——这是「加了新节点但告警不响」的常见根因。

---

# 五、Prometheus 数据储存

## 5.1 TSDB 的结构

```text
写入路径
  样本 → WAL（预写日志，先落盘保证不丢）
       → Head 块（最近 2 小时，常驻内存，WAL 每 2 小时切一次）
       → 持久化块 Persistent Block（2 小时一块，压缩）
       → 每块内按 series 分 chunk（每 chunk 存 120 个样本）
       → 到达 retention 后整块删除

查询路径
  查询 → label 倒排索引（标签值 → series ID 列表）
       → 定位 series → 读它的 chunk（先查内存 Head，再查磁盘块）
       → 合并 → 返回
```

| 概念 | 说明 |
|------|------|
| **chunk** | 一个 series 的一小段连续样本，默认 120 个样本/chunk |
| **Head 块** | 最近约 2 小时，**唯一常驻内存的部分**，WAL 负责崩溃恢复 |
| **持久化块** | Head 切出来的块，**存在磁盘上、已压缩、不可变** |
| **WAL** | 预写日志。Prometheus 先写 WAL 再返回写入成功，崩溃后从 WAL 恢复 |
| **compaction（压实）** | 把小 chunk 合并成大 chunk，压实（compacted）标记的块**可被删除** |

## 5.2 内存怎么算

```text
内存 ≈ 活跃序列数 × 1~2KB（索引 + Head chunk）+ 查询工作区

关键参数：
  --storage.tsdb.head-chunks-size     Head 块内存上限（0 = 自动）
  自动策略：约为进程可用内存的 2/3，上限 2GB
  --storage.tsdb.max-block-duration   持久化块的目标时长（默认 2h）
  --storage.tsdb.retention           数据保留时长（默认 15d）
  --storage.tsdb.retention.size      按体积保留（默认 0 = 不限制）
```

```bash
# 实测当前内存分布
curl -s http://localhost:9090/api/v1/status/tsdb | python -m json.tool | head -30

# 进程实际占用
ps -o pid,rss,cmd -p $(pgrep -f 'prometheus$')
```

> **`retention` 只改 `prometheus.yml` 是无效的**——它不是配置项，是**启动参数**，写在 systemd 的 `ExecStart` 里。常见的「改了配置但数据还是被删」就是这个原因。
>
> 真实案例：某次发现磁盘里 TSDB 只剩 13 天目录，而 `prometheus.yml` 里翻遍没找到 retention。查 `systemctl cat prometheus` 才发现是 `ExecStart` 里的 `--storage.tsdb.retention=15d`，实际目录保留天数还会因为 compactor 有延迟而不完全等于 15。**调 retention 必须改 systemd unit，`daemon-reload` + `restart`。**

## 5.3 磁盘会涨到多大

```text
磁盘占用 ≈ 序列数 × 每样本字节数 × 样本总数 × 压缩比

实测参考（保留 15 天，单机 2000 台、约 40 万条序列）：
  未压缩原始数据 约 1.5~3 GB
  压缩后        约 0.5~1.2 GB
  → 每天新增 30~80 MB，15 天约占 1 GB 左右

经验值：单机监控保留 15 天，磁盘按 10~20 GB 预留比较稳妥。
```

| 降低磁盘的手段 | 代价 |
|---------------|------|
| 缩短 `retention` | 丢历史，长周期趋势（周同比、季节性）没了 |
| `metric_relabel_configs` drop 无用指标 | 丢数据 |
| 调大 `scrape_interval`（15s → 60s） | **直接砍掉 4 倍数据量**，但丢失秒级抖动信息 |
| 只采关心的指标（drop 掉 `go_*`） | 丢数据 |
| 远端存储到 Thanos/VictoriaMetrics | 本地可以只留 1 天，长期数据在远端 |

> **`scrape_interval` 是最有效的降本旋钮**。很多场景 15s 纯属浪费——你 1 分钟才看一次面板，采 15s 和 60s 的信息量差异微乎其微，却让数据量差 4 倍。判断标准是：**你的 for 字段和告警阈值需要多细的时间分辨率**。内存告警要 1 分钟粒度，15s→30s 无所谓；抓请求延迟分位数则可能要 15s。
>
> 另一个隐藏内存杀手：**长查询**。`rate(x[30d])` 这种超长窗口查询会把 30 天数据全拉进内存算，机器直接被 OOM 打挂。Prometheus 3.0 之后会主动拒绝这类查询（`query.max-samples` 限制），老版本只能靠自己自律。

---

# 六、查询和数据结构

## 6.1 两种查询

| 类型 | 界面 | API | 用途 |
|------|------|-----|------|
| **Instant Query**（即时查询） | `/graph` | `/api/v1/query?query=...&time=...` | 查某一时刻的值，告警规则用 |
| **Range Query**（区间查询） | `/graph?g0=..&g1=..` | `/api/v1/query_range?query=..&start=..&step=..` | 画图，step 决定数据点密度 |

```bash
# 即时查询
curl -s -G http://localhost:9090/api/v1/query \
  --data-urlencode 'query=up{job="node"}' | python -m json.tool

# 区间查询：step 太小会把浏览器打爆
curl -s -G http://localhost:9090/api/v1/query_range \
  --data-urlencode 'query=rate(node_network_receive_bytes_total[5m])' \
  --data-urlencode 'start=1735689600' \
  --data-urlencode 'end=1735776000' \
  --data-urlencode 'step=300'            # 300s 一个点
```

> **Range 查询的 step 要算过再填**：`step = (end - start) / 期望点数`。查一年的数据却默认 step=15s，会生成 200 万个点，Prometheus 算得慢、Grafana 画不出来。仪表盘上时间范围拉大时，step 必须同步放大。

## 6.2 必须先背下来的算子

| 算子 | 作用 | 典型写法 | 注意 |
|------|------|---------|------|
| `up` | target 是否抓取成功 | `up{job="node"} == 0` | **排障第一站** |
| `rate()` | Counter 每秒增长速率 | `rate(http_requests_total[5m])` | 窗口建议 ≥ 4 个 scrape_interval |
| `irate()` | 最后两个点的瞬时速率 | `irate(...[5m])` | 单点抖动大，看趋势别用它 |
| `increase()` | 区间内总量 | `increase(x[1h])` | **比 `delta()` 抗 Counter 重启** |
| `sum` / `avg` / `max` / `min` / `count` | 聚合 | `sum(rate(x[5m]))` | Counter 必须先 rate |
| `topk(k, v)` / `bottomk` | 取前/后 k 条 | `topk(10, node_load1)` | 排障找元凶很好用 |
| `histogram_quantile(p, v)` | 从桶算分位数 | 见下 | **必须先 sum by (le) 再 rate** |
| `delta()` | 任意类型的变化量 | `delta(node_load1[5m])` | Gauge 专用，Counter 会被清零打断 |
| `predict_linear(v[d], n)` | 线性外推预测 | `predict_linear(node_filesystem_avail[6h], 4*3600) < 0` | **磁盘将满告警的标准写法** |
| `absent_over_time()` | 判断指标是否消失 | `absent_over_time(up[5m])` | 检测「服务挂了但 up 没更新」 |
| `changes()` / `resets()` | 变化次数 / Counter 重置次数 | `changes(node_load1[1h])` | 判断服务是否重启 |
| `clamp_min` / `clamp_max` | 兜底防负 | `clamp_min(v, 0)` | 图上不要出现负数 |
| `label_replace` | 正则改标签名 | 见第十四章 | label_replace(v, "dst", "$1", "src", "(.*)") |

**分位数必须这样写**（错误写法非常常见）：

```promql
# ✅ 正确：先 rate，再按 le 求和，最后算分位数
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# ❌ 错误：直接对多个实例的桶相加，分位数会失真
histogram_quantile(0.99, sum by (le, instance) (rate(..._bucket[5m])))

# ❌ 错误：对单个 exporter 的整体桶算（没法跨实例聚合）
histogram_quantile(0.99, rate(..._bucket[5m]))
```

## 6.3 聚合与分组

```promql
# by() 保留这些维度，其余聚合掉
sum by (instance) (rate(node_network_receive_bytes_total[5m]))

# without() 排除这些维度，其余保留（维度多时更省事）
sum without (cpu, mode) (rate(node_cpu_seconds_total[5m]))
```

**向量匹配的 `on` / `ignoring`**——两个序列集按标签对齐时用：

```promql
# on(标签) 只用这些标签对齐；ignoring(标签) 排除这些
node_filesystem_avail_bytes{fstype!="tmpfs"}
  / ignoring(device) group_left
  node_filesystem_size_bytes{fstype!="tmpfs"}

# group_left / group_right：声明哪一侧基数小
# 左边多、右边少 → group_left
# 左边少、右边多 → group_right
```

> **`ignoring(device)` 这里的坑**：除法要求两侧 series 数一一对应。磁盘指标带 `mountpoint`、`device` 等多个标签，如果一边多一个标签就匹配不上，返回空。**排障时如果结果为空，先看两边各自的 series 数和标签**（Grafana 里用 Table 面板 + `count by(...)`），十有八九是标签没对齐。
>
> `group_left` / `group_right` 写反了报 `found duplicate series for the match group`，不是静默出错，所以这个错比较好发现。

## 6.4 常用排障表达式

```promql
# —— target 层面 ——
up                                       # 全部存活情况
up == 0                                  # 哪些挂了
up == 0 and on(instance) node_uptime_seconds > 600   # 只看启动 10 分钟内新挂的

# —— CPU ——
1 - avg without(cpu, mode) (sum without(cpu, mode) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) by (instance)
1 - rate(node_cpu_seconds_total{mode="idle"}[5m])          # 单核简化版，超过 1 说明多核已饱和

# —— 内存 ——
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes        # 可用率，比 free 更准
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.9

# —— 磁盘 ——
predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[6h], 4*3600) < 0   # 4 小时后写满
node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.1                       # 剩余不足 10%
rate(node_disk_written_bytes_total[10m])                                              # 写入速率，定位 IO 打满

# —— 网络 ——
rate(node_network_receive_bytes_total{device!="lo"}[5m])
rate(node_network_transmit_bytes_total{device!="lo"}[5m])

# —— 连接数与 fd（服务打满的第一指标）——
node_nf_conntrack_entries / node_netfilter_conntrack_max > 0.8
node_filefd_allocated / node_filefd_maximum > 0.9

# —— 找重启过的服务 ——
resets(process_start_time_seconds[1h]) > 0
```

> **CPU 为什么不能直接 `sum(rate(node_cpu_seconds_total[5m]))`**：`node_cpu_seconds_total` 每核每模式一条，8 核就是 32 条。直接 `sum` 得到的是「所有核所有模式的总秒数」，数值会超过 1，看不出饱和程度。**正确做法是排除 idle 再 `1 -`**——因为「所有非 idle 时间 / 全部时间」天然被限制在 0~1 之间。
>
> 内存用 **MemAvailable 而不是 MemFree**：`MemFree` 只统计完全没被使用的页，Linux 会拿空闲内存做缓存，所以 `free` 常年接近 0，看着像内存告急但其实绰绰有余。`MemAvailable` 才是「不给缓存也能立刻拿出来的内存」，这才是你要的语义。

## 6.5 PromQL 的陷阱

| 陷阱 | 表现 | 正解 |
|------|------|------|
| 忘记 `rate()` | `sum(node_cpu_seconds_total)` 是个巨大的数 | Counter 一律先 rate/increase |
| Counter 直接比较大小 | 永远 `> 0`，告警疯狂触发 | 比的是 `rate()` 的结果 |
| `rate()` 窗口太小 | 窗口 < 2×scrape_interval 时结果剧烈抖动甚至空 | 窗口 ≥ 4×scrape_interval |
| 指标名写错 | 返回空，界面一片空白，**不报错** | 先在 `/graph` 里 `{__name__=~".*cpu.*"}` 模糊搜 |
| label 值是空 | `label=""` 不匹配任何东西，写 `{env=}` 语法错误 | 标签值必须用 `=~` 正则或 `=` 非空 |
| 时间窗口跨越重启 | `rate` 遇到 Counter 重置出现巨大尖峰 | 用 `increase` 或拉长窗口 |
| 精度不足 | 大数指标（字节数）被四舍五入 | 表达式尾部加 `* 1024` 之类换算，或用 `_bytes` 指标 |

---

# 七、Prometheus 角色与集群

## 7.1 单机够用吗

| 序列数 | 配置 | 说明 |
|--------|------|------|
| < 10 万 | 1C1G，单机 + 15 天 retention | 学生/小团队，Docker Desktop 都够 |
| 10 万 ~ 100 万 | 2C4G，SSD，retention 30 天 | 中型集群，能覆盖几百台主机 |
| 100 万 ~ 500 万 | 4C8G+，需要专门的存储规划 | 建议开始考虑联邦或远端存储 |
| > 500 万 | 不要硬扛单机 | Thanos / Cortex / Mimir / VictoriaMetrics |

## 7.2 联邦与远端存储

```yaml
# ---------- 联邦：子 Prometheus 汇总到父 Prometheus ----------
# 子：/etc/prometheus/prometheus.yml
scrape_configs:
  - job_name: 'federate'
    honor_labels: true                 # 保留子实例原始 label
    http_sd_configs:
      - url: http://parent:9090/federate
        refresh_interval: 30s

# 父：监听 federation label，不抓自己（避免指标重复）
scrape_configs:
  - job_name: 'federate'
    honor_labels: true
    metrics_path: /federate
    params:
      'match[]':
        - '{job=~".+"}'
        - != 'job="federate"'          # 排除自身
    static_configs:
      - targets: ['sub01:9090', 'sub02:9090']
```

```yaml
# ---------- remote write：本地只留 1 天，长期数据进远端 ----------
remote_write:
  - url: "http://thanos-receive:19291/api/v1/receive"
    remote_timeout: 30s
    queue_config:
      capacity: 10000
      max_shards: 10
      min_shards: 1
    metadata_config:
      send: true
```

| 方案 | 定位 | 适合 |
|------|------|------|
| **联邦** | 树形汇总，子查父 | 机房/多网络分区，各自可达本地 Prometheus |
| **remote write** | 写远端存储，本地短期缓存 | 长期留存 + 用 Thanos/Mimir 的统一查询层 |
| **Thanos** | 存长期 + 跨集群聚合查询 | 主流开源方案，兼容 PromQL |
| **Cortex / Mimir** | 多租户 + 长存储 | 更快、存 Blob（S3） |
| **VictoriaMetrics** | 资源占用极低 | 单机想存很久、内存吃紧 |

## 7.3 集群不等于高可用

> **必须记住**：Prometheus 官方文档明说「Prometheus 集群不实现高可用」。它的定位是**横向切分**（不同团队管不同集群，各自一套，最后汇总），不是纵向冗余。
>
> 单机 Prometheus 挂了 = 那段时间的指标永久丢失，补不回来。想要真正不丢数据，逻辑是：
>
> ```text
> Prometheus × 2（互相独立抓取，非主备）
>        ↓ remote write
>   Thanos Receive / VictoriaMetrics（数据在这里才真正持久）
> ```
>
> 两个 Prometheus 各抓各的、各自远端写，谁挂了另一个继续；因为抓的是同一批 target，数据最终在远端汇合。**没有「主备切换」这个概念**——这是很多人对 Prometheus 集群最大的误解。
>
> Alertmanager 倒是有官方 HA 方案：多个实例之间用 **gossip 集群**（端口 9094）互相广播告警状态，保证同一条告警不会重复发、也不会因为某个实例挂了而漏发。配置是 `--cluster.peer=alertmanager:9094 --cluster.listen-address=`。

---

# 八、K8s 部署与监控（helm operator）

## 8.1 kube-prometheus-stack 组件清单

速记里「2 个 StatefulSet / 3 个 Deployments / 1 个 DaemonSet / 3 个 ReplicaSet」说的就是 `kube-prometheus-stack` 这个 Helm Chart 装出来的东西：

| 数量 | 名称 | 类型 | 作用 |
|------|------|------|------|
| **1** | `prometheus` | **StatefulSet** | 采集 + 存储 + 查询 + 规则评估。有状态（要保留 TSDB 数据卷） |
| **2** | `alertmanager` | **StatefulSet** | 告警分组、抑制、路由。有状态（要保留 silences / notification log） |
| **3** | `grafana` | Deployment | 可视化。无状态 |
| **4** | `prometheus-operator` | Deployment | CRD 控制器，把「期望配置」翻译成 Prometheus 配置并热加载 |
| **5** | `kube-state-metrics` | Deployment | 读 API Server，把 K8s 对象状态（副本数、Pod 相位…）变成指标 |
| **6** | `node-exporter` | **DaemonSet** | 每节点一个，暴露主机级指标 |

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo prometheus-community/kube-prometheus-stack

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword='<强密码>' \
  --set alertmanager.enabled=true \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.resources.requests.memory=2Gi \
  --set prometheus.prometheusSpec.resources.requests.cpu=500m \
  -f values-2c2g.yaml      # 2C2G 小集群务必覆盖资源值，见下方说明
```

```yaml
# values-2c2g.yaml：2C2G 集群的降配（默认值会 OOM）
prometheus:
  prometheusSpec:
    retention: 7d                     # 15d → 7d，磁盘和内存都省
    scrapeInterval: 30s               # 15s → 30s，数据量直接砍半
    resources:
      requests: { memory: 1Gi, cpu: 300m }
      limits:   { memory: 1500Mi, cpu: "1" }
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: <你的本地 StorageClass>
          resources: { requests: { storage: 8Gi } }
alertmanager:
  resources:
    requests: { memory: 100Mi, cpu: 50m }
grafana:
  resources:
    requests: { memory: 150Mi, cpu: 50m }
kubeStateMetrics:
  resources:
    requests: { memory: 100Mi, cpu: 50m }
```

> **2C2G 集群装 kube-prometheus-stack 是会 OOM 的**。这套组件自身开销：Grafana 约 370MB + 插件子进程约 120MB + Prometheus 约 100MB + node_exporter 约 25MB ≈ **600MB，占 33%**。而且每个 Grafana 插件是**独立子进程**，装 5 个插件就是 5 个常驻进程各吃几十 MB——插件装得越多，内存涨得越快。
>
> 更麻烦的是它会和业务抢内存：Calico 每个节点常驻、Dashboard 的 kong、metrics-server、kube-system 组件都要吃。如果 master01 空载已经用到 70% 以上，再叠这套就会触发级联驱逐——典型症状是 Dashboard 突然 502、Prometheus 容器 OOMKilled。
>
> **2C2G 的可行做法**：① `retention` 降到 7d、`scrapeInterval` 放慢到 30s；② 显式设 `resources.requests`（否则默认无限制，节点压力一大就被驱逐）；③ **用 helm 指定 `--set grafana.sidecar.dashboards.enabled=false` 关闭 dashboard sidecar**（它会 watch 所有 ConfigMap，在大集群里很吃内存）；④ 内存实在不够就分两步装：先 `prometheus + node-exporter + kube-state-metrics`（不加 Grafana、不加 Alertmanager），UI 用 `kubectl port-forward` 临时看。

```bash
# 验证安装：对照速记里的组件数量
kubectl -n monitoring get statefulsets,deployments,daemonsets
# 期望：2 个 STS（prometheus、alertmanager-alertmanager）
#      3~4 个 Deployment（grafana、prometheus-operator、kube-state-metrics）
#      1 个 DaemonSet（prometheus-node-exporter）
#      ReplicaSet 数量取决于 operator/grafana 的副本与滚动策略
```

## 8.2 Operator 与四个 CRD

传统做法是手动改 `prometheus.yml`，改完 `kubectl rollout restart`。Operator 的思路是**把配置变成 K8s 对象**：

```text
传统：  你手写 prometheus.yml  →  restart Pod  →  生效（有中断）
Operator： 你写 CRD（声明式）  →  Operator 监听变化  →  渲染成配置  →  热加载（无中断）
```

| CRD | 作用 | 谁消费它 |
|-----|------|---------|
| **ServiceMonitor** | 定义「抓哪个 Service 的哪个端口」 | Prometheus |
| **PodMonitor** | 定义「抓哪些 Pod」（不走 Service） | Prometheus |
| **PrometheusRule** | 定义告警/记录规则（`spec.groups`） | Prometheus |
| **Probe** | 定义黑盒探测（`spec.prober` + `spec.probeSpec`） | Prometheus |
| **AlertmanagerConfig**（新版） | 往 Alertmanager 里注入接收器配置 | Alertmanager |

```yaml
# ServiceMonitor：抓 redis-exporter 的 Service
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: redis-monitor
  labels:
    release: monitoring          # ← 这个 label 决定被哪个 Prometheus 选中
spec:
  selector:
    matchLabels:
      app: redis-exporter
  endpoints:
    - port: metrics              # 对应 Service 里 port 的 name，不是端口号！
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
  namespaceSelector:
    matchNames: [default]
```

```yaml
# PrometheusRule：告警规则（注意字段和原生 rules 文件几乎一样）
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: node-alerts
  labels:
    release: monitoring
spec:
  groups:
    - name: node
      rules:
        - alert: NodeHighCpuLoad
          expr: node:node_cpu_utilisation:rate5m > 0.85
          for: 10m
          labels: { severity: warning }
          annotations:
            summary: "CPU 高负载 {{ $labels.instance }}"
```

> **ServiceMonitor 里 `port` 写端口号是最常见的错误**。它匹配的是 **Service `spec.ports[].name`**，不是 `port` 数字也不是 `targetPort`。
>
> ```yaml
> # Service
> ports:
>   - name: metrics        # ← 名字在这
>     port: 9121           # Service 端口
>     targetPort: 9121
> # ServiceMonitor
> endpoints:
>   - port: metrics        # ← 要写 name，不是 9121
> ```
> 写错的话 Operator 会正常渲染出配置，但 Prometheus 一个 target 都发现不了——`kubectl get servicemonitor` 一切正常，target 页却是空的。**排查时直接 `kubectl get prometheus -o yaml` 看渲染出来的 `scrapeConfigs`，那是最终真相。**
>
> 另外 ServiceMonitor 自己不生效还要看第二道门：`metadata.labels` 里的 `release: monitoring` 必须和 Prometheus CR 的 `spec.serviceMonitorSelector` 匹配上。默认是 `matchLabels: {release: <helm release 名>}`，helm 装的时候 release 名改了，这里就得跟着改。

## 8.3 instance / job 的区分

速记里专门列了这两个概念，因为在 K8s 里它们是自动生成的，不理解就没法做聚合和过滤。

| 标签 | 谁生成 | 取值 | 用途 |
|------|-------|------|------|
| **`job`** | scrape 配置里的 `job_name` | `serviceMonitor/<namespace>/<name>/<idx>`（Operator 生成）或你自己写的名字 | **标识「这是哪一类 job」**，用来做汇总 |
| **`instance`** | 目标地址 | `10.244.1.17:9100`，或 `podname_namespace_ipport` | **标识「这是哪一台机器」**，用来告警和定位 |

```promql
# 按 job 汇总存活数
sum by (job) (up)
# 结果： {job="node"} 3   {job="redis"} 1   {job="mysql"} 2

# 找出所有 down 的具体机器
up == 0
# 结果： {instance="10.244.2.8:9100", job="node"} 0

# 按 job 分组算总流量
sum by (job) (rate(node_network_receive_bytes_total[5m]))
```

> **`instance` 在 K8s 里会带 Pod IP，所以 Pod 一重建，instance 就变了**。后果：告警里出现一堆「不同 instance」的重复告警，看不出是同一台机器；按 instance 聚合的历史曲线会断成两段。
>
> 两个解法：
> **① 用 relabel 把 Pod 名/节点名写回 instance**（更可读）：
> ```yaml
> relabel_configs:
>   - source_labels: [__meta_kubernetes_pod_name]
>     target_label: instance            # 覆盖掉 IP:port
>   - source_labels: [__meta_kubernetes_pod_node_name]
>     target_label: node
> ```
> **② 保持 IP，但按 node 聚合**（更适合长期趋势）：用 `by (node)` 而不是 `by (instance)`。
>
> 一句话判断：**instance 是「这一条抓取连接」，不是「这一台机器」**。理解了这点，聚合和告警的写法就都通了。

## 8.4 endpoint 角色

K8s 服务发现的 `role` 决定「去哪儿找 target」：

| role | 发现的 target | 生成的 `__meta_*` 常用标签 |
|------|--------------|------------------------|
| `node` | 集群节点 | `__meta_kubernetes_node_name` `..._node_label_*` |
| `pod` | 所有 Pod（默认只 Running 且非 completed） | `..._pod_name` `..._pod_namespace` `..._pod_node_name` `..._pod_ip` `..._pod_container_port_name` |
| `service` | Service（需配合 role: endpoints） | `..._service_name` `..._service_namespace` |
| `endpoints` | Service 后端 Endpoints（旧 API） | `..._endpoint_port_name` |
| `endpointslice` | EndpointSlice（**推荐**，新集群默认） | `..._endpointslice_endpoint_address_target_kind` |
| `ingress` | Ingress 对象 | `..._ingress_scheme` `..._ingress_host` |

```yaml
kubernetes_sd_configs:
  - role: pod            # 直接发现 Pod，不经 Service
    namespaces:
      names: [monitoring]
    relabel_configs:
      # 只有带这些 annotation 的 Pod 才抓，避免抓一堆无关 Pod
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      # 从 annotation 里读端口（比硬编码灵活）
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        target_label: __address__
        regex: "(.+)"
        replacement: "$1"          # 需配合 __address__ 初始化为 ip:0
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        target_label: __metrics_path__
        regex: "(.+)"
```

> **`role: pod` 抓全集群 Pod 会产生大量无效 target**。每个 target 都要发 HTTP 请求，200 个 Pod 里 190 个没暴露 `/metrics`，就是 190 次注定 404 的请求，还污染 `up` 指标。**必须用 `keep` 规则按 annotation 过滤**，让 `up == 0` 真正代表「监控对象挂了」而不是「这个 Pod 压根不该被抓」。
>
> 这是**「up == 0 有两种截然不同的含义」**的典型场景：不加过滤时，`up == 0` 满屏都是噪音，久了就没人看了；加了过滤，它才重新变成一个可信的信号。

## 8.5 相同目标的 collection

速记这句指的是：**当多个 target 需要共享同一组标签时，用 relabel 统一打标**，而不是在每条 scrape 配置里复制粘贴。

```yaml
  - job_name: 'node'
    static_configs:
      - targets: ['10.0.0.1:9100', '10.0.0.2:9100', '10.0.0.3:9100']
        labels:
          region: 'cn-east'
          tier: 'prod'
          team: 'infra'
    # 三个 target 共享上面 3 个标签，下面的规则对所有 target 统一生效
    relabel_configs:
      - source_labels: [region]
        target_label: 'env'
        regex: 'cn-east'
        replacement: 'production'
```

**「去重」场景的典型用法**：Operator 自己也会去抓 Prometheus 自身的 `/metrics`，导致同一指标出现两份（instance 不同）。排除它：

```yaml
    metric_relabel_configs:
      - source_labels: [job]
        regex: 'prometheus|alertmanager|operator'
        action: drop
```

**「合并成一条 series」场景**：想看全集群总流量时，需要剥掉 instance 维度：

```promql
# 先在 relabel 里丢掉 instance，再在 PromQL 里 sum，得到全集群一条曲线
relabel_configs:
  - action: labeldrop
    regex: instance
```

> **`labeldrop` 是控制高基数的终极武器**，配合 `keep` 一起用效果最好：
> ```yaml
> relabel_configs:
>   - source_labels: [__meta_kubernetes_pod_label_app]
>     action: keep
>     regex: 'my-service'          # 只留一个应用的 Pod
>   - action: labeldrop
>     regex: 'pod_id|container_id' # 丢掉高基数且无分析价值的标签
> ```
> 挑 label 的标准很简单：**这个标签的值集合是不是有限的？** `app`、`env`、`namespace` 这种有限集合留着；`pod_id`、`trace_id`、`request_path` 这种无限集合一律 drop 或归一化。

## 8.6 开放可视化 UI

| 方式 | 命令 | 暴露面 | 适用 |
|------|------|--------|------|
| **port-forward**（最推荐） | `kubectl -n monitoring port-forward svc/prometheus-operated 9090:9090` | 只有本机 | 临时查东西，零风险 |
| **kubectl proxy → API Server** | `kubectl proxy` 然后访问 `/api/v1/namespaces/monitoring/services/http:prometheus-operated:9090/proxy/graph` | 需要 API Server 认证 | 浏览器直接开，无本地端口 |
| **NodePort** | `kubectl patch svc ... -p '{"spec":{"type":"NodePort"}}'` | 整个集群网 | 多人共享 |
| **Ingress + 认证** | helm `--set ingress.enabled=true` + basic auth | 公网可达 | 长期对外，**必须配认证** |
| **SSH 隧道** | `ssh -L 9090:127.0.0.1:9090 user@host` | 只有本机 | Grafana 这类 UI 的推荐做法 |

> **Prometheus 和 Grafana 都不应该裸暴露公网**。Prometheus 的 UI 里包含全部内部指标（机器名、内网 IP、可能还有配置里的敏感 label），Grafana 更不必说。常见做法是 Grafana 只监听 `127.0.0.1`，用 SSH 端口转发访问：
>
> ```bash
> ssh -N -L 3000:127.0.0.1:3000 -L 9090:127.0.0.1:9090 monitor@<A机IP>
> # 之后本机浏览器开 http://127.0.0.1:3000
> ```
>
> 我自己的双机监控就是这么做的：Grafana 改 `http_addr = 127.0.0.1` 只监听本机，然后从 Windows 用 SSH 隧道 + 浏览器访问。**安全组不需要为 Grafana 开任何入站端口**，这是最省事也最不容易出错的做法。

---

# 九、Grafana

## 9.1 数据源

| 类型 | 说明 |
|------|------|
| Prometheus | 主数据源，配 `Prometheus` 类型的 Data source |
| Instrumental | 「一个指标一个面板」的轻量模式，不写 PromQL |
| Loki | 日志，配好后可以在同面板叠加日志 |
| MySQL / ES / CloudWatch | Grafana 作为统一看板 |

```yaml
# 纯配置化（provisioning）——推荐，别在 UI 里手点
# /etc/grafana/provisioning/datasources/datasources.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus-operated.monitoring.svc:9090
    isDefault: true
    editable: false
    jsonData:
      httpMethod: POST      # 长查询走 POST，避免 URL 过长被截断
      timeInterval: 15s     # 与 scrape_interval 对齐，否则 Grafana 的 min step 会小于数据密度
```

> **Grafana 8+ 必须显式写 `uid`**。这是纯配置化最大的坑：YAML 里定义的 dashboard 引用数据源是**按 uid 引用**的（`datasource: { uid: 'prometheus' }`），而 UI 手建的数据源默认 uid 是随机 64 位字符串。YAML 里写 `${DS_PROMETHEUS}` 变量导入时看起来正常，一改成固定 uid 就报 `Datasource not found`。
>
> **另一个坑**：`grafana.db`（SQLite）默认权限是 `-rw-r----- grafana:grafana`，**普通用户读不了**。所以用 provisioning 之前必须 `sudo chown grafana:grafana /var/lib/grafana/grafana.db`；改完必须 `systemctl restart grafana` 才生效。改权限不重启是「配置改了没反应」的头号原因。
>
> 顺带一句版本相关的：Grafana 9.3+ 移除了旧版 Angular 下拉插件支持，Grafana 11 是大版本跃迁，插件市场里相当一批旧插件已不可用。装插件前先确认兼容版本，否则面板加载不出来。

## 9.2 仪表盘

| 面板类型 | 用途 | 注意 |
|---------|------|------|
| **Time series** | 折线图，最常用 | **务必设 `Min step = scrape_interval`**，否则数据量不够会画出不连续的锯齿 |
| **Stat** | 单个大数字 | 配 `Reduce → Last` 显示最新值 |
| **Gauge** | 百分比 | 后端要支持 `min/max` |
| **Table** | 明细表 | 配 Instant query，不要用 Range |
| **Heatmap** | 直方图 | 表达分位数最直观 |
| **Logs** | Loki 日志 | 需配 Loki 数据源 |
| **Alert list** | 当前告警 | 需配统一告警 |

```
仪表盘实操三板斧：
① 变量（Variables）→ 让一个 dashboard 服务多台机器
   变量类型：Query（从指标里取值）/ Custom（手填）/ Constant / Interval（$__interval）
   用法：query variable 写 `label_values(node_uname_info, instance)`
② 复用片段（Dashboard → Snippets）→ 公共查询块抽出来复用
③ 导出/导入（JSON）→ 版本化、跨环境迁移
```

```promql
# 变量查询的常用写法
label_values(node_uname_info, instance)        # 取所有 instance
label_values(up{job="node"}, job)              # 取 job 列表
count_values("up", up)                         # 按取值统计
```

> **`Min step` 忘了设是最常见的图表失真原因**。Grafana 默认的 Min step 是自动的，但如果你把 `scrapeInterval` 改到 30s 而 dashboard 缓存的 Min step 还是 15s，画出来的曲线会在两个点之间锯齿抖动——**那不是业务抖动，是数据密度不匹配产生的假象**。设成 `15s` 之后再看，曲线立刻变平滑。
>
> 另一个失真源是**时间范围和 step 不匹配**：Range query 的 step 太小 → 加载慢；太大 → 折线失去细节。Grafana 会自动算，但导出 JSON 分享给别人时容易带着过小的 step。

## 9.3 告警与权限

| 主题 | 要点 |
|------|------|
| **两种告警** | ① 传统 dashboard alerts（面板里配，只在该面板可见）② **Unified Alerting**（统一告警，跨 dashboard、支持 Grafana-managed 规则） |
| **推荐** | 用 Unified Alerting，规则写成文件做 provisioning，**别在 UI 里点** |
| **权限模型** | Viewer（只读）→ Editor（改 dashboard）→ Admin（改数据源/告警/用户） |
| **最小权限** | 业务方给 Viewer；给人建 dashboard 给 Editor；**Admin 只留 1~2 个** |
| **团队协作** | Grafana 本身有 folder 权限，但更推荐「datasource 用统一 uid + dashboard 全部 provisioning」的仓库化方式 |

## 9.4 dashboard 迁移

```bash
# 导出（从 UI 或 API）
curl -s -H "Authorization: Bearer <token>" \
  "http://localhost:3000/api/dashboards/uid/<uid>" > dash.json

# 导入（注意 data source uid 冲突是最大的坑）
curl -s -X POST -H "Content-Type: application/json" \
  http://localhost:3000/api/dashboards/db \
  -d '{"dashboard": <json>, "overwrite": true, "folderUid": "<folder>"}'
```

> **迁移最容易翻车的是 datasource uid 不匹配**。从 A 环境导出的 dashboard 里数据源是 `{ "uid": "P1234ABCD" }`，导到 B 环境时 B 环境恰好没有这个 uid，面板就变成红色 `Datasource not found`。
>
> 规避办法两个：**① 迁移前先统一 uid**（把目标环境的 Prometheus 数据源 uid 显式设成 `prometheus`）；**② 用 `$DS_PROMETHEUS` 变量**——Grafana 导入时会提示「把变量替换为某个数据源」，批量确认即可。前者一劳永逸，适合长期维护的环境。

---

# 十、第三方应用程序监控和部署

## 10.1 exporter 是什么

exporter 的定位可以用一句话概括：**让不懂 Prometheus 的应用，用自己的语言把数据交出来**。

```text
[MySQL]  ← exporter 用 MySQL 协议查 status/ 变量  → 翻译成 Prometheus 格式
         redis_exporter / mysqld_exporter / postgres_exporter / nginx-prometheus-exporter
[你的 Go 服务]  ← 引入官方 client 库，代码里主动 expose  → 少一层代理
         prometheus/client_golang
[黑盒]  ← blackbox_exporter 拨端口、发 HTTP 探活
         blackbox_exporter
```

| exporter | 监控对象 | 默认端口 | 备注 |
|----------|---------|---------|------|
| `node_exporter` | 主机 | 9100 | 官方 |
| `mysqld_exporter` | MySQL | 9104 | 需授权账号 |
| `redis_exporter` | Redis | 9121 | 注意 `--redis.password` |
| `postgres_exporter` | PostgreSQL | 9187 | |
| `nginx-prometheus-exporter` | Nginx | 9113 | 需开 `stub_status` |
| `blackbox_exporter` | 任意目标（HTTP/TCP/ICMP/DNS） | 9115 | 黑盒探测 |
| `kube-state-metrics` | K8s 对象 | 8080 | 官方，但部署在集群内 |

## 10.2 client 库：不必在意如何暴露服务

速记里「**不必在意如何暴露服务，devops 只关心数据**」这句话是 exporter 生态的核心思想：

```go
// Go 官方 client 库：业务代码里主动暴露指标
import "github.com/prometheus/client_golang/prometheus"

var requestDuration = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "请求耗时分布",
        Buckets: prometheus.DefBuckets,   // 5ms~10s
    },
    []string{"method", "path", "status"},  // ← 标签控制在低基数
)

func handler(w http.ResponseWriter, r *http.Request) {
    start := time.Now()
    doWork()
    requestDuration.WithLabelValues(
        r.Method, normalize(r.URL.Path), "200",   // ← 必须归一化，不能用原始 path
    ).Observe(time.Since(start).Seconds())
}
```

> **用 client 库时最容易犯的错是标签里塞原始值**。把 `r.URL.Path` 直接当标签，一万个不同订单号就是一万条序列——服务上线半小时内存就爆。**正确做法是在写入前归一化**：`/api/orders/12345` → `/api/orders/{id}`。这个函数在每个做监控的 Go 项目里都会出现，值得抄一份到自己的工具库。
>
> 第二个是**标签组合爆炸**：`method(4) × path(100) × status(20) = 8000`，还只是这一个 handler。标签设计要在接入前先算一遍「可能取值的乘积」，超过几千就要砍。

## 10.3 定义衡量标准：RED 与 USE

速记里「定义衡量标准」「在逻辑关系追踪值」，说的是**采什么指标**，这是比技术选型更重要的部分。

| 方法 | 面向 | 三个指标 |
|------|------|---------|
| **RED** | 面向**用户请求**的服务 | **R**ate（请求速率）、**E**rror（错误率）、**D**uration（耗时） |
| **USE** | 面向**资源**（主机、容器、磁盘） | **U**tilization（利用率）、**S**aturation（饱和度）、**E**rrors（错误） |

```promql
# RED —— 任何提供 HTTP/gRPC 的服务都该有的三个面板
sum(rate(http_requests_total[5m]))                                    # Rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))   # Error 率
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))  # Duration p99

# USE —— 资源饱和度
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes                    # Utilization
node_load5 / count without(cpu,mode)(node_cpu_seconds_total{mode="idle"})      # Saturation（load / 核数）
rate(node_disk_io_time_seconds_total[5m])                                     # 错误/超时（IO 等待占比）
```

**在逻辑关系追踪值**——把中间件指标和用户体验关联起来：

```text
用户说「网站变慢了」→ 往哪查？

    入口层        指标：request_duration p99        慢了吗？
      ↓ 关联：哪个接口慢
    应用层        指标：按 path 拆的 duration        是所有接口都慢还是特定接口？
      ↓ 关联：是特定接口 → 查它依赖什么
    中间件层      指标：redis 命令延迟 / db 查询延迟 / 连接池等待
      ↓ 关联：是数据库慢 → 查具体慢查询
    资源层        指标：CPU / IO 等待 / 磁盘 await

关键在于每层都要带「能追到下一层的标签」：
  应用层指标带 path
  数据库指标带 db / 表名（pg_stat_activity 的 query）
  这样才能从 p99 一路下钻，而不是在四层面板之间盲猜
```

> **中间件的「等待」指标比「使用率」更能提前发现问题**。Redis 的 `used_memory` 到 90% 才需要关心，但 `blocked_clients` 一旦大于 0 就已经在出问题了。数据库的连接池 `wait_count` 同样如此。
>
> 提前把这些指标**和业务指标画在同一张图上**（Grafana 的 Graph 面板可以叠多条 query + 左右 Y 轴），相关性一目了然。比如「接口耗时」和「Redis 延迟」画一张图，Redis 一卡就能直接看到耗时跟着抬头——这就是「在逻辑关系追踪值」的具体落地。

## 10.4 nginx 为例

```bash
# ① nginx 侧：只开 stub_status，不额外装模块（如果是官方包，通常已内置）
#    在 server 块里加
location /stub_status {
    stub_status;
    access_log off;
    allow 127.0.0.1;          # ← 只允许本机 exporter 访问
    deny all;
}
nginx -t && nginx -s reload
curl -s http://127.0.0.1/stub_status
# Active connections: 291
# server accepts handled requests
#  16630948 16630948 31070465
# Reading: 6 Writing: 179 Waiting: 106
```

```bash
# ② exporter 侧
nginx-prometheus-exporter \
  --nginx.scrape-uri=http://127.0.0.1:80/stub_status \
  --web.listen-address=:9113
```

```promql
# ③ 关键指标解读（stub_status 只有一个文本块，必须自己换算）
# 连接数三个维度
nginx_connections_active            # 当前活动连接
nginx_connections_reading            # 正在读请求（≈正在处理的）
nginx_connections_waiting            # 空闲等待（keep-alive 堆积，非并发高）
# 请求累计
rate(nginx_http_requests_total[5m])  # QPS
# 从 nginx_upstream 拿后端状态（需 lua 或 stub 扩展）
nginx_upstream_response_duration_seconds   # 后端耗时，必须配 histogram
```

> **nginx 的 stub_status 只有 6 行纯文本**，exporter 做的唯一工作是把它翻译成指标。想拿更多（Nginx Plus 的详细指标、SLA 分位、upstream 耗时），需要编译 `nginx-module-vts`（Very Tiny Stub）模块。所以：**先分清你要的是「QPS + 连接数」（stub 够用）还是「后端响应时间分布」（必须 vts 或 OpenResty）**。
>
> `waiting` 数值高**不等于**有压力——它主要统计 keep-alive 空闲连接。判断并发压力要看 `active`，判断排队要看 `reading`。把 waiting 当并发看是常见误读。

## 10.5 Redis 为例

```yaml
# redis_exporter + ServiceMonitor
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-exporter
  labels: { app: redis-exporter }
spec:
  replicas: 1
  selector:
    matchLabels: { app: redis-exporter }
  template:
    metadata:
      labels: { app: redis-exporter }
    spec:
      containers:
        - name: redis-exporter
          image: oliver006/redis_exporter:v1.67.0
          args:
            - --redis.addr=redis.default.svc:6379
            - --redis.password-file=/etc/redis/password
            - --web.listen-address=:9121
          ports: [{ containerPort: 9121, name: metrics }]
          resources:
            requests: { memory: 32Mi, cpu: 10m }
---
apiVersion: v1
kind: Service
metadata:
  name: redis-exporter
  labels: { app: redis-exporter }
spec:
  selector: { app: redis-exporter }
  ports:
    - name: metrics              # ← name 必须和 ServiceMonitor 的 port 对上
      port: 9121
      targetPort: 9121
```

```promql
# 必看的几个
redis_memory_used_bytes / redis_config_maxmemory_bytes          # 内存水位
redis_connected_clients                                          # 连接数
redis_evicted_keys_total                                         # 淘汰计数，有值说明内存不够
rate(redis_command_duration_seconds_sum[5m]) / rate(redis_commands_total[5m])   # 平均命令耗时
redis_key_size                                 # 每个 key 的分布（⚠️ 高基数风险）
```

> **`redis_key_size` 是 redis_exporter 里最危险的指标**：它按 key 名输出一条序列，百万级 key 就是百万条序列，直接把 Prometheus 内存打爆。**生产环境务必 drop**：
> ```yaml
> metric_relabel_configs:
>   - source_labels: [__name__]
>     regex: 'redis_key_size'
>     action: drop
> ```
> 需要看 key 大小时，用 `redis_memory_used_bytes` 配合 `--check-keys`（**注意：生产上这个检查会遍历全库，很慢，务必限制频率**）。
>
> 另外密码别写进 `--redis.password`（会出现在 `ps` 输出和容器 inspect 里），用 `--redis.password-file` 挂 Secret。

## 10.6 k8s charts 部署 exporter

```bash
# 通用三件套：Deployment + Service + ServiceMonitor
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install redis-exporter bitnami/redis-exporter \
  -n monitoring \
  --set serviceMonitor.enabled=true \
  --set serviceMonitor.labels.release=monitoring
```

**自建 exporter 的最小清单**（照这个抄就不会漏）：

```text
① Deployment     —— exporter 进程，含 resources.requests（2C2G 集群必填）
② Service        —— 端口必须带 name
③ ServiceMonitor —— labels.release 对上 Prometheus
④ PrometheusRule —— 告警规则（可选但建议）
⑤ NetworkPolicy  —— 限制来源（可选，安全加分）
```

> **给 2C2G 集群部署 exporter 的三条硬规矩**：
> ① **每个 exporter 都必须写 `resources.requests`**（哪怕只给 10m CPU / 32Mi 内存）。不写就是无限制，节点内存一紧张 kubelet 第一个杀它，而且往往是「监控组件被杀 → 看不到告警 → 没人知道为什么挂了」这种静默故障。
> ② **replicas 保持 1**。exporter 都是无状态的，多副本纯属浪费。
> ③ **优先用 DaemonSet 只在需要主机指标时**，中间件指标用 Deployment 单副本就够。
>
> 另外，指标抓取失败是静默的：exporter 崩了但没被监控，所以**「exporter 自己的存活」必须由 Prometheus 的 `up` 指标来管**（`up{job="redis-exporter"} == 0`），而不是靠 pod status——这套告警本身要写在 Prometheus 侧才有效。

## 10.7 endpoint 发现

| 方式 | 配置 | 适用 |
|------|------|------|
| **静态**（static_configs） | 写死 IP:端口 | 目标少、IP 固定（自建机房、单机） |
| **文件发现**（file_sd） | 定期读 JSON/YAML 文件，**改文件不用重启** | 目标动态但不想上 K8s；CMDB 驱动 |
| **DNS 发现**（dns_sd） | DNS A 记录 | 目标少、靠 DNS 管理 |
| **Consul / etcd / Eureka** | 注册中心原生 | 已有注册中心 |
| **K8s SD** | role: pod/service/node | 容器环境唯一选择 |

```yaml
  - job_name: 'file-sd-demo'
    file_sd_configs:
      - files: ['/etc/prometheus/targets/*.json']
        refresh_interval: 30s            # 文件变了自动加载
```

```json
[
  { "targets": ["10.0.0.1:9104"], "labels": { "service": "mysql", "env": "prod" } }
]
```

> **`file_sd` 是被严重低估的功能**。很多人还在为「加一台机器要 restart Prometheus」发愁，其实一个 JSON 文件 + `refresh_interval: 30s` 就解决了。**目标文件用 JSON 而不是 YAML**，因为 Prometheus 会在文件解析失败时**保留上一次的有效配置并继续运行**（日志里报解析错误），这个容错行为比想象中重要——别用一个写坏的 YAML 把整个监控干掉。
>
> 静态配置里 target 写错 IP 同样只是这一个 target 变 `up == 0`，不会影响其他 target。**Prometheus 对局部错误的容错很好**，这一点和它对配置文件的态度正好相反（配置错 = 启动失败）。

## 10.8 数据源如何引入 Grafana dashboard

```text
方式① 官方 dashboard ID（Grafana.com → Dashboards → 复制 ID）
      import 页 → 填 ID → 选择数据源 → Import
      优点：省事；缺点：数据源 uid 可能不匹配，面板要手工修

方式② JSON 文件（推荐，可版本化）
      import 页 → Upload JSON file
      或 API 批量导入（见 9.4）
      优点：进 git，能 code review；缺点：uid 要预先统一

方式③ 插件
      Grafana 官方 + 社区插件（如 grafana-piechart-panel）
      优点：功能强；缺点：版本兼容是雷区
```

> **官方 dashboard 直接 import 最容易踩的坑是数据源不匹配**。官方 dashboard 引用的是它们自己的数据源 uid（如 `prometheus` 恰好一致的概率不高）。导入后一堆红色 `Datasource not found`，得逐个面板改 datasource。
>
> 一次修完的办法：dashboard JSON 里把所有 `"datasource": {"uid": "旧uid"}` 全局替换成你的 uid，再导入。**或者反过来——把自己的数据源 uid 设成官方 dashboard 常用的那个**。后者一劳永逸。

---

# 十一、实战：双机 + 隧道监控架构

这是我在做的双机部署里的实际形态，记录下来作为「网络不通时 Prometheus 怎么抓」的参考解法。

```text
                     ┌──────────── A 机（监控机，公网可达）─────────────┐
                     │  Prometheus :9090     Grafana 127.0.0.1:3000      │
                     │  node_exporter :9100                           │
                     └───────┬────────────────────┬───────────────────┘
                             │ WireGuard 隧道      │ 本地
                    10.10.0.2│                    └─→ scrape A 10.10.0.2:9100
                             │
                    10.10.0.1│WireGuard
                     ┌───────┴──────── B 机（应用机，备案主体）──────────┐
                     │  node_exporter :9100                           │
                     │  Docker: we1l-backend / we1l-nginx              │
                     └────────────────────────────────────────────────┘
```

```yaml
# A 机 /etc/prometheus/prometheus.yml
global:
  scrape_interval: 30s
  scrape_timeout: 10s
  evaluation_interval: 30s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['10.10.0.2:9100', '10.10.0.1:9100']    # 隧道两端都在同一份配置里
        labels:
          env: prod
```

```bash
# 关键点：B 机的 9100 绝对不能监听公网 IP，只监听隧道地址
# B 机 node_exporter.service
ExecStart=/usr/local/bin/node_exporter --web.listen-address=10.10.0.1:9100

# A 机验证隧道通不通（ping 不通不代表端口不通，先 ping 再 curl）
ping -c 2 10.10.0.1
curl -s http://10.10.0.1:9100/metrics | head -1

# Windows 本机看 UI：只做端口转发，不在安全组开 Grafana 入站
ssh -N -L 3000:127.0.0.1:3000 -L 9090:127.0.0.1:9090 monitor@<A机公网IP>
```

> **这套架构里踩过的坑，按出现频率排**：
>
> ① **`retention` 改配置文件不生效**——它在 `systemd` 的 `ExecStart` 里，不在 `prometheus.yml` 里。改完要 `daemon-reload` + `restart`。
>
> ② **监控自身开销被低估**——Grafana 377M + 插件子进程 120M + Prometheus 98M + node_exporter 24M ≈ **620M，2G 机器上就是 33%**。加上 Calico / Dashboard kong / metrics-server，空载就到 73%。加监控前先算内存预算。
>
> ③ **Grafana 的 `grafana.db` 权限**——`-rw-r----- grafana:grafana`，普通用户读不到，provisioning 之前必须 `sudo chown` **并重启**，不重启就是「配置改了没反应」。
>
> ④ **B 机不需要装 Alertmanager**（如果只想 A 机统一告警），但要注意：**Prometheus 如果配置了 `alerting.alertmanagers` 却没装 Alertmanager，告警会一直处于 Pending 状态**——规则正常评估，但通知发不出去，而且**没有任何报错提示你「我发不出去」**。要么装，要么把 `alerting` 段整段删掉。
>
> ⑤ **跨网络抓取时，防火墙要放行的是隧道网段**（`10.10.0.0/24`），不是公网 IP，也不是 `0.0.0.0/0`。

---

# 十二、告警规则与 Ruler

> **Ruler 就是 Prometheus Server**（官方文档中把 rule 文件的组织方式叫 "rules"，而 Grafana 生态里把 Grafana 里的规则编辑器叫 Ruler）。这里指**规则文件的组织与加载方式**。

```yaml
# /etc/prometheus/rules/node-alerts.yml
groups:
  - name: node-alerts
    interval: 30s
    rules:
      - alert: NodeDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "实例 {{ $labels.instance }} 失联"
          description: "job={{ $labels.job }} 已 1 分钟无法抓取"

      - alert: NodeHighCpuLoad
        expr: node:node_cpu_utilisation:rate5m > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "CPU 高于 85%（{{ $labels.instance }}）"

      - alert: NodeMemoryWillFill
        expr: |
          predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[6h], 4*3600) < 0
          and node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.2
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "磁盘将在 4 小时内写满（{{ $labels.instance }}）"

      - alert: NfsMountLost
        expr: nfs_mount_ok == 0          # textfile collector 自定义指标
        for: 5m
        severity: critical
        annotations:
          summary: "NFS 挂载丢失：{{ $labels.mountpoint }}"
```

```bash
promtool check rules /etc/prometheus/rules/*.yml     # 语法自检
curl -s http://localhost:9090/api/v1/rules | python -m json.tool | head -40
# 或直接看 UI 的 /alerts 页面，能看到每条规则的状态和最近触发时间
```

**告警状态的三段流转**（理解这个才能懂 `for` 的意义）：

```text
Inactive（表达式为假）
    │  表达式变成真
    ▼
Pending（已经开始满足条件，正在计时 for）
    │  持续满足满 for
    ▼
Firing（真正触发，发给 Alertmanager）
    │
    │  表达式变回假
    ▼
Inactive
```

| 字段 | 作用 | 写法要点 |
|------|------|---------|
| `alert` | 告警名 | **一个项目里必须全局唯一**，同名会被覆盖 |
| `expr` | PromQL | 尽量基于 recording rule，减少重复计算 |
| `for` | 持续多久才 Firing | **没有 for 就会毛刺误报**；内存/CPU 至少 5~10 分钟 |
| `labels` | 告警标签 | `severity` 必加，用于路由和分级 |
| `annotations` | 通知内容 | 用 `{{ $labels.xxx }}` 和 `{{ $value }}` 模板化 |

```yaml
# 模板变量的正确写法（写在 annotations 里，不写在 labels 里）
annotations:
  summary: "磁盘剩余 {{ $value | humanizePercentage }}（{{ $labels.instance }}）"
  runbook_url: "https://wiki.internal/runbook/disk-full"   # 处置手册，强烈建议加
```

> **`for` 是告警质量的命门**。没有 `for` 的规则在网络抖动、GC 停顿、指标瞬时为 0 时会疯狂误报，几次之后**没人再看告警**，整套监控就废了——这是「告警疲劳」的经典死法。
>
> 经验值：`for: 1m` 只给「实例失联」这种确定性的场景用；资源类（水位、饱和度）至少 5~10 分钟；「预测 4 小时后写满」这种要 30 分钟。
>
> **另一个关键点：`expr` 必须能返回空值。** 比如 `node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.1`，某台机器没这个指标时结果里就没有它——这是对的。**但如果你写 `predict_linear(...[6h], 0) < 0 and ...` 这种组合，任何一侧为空整个表达式就为假**，看起来很安全，实际是「有问题的机器反而因为别的原因没被告警」。写告警表达式时要想清楚空值语义。
>
> **必须加 `runbook_url`**。半夜被叫醒时，annotations 里有一行处置链接和没有，处置时间差 10 倍。这不是形式主义，是把「告警」变成「可执行的信息」。

---

# 十三、Alertmanager

## 13.1 为什么需要它

Prometheus 只会「算出告警」，**发通知是 Alertmanager 的事**。中间这一层不是多余的，它负责四件 Prometheus 做不了的事：

| 能力 | 说明 | 没有它会怎样 |
|------|------|------------|
| **分组** | 同一台机器 20 条告警合成 1 封邮件 | 一晚上收 200 封邮件，然后全部忽略 |
| **抑制** | 机器挂了（`NodeDown`）时，抑制它身上所有衍生告警 | 一次宕机收到 15 条告警，真正的原因被淹没 |
| **静默** | 计划内维护时临时静音 | 维护期间告警狂响，团队开始关机重启服务 |
| **路由** | 不同 severity 发到不同接收器 | 关键告警和测试告警混在同一个邮箱 |

## 13.2 配置文件

```yaml
# /etc/alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m                     # 告警多久没恢复算 resolved
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alert@example.com'
  smtp_auth_username: 'alert@example.com'
  smtp_auth_password: 'AppSpecificPassword'
  smtp_require_tls: true                  # ← 不写这条密码会明文传输

route:
  receiver: 'default'                     # 兜底接收器
  group_by: ['alertname', 'instance']     # 这两个相同的告警合成一组
  group_wait: 30s                         # 出现后等 30s 再发（等更多同类告警汇齐）
  group_interval: 5m                      # 同组内新增告警的最小通知间隔
  repeat_interval: 4h                     # 未恢复的告警每 4h 重发一次
  routes:
    - matchers:
        - severity="critical"
      receiver: 'oncall'
      continue: false                     # 处理完不再往下一级匹配
    - matchers:
        - severity="warning"
      receiver: 'default'

receivers:
  - name: 'default'
    email_configs:
      - to: 'ops@example.com'
        send_resolved: true               # 恢复时也发一封（否则没人知道好了）

  - name: 'oncall'
    email_configs:
      - to: 'oncall@example.com'
        send_resolved: true
    webhook_configs:
      - url: 'https://hooks.example.com/prometheus'
        send_resolved: true

inhibit_rules:
  - source_matchers: [ alertname="NodeDown" ]
    target_matchers: [ severity="critical" ]
    equal: ['instance']                   # 同一 instance 的 critical 全被抑制
```

```bash
amtool check-config /etc/alertmanager/alertmanager.yml   # 语法自检
amtool alert add --alertmanager.url=http://localhost:9093 \
  InstanceDown instance=10.0.0.1:9100 severity=critical     # 手动注入一条告警做端到端测试
curl -s http://localhost:9093/api/v2/alerts | python -m json.tool
```

> **抑制（inhibit_rules）是 Alertmanager 最被低估的功能**。没有它，一次主机故障会同时触发 `NodeDown` + `NodeHighCpu` + `NodeMemoryHigh` + `DiskWillFill` + `NfsMountLost` 五六条通知，接收人根本不知道哪个是根因——而根因那条往往被淹没了。
>
> 规则要写成「**根因抑制衍生**」：`NodeDown` 是根因，它在场时抑制同 instance 的其他告警。顺序反过来写（衍生抑制根因）会把真正的原因隐藏掉。
>
> `equal` 字段是**抑制生效的分组键**：只抑制 `instance` 相同的那一组。写 `['instance']` 意味着 A 机的告警不会抑制 B 机的——这正是想要的。

> **邮箱收不到邮件的排查顺序**：① `smtp_require_tls: true` 有没有写；② 邮箱服务商是否要求**授权码**而不是登录密码（QQ 邮箱、163 邮箱都是）；③ 465 端口要 `smtp_smarthost: 'host:465'` 且通常不能带 `require_tls`（465 是隐式 TLS，587 是 STARTTLS，**两个不能混**）；④ 看 Alertmanager 日志 `journalctl -u alertmanager -f`，连接失败会有明确报错。
>
> **最常见的坑是「Prometheus 根本没配 Alertmanager」**：`prometheus.yml` 里没有 `alerting.alertmanagers` 段，或者 targets 指错地址，那么规则照样评估、状态照样变 Firing，但**通知永远发不出去，而且日志里没有任何提示**。排查方法：`http://localhost:9090/alerts` 页面的通知列会明确显示通知发送失败。**告警系统上线后第一件事：手动触发一条测试告警，确认真的收到通知。**

## 13.3 完整的告警链路

```text
   采集              评估                通知                 接收
node_exporter ──> Prometheus ──> Alertmanager ──> email/webhook/钉钉/电话
 :9100              :9090                 :9093                    │
                    │  alertname+labels       │                     │
                    └── 规则评估（每 15s）─────┘                     │
                        state: Firing ─────────────────────────────>─┘

反向依赖：Alertmanager 挂了会怎样？
  → Prometheus 继续评估、Firing、继续发（发不出去）
  → 告警状态还在，但没人收到
  → 所以 Alertmanager 也需要监控（monitor 自己那套监控要覆盖它）
```

> **别监控自己这条链条的三个环节**：
> ① **Prometheus 自己**——用 `up{job="prometheus"} == 0` 是自己抓自己（**自己抓自己时 `up` 永远是 1**，因为进程挂了就没谁来报告它死了）；正确做法是用**外部黑盒探测**（`blackbox_exporter` 从另一台机器拨过来）。
> ② **Alertmanager**——`amtool alert add` 手动注入测试告警，验证通知链路通。
> ③ **磁盘**——TSDB 写满时 Prometheus 会**只读**（日志报 `TSDB WAL: out of space`），采集继续但数据不再写入。`predict_linear` 的磁盘告警就是防这个。
>
> 这三条配齐了，监控才算闭环。**一个没人监控的监控系统，本身就是最大的单点。**

---

# 十四、排障速查

| 症状 | 可能根因 | 处理 |
|------|---------|------|
| target 页 `up = 0` | 网络不通 / 端口没监听 / 路径不对 | `curl http://target:port/metrics` 在 Prometheus 主机上验证 |
| target 显示 `down` 但 curl 正常 | 认证头 / TLS 校验 / 超时太短 | 检查 `scrape_timeout`；加 `tls_config` / `authorization` |
| 指标查不到，界面空白 | 指标名拼错（**不报错**） | `{__name__=~".*关键字.*"}` 模糊搜；`/api/v1/label/__name__/values` |
| CPU 值超过 100% | 直接 sum 了所有核和模式 | 排除 idle 再 `1 - avg without(cpu,mode)(sum without(cpu,mode)(...))` |
| Counter 求和得到巨大数字 | 忘了 `rate()` | 一律先 `rate()`/`increase()` |
| p99 算出来是 0 或怪值 | 忘了 `sum by (le)` | 必须 `histogram_quantile(p, sum by(le)(rate(_bucket[5m])))` |
| 序列数暴涨 / OOM | 高基数标签（URL、user_id、pod_ip+port） | `predict` 查 top 指标，`metric_relabel_configs` drop 或归一化 |
| PromQL 结果为空但指标存在 | 标签值不匹配 / 用了 `=` 匹配空值 | 确认标签值非空；`{env=~".+"}`；用 Table 面板看 label |
| 向量匹配返回空 | 两边标签没对齐 | Table 面板看两边 series 的标签差异；`ignoring()` / `on()` |
| 规则不触发 | 表达式太严 / 标签过滤掉了 / `for` 太长 | `/alerts` 页面看规则状态和最近评估时间 |
| 告警一直 Pending | Alertmanager 没配或不可达 | `prometheus.yml` 的 `alerting` 段；`/alerts` 页看通知状态 |
| 告警不发送 | 没装 Alertmanager / 邮件配置错 | `journalctl -u alertmanager -f`；`amtool check-config` |
| 告警邮件只有部分发出 | 被 inhibit_rules 抑制了 | 查 `/api/v2/alerts` 的 `suppressedBy` 字段 |
| 配置改了没生效 | 没重启 / `prometheus --reload` 未开 | `systemctl reload prometheus`（需启动参数 `--web.enable-lifecycle`）或 restart |
| Prometheus 启动失败 | YAML 语法错 | **改完先 `promtool check config`**，别直接重启 |
| retention 改了没变 | 它是启动参数不是配置项 | 改 systemd `ExecStart` + `daemon-reload` + restart |
| TSDB 磁盘涨太快 | scrape_interval 太密 / 序列太多 | 放慢到 30s、drop 无用指标、缩短 retention |
| 磁盘满后采集停了 | WAL 写不进去，**只读模式** | 扩盘或缩短 retention；清 WAL 目录前先停服务 |
| Grafana 面板 `Datasource not found` | 数据源 uid 不匹配 | 统一数据源 uid，或导入时映射变量 |
| Grafana 配置改了没反应 | `grafana.db` 权限 / 没重启 | `chown grafana:grafana` **并 restart** |
| Min step 小于 scrape_interval | 图表锯齿抖动是假象 | Min step 设为 scrape_interval |
| ServiceMonitor 配了但无 target | `port` 写成了端口号 / `release` label 不匹配 | 写 Service 的 port **name**；`kubectl get prometheus -o yaml` 看渲染结果 |
| 告警发到一半断了 | 分组 + `repeat_interval` 干扰 | 调大 `group_wait`，确认 `send_resolved` 策略 |

```bash
# 排障三板斧（按此顺序，能定位九成问题）
# ① 目标层：这个 target 的标签长什么样
curl -s http://localhost:9090/api/v1/targets | python -m json.tool | head -40

# ② 规则层：规则在不在、状态是什么
curl -s http://localhost:9090/api/v1/rules | python -m json.tool | head -40

# ③ 通知层：告警生成了吗、发出去没有
curl -s http://localhost:9093/api/v2/alerts | python -m json.tool
curl -s http://localhost:9090/api/v1/alerts | python -m json.tool | grep -A3 notifications
```

---

# 十五、常见误区

| 误区 | 真相 |
|------|------|
| 「装了 Prometheus 就等于有监控了」 | 装了只是有了**能力**。没有对应角色的仪表盘和告警规则，等于没装 |
| 「指标越多越好」 | 没人看的指标是负债，还占存储。指标的落点是「能触发一个动作」 |
| 「内存看 free」 | 看 `MemAvailable`。Linux 拿空闲内存做缓存，`free` 常年接近 0 |
| 「CPU 直接 sum」 | 排除 idle 再 `1 - avg without(cpu,mode)(sum without(cpu,mode)(...))` |
| 「p99 用 exporter 给的 Summary」 | Summary 的分位数不可跨实例聚合，用 Histogram + `histogram_quantile` |
| 「高基数交给 Prometheus 想办法」 | 它的设计就是「你给什么标签它存什么」，必须自己在 exporter 和 relabel 层解决 |
| 「Prometheus 集群自带高可用」 | 官方明说**不实现高可用**。它是横向切分，冗余要靠多实例 + 远端存储 |
| 「Alertmanager 可以不装」 | 可以，但配了 `alerting` 段却没装 = 告警永远 Pending 且**无任何报错** |
| 「retention 在 prometheus.yml 里改」 | 它是 systemd 启动参数 `--storage.tsdb.retention` |
| 「Prometheus UI 可以开公网」 | 它包含全部内部指标（机器名、内网 IP、敏感 label）。用 SSH 隧道或 Ingress + 认证 |
| 「kube-prometheus-stack 直接装 2C2G 集群」 | 自身约 620M（33% 内存），必须降配：retention 降、scrape 放慢、设 requests、关 sidecar |
| 「告警越多越安全」 | 告警疲劳 = 没人看 = 整套监控失效。宁少勿滥，`for` 一定要给 |

---

## 小结

> **总的说**：Prometheus 用 Pull 抓 exporter 的 `/metrics`，把「指标名 + 标签」当唯一标识存进 TSDB，用 PromQL 查，Grafana 画，Alertmanager 分组抑制后发通知。K8s 里由 Operator 把这些配置变成 CRD，helm 装出来就是 2 个 StatefulSet（Prometheus、Alertmanager）+ 3 个 Deployment（Grafana、Operator、kube-state-metrics）+ 1 个 DaemonSet（node-exporter）。

**十五条**：

1. **先分清数据类型**：指标进 Prometheus，日志进 Loki/ELK，链路进 Jaeger，事件进审计日志。四套拼起来才叫可观测性。
2. **数据模型是地基**：序列 = 指标名 + 标签集合。**值永远不能进标签，标签值集合必须有限**。
3. **四种指标类型**：Counter 要 `rate()`，Gauge 直接取值，分位数必须用 Histogram + `histogram_quantile`。
4. **高基数是最贵的坑**：百万序列直接 OOM。`/api/v1/status/tsdb` 查序列数，`labeldrop` 砍标签。
5. **target / endpoint / exporter 三个词分开看**：exporter 产生数据，target 是被抓对象，endpoint 是地址 + 标签。
6. **Pull 是默认选择**：服务能被抓就用 Pull，`up == 0` 的价值无可替代；只有「跑完就没」的 Job 才用 Pushgateway，且用完必须 DELETE。
7. **node_exporter + textfile collector 是主机监控底线**：自定义脚本写成 `.prom` 文件丢进目录即可，落地要 `mv` 避免半截文件。
8. **端口要规划**：监控自己占 9100~9199 / 9090~9099，和业务端口分开；K8s 里 ServiceMonitor + Pod IP 直连，不碰宿主机端口。
9. **relabel 两个时机**：`relabel_configs` 抓前改 target，`metric_relabel_configs` 抓后改样本，作用完全不同。
10. **存储**：Head 块常驻内存（约 1~2KB/序列），持久块在磁盘。**retention 是启动参数不是配置项**。`scrape_interval` 是最有效的降本旋钮。
11. **PromQL 必背**：`up` 排障第一站，CPU 用「排除 idle 再 1 -」，磁盘将满用 `predict_linear`。
12. **告警三条命门**：`for` 不给会毛刺误报；根因告警要 inhibit 衍生告警；annotations 必须带 `runbook_url`。
13. **告警链路要端到端测过**：`amtool alert add` 手动打一条，确认真的收到通知。**监控系统自己也要被监控**（用外部 blackbox 探 Prometheus 自己）。
14. **Operator 的两个必踩坑**：ServiceMonitor 的 `port` 写 Service 的 port **name** 不是端口号；CRD 的 `labels.release` 要和 Prometheus 的 selector 匹配。排查看 `kubectl get prometheus -o yaml` 渲染结果。
15. **Grafana 走 provisioning + 固定数据源 uid**；`grafana.db` 权限要 chown 且**必须重启**；Min step 要等于 scrape_interval。UI 默认只监听 127.0.0.1 是有意的，远程访问用 SSH 隧道。

> 踩坑优先级最高的是这五条：**高基数打爆内存**、**高基数残留**（retention 是启动参数）、**Alertmanager 没装导致告警静默不发**、**告警没给 for 导致疲劳**、**2C2G 集群装全栈监控 OOM**。
