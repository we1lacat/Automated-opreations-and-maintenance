# Linux 排障手册

> 本文件收录从 `Linux/Linux-note.md` 抽离出来的**问题排查 / 避坑 / 事故复盘**内容。
> 主笔记只保留 RHCSA 考试范围内的必要注意；所有「为什么错了、怎么查、怎么修」的展开都归这里。

**使用方式**：按症状反查。每条都是「现象 → 根因 → 处理」三段式，急着修复就直接看最后的命令。

**目录**

1. [权限与属主](#一权限与属主)
2. [文件与目录操作](#二文件与目录操作)
3. [文本三剑客（grep / sed / awk）](#三文本三剑客grep--sed--awk)
4. [磁盘、挂载与 LVM](#四磁盘挂载与-lvm)
5. [进程、服务与日志](#五进程服务与日志)
6. [防火墙与网络](#六防火墙与网络)
7. [登录、会话与长任务](#七登录会话与长任务)
8. [用户与密码](#八用户与密码)

## 一、权限与属主

### 报 `Operation not permitted`

**现象**：`chmod` / `chown` 报 `Operation not permitted`。

**根因**：你不是文件属主、也不是 root。**`chmod` 不能提权**。

**处理**：
```bash
sudo chmod 755 file          # 管理员执行
sudo chown user:group file   # 属主不对时，先改属主再改权限
```

### 改了权限还是不能写

**现象**：明明 `chmod` 成功了，写文件仍失败。

**根因**：属主不对——权限位再全，属主不是你一样进不去。

**处理**：先 `ls -l` 看第三列属主，不对就 `chown`：
```bash
ls -l file                    # 看属主 / 属组
chown $(id -un) file          # 改回自己
```

### 给目录加了 `w` 仍进不去

**现象**：`chmod u+w dir` 后仍报 `Permission denied`。

**根因**：目录缺 `x`。目录至少要 `r-x`（5）才可用。

**处理**：
```bash
chmod u+rwx dir               # w 和 x 一起补
```

### `cp` 过来属主变了

**现象**：root `cp` 给普通用户的文件，用户改不动。

**根因**：`cp` 默认把属主设成**当前操作用户**。

**处理**：
```bash
cp -p src dst                 # 保留权限与时间戳（属主本身仍需 root 才能改）
chown user:group dst          # root 下手动改回去
```

### 图省事用 `-R 777`

**现象 / 后果**：整个目录树权限全开，源码也带上执行位，`ls` 一片高亮，安全归零。

**正确做法**：目录 755 + 文件 644 + 需要执行的单独设：
```bash
find dir -type d -exec chmod 755 {} +
find dir -type f -exec chmod 644 {} +
chmod 755 dir/needs_exec
```

> ⚠️ `chmod -R 777 /` 是**绝对红线**：系统权限全开且没有一次性回滚的办法。

## 二、文件与目录操作

### `cd` 报 `Not a directory`

**现象**：`cd ./makefile` 报 `-bash: cd: ./makefile: Not a directory`。

**根因**：目标是普通文件（`ls -l` 首字符是 `-`），不是目录。

**处理**：不用等报错，`ls -l` 看第一个字符就能提前判断：
```bash
ls -l ./makefile
# -rw-r--r-- 1 monitor monitor 93 Sep 17 14:30 ./makefile   ← 首字符 -，普通文件
```

> `ls -lR ./makefile` 能列出它，是因为 `-R` 只对目录递归、对文件就是正常列出——别被输出误导。

### 隐藏文件看不到

**现象**：`ls` 看不到某些文件。

**根因**：`.` 开头的文件是隐藏文件。

**处理**：`ls -a`（配合 `-l` 即 `ls -al`）。

## 三、文本三剑客（grep / sed / awk）

### awk：`print $1 $3` 与 `print $1,$3` 结果不一样

**现象**：两列内容连成一坨，像数据错乱。

**根因**：**逗号是字段分隔符**（会被替换成 `OFS`，默认空格）；不写逗号就是**直接拼接**。这是 awk 的第一号坑。

**处理**：
```bash
awk '{print $1 $3}' file      # 直接拼接：a1b1
awk '{print $1,$3}' file      # 带空格：a1 b1
awk 'BEGIN{FS=":";OFS="|"} {print $1,$3}' file   # 自定义分隔符
```

### awk 默认分隔符是「连续空白」

**现象**：用 `awk` 切 `ls -la` 这种多空格对齐的输出，列对不上。

**根因**：awk 默认 `FS` 是**连续空白**（空格 / Tab 混合都算一个分隔符），与 `cut -d' '` 按单个空格切的行为不同。

**处理**：多空格对齐的列用 awk 反而更稳；要按固定单字符切再用 `cut -d' '`。

### `print` 与 `printf` 的逗号含义不同

**现象**：`printf` 里用了逗号，输出一堆莫名其妙的拼接结果。

**根因**：`print` 的逗号是**字段分隔符**（转 `OFS`），`printf` 的逗号是**参数分隔符**。

**处理**：`printf` 用格式串控制，不要用逗号拼：
```bash
awk '{printf "%s|%s\n", $1, $3}' file    # 正确
```

### grep 匹配不到 / 匹配太多

**现象**：`grep` 结果为空或包含无关行。

**处理**：
```bash
grep -i pattern file          # 忽略大小写
grep -v pattern file          # 反选（排除）
grep -n pattern file          # 带行号，定位用
grep -E 'a|b' file            # 扩展正则，或不用转义
```

## 四、磁盘、挂载与 LVM

### ⚠️ 事故复盘：改 `/etc/fstab` 污染 SELinux 标签 → 开机根分区只读

> 亲历案例，排查花了很久，代价不小，**务必记住这条铁律**。

**现象**：改完 fstab 重启后，根分区停在 `ro`，auditd / kdump / dbus / NetworkManager 连锁起不来，网卡 DOWN。控制台下 root 怎么看文件都正常，但开机却读不了 fstab。

**根因**：往 `/etc/fstab` 写内容时用了 `/tmp` 中转 + `mv`：
```bash
grep -vE 'vgdata' /etc/fstab > /tmp/fstab.new   # /tmp 下生成的文件带 user_tmp_t 标签
echo '...' >> /tmp/fstab.new
mv /tmp/fstab.new /etc/fstab                     # mv 保留源文件标签 → /etc/fstab 变成 user_tmp_t
```
开机时 `systemd-fstab-generator`（受限域）读到带 `user_tmp_t` 标签的 `/etc/fstab` 被 SELinux **拒绝** → `systemd-remount-fs` 失败 → 根停留 `ro` → 全家桶挂掉。root 控制台是 `unconfined_t` 不受限，所以表面看不出异常——**只有 `ls -Z` 能现形**：
```bash
ls -Z /etc/fstab
# 异常：unconfined_u:object_r:user_tmp_t:s0        （正常应为 etc_runtime_t:s0 或 etc_t:s0）
```

**修复**：
```bash
restorecon -v /etc/fstab            # 一条命令把标签改回合法类型，重启即恢复
```

**铁律（以后写系统目录文件）**：
- **禁止 `/tmp` 中转 + `mv` 到 /etc 等系统目录**。要么原地改（`sed -i`），要么 `>>` 追加，要么改完补 `restorecon`。
- 改完 fstab 务必 `mount -a` 验一遍；怕标签问题就在脚本末尾统一 `restorecon -v /etc/fstab`。

### `umount` 报 `target is busy`

**现象**：卸载时报 `umount: /xxx: target is busy`。

**根因**：还有进程在用这个目录（当前 shell 的 cwd 就在里面最常见）。

**处理**：
```bash
cd /                              # 先退出目标目录
lsof +D /db-bak/                  # 查谁在用
fuser -vm /db-bak/                # 另一种查法
umount /db-bak/
```

> 正在被写入时强行卸载容易丢数据，**卸载前先 `sync`**。

### 改完 fstab 不生效

**现象**：`mount -a` 没挂上，或提示 `systemd still uses the old version`。

**处理**：
```bash
mount -a                           # 先试这个
systemctl daemon-reload            # systemd 仍认旧配置时再刷
```

### 开机进救援模式 / 根分区只读

| 症状 | 根因 | 处理 |
| --- | --- | --- |
| 根分区 `ro`，服务全挂 | fstab 根行 `options` 误写 `ro` | 进救援模式改回 `defaults` |
| 根分区 `ro`，`ls -Z` 标签异常 | SELinux 标签被污染 | `restorecon -v /etc/fstab`，见上方事故复盘 |
| 起不来但能进 shell | fstab 设备名写错 / 设备缺失 | 检查设备是否存在，建议改用 `UUID=` |

> 改 fstab 前先备份：`cp /etc/fstab /etc/fstab.bak`。

### 易错点速查

| 易错点 | 正确做法 |
| --- | --- |
| `unmount` 命令不存在 | 卸载是 `umount`（少一个 n） |
| 二级挂载点的 systemd 单元名 | `/mnt/mydata` 的单元名是 `mnt-mydata.mount`（路径化转义），不是 `mydata.mount`；查状态用 `systemctl is-active mnt-mydata.mount` |
| 逻辑分区不能单独 `w` / 格式化 | `/dev/sda5` 要通过 `/dev/sda4` 才能访问，先格式化扩展分区 |
| fdisk 用 `-t` 换分区表格式 | `g` 才是换分区表格式；`t` 是改单个分区类型，`l` 是列出可用类型 |

### LVM 相关

**逻辑卷扩容了，`df` 看还是老容量**：
```bash
xfs_growfs /db-bak       # xfs：扩 LV 后必须再扩文件系统
resize2fs /dev/vgdata/mydata   # ext4 同理
```

**删除时提示卷组 / 物理卷仍在使用中**：创建顺序 PV→VG→LV，删除反过来 LV→VG→PV，且删 LV 前必须已卸载。

**快照失效（`Invalidated`）**：原 LV 持续写入、改动块累积超过快照容量。容量给原 LV 的 10%~20% 起，别长期挂着。

**xfs 缩不了容**：xfs 只能扩不能缩，要缩小只能重新格式化再拷数据。

## 五、进程、服务与日志

### 服务起不来

**处理**：
```bash
systemctl --failed                # 排障第一步：先看谁挂了
systemctl status 服务名            # 看状态 + 最近日志
journalctl -u 服务名 -n 50         # 看该服务最近的 50 行日志
journalctl -p err --since "1 hour ago"   # 只看最近一小时的 error
```

### 进程占满资源

**处理**：
```bash
top                               # 按 P 排 CPU、按 M 排内存
ps aux --sort=-%cpu | head        # 快照式找最耗 CPU 的进程
kill -15 PID                      # 先礼后兵；-9 是兜底
```

### 时间不同步引发的怪事

**现象**：Kerberos 认证失败、证书校验报错、日志时间线错乱。

**处理**：
```bash
timedatectl                       # 看当前时间与时区
systemctl enable --now chronyd    # 启用时间同步
chronyc sources                   # 确认在同步
```

> K8s 集群对节点时间差尤其敏感，etcd 依赖一致的时间基线。

## 六、防火墙与网络

### 服务起来了但连不上

**处理**（按顺序查）：
```bash
ss -aptuln | grep 端口            # ① 服务真的在监听吗
systemctl status firewalld        # ② 防火墙开着吗
firewall-cmd --list-all           # ③ 当前 zone 放行了什么
firewall-cmd --add-service=http --permanent   # ④ 没放行就加
firewall-cmd --reload             # ⑤ permanent 必须 reload 才生效
```

> 少加 `--permanent` 的话重启就丢。

### SSH 连接被拒 / 超时

**处理**：
```bash
systemctl status sshd             # 服务在跑吗
ss -aptuln | grep 22              # 在监听吗
firewall-cmd --list-all           # 22 放行了吗
journalctl -u sshd -n 50          # 看拒绝原因
```

> 改 `sshd_config` 前先**保留一个已连接的会话**，改错了还能从旧会话改回来——否则可能把自己锁在外面。

## 七、登录、会话与长任务

> 相关背景见主笔记「进阶专题：退出登录后，下载任务为什么会停」。

### `nohup` 没保住任务

**现象**：明明加了 `nohup ... &`，退出登录后进程还是死了。

**根因**：`nohup` 只挡 `SIGHUP`，挡不住 logind 的 cgroup 清理。

**处理**：加 `setsid` 或走 systemd：
```bash
setsid wget URL > /tmp/dl.log 2>&1 < /dev/null &
```

### 管道里只有第一个命令被保护

**现象**：`nohup wget URL | tee log &` 中 `wget` 活着、`tee` 死了。

**根因**：`nohup` 只作用于紧跟它的那一个命令。

**处理**：整条管道包一层：
```bash
nohup bash -c 'wget URL | tee log' &
```

### 后台任务输出异常

**现象**：任务阻塞不动，或目录里冒出巨大的 `nohup.out`。

**根因**：输出没重定向，写满或阻塞在已消失的终端上。

**处理**：启动时就把三个流都接走：
```bash
setsid wget URL > /tmp/dl.log 2>&1 < /dev/null &
```

### `wget -c` 断点续传不生效

**现象**：重新下载或报错。

**根因**：服务端不支持 Range 请求。

**处理**：
```bash
curl -I URL | grep -i accept-ranges    # 确认 Accept-Ranges: bytes
```

### tmux 会话也死了

**根因**：tmux server 本身也在用户会话里，若被 logind 回收同样会死。

**处理**：配合 systemd linger 使用最稳（`loginctl enable-linger 用户名`）。

### linger 的副作用

**现象**：`user@UID.service` 一直存在，内存被长期占用。

**处理**：不用时 `loginctl disable-linger 用户名`，或 `systemctl --user stop` 掉托管的进程。

### 别用 `kill -9` 收尾

**现象**：续传元信息丢失，`.part` / `.aria2` 文件残留。

**处理**：先 `kill -TERM` 让下载器正常落盘，`-9` 只作兜底。

### SSH 侧加固（减少假死断连）

```bash
# ~/.ssh/config
Host *
    ServerAliveInterval 30
    ServerAliveCountMax 3
    TCPKeepAlive yes
```

## 八、用户与密码

### 密码相关

**处理**：
```bash
passwd                 # 改当前用户密码
passwd 用户名           # root 改别人密码
chage -d 0 用户名       # 强制用户下次登录改密码
```

> 密码规范且能记住即可；**忘记不能查询**（`/etc/shadow` 里是哈希，不可逆），只能 root 重置。

### `su` 切换失败

**现象**：`su - 用户名` 报 `Authentication failure`。

**根因**：高级切低级不需要密码，**低级切高级需要目标用户密码**。

**处理**：确认当前身份（`whoami`）与目标用户，或用 `sudo -u 目标用户`。

