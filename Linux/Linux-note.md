# Linux 学习笔记

> 适用范围：Linux 入门基础 + 常用运维专题。
> 关键词：一切皆文件、目录结构、文件操作、文本三剑客（grep / sed / awk）、用户与用户组、权限与属主、运行级别、会话与进程脱钩。

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
cp [选项] source destdir     # 基本语法
cp /opt /home/apple          # 拷贝到指定目录
cp -r /home/apple /opt       # -r：递归复制整个文件夹
\cp -r /home/apple /opt      # \cp：强制覆盖不提示
cp -p /home/apple/a.txt /tmp # -p：连同权限、属主、时间戳一起复制
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

#### ls

```bash
ls                  # 列出当前目录下的文件名
ls -l               # 长格式：权限、属主、大小、修改时间
ls -a               # 显示隐藏文件（以 . 开头）
ls -la              # 最常用的组合：长格式 + 隐藏文件
ls -lh              # 大小带单位（K/M/G），人眼友好
ls -lR              # 递归列出所有子目录内容
ls -lt              # 按修改时间排序（新的在前）
ls -ld dir          # 看目录本身的信息，而不是目录里面的东西
```

`ls -l` 每一列的含义：

```text
-rwxr-xr-x  1  root  root  26112  Sep 17 14:38  stuscore
│└┬┘└┬┘└┬┘  │   │     │      │        │         └─ 文件名
│ │  │  └── 其他人 o：r-x（可读可执行、不可写）   │
│ │  └───── 属组 g：r-x                          └─ 最后修改时间
│ └──────── 属主 u：rwx
└────────── 类型位：- 普通文件 / d 目录 / l 软链接      链接数（硬链接数）
                                                        大小（字节）
                           属主名   属组名
```

| 首字符 | 类型 |
|--------|------|
| `-` | 普通文件 |
| `d` | 目录 |
| `l` | 软链接 |
| `c` / `b` | 字符设备 / 块设备 |
| `s` / `p` | socket / 管道 |

实例（同一台机器、同一个 `makefile` 目录）：

```text
$ ls
cal.c  in.c  makefile  out.c  stuscore  stuscore.c  stuscore.h

$ ls -lah
总用量 60K
drwxr-xr-x 2 root    root    4.0K 9月  17 14:38 .
drwx------ 5 monitor monitor 4.0K 9月  20 20:08 ..
-rw-r--r-- 1 root    root     169 9月  17 14:37 cal.c
-rw-r--r-- 1 root    root     432 9月  17 14:37 in.c
-rw-r--r-- 1 root    root      93 9月  17 14:30 makefile
-rw-r--r-- 1 root    root     264 9月  17 14:37 out.c
-rwxr-xr-x 1 root    root     26K 9月  17 14:38 stuscore
-rw-r--r-- 1 root    root     274 9月  17 14:37 stuscore.c
-rw-r--r-- 1 root    root     187 9月  17 14:37 stuscore.h
```

> 三处值得留意：① `stuscore` 是编译产物，权限带 `x`，可直接 `./stuscore` 执行；`stuscore.c` 是源码，没有 `x`。② `.` 和 `..` 的权限分别是 `drwxr-xr-x` 和 `drwx------`，**目录的 `x` 决定别人能不能进去、`w` 决定能不能在里面增删文件**（详见第七章）。③ 这一天文件属主全是 `root`，普通用户 `monitor` 改不动——属主问题的处理也见第七章。

`ls` 用错类型的坑：

```text
$ ls -lR ./makefile
-rw-r--r-- 1 monitor monitor 93 Sep 17 14:30 ./makefile

$ cd ./makefile
-bash: cd: ./makefile: Not a directory
```

> `makefile` 是个**普通文件**（首字符是 `-`），不是目录，所以 `cd` 进去会报 `Not a directory`。看一眼 `ls -l` 的第一个字符就能提前避免，不用等报错——`ls -lR` 之所以能列出它，是因为 `-R` 只对目录递归，对文件就是正常列出。

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

### 3.5 文本三剑客（grep / sed / awk）与管道符

> 这一节是 Linux 的生产力工具：**grep 找行、sed 改行、awk 拆列**。日志分析、进程排查、批量改配置全靠它们三个加一根管道。
>

| 工具 | 定位 | 核心动作 | 最常用在 |
|------|------|---------|---------|
| `grep` | 文本搜索工具，按用户指定的模式匹配并打印结果 | **筛选行** | 日志分析、查进程 / 端口 / 用户 |
| `sed` | 字符流编辑器（stream editor） | 按行**过滤与取行**、批量替换 | 改配置、删注释、批量替换 |
| `awk` | 格式化输出程序，命令行里的 Excel | 按列**拆分与统计** | 取列、求和、格式化对齐 |

```bash
ls -la > ./mfdir      # 包含隐藏文件
mv mfdir dir.txt      # 改个好记的名字
cat dir.txt
```

```text
总用量 64
drwxr-xr-x 2 monitor monitor  4096 9月  20 23:06 .
drwxr-xr-x 5 monitor monitor  4096 9月  20 20:48 ..
-rw-r--r-- 1 monitor monitor   169 9月  17 14:37 cal.c
-rw-r--r-- 1 monitor monitor   432 9月  17 14:37 in.c
-rw-r--r-- 1 monitor monitor    93 9月  17 14:30 makefile
-rw-r--r-- 1 monitor monitor     0 9月  20 23:12 mfdir
-rw-r--r-- 1 monitor monitor   264 9月  17 14:37 out.c
-rw-rw-r-- 1 monitor monitor     0 9月  20 23:05 pwd.txt
-rwxr-xr-x 1 monitor monitor 26112 9月  17 14:38 stuscore
-rw-r--r-- 1 monitor monitor   274 9月  17 14:37 stuscore.c
-rw-r--r-- 1 monitor monitor   187 9月  17 14:37 stuscore.h
-rw-r--r-- 1 monitor monitor    38 9月  20 20:48 target.txt
```

---

#### 3.5.1 正则表达式：基本表达式（BRE）与扩展表达式（ERE）

三个工具的模式匹配都建立在正则之上，但**默认方言不同**

| 元字符 | 含义 | 基本正则 BRE | 扩展正则 ERE |
|--------|------|-------------|-------------|
| `.` | 任意一个字符 | 直接写 | 直接写 |
| `*` | 前一个字符重复 0 次或多次 | 直接写 | 直接写 |
| `^` / `$` | 行首 / 行尾 | 直接写 | 直接写 |
| `[]` / `[^]` | 字符集合 / 取反 | 直接写 | 直接写 |
| `?` | 前一个字符出现 0 或 1 次 | `\?` | `?` |
| `+` | 前一个字符出现 1 次或多次 | `\+` | `+` |
| `\|` | 或 | `\|` | `|` |
| `\(\)` | 分组 | `\(\)` | `()` |
| `\{n,m\}` | 重复 n 到 m 次 | `\{n,m\}` | `{n,m}` |

> **BRE 里 `?` `+` `|` `()` `{}` 必须加反斜杠，ERE 里直接写。**
> `grep` / `sed` **默认是 BRE**，加 `-E`（等价于老的 `egrep`）才切到 ERE。记不住就无脑加 `-E`——多打两个字符，不用记住那一堆转义。

字符类（POSIX 写法比手写 `[a-z]` 更可靠，不受 locale 影响）：

| 写法 | 含义 | 写法 | 含义 |
|------|------|------|------|
| `[[:digit:]]` | 数字 | `[[:space:]]` | 空白（空格 / Tab / 换行） |
| `[[:alpha:]]` | 字母 | `[[:upper:]]` `[[:lower:]]` | 大写 / 小写字母 |
| `[[:alnum:]]` | 字母 + 数字 | `[[:punct:]]` | 标点符号 |

```bash
grep -E '^[[:digit:]]{3}$' file    # 整行恰好是 3 位数字
```

> 要匹配**字面量**的特殊字符转义：`grep '\.' file` 才是找含 `.` 的行；不写 `\` 时 `.` 表示「任意一个字符」，几乎会匹配到所有行。`.` 和 `*` 是最高频的误用点。

---

#### 3.5.2 grep：按行筛选

```bash
grep [选项] '模式' 文件
```

| 选项 | 作用 | 选项 | 作用 |
|------|------|------|------|
| `-i` | 忽略大小写 | `-c` | 只输出匹配的行数 |
| `-v` | **反向匹配**，输出不匹配的行 | `-l` / `-L` | 只列文件名（含 / 不含） |
| `-n` | 显示匹配行的行号 | `-w` | 全词匹配 |
| `-o` | 只输出匹配到的那部分 | `-r` / `-R` | 递归搜索整个目录 |
| `-A n` `-B n` `-C n` | 额外显示后 / 前 / 前后 n 行 | `-E` / `-F` | 扩展正则 / 固定字符串（不解析正则） |
| `-q` | 静默模式，只用退出码判断 | `--color=auto` | 匹配处高亮 |

**实例（全部基于上面造的 `dir.txt`）**：

```bash
grep "cal" dir.txt -n        # 4:-rw-r--r-- ... cal.c        ← -n 显示行号
grep "cal" dir.txt -n -i     # 4:...cal.c  14:...Cal.txt     ← -i 后大小写都命中
grep '^d' dir.txt            # drwxr-xr-x ... .   ← ^d：只留目录行
grep '^d' dir.txt -v         # 反过来：排除所有目录行，只看文件
```

> `-i` ：加 `-i` 前只命中 1 行，加上后变成 2 行——文件里被 `vim` 手工加了一行 `Cal.txt`。**忽略大小写后 `cal` 与 `Cal` 都算命中。**

运维里真正高频的四类用法：

```bash
# ① 查日志里的报错，带上下文
grep -n -A2 -B2 "ERROR" /var/log/app.log

# ② 排除注释行与空行，只看有效配置
grep -vE '^\s*(#|$)' /etc/nginx/nginx.conf

# ③ 查进程 / 查端口
ps -ef | grep java
ss -lntp | grep 8080

# ④ 查用户（排除系统伪用户）
grep -v '/sbin/nologin' /etc/passwd
```

> **`ps -ef | grep xxx` 会把自己也匹配出来**——因为 `grep xxx` 这个进程的命令行里就带着 `xxx`。两种解法：
> ```bash
> ps -ef | grep -v grep | grep java   # 再过滤掉 grep 自身
> ps -ef | grep '[j]ava'              # 更优雅：正则 [j]ava 匹配 "java"，但不匹配 "[j]ava"
> ```

---

#### 3.5.3 sed：字符流编辑器

sed 的工作流程：

```text
逐行读取文件
    │
    ▼
模式空间（pattern space，存放当前这一行的临时缓冲区）
    │
    ▼
按「地址定界」判断这一行要不要处理 ── 不匹配 ──→ 直接输出
    │ 匹配成功
    ▼
执行命令（p 打印 / d 删除 / s 替换 / a i c 追加插入替换 …）
    │
    ▼
输出（默认每行都会打印一遍，用 -n 抑制）
```

- **重要用途**：过滤与取行、批量替换。
- **`-n` 与 `p` 配合是核心机制**：sed 默认会把每一行原样打印；
- `-n` 关掉默认输出后，**只有被 `p` 命令选中的行才打印**——所以 `sed -n '/模式/p'` 的语义就是「只打印匹配行」。

| 选项 | 作用 |
|------|------|
| `-n` | 取消默认输出，通常与 `p` 搭配 |
| `-e` | 指定多条命令（`-e 'cmd1' -e 'cmd2'`） |
| `-i` | **直接修改原文件**（默认只输出到屏幕，不落盘） |
| `-i.bak` | 修改前先把原文件备份成 `file.bak` |
| `-r` / `-E` | 使用扩展正则 |
| `-f` | 从脚本文件里读取 sed 命令 |

| 命令 | 作用 | 示例 |
|------|------|------|
| `p` | 打印 | `sed -n '1,5p' f` |
| `d` | 删除（即不输出） | `sed '/^#/d' f` |
| `s///` | 替换 | `sed 's/old/new/g' f` |
| `a` / `i` / `c` | 行后追加 / 行前插入 / 整行替换 | `sed '3a hello' f` |
| `y` | 字符转换（逐字符映射） | `sed 'y/abc/xyz/' f` |
| `q` | 处理到某行就退出 | `sed '5q' f` |

地址定界（决定命令作用在哪几行）：

| 写法 | 含义 | 写法 | 含义 |
|------|------|------|------|
| `3p` | 第 3 行 | `$p` | 最后一行 |
| `3,7d` | 第 3~7 行 | `1,+3p` | 第 1 行及其后 3 行 |
| `/正则/p` | 匹配正则的行 | `/开始/,/结束/d` | 从匹配「开始」到匹配「结束」的区间 |

**实例（基于 `dir.txt`）**：

```bash
sed "/conf/p" dir.txt -n         # 只打印含 conf 的行（-n 抑制默认输出）
sed "/sys/d" dir.txt -n          # 删除含 sys 的行（配 -n 时什么都不剩）
sed "/sys/d" dir.txt             # 不配 -n：打印「除 sys 行以外的所有行」
sed -n '1,5p' dir.txt            # 只取前 5 行
sed -n '$p' dir.txt              # 只取最后一行
sed 's/cal/CAL/g' dir.txt        # 把 cal 全部换成 CAL（只输出屏幕，不改文件）
sed -i.bak 's/monitor/MONITOR/g' dir.txt   # 真正改文件，且先留一份 .bak
```

> **`-i` 是 sed 里最需要小心的参数**：它直接落盘修改，没有任何确认。正确方法是**先不带 `-i` 跑一遍看输出，确认无误再加 `-i`**；更稳妥的写法是 `-i.bak`，它会先把原文件备份成 `dir.txt.bak`。
> **`s///` 不加 `g` 只替换每行的第一处**——这是「替换命令跑了但没全换掉」的最常见原因。
> 分隔符不限于 `/`，路径里斜杠多的时候换成 `#` 或 `|` 更清爽：`sed 's#/usr/local#/opt#g' f`。
> ⚠️ sed 用的是 BRE，**不支持 `\d` `\w` 这类 PCRE 简写**，写 `[0-9]`、`[a-zA-Z]` 才稳。

---

#### 3.5.4 awk：按列处理与格式化输出

awk 的定位：**拥有强大处理能力的格式化输出程序，命令行的 Excel**。强项是「按列拆分 + 统计 + 格式化」。

```bash
awk [选项] '模式{动作}' 文件
```

处理流程：

```text
BEGIN{ }        ← 读文件之前执行一次（初始化变量、打印表头）
    │
逐行读入，默认按「空白」拆分成 $1 $2 … $NF
    │
模式匹配，命中就执行 {动作}
    │
END{ }          ← 全部读完后执行一次（汇总、求平均）
```

| 选项 | 作用 |
|------|------|
| `-F` | **指定输入分隔符**（`-F:` 按冒号切、`-F'\t'` 按 Tab 切） |
| `-v` | **定义或修改一个 awk 内部变量**（如 `-v OFS=,`） |
| `-f` | 从脚本文件读取 awk 命令 |

**输入分隔符 / 输出分隔符**：

| 变量 | 含义 | 说明 |
|------|------|------|
| `FS` | 输入字段分隔符（Field Separator） | 与 `-F` 等价，`-F:` 就是 `FS=":"` |
| `OFS` | 输出字段分隔符（Ouput Field Separator） | 决定 `print $1,$2` 里逗号输出成什么，默认空格 |

```bash
awk -F: '{print $1}' /etc/passwd                 # 按冒号切，取第一列
awk -F: -v OFS=',' '{print $1,$3}' /etc/passwd   # 输入按冒号切、输出用逗号拼
awk 'BEGIN{FS=":";OFS="|"} {print $1,$3}' /etc/passwd   # 在 BEGIN 里设，效果相同
```

> **`FS` 「进来怎么切」、`OFS` 「出去怎么拼」**，输入输出。
> 注意 awk 的默认分隔符是**连续空白（空格 / Tab 混合都行）**，这和 `cut -d' '` 按单个空格切的行为不同——`ls -la` 这种多空格对齐的输出。

**awk 的内置变量**：

| 变量 | 含义 | 变量 | 含义 |
|------|------|------|------|
| `$0` | 整行内容 | `NF` | 当前行的**字段数**（`$NF` 即最后一个字段） |
| `$1`…`$n` | 第 1…n 个字段 | `NR` | 已读入的总行号（多文件时累加） |
| `FS` | 输入字段分隔符 | `FNR` | 当前文件内的行号（每个文件重新计数） |
| `OFS` | 输出字段分隔符 | `FILENAME` | 当前正在处理的文件名 |
| `RS` | 输入**行**分隔符（默认 `\n`） | `ARGC` | 命令行参数个数 |
| `ORS` | 输出**行**分隔符（默认 `\n`） | `ARGV` | 命令行参数数组（`ARGV[0]` 是 awk 本身） |

```bash
awk '{print FILENAME, $0}' a.txt b.txt        # 多文件时标明每行来自哪个文件
awk '{print NR, FNR, $0}' a.txt b.txt         # NR 累加、FNR 每文件归零
awk 'BEGIN{ORS=", "} {print $NF}' dir.txt     # 改了 ORS：多行拼成一行输出
awk 'BEGIN{print ARGC; print ARGV[0], ARGV[1]}' dir.txt
```

> **`ORS` 是输出行分隔符**（默认换行），把它改成 `", "` 就能让 `print` 的结果横排成一行——这是「awk 输出结果想拼成单行」的标准做法。
> **`NR` 与 `FNR` 的区别只在处理多个文件时才显现**：`NR` 一直累加，`FNR` 每换一个文件就归零。
> **`FILENAME` 在多文件处理时定位问题特别好用**，等价于给输出自动带上来源。
> **`ARGC` / `ARGV`** 用来读命令行参数：`ARGC` 是参数个数（含 `awk` 本身），`ARGV[0]` 是 `awk`，从 `ARGV[1]` 开始才是真正的文件或变量。

**自定义变量**：

```bash
awk -v n=3 '{print $n}' dir.txt                          # -v：从命令行传进去
awk 'BEGIN{sum=0} {sum+=$5} END{print sum}' dir.txt      # BEGIN 里初始化
awk '{sum+=$5} END{print sum}' dir.txt                   # 不初始化也能用（空值参与运算按 0）
```

> 用 `-v` 传进去的变量在 BEGIN 块里就可用；而 `-v` 之外的变量有两种定义位置——`BEGIN{}` 里、或直接在动作里用（awk 变量**不需要声明类型**，字符串与数字自动转换，所以 `sum+=$5` 这种没初始化的写法也能跑）。

**常用模式**：

| 写法 | 含义 | 写法 | 含义 |
|------|------|------|------|
| `NR==2` | 第 2 行 | `/正则/` | 匹配正则的行 |
| `NR==2,NR==5` | 第 2 到第 5 行 | `$3>100` | 第 3 列大于 100 |
| `NR>1` | 跳过表头 | `$NF=="x"` | 最后一列等于 x |

**实例（基于 `dir.txt`）**：

```bash
awk '{print $1}' dir.txt                    # 只取第一列（权限）
awk '{print $0}' dir.txt                    # $0 = 整行，等价于 cat
awk '{print "该行是"$0}' dir.txt             # 字符串与字段拼接
awk '{print $1 $3}' dir.txt                 # 两列直接拼接（中间没有任何分隔符！）
awk '{print $1,$3}' dir.txt                 # 带逗号 → 以 OFS（默认空格）分隔
awk 'NR==2{print}' dir.txt                  # 只打印第 2 行
awk 'NR==2,NR==5' dir.txt                   # 打印第 2~5 行
awk 'NR==2,NR==5{print NR $0}' dir.txt      # 带行号打印第 2~5 行
awk '{print $1,$(NF-1),$(NF-2)}' dir.txt    # 靠 NF 反向取列：末列、倒数第二、倒数第三
```

> **`print $1 $3` 与 `print $1,$3` 的区别是 awk 的第一号坑**：逗号会被替换成 `OFS`（默认空格），**不写逗号就是直接拼接**——两列内容会连成一坨，看起来像数据错乱。
> **`$(NF-1)` 这种「用 NF 反向定位」的写法**在处理列数不固定的文本时极其实用（尤其 `ls -la`）：最后一列是文件名、倒数第三列是大小，不用数第几列。

**awk 统计**：

```bash
# 统计 dir.txt 里所有文件大小之和（第 5 列；NR>1 跳过「总用量」行）
awk 'NR>1{sum+=$5} END{print "总计:", sum, "字节"}' dir.txt

# 统计行数（等价于 wc -l）
awk 'END{print NR}' dir.txt

# 按第 1 列（权限）分组计数
awk '{count[$1]++} END{for (k in count) print k, count[k]}' dir.txt

# 列出大于 1000 字节的文件
ls -la | awk 'NR>1 && $5>1000 {print $NF, $5}'
```

> awk 的数组是**关联数组**（下标可以是字符串），`count[$1]++` 就是「以第一列为 key 计数」——日志分析里的「按 IP 统计访问次数」「按状态码分组」用的都是。
> 统计时**别忘跳过表头/汇总行**：`ls -la` 的第一行「总用量」不是文件，`NR>1` 简单过滤。

**格式化输出：print 与 printf**

| 对比项 | `print` | `printf` |
|--------|---------|----------|
| 换行 | **自动**换行 | **不换行**，要手写 `\n` |
| 多字段 | 用逗号分隔，输出时变成 `OFS` | 必须由格式串显式指定 |
| 格式化能力 | 无 | 有（`%s` `%d` `%f` …） |
| 适用 | 快速取列 | 对齐成表格 |

```bash
awk '{print $NF}' dir.txt                       # 输出文件名，自动换行
awk '{printf "%s\n", $NF}' dir.txt              # 等价写法，但 \n 必须自己写
awk '{printf "%-30s %10s\n", $NF, $5}' dir.txt  # 文件名左对齐宽 30、大小右对齐宽 10
```

printf 的格式化输出方法（**类比 C 语言，很好理解**）：

| 格式符 | 含义 | 示例输出 |
|--------|------|---------|
| `%s` | 字符串 | `cal.c` |
| `%d` | 整数 | `169` |
| `%f` | 浮点数 | `169.000000` |
| `%.2f` | 保留两位小数 | `169.00` |
| `%-10s` | 左对齐、宽度 10 | `cal.c     ` |
| `%10s` | 右对齐、宽度 10 | `     cal.c` |
| `%5d` | 整数右对齐、宽度 5 | `  169` |

> **`printf` 忘了写 `\n` 最经典**：所有输出会挤成一整行，看着像命令没生效。
> 注意：**`print` 里的逗号是「字段分隔符」**（会被转成 `OFS`），**`printf` 里的逗号是「参数分隔符」**——两者含义不同，混用会输出一堆莫名其妙的拼接结果。

---

#### 3.5.5 管道符与组合技

`|` 把**前一个命令的标准输出**接到**后一个命令的标准输入**，是三剑客能串成流水线的关键。

和重定向的区别（这两个容易混，一起记）：

| 符号 | 作用 | 数据流向 |
|------|------|---------|
| `>` / `>>` | 重定向到**文件** | 命令 → 文件（覆盖 / 追加） |
| `<` | 从文件读入 | 文件 → 命令 |
| `2>` | 重定向**错误输出** | 命令的 stderr → 文件 |
| `&>` | 标准输出 + 错误输出一起重定向 | 命令 → 文件 |
| `\|` | **管道**：交给下一个命令 | 命令 → 命令 |
| `tee` | 既输出到屏幕、也写入文件 | `cmd \| tee file` |

常见组合（每段都可单独替换）：

```bash
# ① 进程 / 端口排查
ps -ef | grep '[j]ava'
ss -lntp | grep 8080

# ② 从一堆文件里筛出关键字所在文件，去重
grep -rn "timeout" /etc/nginx/ | awk -F: '{print $1}' | sort -u

# ③ 目录里最大的 5 个文件
ls -la | awk 'NR>1{print $5, $NF}' | sort -rn | head -5

# ④ 日志统计：按状态码分组计数（经典 awk | sort | uniq -c 组合）
awk '{print $9}' access.log | sort | uniq -c | sort -rn | head

# ⑤ 实时追日志并过滤
tail -f app.log | grep --line-buffered "ERROR"

# ⑥ 三剑客串起来：grep 选行 → sed 清注释 → awk 取列统计
grep -v '^#' nginx.conf | sed 's/#.*//' | awk '{sum+=$2} END{print sum}'
```

> **`tail -f | grep` 不实时输出**：管道默认带缓冲，grep 会攒够一个块才往外吐，看着像「日志没更新」。加 `grep --line-buffered`（或 awk 里调用 `fflush()`）就能实时。
> **`cat file | grep xxx` 是多余的**：`grep xxx file` 就够了，多起一个进程纯属浪费（这种写法叫 UUOC——无用地使用 cat）。当然数据本来就来自管道时，继续用管道是对的。
> 记住这条主线即可：**grep 决定「哪些行」→ sed 决定「行变成了什么样」→ awk 决定「取哪些列、怎么算」**，中间插 `sort` / `uniq -c` / `head`，就是命令行文本处理的全部基本功。

---

#### 3.5.6 小结

> grep 选行、sed 改行、awk 拆列；用 `|` 把它们串起来，就是命令行里最锋利的一套文本处理组合。

**可以总结**：

1. **正则方言分清**：grep / sed 默认 BRE（`?` `+` `|` `()` `{}` 必须转义），加 `-E` 切 ERE；
2. **grep 四个高频**：`-n` 行号、`-i` 忽略大小写、`-v` 反向、`-c` 计数；上下文用 `-A/-B/-C`；
3. `ps -ef | grep xxx` 会匹配到 grep 自身，用 `grep -v grep` 或 `'[x]xx'` 规避；
4. **sed = 逐行读取 + 模式空间 + 匹配则执行命令**；`-n` 抑制默认输出，配 `p` 才是「只打印匹配行」；
5. **`sed -i` 直接改文件**，先不带 `-i` 预览或加 `.bak`；`s///` 不写 `g` 只替换每行第一处；
6. **awk = 命令行的 Excel**：`-F` 定输入分隔符、`-v` 传变量、`-f` 读脚本；`FS`/`OFS` 管进出、`NF`/`NR` 管字段数与行号；
7. **`print $1,$3`（逗号 → OFS）和 `print $1 $3`（直接拼接）不是一回事**；`printf` 必须手写 `\n`；
8. **管道是串联器**：grep 选行 → sed 改行 → awk 拆列统计，实时日志记得 `--line-buffered`。

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

## 七、权限管理（chmod / chown）

> `chmod` = **ch**ange **mod**e，修改文件或目录的权限。只有**文件属主**或 **root** 能改，普通用户**不能靠它提权**。
> `chown` / `chgrp` 负责改**属主、属组**。「权限位 + 属主 + 属组」三样合起来，才回答得了「谁能对这个文件做什么」。

### 7.1 权限怎么读：九个字符 + 三类身份

`ls -l` 第一列（如 `-rwxr-xr-x`）去掉开头的类型位，剩下 **9 个字符、每 3 个一组**：

| 分组 | 身份 | 简写 | 说明 |
|------|------|------|------|
| 第 1 组 | 属主 | `u`（user） | 文件的拥有者 |
| 第 2 组 | 属组 | `g`（group） | 文件所属组的成员 |
| 第 3 组 | 其他人 | `o`（other） | 既不是属主、也不在属组里的用户 |
| — | 所有人 | `a`（all） | `u`+`g`+`o` 的简写，只在改权限时用 |

```text
-rwxr-xr-x
 └┬┘└┬┘└┬┘
  │  │  └── o：其他人 → r-x
  │  └───── g：属组   → r-x
  └──────── u：属主   → rwx
```

### 7.2 同一个权限位，落在文件上和目录上不是一回事

这是最容易被含糊过去的一点：

| 权限 | 对**文件** | 对**目录** |
|------|-----------|-----------|
| `r` | 可以读取内容（`cat`） | 可以列出里面有哪些文件（`ls`） |
| `w` | 可以修改内容 | 可以**创建 / 删除 / 重命名**里面的文件 |
| `x` | 可以作为程序执行（`./a.sh`） | 可以进入该目录（`cd`）、访问里面的文件 |

> **只给目录 `w` 不给 `x` 是无效的**：能建文件却进不去、也删不掉。目录至少要 `r-x`（即 5）才可用。
> 反过来，**能不能删掉一个文件，看的是它所在目录的 `w`，而不是文件自己的 `w`**——这一点与 Windows 的直觉相反。所以防误删的有效手段是把目录设成 `555`，而不是给文件去掉写权限。

### 7.3 数字模式

`r=4`、`w=2`、`x=1`，三位相加得到一个数字，三个数字依次对应 u / g / o：

```text
7 = rwx      6 = rw-      5 = r-x      4 = r--
3 = -wx      2 = -w-      1 = --x      0 = ---
```

`chmod 755 dir` 就是：属主 `rwx`、属组 `r-x`、其他人 `r-x`。

| 命令 | 等价权限 | 典型用途 |
|------|---------|---------|
| `chmod 644 file` | `rw-r--r--` | 普通文件（网页、配置、源码），**最常用** |
| `chmod 600 file` | `rw-------` | 私密文件，如 `id_rsa` 私钥 |
| `chmod 755 dir` | `rwxr-xr-x` | 目录、可执行程序 |
| `chmod 700 dir` | `rwx------` | 私有目录（家目录默认就是这个） |
| `chmod 777 file` | `rwxrwxrwx` | ⚠️ 所有人全权限，危险，别用 |

### 7.4 符号模式

格式：

```bash
chmod [ugoa][+-=][rwxXst] 文件
```

| 部分 | 取值 | 含义 |
|------|------|------|
| 身份 | `u` / `g` / `o` / `a` | 不写默认 `a`（所有人） |
| 操作 | `+` 加 / `-` 减 / `=` 设为 | `=` 会把没列出的权限一并清掉 |
| 权限 | `r` `w` `x` | 此外还有 `X` `s` `t`，见下 |

```bash
chmod +x script.sh          # 给所有人加执行权限（脚本变可执行）
chmod u+x script.sh         # 只给属主加执行
chmod go-w file             # 去掉属组和其他人的写权限
chmod a+r file              # 所有人可读
chmod u=rwx,g=rx,o= file    # 等价于 chmod 750
chmod -R u+rwX,go+rX dir    # 递归设置目录树的常用组合
```

> 大写 **`X`** 是小写 `x` 的「条件版本」：**只有目录、或原本就带执行权限的文件**才会被加上 `x`。递归处理目录树时必须用它——否则一条 `-R a+x` 会把 `.c`、`.h`、`.txt` 全变成「可执行文件」，既难看又给源码平白加了执行位。

### 7.5 特殊权限：setuid / setgid / sticky

四位数字模式的第一位就是特殊权限位：

| 数字 | 名称 | 符号写法 | 作用 | 在 `ls -l` 里长这样 |
|------|------|---------|------|-------------------|
| 4 | **setuid**（SUID） | `chmod u+s file` | 执行时**以文件属主的身份**运行，而不是调用者身份 | `-rwsr-xr-x` |
| 2 | **setgid**（SGID） | `chmod g+s dir` | 目录下新建的文件**继承该目录的属组** | `drwxr-sr-x` |
| 1 | **sticky** | `chmod +t dir` | 目录里只有**文件属主、目录属主或 root** 能删除文件 | `drwxrwxrwt` |

```bash
chmod 4755 file    # setuid
chmod 2755 dir     # setgid：目录里新建文件自动归属该组，团队共享目录常这么配
chmod 1777 /tmp    # sticky：人人可写、但只能删自己的文件
```

> **`/tmp` 的 `1777` 就是 sticky 的经典应用**——所以谁都能在 `/tmp` 里建文件，却删不掉别人的。
> `setuid` 是把双刃剑：`passwd` 让普通用户也能改密码（它要写 `/etc/shadow`），靠的正是 setuid。但别自己随手给程序加 setuid，一旦程序有输入漏洞，攻击者就能借它拿 root。

### 7.6 改属主与属组：chown / chgrp

权限位和属主是两件独立的事：**属主不对，`chmod` 给再多权限也没用**。

| 命令 | 作用 | 示例 |
|------|------|------|
| `chown 用户 文件` | 改属主 | `chown monitor /home/monitor/makefile` |
| `chgrp 组 文件` | 改属组 | `chgrp monitor /home/monitor/makefile` |
| `chown 用户:组 文件` | 同时改属主与属组 | `chown monitor:monitor file` |
| `chown -R 用户:组 目录` | 递归改整个目录树 | `chown -R monitor:monitor .` |
| `stat -c '%U %G' 文件` | 只看属主与属组 | `stat -c '%U %G' makefile` |

```text
# 改之前
$ stat -c '%U %G' makefile
root root

$ chown monitor /home/monitor/makefile
$ chgrp monitor /home/monitor/makefile

# 改之后
$ stat -c '%U %G' makefile
monitor monitor
```

> **只有 root 能把文件「送」给别人。** 普通用户不能 `chown` 给他人（哪怕是想把自己的文件交出去也不行，这是防止绕开配额与审计的机制），所以 `chown` 实际都由 root / sudo 执行。

### 7.7 实战：把 root 建的目录交还给普通用户

典型场景：用 root 编译或拷贝了一堆文件到某个用户的目录下，属主全是 `root`，用户自己改不了也删不掉。

```text
$ ls -l /home/monitor/makefile
-rw-r--r-- 1 root root   169 Sep 17 14:37 cal.c
-rw-r--r-- 1 root root   432 Sep 17 14:37 in.c
-rw-r--r-- 1 root root    93 Sep 17 14:30 makefile
-rw-r--r-- 1 root root   264 Sep 17 14:37 out.c
-rwxr-xr-x 1 root root 26112 Sep 17 14:38 stuscore      ← 可执行文件
-rw-r--r-- 1 root root   274 Sep 17 14:37 stuscore.c
-rw-r--r-- 1 root root   187 Sep 17 14:37 stuscore.h
```

处理顺序是「**先改属主 → 再按目录/文件分别设权限 → 最后单独补可执行位**」：

```bash
# ① 递归改属主属组（这是根因；权限位再对，属主不对也没用）
chown -R monitor:monitor .

# ② 目录统一 755、文件统一 644（用 find 区分类型，而不是一把 -R 755）
find . -type d -exec chmod 755 {} +
find . -type f -exec chmod 644 {} +

# ③ 只给需要执行的那个文件补执行位
chmod 755 stuscore

# ④ 核对
ls -la
```

```text
-rw-r--r-- 1 monitor monitor   169 Sep 17 14:37 cal.c
-rw-r--r-- 1 monitor monitor   432 Sep 17 14:37 in.c
-rw-r--r-- 1 monitor monitor    93 Sep 17 14:30 makefile
-rw-r--r-- 1 monitor monitor   264 Sep 17 14:37 out.c
-rwxr-xr-x 1 monitor monitor 26112 Sep 17 14:38 stuscore     ← 执行位保留
-rw-r--r-- 1 monitor monitor   274 Sep 17 14:37 stuscore.c
-rw-r--r-- 1 monitor monitor   187 Sep 17 14:37 stuscore.h
```

> **为什么不直接 `chmod -R 755 .`**：那样 `.c`、`.h`、`makefile` 全带上执行位，`ls` 输出一片高亮，看着就乱，给源码加执行位也毫无意义。「**目录 755 + 文件 644 + 需要执行的单独设**」这套三步法适用于绝大多数目录树。
> `find ... -exec chmod ... {} +` 结尾的 `{} +` 表示**把所有匹配到的路径一次性传给同一个 chmod 进程**；换成 `{} \;` 则是每个文件起一个进程，文件多了会慢一个量级。

### 7.8 避坑清单

| 坑 | 说明 |
|----|------|
| `chmod -R 777 /` | ⚠️ **绝对不能执行**。整个系统权限全开、安全归零，且没有一次性回滚的办法。 |
| 报 `Operation not permitted` | 你不是文件属主、也不是 root。**`chmod` 不能提权**，只能由管理员用 `sudo` / `chown` / `setfacl` 处理。 |
| 改了权限还是不能写 | 先看属主对不对（`ls -l` 第三列）；属主不对就先 `chown`。 |
| 给目录加了 `w` 仍进不去 | 目录缺 `x`，补上（`chmod u+rwx dir`）。 |
| `cp` 过来属主变了 | `cp` 默认把属主设成**当前操作用户**；要保留权限与时间戳加 `-p`，属主本身仍需 root 才能改。 |
| 图省事用 `-R 777` | 反面教材。该改的是属主与属组，不是把门全打开。 |

### 7.9 小结

> 权限 = 「属主 / 属组 / 其他人」各 3 位；读 `r`(4)、写 `w`(2)、执行 `x`(1)；落到目录上分别意味着「能列出文件」「能增删文件」「能进去」。**属主不对先 `chown`，权限不对再 `chmod`。**

**六条核心**：

1. `chmod` 改权限位，`chown` / `chgrp` 改属主属组，**属主是前提**；
2. 三位一组依次对应 u / g / o，`a` 是三者之和；
3. 目录的 `w` 管「能不能删里面的文件」，文件的 `w` 管「能不能改内容」——**删文件看目录**；
4. 目录必须有 `x` 才能进得去、才能访问里面的文件；
5. 常用档位：普通文件 `644`、私密文件 `600`、目录与可执行 `755`、私有目录 `700`；
6. **`chmod -R 777` 是红线**；批量整目录用「目录 755 + 文件 644 + 单独补 `x`」三步走。

---

## 八、运行级别

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

## 九、帮助命令

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

## 十、进阶专题：退出登录后下载任务停止：原因与解决/避免方法

> 适用范围：SSH 远程登录、本地终端、`su`/`sudo` 切换的会话。
> 关键词：SIGHUP、会话（session）、进程组、systemd-logind、断点续传。

### 10.1 为什么会停：三条独立的"kill 路径"

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

### 10.2 如何判断进程是否真的"脱钩"

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

### 10.3 解决方案（按推荐度排序）

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

### 10.4 方案选择速查

| 场景 | 推荐做法 | 抗 ①②③ |
|------|---------|--------|
| 临时小文件，马上就完 | `nohup wget -c URL &` | ①② |
| 大文件 / 长时间下载 | `tmux` + `aria2c -c` | ①②（配 linger 可抗 ③） |
| 无人值守、断线要自愈 | systemd service + `Restart=on-failure` + `enable-linger` | ①②③ |
| 已经在跑、来不及重启 | `Ctrl-Z` → `bg` → `disown -h %1` | ①② |
| root 长期任务 | 系统级 systemd unit | ①②③ |

### 10.5 避坑清单

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

### 10.6 一句话结论

> 退出登录后下载停止，**根因是进程仍绑定在登录会话上**（收 `SIGHUP`）或**被 logind 随会话回收**。
> 对策就三层：**`setsid`/`nohup` 脱钩 → `tmux` 保活 → `systemd + linger` 托管**；
> 同时**永远给下载器加断点续传参数（`-c` / `-C -`）**，这样即便被杀也不会前功尽弃。
