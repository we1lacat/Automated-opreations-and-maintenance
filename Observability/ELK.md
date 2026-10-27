# ELK（Elasticsearch + Logstash + Kibana + Filebeat）

> **定位**：ELK 是**日志聚合**的事实标准——把散落在各机器上的日志**集中采集 → 解析 → 存储 → 可视化**。它解决的是「**日志里发生了什么**」，不解决「现在有没有事」。
>
> **适用边界**：ELK 只吃**日志/事件流**（文本 + 结构化字段）。指标监控归 Prometheus，链路追踪归 Jaeger/Tempo。硬拿 ES 存高频数值指标，写入放大和存储成本都会失控。
>
> **与其他方案的取舍**：ES 全文检索能力最强但吃内存（常驻堆 + 文件缓存）；轻量场景可考虑 **Loki + Grafana**（只索引标签、不索引全文，存储省一个量级），代价是复杂聚合和长文本检索能力弱。
>
> **本文构成**：第一至七章是知识框架（选型原理、学习重点、运维要点、面试高频）；第八至十一章是本机 `jenkins-server`（192.168.128.133）的**实战部署与踩坑记录**，配置片段均已实测可跑。
>
> **相关阅读**：指标侧见 [`Prometheus.md`](./Prometheus.md)；容器侧见 [`docker-note.md`](../container/docker-note.md)。

---

## 目录

**一、为什么选 ELK**
1. 要解决的问题
2. 核心价值
3. 选型取舍

**二、ELK 是什么：组件职责与数据流**

**三、学习重点：Elasticsearch（绝对核心）**

**四、日志采集与解析**

**五、索引生命周期管理（ILM）**

**六、查询与可视化**

**七、运维重点**

**八、实战架构与 Docker Compose 部署**

**九、关键配置文件（实测可用）**

**十、启动与验证**

**十一、踩坑记录（本次实战）**

**十二、面试怎么讲这个项目**

**十三、后续进阶练习**

---

## 一、为什么选 ELK

### 1. 要解决的问题

单机时代查日志的痛点：日志散落在 N 台机器上、轮转后被删、格式各不相同、只能一台台 `grep`。集中化之后才有「全文检索 + 趋势分析 + 告警」的能力。

### 2. 核心价值

| 能力 | 说明 |
|------|------|
| 日志汇总 | 多机器、多来源日志统一入口 |
| 持久化储存 | 集中存储，支持快照（Snapshot）与跨集群同步 |
| 高可用 | 主分片 + 副本分片，节点故障自动故障转移 |
| 多平台采集 | Filebeat 轻量采集器，跨 Linux/Windows/容器 |
| 可视化分析 | Kibana  Discover / Dashboard / Dev Tools |
| 扩容 | 水平加节点即可线性扩容量与查询能力 |
| 稳定性 | 单节点故障不丢数据（前提是副本 ≥ 1） |
| 数据安全 | RBAC + TLS（企业版/付费）或网络隔离 |

### 3. 选型取舍

| 维度 | ELK | Loki + Grafana | 说明 |
|------|-----|----------------|------|
| 全文检索 | 强 | 弱（只索引标签） | 复杂查询选 ELK |
| 存储成本 | 高（存全文倒排） | 低（可只存日志原文 + 标签索引） | 日志量大选 Loki |
| 资源占用 | 高（吃内存） | 低 | 小机器选 Loki |
| 生态成熟度 | 高 | 中 | 长期维护选 ELK |
| 部署复杂度 | 中高 | 低 | 快速试点选 Loki |

**结论**：中小规模、需要全文检索 → ELK；日志量极大、只做标签级聚合 → Loki。个人项目和中小生产环境 ELK 完全够用。

> **纠偏**：「自建/托管」不是二选一。企业用 Elastic Cloud 省运维；个人项目自建即可，成本几乎为零但要自己扛 JVM 调优和磁盘水位。本机这套就是纯自建（`xpack.security.enabled=false`，**仅限内网，切勿直接暴露到公网**）。

---

## 二、ELK 是什么：组件职责与数据流

| 组件 | 作用 | 本项目对应 |
|------|------|-----------|
| **Elasticsearch**（ES） | 存储、搜索、分析日志 | `es` 容器 :9200 |
| **Logstash** | 采集、解析（Grok）、转发 | `logstash` 容器 :5044 |
| **Kibana** | 可视化查询日志 | `kibana` 容器 :5601 |
| **Filebeat** | 轻量级日志采集器，替代 Logstash 做采集端 | `filebeat` 容器 |

> **关键理解**：Filebeat 出现后，**Logstash 退居「解析 + 转发」**。传统做法是 Logstash 一手采集一手解析，但在单机上跑一个 JVM（几百 MB 起）太重，于是拆成：Filebeat（纯 Go，几十 MB，无 JVM）负责采集与轻量处理 → Logstash 负责重解析（Grok / enrich / 聚合）→ ES 存储。

**数据流：**

```
Nginx 产生日志
  -> Filebeat 采集（/var/log/nginx/*.log）
  -> Logstash 解析（Grok 拆字段 / 多行合并）
  -> Elasticsearch 存储（按天建索引）
  -> Kibana 可视化
```

---

## 三、学习重点：Elasticsearch（绝对核心）

| 主题 | 要掌握的内容 |
|------|-------------|
| **倒排索引** | 为什么搜索快；与 MySQL B+ 树的本质区别（字段值 → 文档 ID 的映射） |
| **分片与副本** | 主分片、副本分片；分片数怎么定（一旦创建不可改） |
| **集群状态** | Green / Yellow / Red 的含义与排查路径 |
| **写入流程** | refresh（近实时可见）、translog（崩溃恢复）、flush（落盘 + 清 translog） |
| **查询流程** | Query 阶段（打分/选 doc）vs Fetch 阶段（真正取 `_source`） |
| **Mapping** | `keyword`（不分词、可聚合）vs `text`（分词、不可聚合）的区别 |

### 集群状态速查

| 状态 | 含义 | 常见原因 |
|------|------|----------|
| 🟢 Green | 所有主分片和副本都已分配 | 正常 |
| 🟡 Yellow | 主分片正常，**有副本未分配** | 单节点 + `number_of_replicas: 1` 时**必然 Yellow，属正常** |
| 🔴 Red | 有主分片未分配，**数据不完整** | 磁盘满、节点挂掉、分片损坏 |

> **实战对照**：本机单节点 ES 的 `_cluster/health` 返回 `yellow`、`active_shards: 29`、`unassigned_shards: 1`——**这是单节点部署的预期状态，不是故障**。看到 Yellow 先确认是不是「单节点 + 副本数 ≥ 1」，再去排查。

### Mapping 类型对照

| 类型 | 分词 | 可聚合 | 典型用途 |
|------|------|--------|----------|
| `keyword` | 不分词，整体存储 | ✅ | 状态码、IP、路径、用户名、精确匹配 |
| `text` | 按 analyzer 分词 | ❌ | 日志正文、错误堆栈（需要全文检索时） |

> 默认动态映射会把字符串映射成 `text + keyword` 双字段，所以直接对「未指定 mapping 的文本字段」做聚合会报 `Fielddata is disabled on text fields by default`。**日志字段应在 index template 里显式声明 mapping**，不靠动态推断。

---

## 四、日志采集与解析

| 组件 | 关键点 |
|------|--------|
| **Filebeat** | 轻量（无 JVM）、自动续传（registry 记录 offset）、`filestream` input |
| **Logstash** | Grok 解析、多行日志合并、enrich、聚合 |
| **Grok** | `%{COMBINEDAPACHELOG}`、`%{COMMONAPACHELOG}` 等内置模式解析 Nginx 访问日志 |
| **多行合并** | Java 异常栈的 `at com.foo.Bar(...)` 多行需合并成**一条**事件 |

### 8.x 变更：log input 已废弃

```yaml
# ❌ 8.x 已废弃，日志会打 DEPRECATED 警告
filebeat.inputs:
  - type: log
    paths: [/var/log/nginx/access.log]

# ✅ 8.x 推荐
filebeat.inputs:
  - type: filestream
    id: nginx-access
    enabled: true
    paths:
      - /var/log/nginx/access.log
    prospector.scanner.fingerprint.enabled: false
```

### 多行合并示例

```conf
input {
  beats { port => 5044 }
  file {
    path => "/var/log/app/*.log"
    codec => multiline {
      pattern => "^%{TIMESTAMP_ISO8601}"
      negate => true
      what => "previous"
    }
  }
}
```

---

## 五、索引生命周期管理（ILM）

| 阶段 | 作用 |
|------|------|
| **Hot（热）** | 频繁写入，存最新索引（`nginx-access-2026.10.05`） |
| **Warm（温）** | 写入停止，滚动 / 合并优化 |
| **Cold（冷）** | 降副本、冷存储 |
| **Delete（删除）** | 到期删除 |

**目的**：按天/按大小滚动索引，**防止磁盘被打满**。生产上必须配 ILM（或至少配一个按天 rollover + 保留 N 天的策略），否则日志索引会无限增长直到 `read_only` 保护触发。

---

## 六、查询与可视化

| 工具 | 用途 |
|------|------|
| **Kibana Discover** | 交互式检索原始日志，最常用来「快速查错」 |
| **Visualize** | 基于保存的查询生成图表 |
| **Dashboard** | 组合多个图表做常驻监控面板 |
| **KQL** | Kibana Query Language：`status:500 and client_ip:"1.2.3.4"` |
| **Lucene** | 更底层的查询语法，KQL 表达不了的用它 |

**常用排查查询：**

```bash
# 某 IP 的访问记录
curl -s "http://localhost:9200/nginx-access-*/_search?q=client_ip:1.2.3.4&size=20&pretty"

# 5xx 错误日志
curl -s "http://localhost:9200/nginx-access-*/_search?q=status:[500 TO 599]&size=20&pretty"

# 聚合：Top 20 IP / 状态码分布
curl -s "http://localhost:9200/nginx-access-*/_search?size=0&pretty" -H 'Content-Type: application/json' -d '{
  "aggs": {
    "top_ip": { "terms": { "field": "client_ip", "size": 20 } }
  }
}'
```

> **前置条件**：这些字段级查询和聚合**必须先在 index template 里把字段定义成 `keyword`**。用默认动态映射的话，`client_ip` 会被建成 `text`，聚合直接报错。

---

## 七、运维重点

| 项 | 要点 |
|----|------|
| **ES 内存** | JVM 堆**不超过 32G**（超过就失去指针压缩优势），剩余内存全留给**文件缓存**（FS Cache 决定查询速度） |
| **分片规划** | 太多（碎片多、元数据爆炸、单分片过小）太少（单分片过大、恢复慢）都有问题；**单分片建议 20–50G**；分片数**创建后不可改**，只能 reindex |
| **磁盘水位** | **85% 写报警告 / 90% 触发写入限流 / 95% 全部索引转 read-only** |
| **集群运维** | 节点角色划分、脑裂（`discovery.zen.ping.unicast` 配好）、滚动重启（加 `cluster.routing.allocation.enable` 约束）、Snapshot 备份 |
| **故障排查** | Red 集群、写入变慢、查询变慢、磁盘打满 |

> **磁盘打满的连锁反应**：一旦越过 95%，ES 会把索引置为 `read_only_allow_delete`（熔断保护），此时**写入直接失败**，看起来像 Filebeat/Logstash 挂了，根因却在磁盘。排查顺序：先 `df -h` → 再 `_cat/allocation?v` 看哪个节点水位最高。

---

## 八、实战架构与 Docker Compose 部署

### 实战架构（2026-10-05 实测，kenkins-server / 192.168.128.133）

```
Nginx (:8081) -> Filebeat -> Logstash (:5044) -> Elasticsearch (:9200) -> Kibana (:5601)
                              ^
                        Jenkins (:8080) 同机并存
```

| 组件 | 镜像 | 端口 | 内存占用（实测） |
|------|------|------|-----------------|
| ES | `elasticsearch:8.13.0` | 9200 | ~1014 MiB |
| Kibana | `kibana:8.13.0` | 5601 | ~665 MiB |
| Logstash | `logstash:8.13.0` | 5044 | ~704 MiB |
| Filebeat | `filebeat:8.13.0` | — | ~44 MiB |
| Nginx | `nginx:latest` | 8081 | ~5 MiB |
| Jenkins | `jenkins/jenkins:lts` | 8080 / 50000 | ~802 MiB |

> **端口避让**：Jenkins 容器已占用 **8080**，所以 Nginx 映射到 **8081:80**，这是本部署端口选择的直接原因。

### docker-compose.yml（实测可用）

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    container_name: es
    restart: unless-stopped          # 自愈：无 restart 策略则容器退出后永久躺尸
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false  # 仅限内网！公网必须开启并配 TLS/密码
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    ulimits:
      memlock: { soft: -1, hard: -1 }
      nofile:  { soft: 65536, hard: 65536 }
      nproc: 4096

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.0
    container_name: kibana
    restart: unless-stopped
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on: [elasticsearch]

  logstash:
    image: docker.elastic.co/logstash/logstash:8.13.0
    container_name: logstash
    restart: unless-stopped
    environment:
      - LS_JAVA_OPTS=-Xms512m -Xmx512m   # 默认 1G，本机内存有限，收一收
      - xpack.monitoring.enabled=false
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - "5044:5044"
    depends_on: [elasticsearch]

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.13.0
    container_name: filebeat
    restart: unless-stopped
    user: root
    # ⚠️ 三段式写法：命令名 + 参数，不能只写参数（否则覆盖 entrypoint 直接起不来）
    command: ["filebeat", "-e", "--strict.perms=false"]
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - ./nginx/logs:/var/log/nginx:ro
    depends_on: [logstash]

  nginx:
    image: nginx:latest
    container_name: nginx
    restart: unless-stopped
    ports:
      - "8081:80"
    volumes:
      - ./nginx/logs:/var/log/nginx

volumes:
  es-data:
```

### 宿主机必须补的系统级配置

ES 的内核参数和进程限制**不会自动持久化**，缺失会导致重启后 bootstrap check 失败：

```bash
# /etc/sysctl.d/99-elasticsearch.conf
vm.max_map_count=1048576
vm.swappiness=1

# /etc/security/limits.d/99-elasticsearch.conf
jenkins soft memlock unlimited
jenkins hard memlock unlimited
jenkins soft nofile 65536
jenkins hard nofile 65536
jenkins soft nproc  4096
jenkins hard nproc  4096
root    soft memlock unlimited
root    hard memlock unlimited
root    soft nofile 65536
root    hard nofile 65536

# 生效
sudo sysctl --system
```

> **注意**：limits 改动**只对新登录的 shell / 新启动的进程生效**，`ulimit -n` 仍是 1024 时需要重新登录。这也是为什么容器里还要用 `ulimits:` 显式声明。

---

## 九、关键配置文件（实测可用）

### 1. `~/elk/filebeat/filebeat.yml`

```yaml
filebeat.inputs:
  - type: filestream
    id: nginx-access
    enabled: true
    paths:
      - /var/log/nginx/access.log
    prospector.scanner.fingerprint.enabled: false

output.logstash:
  hosts: ["logstash:5044"]
```

### 2. `~/elk/logstash/pipeline/nginx.conf`

**（A）本机当前部署的精简版**——只打 tag，全文进 `message`：

```conf
input {
  beats { port => 5044 }
}

filter {
  if [message] {
    mutate { add_field => { "service" => "nginx" } }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "nginx-access-%{+YYYY.MM.dd}"
  }
}
```

**（B）进阶版**——用 Grok 拆出结构化字段，才能按状态码 / IP / 路径做 Dashboard：

```conf
input {
  beats { port => 5044 }
}

filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
    tag_on_failure => ["_grokparsefailure"]
  }
  date {
    match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "nginx-access-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

> **落地建议**：先跑 (A) 打通链路（已验证可用），确认数据能进来后，再升级到 (B) 做可视化。直接上 (B) 的话，Grok 匹配失败会进 `_grokparsefailure` 标签，需要配 dead letter queue 兜底。

---

## 十、启动与验证

```bash
# 1. 启动
cd ~/elk && docker compose up -d

# 2. 检查容器（应全部 Up，且带 RestartPolicy=unless-stopped）
docker ps --format "table {{.Names}}\t{{.Status}}"
for c in $(docker ps -aq); do \
  docker inspect --format '{{.Name}} restart={{.HostConfig.RestartPolicy.Name}} state={{.State.Status}}' $c; done

# 3. 产生日志
curl http://localhost:8081
curl http://localhost:8081/hello

# 4. 等 20 秒，查索引与文档数
curl "http://localhost:9200/_cat/indices?v"
curl "http://localhost:9200/nginx-access-*/_count?pretty"

# 5. 打开 Kibana
# http://<服务器IP>:5601
# Stack Management -> Data Views -> 创建 nginx-access-* -> Discover
```

**成功标志：**

| 检查点 | 期望输出 |
|--------|----------|
| ES 索引 | 出现 `nginx-access-2026.10.05` |
| ES 文档数 | 打 N 条请求后 `_count` 恰好 +N |
| Logstash 日志 | `Pipeline started {"pipeline.id"=>"main"}` |
| Logstash 运行中 | `Pipelines running {:count=>1, :running_pipelines=>[:main]}` |
| Filebeat 日志 | `Input 'filestream' starting` + `Connection to ... logstash:5044 established` |

> **注意**：ES 冷启动约需 30–60 秒。`curl localhost:9200` 一开始不通是正常的，**不要在此时就判定 ES 挂了**——用 `docker exec es curl ...` 从容器内测更准确。

---

## 十一、踩坑记录（本次实战）

| 问题 | 原因 | 解决 |
|------|------|------|
| 端口 8080 起不了 | Jenkins 容器占用 8080 | Nginx 改 `8081:80` |
| Filebeat 报 `must be owned by uid=0 or root` | `user: root` 运行时默认 `strict.perms=true`，强制校验配置属主 | compose 加 `command: ["filebeat","-e","--strict.perms=false"]`（保留文件归 jenkins 可编辑） |
| Logstash `No config files found` 后退出 | pipeline 目录为空，Logstash 无业务 pipeline 即正常退出（退出码 0） | 补 `logstash/pipeline/*.conf` |
| Logstash 退出后不复活 | ELK 服务全无 `restart` 策略（只有 Jenkins 容器有） | 全部服务加 `restart: unless-stopped` |
| `Permission denied` 写 config | `~/elk/` 下 4 处 root 属主，filebeat.yml 是 `600 root:root` | `sudo chown -R jenkins:jenkins ~/elk`，目录 755 / 文件 644 |
| Filebeat `DNS lookup failure logstash` | Logstash 已退出 → 容器名无法解析 | 先修 Logstash，链路恢复即自动正常 |
| vim 写配置不完整 | 编辑中断留下 `.swp` 残留 | 删除 `.docker-compose.yaml.swp`；改用 `cat > file << 'EOF'` 写入 |

> **踩坑：命令覆盖 entrypoint**
> 写 `command: ["--strict.perms=false"]` 是**错的**——compose 的 `command` 会**整体覆盖**镜像 entrypoint，容器会试图把 `--strict.perms=false` 当可执行文件，秒退。正确写法必须带上命令名：`["filebeat", "-e", "--strict.perms=false"]`。
>
> **踩坑：属主修复会反噬**
> 把 `filebeat.yml` 从 `root:root` 改成 `jenkins:jenkins` 之后，filebeat 反而进了重启循环——因为它以 root 运行却读到非 root 属主的配置。**不要为了让容器跑而把配置文件改回 root 属主**（那会退回「用户改不了自己配置」的原问题），正确解法是保留文件归 jenkins + 关掉 strict.perms 校验，两个需求同时满足。
>
> **踩坑：单节点 Yellow 不是故障**
> 单节点 ES + 默认 `number_of_replicas: 1` 必然 Yellow（副本无处可放）。`active_shards_percent: 96.67%` + `unassigned_shards: 1` 就是这个状态。**排查 Red/Yellow 前先确认副本数与节点数**。

---

## 十二、面试怎么讲这个项目

> "我用 Docker Compose 搭了一套 ELK + Filebeat 的日志采集链路。Filebeat 采集 Nginx 日志，Logstash 用 Grok 解析，写入 ES 按天建索引，Kibana 做可视化。过程中处理过端口冲突、Filebeat 配置文件属主校验、Logstash 挂载路径为空导致找不到配置、以及容器缺 restart 策略无法自愈等问题。"

**可深挖的追问准备：**

| 追问 | 答法 |
|------|------|
| ES 为什么快？ | 倒排索引，字段值 → 倒排表 → docId，避免全表扫描 |
| 分片和副本作用？分片数怎么定？ | 分片提供水平扩展能力、副本提供故障转移；单分片 20–50G，创建后不可改 |
| Green / Yellow / Red 区别？ | Green 全分配；Yellow 主分片正常、副本未分配；Red 主分片未分配、数据不完整 |
| refresh / flush / translog？ | refresh 让数据近实时可见；flush 落盘并清 translog；translog 记录未提交写入用于崩溃恢复 |
| keyword 和 text 区别？ | keyword 不分词可聚合；text 分词用于全文检索 |
| 日志量太大怎么办？ | ILM：热/温/冷/删除四阶段，按天 rollover + 到期删除 |
| Filebeat 和 Logstash 区别？ | Filebeat 无 JVM、轻量、只做采集；Logstash 吃 JVM、做 Grok 解析和富化 |
| 深分页怎么解决？ | `search_after` / PIT，禁止超大 `from + size`；聚合用 `composite` 分页 |
| ES 和 MySQL 区别？何时用 ES？ | MySQL 事务型、精确查询；ES 全文检索、日志分析、近实时聚合 |

---

## 十三、后续进阶练习

1. **Kibana Dashboard**：访问量趋势、状态码分布、Top IP / Top 路径
2. **错误日志链路**：采集 `error.log`，Grok 解析错误级别与堆栈
3. **告警**：配置 Kibana Alerting，5xx 比例超阈值告警
4. **方案对比**：同样场景搭一套 Loki + Grafana，做存储与查询成本对比
5. **应用日志接入**：Java（Logstash Log4j/Logback appender）、Python、Node
6. **ILM 实操**：给 `nginx-access-*` 配 ILM，观察滚动与删除
7. **K8s 部署 ELK**：DaemonSet 跑 Filebeat、ConfigMap 下发配置、StatefulSet 跑 ES
8. **高可用验证**：加第二个 ES 节点，观察 Yellow → Green 的副本分配过程
