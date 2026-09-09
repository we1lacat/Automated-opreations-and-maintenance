# Linux 学习笔记

> 适用范围：Linux 入门基础 + 常用运维专题。
> 关键词：一切皆文件、目录结构、文件操作、用户管理、运行级别、会话与进程脱钩。

---

## 一、核心思想：一切皆文件

```
在 Linux 中把硬件映射为文件，一切皆文件
```

---

## 二、目录结构

| 目录 | 说明 |
|------|------|
| `/` | 根目录 |
| `/root` | 超级管理员家目录 |
| `/home` | 存放不同用户的家目录 |
| `/bin` | Binary — 存放常用指令（指令集） |
| `/sbin` | 超级管理员的系统管理程序存放位置 |
| `/etc` | 配置文件存放 |
| `/usr` | 类似 Program Files，用户程序存放 |
| `/boot` | Linux 启动的核心文件，包括连接文件和镜像文件等 |
| `/proc` | 虚拟目录，系统内存映射（**不可修改**） |
| `/srv` | 服务数据目录（**不可修改**） |
| `/sys` | 系统设备映射（**不可修改**） |
| `/tmp` | 临时文件 |
| `/dev` | 设备管理，把所有硬件以文件形式存放 |
| `/media` | 识别设备后挂载在此处 |
| `/mnt` | 挂载外部文件系统，挂载后进入该目录查看 |
| `/opt` | 给主机额外安装软件所存放的目录 |
| `/var` | 不断扩充文件的存储位置（日志等） |
| `/selinux` | 安全子系统 |

---

## 三、文件与目录操作

### 3.1 路径

| 类型 | 定位起点 | 示例 |
|------|---------|------|
| 绝对路径 | 从根目录 `/` 开始定位 | `/nono/etc/good apple.txt` |
| 相对路径 | 从当前目录开始定位 | `etc/good apple.txt` |

### 3.2 目录操作

#### cd

```bash
cd [target]        # 跳转目标目录
cd /               # 根目录
cd ~               # 原始家目录
cd ..              # 回到上级目录
cd ../../root      # 从 nono 回到 root（多级向上）
```

#### mkdir

```bash
mkdir /home/zipmaker/fomal    # 创建单级目录（父目录不存在会报"没有那个文件或目录"）
mkdir -p /home/zipmaker/fomal # -p：创建多级目录
```

#### pwd / rmdir / rm

```bash
pwd                # 显示当前工作目录
rmdir dir          # 删除空目录（有内容则删除失败）
rm -rf target      # 强制递归删除，慎用！
```

#### touch

```bash
touch route/filename    # 创建一个空文件
```

#### cp

```bash
cp [选项] source dest        # 基本语法
cp /opt /home/apple          # 拷贝到指定目录
cp -r /home/apple /opt       # -r：递归复制整个文件夹
\cp -r /home/apple /opt      # \cp：强制覆盖不提示
```

#### mv

```bash
mv oldnamefile newnamefile           # 重命名文件
mv /temp/movefile /target/route      # 移动文件
```

### 3.3 文件查看

#### cat / more / less

```bash
cat [目标]         # 只读模式查看文件
cat -n file        # -n 显示行号
cat file | more    # 分页浏览文件
```

#### echo

```bash
echo [选项] [内容]     # 输出内容到控制台
echo $HOSTNAME        # 输出主机名
```

#### head / tail

```bash
head -n num file      # 查看文件前 num 行（默认前十行）
tail -n num file      # 查看文件最后 num 行（默认后十行）
tail -f file          # 实时追踪文件更新（看日志常用）
```

### 3.4 输出重定向

| 符号 | 作用 |
|------|------|
| `>` | 输出重定向（覆盖） |
| `>>` | 输出重定向（追加） |

```bash
ls > a.txt        # 覆盖写入
ls >> a.txt       # 追加写入
```

---

## 四、Vim 编辑器

| 模式 | 进入方式 |
|------|---------|
| 插入模式 | 按 `i / I / o / O / a / A / R` 任意键进入，可自由输入 |
| 命令行模式 | `Esc` 退出插入模式后输入命令（`:wq` 保存退出等） |

---

## 五、登录注销与关机重启

```bash
shutdown -h now          # 立即关机
halt                     # 立即关机（同上）
shutdown -h 1 "hello"    # 1 分钟后关机并广播提示
shutdown -r now          # 立即重启
```

---

## 六、用户与用户组管理

> root 用户可创建多用户并进行管理（用户目录在 `/home` 中）。

### 6.1 用户操作

| 命令 | 作用 | 示例 |
|------|------|------|
| `useradd username` | 添加用户（默认创建同名家目录与同名组） | `useradd king` |
| `useradd -d 路径 username` | 指定家目录创建用户 | `useradd -d /home/test king` |
| `passwd username` | 设置/修改密码（不写用户名默认改当前用户） | `passwd king` |
| `userdel username` | 删除用户但保留家目录（一般推荐保留） | `userdel apple` |
| `userdel -r username` | 删除用户并删除家目录 | `userdel -r orange` |
| `id username` | 查询用户是否存在及其信息 | `id king` |
| `su - username` | 切换用户 | `su - root` |
| `whoami` / `who am i` | 查询当前登录系统的用户信息 | — |

> 权限不足时通过 `su -` 切换到高权限用户。**高权限切换低权限不需要输入密码，低权限切换高权限需要输入密码。**

### 6.2 用户组

> 类似角色，对具有共性/相同权限的用户进行统一管理。

| 命令 | 作用 | 示例 |
|------|------|------|
| `groupadd groupname` | 添加用户组 | `groupadd meme` |
| `groupdel groupname` | 删除用户组 | `groupdel meme` |
| `useradd -g groupname username` | 添加用户到指定组 | `useradd -g huanglong jinxi` |
| `usermod -g usergroup username` | 更改用户所属组 | — |

> 未指定组时，`useradd` 会默认创建与用户同名的组。

---

## 七、运行级别

| 级别 | 说明 |
|------|------|
| 0 | 关机 |
| 1 | 单用户（找回丢失密码） |
| 2 | 多用户无网络服务 |
| 3 | 多用户有网络服务（服务器常用） |
| 4 | 系统未使用，保留给用户 |
| 5 | 图形界面 |
| 6 | 系统重启 |

```bash
init [0123456]       # 切换不同运行级别
```

> 单用户模式下可以找回丢失的 root 密码：重启进入运行级别 1，直接用 `passwd` 修改密码（需在物理机/控制台操作）。

---

## 八、帮助命令

### man

```bash
man ls        # 获得命令或配置文件的帮助信息
```

> Linux 中以 `.` 开头的文件是隐藏文件，可组合参数使用，例如 `ls -al /root`。

### help

```bash
help cd       # 获得 shell 内置命令的帮助信息
```

---

## 九、进阶专题：退出登录后下载任务停止：原因与解决/避免方法

> 适用范围：SSH 远程登录、本地终端、`su`/`sudo` 切换的会话。
> 关键词：SIGHUP、会话（session）、进程组、systemd-logind、断点续传。

### 9.1 为什么会停：三条独立的"kill 路径"

| # | 触发机制 | 原理 | 影响范围 | 是否常见 |
|---|---------|------|---------|---------|
| ① | **SIGHUP（终端挂断信号）** | SSH 断开 / 终端窗口关闭 → 内核认为控制终端"挂断"，向前台进程组发送 `SIGHUP`。进程默认动作是终止 | 当前终端的前台进程组 | 最常见 |
| ② | **shell 作业清理** | `bash`/`zsh` 退出时清理作业表；若开启 `huponexit`（`shopt -s huponexit`），会向**所有**作业发 `SIGHUP` | 该 shell 的所有后台作业 | 常见 |
| ③ | **systemd-logind 会话回收** | 最后一个会话结束时，logind 销毁 `session-N.scope` / `user@UID.service`；`KillUserProcesses=yes` 或 `RemoveIPC=yes` 会连带清掉残留进程 | 该用户**所有**进程（含后台） | 图形会话/部分发行版 |

关键细节：

- `SIGHUP` 的**默认动作就是终止进程**，所以不做任何保护的 `wget`/`curl` 一定会死。
- 信号发给的是**进程组**，不是单个进程。管道 `a | b | c` 里的三段都在同一组，会一起被杀。
- `nohup` **只忽略 SIGHUP**，对 ③（logind 直接 kill / cgroup 销毁）无效 —— 这是最常见的误解。
- `&` 放到后台**不等于**脱离会话：它仍在同一 session、同一进程组，照样收 `SIGHUP`。
- 如果只是网络断了（SSH TCP 超时），本地 `wget` 直连其实还能跑；真正致命的是终端/会话消失。

### 9.2 如何判断进程是否真的"脱钩"

```bash
ps -o pid,ppid,pgid,sid,tty,stat,cmd -p <PID>
```

判定标准：

| 字段 | 期望值 | 说明 |
|------|--------|------|
| `TTY` | `?` | 无控制终端 → 已脱离 |
| `SID` | ≠ 登录 shell 的 SID | 独立会话 |
| `PPID` | `1`（或 systemd） | 被 init 收养，父 shell 已消失 |

若 `TTY` 仍显示 `pts/0`、`pts/1`，说明它**还挂在这个终端上**，退出即死。

再看会话与 logind 设置：

```bash
loginctl list-sessions                 # 当前会话
loginctl show-session $XDG_SESSION_ID  # 看 KillProcesses / Scope
grep -E 'KillUserProcesses|RemoveIPC' /etc/systemd/logind.conf
```

### 9.3 解决方案（按推荐度排序）

#### 方案 A：进程脱钩（最快，适合一次性任务）

```bash
# 1) 最稳妥组合：新会话 + 忽略挂断
setsid nohup wget -c -t 0 -o dl.log "URL" &

# 2) 只忽略 SIGHUP
nohup curl -C - -O "URL" > dl.log 2>&1 &

# 3) 只脱离终端（推荐，比 nohup 更彻底）
setsid wget -c "URL" &
```

| 命令 | 作用 | 局限 |
|------|------|------|
| `nohup` | 忽略 `SIGHUP`，stdout 自动重定向到 `nohup.out` | 不脱离终端；不防 logind 清理 |
| `setsid` | 新建 session，彻底脱离控制终端 | 不忽略其它信号（如 `SIGTERM`） |
| `disown` | 从 shell 作业表移除，shell 不再管它 | 只对**已启动**的作业有效；不防 ③ |
| `trap '' HUP` | 脚本内忽略 HUP | 仅限脚本内部 |

补救已经在前台跑起来的任务：

```bash
Ctrl-Z                 # 暂停
bg %1                  # 转后台
disown -h %1           # 标记：shell 退出时不发 SIGHUP
```

#### 方案 B：终端复用器（交互式任务首选）

```bash
# tmux
tmux new -s dl
# 在 tmux 里启动下载，然后 Ctrl-b d 分离
tmux ls
tmux attach -t dl      # 下次登录重连，进度还在

# screen
screen -S dl
# Ctrl-a d 分离
screen -r dl

# 直接后台建会话（非交互式）
screen -dmS dl bash -c 'wget -c URL; exec bash'
```

> 注意：tmux server 本身也在用户会话里，若被 ③ 回收同样会死。**配合方案 C 的 linger 使用最稳。**

#### 方案 C：systemd 托管（长期/服务器/需自动重启，最健壮）

```bash
# 1) 允许用户进程在登出后存活（关键！）
sudo loginctl enable-linger $USER
# 验证
ls /var/lib/systemd/linger/

# 2) 跑一个临时托管任务
systemd-run --user --unit=mydl --remain-after-exit \
  /usr/bin/wget -c -t 0 -O /data/big.iso "URL"

# 查看 / 跟进
systemctl --user status mydl
journalctl --user -u mydl -f
```

写成正式服务单元（开机/断线可自愈）：

```ini
# ~/.config/systemd/user/dl.service
[Unit]
Description=Long download job
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/aria2c -c -x 16 -s 16 -d /data "URL"
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now dl.service
```

系统级（root，不受任何用户会话影响）：

```bash
sudo tee /etc/systemd/system/dl.service >/dev/null <<'EOF'
[Unit]
Description=Download job
After=network-online.target
[Service]
Type=simple
User=ops
WorkingDirectory=/data
ExecStart=/usr/bin/aria2c -c -d /data "URL"
Restart=on-failure
RestartSec=10
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable --now dl.service
```

#### 方案 D：下载器自身的断点续传（兜底，必配）

即使被杀，也能从断点继续，**前提是服务器支持 Range 请求**：

```bash
curl -I "URL" | grep -i accept-ranges    # 必须返回 bytes
```

| 工具 | 续传命令 | 备注 |
|------|---------|------|
| wget | `wget -c -t 0 --timeout=30 URL` | `-t 0` 无限重试；`-b` 后台 + `-o log` |
| curl | `curl -C - -O --retry 10 --retry-delay 5 URL` | `-C -` 自动续传 |
| aria2c | `aria2c -c -x 16 -s 16 -d /data URL` | 多线程，大文件首选；`-D` 守护化 |
| axel | `axel -a -n 16 URL` | 轻量多线程 |
| rsync | `rsync -P --append user@host:file ./` | `-P` = `--partial --progress` |

校验完整性（续传后务必做）：

```bash
sha256sum big.iso          # 与官方 checksum 比对
wget -c --spider URL       # 探测是否支持续传
ls -l --time-style=full-iso big.iso   # 观察大小是否仍在增长
```

### 9.4 方案选择速查

| 场景 | 推荐做法 | 抗 ①②③ |
|------|---------|--------|
| 临时小文件，马上就完 | `nohup wget -c URL &` | ①② |
| 大文件 / 长时间下载 | `tmux` + `aria2c -c` | ①②（配 linger 可抗 ③） |
| 无人值守、断线要自愈 | systemd service + `Restart=on-failure` + `enable-linger` | ①②③ |
| 已经在跑、来不及重启 | `Ctrl-Z` → `bg` → `disown -h %1` | ①② |
| root 长期任务 | 系统级 systemd unit | ①②③ |

### 9.5 避坑清单

1. **`nohup` 不是万能的** —— 它只挡 `SIGHUP`，挡不住 logind 的 cgroup 清理。长任务请加 `setsid` 或走 systemd。
2. **管道里只有第一个命令被保护**：`nohup wget URL | tee log &` 中 `nohup` 只作用于 `wget`，`tee` 仍会死。整条管道请包一层：`nohup bash -c 'wget URL | tee log' &`
3. **`wget -c` 需要服务端支持 Range**，否则会重新下载或报错；先用 `curl -I` 确认 `Accept-Ranges: bytes`。
4. **后台任务的输出要重定向**，否则写满 `nohup.out` 或阻塞在已消失的终端上。
5. **开启 linger 后 `user@UID.service` 会一直存在**，注意它托管的进程会长期占用内存。
6. **不要用 `kill -9` 收尾**：先 `kill -TERM` 让下载器正常落盘（`.part`/`.aria2` 文件），否则续传元信息可能丢失。
7. **SSH 侧也可加固**，减少"假死断连"：

   ```bash
   # ~/.ssh/config
   Host *
       ServerAliveInterval 30
       ServerAliveCountMax 3
       TCPKeepAlive yes
   ```

### 9.6 一句话结论

> 退出登录后下载停止，**根因是进程仍绑定在登录会话上**（收 `SIGHUP`）或**被 logind 随会话回收**。
> 对策就三层：**`setsid`/`nohup` 脱钩 → `tmux` 保活 → `systemd + linger` 托管**；
> 同时**永远给下载器加断点续传参数（`-c` / `-C -`）**，这样即便被杀也不会前功尽弃。
