# Ansible 学习笔记（RHCE EX294）

> 适用范围：RHCE EX294（基于 RHEL 10）+ Ansible 自动化运维。
> 关键词：控制节点、受管节点、inventory、playbook、模块、变量、角色、Vault、Collections。
>
> **考纲基线**：官方 EX294 已基于 RHEL 10，自动化考题占比约 55%–75%（Ansible 是绝对主角）。
> 所有配置必须**重启后无需人工干预仍然生效**——写完 playbook 务必 `reboot` 验证。

---

## 一、架构与安装（控制节点）

### 1.1 架构：控制节点 vs 受管节点

Ansible 是**无代理（agentless）**架构：控制节点装好 Ansible 即可，受管节点**不需要装任何客户端**，
只要求能被 SSH 连上、且装了 Python 3。

| 组件 | 职责 |
| --- | --- |
| 控制节点（control node） | 装 Ansible，写 playbook，发起任务 |
| 受管节点（managed node） | 被管理的机器，只需 SSH + Python 3 |
| inventory | 记录受管节点清单（IP / 分组 / 变量） |
| module | 实际干活的单元（copy、service、user…），通过 SSH 推到受管节点执行 |

> 「无代理」是 Ansible 相对 Salt / Puppet 的最大优势：受管端零依赖，下线即干净。
> 代价是每次执行都走 SSH，大规模场景要调 `forks` 提高并发（见 §8.3）。

### 1.2 安装

```bash
# RHEL 10 / 9：装完整版（含 ansible-navigator、社区 collections）
dnf install -y ansible

# 只装核心（最小化，常用模块都在）
dnf install -y ansible-core

ansible --version                # 确认版本与配置文件路径
ansible --version | head -2      # 顺带看用的是哪个 ansible.cfg
```

> `ansible` 包 = `ansible-core` + 大量社区 collections；考纲要求「安装必需的包」，装哪个都行。
> 控制节点**不能是 Windows**，只能是 Linux / macOS（Windows 只能当受管节点）。

### 1.3 配置文件 ansible.cfg

四个位置按优先级从高到低（**前者覆盖后者**）：

| 优先级 | 位置 | 说明 |
| --- | --- | --- |
| 1（最高） | 环境变量 `ANSIBLE_CONFIG` | `ANSIBLE_CONFIG=./my.cfg ansible-playbook site.yml` |
| 2 | 当前目录 `./ansible.cfg` | **项目级配置，最常用** |
| 3 | `~/.ansible.cfg` | 用户级 |
| 4（最低） | `/etc/ansible/ansible.cfg` | 系统默认 |

```ini
# ansible.cfg 最小可用配置
[defaults]
inventory      = ./inventory/hosts.ini   # 指定清单文件
remote_user    = ops                     # 默认 SSH 用户
host_key_checking = False                # 首次连接不验证指纹（实验室省事，生产不建议）
forks          = 20                      # 并发数，默认 5
log_path       = ./ansible.log           # 日志落盘

[privilege_escalation]
become         = True                    # 默认提权
become_method  = sudo
become_user    = root
become_ask_pass = False
```

> `host_key_checking = False` 在考试里很有用（第一次连新节点不卡在指纹确认），生产环境应该保持 `True`。
> 不确定当前生效的是哪个配置：`ansible-config view` 看最终合并结果，`ansible-config dump --only-changed` 只看与默认不同的项。

### 1.4 inventory：静态清单

INI 格式（最常用）：

```ini
# inventory/hosts.ini
# 单主机
node1.example.com

# 分组
[web]
node1.example.com
node2.example.com

[db]
db1.example.com
db2.example.com

# 组内变量（组里所有主机生效）
[web:vars]
http_port=80
ansible_user=ops

# 嵌套组：web 和 db 都归 datacenter 管
[datacenter:children]
web
db

# 主机级变量（内联，只影响这一台）
[db]
db1.example.com  http_port=8080
```

YAML 格式（复杂场景更清晰）：

```yaml
# inventory/hosts.yml
all:
  children:
    web:
      hosts:
        node1.example.com:
          http_port: 80
      vars:
        ansible_user: ops
    db:
      hosts:
        db1.example.com:
```

验证清单写得对不对：

```bash
ansible-inventory --list -y          # 打印解析后的清单（排障第一步）
ansible-inventory --graph            # 树状图看分组
ansible all --list-hosts             # 列出所有主机
ansible web --list-hosts             # 只看某组
```

> 组名别用 `all`、`ungrouped`、`localhost`——这三个是 Ansible 保留字，自定义会撞车。

---

## 二、受管节点与连通性

### 2.1 SSH 密钥分发

考纲明确要求「在受管节点上创建并分发 SSH 密钥」：

```bash
# 控制节点生成密钥
ssh-keygen -t ed25519 -f ~/.ssh/ansible_key -N ""

# 把公钥推到每台受管节点
ssh-copy-id -i ~/.ssh/ansible_key.pub ops@node1.example.com
ssh-copy-id -i ~/.ssh/ansible_key.pub ops@node2.example.com

# inventory 里指定私钥
```

```ini
[all:vars]
ansible_user=ops
ansible_ssh_private_key_file=~/.ssh/ansible_key
```

### 2.2 sudo 提权

受管节点上给运维账号配 sudo（RHCE 提权 = sudo，不是直接给 root 密码）：

```bash
# 在受管节点上执行
echo 'ops ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/ops
chmod 440 /etc/sudoers.d/ops
```

Ansible 侧对应：

```ini
# ansible.cfg
[privilege_escalation]
become=True
become_method=sudo
become_user=root
become_ask_pass=False
```

或在 playbook 里按需开启：

```yaml
- hosts: web
  become: yes                # 整个 play 提权
  tasks:
    - name: 某个不需要提权的任务
      ansible.builtin.command: uptime
      become: no             # 单个任务关闭
```

### 2.3 ad-hoc 命令：快速验证与一次性操作

不用写 playbook，一条命令直接对多台机器操作：

```bash
ansible all -m ping                              # ① 验证连通性（最常用的第一条）
ansible web -m ping                              # 只测某组
ansible web -a "uptime"                          # ② 执行命令（默认 command 模块）
ansible web -m command -a "df -h"                # 同上，显式写模块
ansible db -m shell -a "ps aux | grep mysql"     # 需要管道/重定向时用 shell
ansible web -m copy -a "src=/tmp/a.txt dest=/tmp/"   # ③ 传文件
ansible all -m dnf -a "name=httpd state=latest" --become   # ④ 装包（提权）
ansible web -m service -a "name=httpd state=restarted" --become
ansible web -m user -a "name=king state=present" --become
ansible all -m setup                             # 收集 facts（信息量大，配合 filter）
ansible all -m setup -a "filter=ansible_distribution*"     # 只看发行版相关
```

> 考试里 `ansible all -m ping` 是**连通性验证的标准第一步**，不过就说明 SSH / Python / inventory 至少一处有问题。
> `command` 模块不支持管道和重定向（`|` `>` `<` 会被当成普通字符）；需要这些用 `shell`。

#### 用 shell 脚本跑 ad-hoc（考纲明确点）

把多条 ad-hoc 命令串成脚本，批量执行：

```bash
#!/bin/bash
# deploy.sh —— 一条命令完成多台机器的安装与启动

set -e
ansible web -m dnf -a "name=httpd state=latest" --become
ansible web -m service -a "name=httpd state=started enabled=yes" --become
ansible web -m copy -a "src=./index.html dest=/var/www/html/" --become
ansible web -m uri -a "url=http://{{ inventory_hostname }}/ return_content=yes"
echo "部署完成"
```

> 超过两三台机器、或要重复执行时，就该把 ad-hoc 脚本升级成 playbook——脚本没有幂等性检查，重复跑可能装重复。

---

## 三、Playbook 基础

### 3.1 YAML 结构与 play / task

```yaml
---
# 一个 playbook 可以包含多个 play
- name: 配置 web 服务器           # play 名（可选但建议写）
  hosts: web                    # 作用于哪组主机
  gather_facts: yes             # 是否收集 facts（默认 yes）
  become: yes                   # 是否提权

  tasks:                        # 任务列表，顺序执行
    - name: 安装 httpd           # task 名（建议写，日志里就是它）
      ansible.builtin.dnf:
        name: httpd
        state: present

    - name: 启动并启用 httpd
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: yes

- name: 配置数据库               # 第二个 play
  hosts: db
  tasks:
    - name: ...
```

要点：

- `---` 开头是 YAML 文档起始标记，建议写上；
- **playbook 是任务导向**：跑过的任务不会重复跑（模块内部做幂等检查）；
- `tasks` 里的任务**严格按顺序执行**，前一个失败默认终止整个 play；
- 一个 play 可以含多个 task，一个 task 对应一个模块调用；
- 缩进必须是空格，**不能用 Tab**。

### 3.2 幂等性

同一个 playbook 跑两次，结果应该和跑一次一样——**已满足的跳过，未满足的才执行**：

```bash
ansible-playbook site.yml          # 第一次：装 httpd、起服务
ansible-playbook site.yml          # 第二次：httpd 已装、服务已起，显示 ok / skipping
```

> 幂等性是 Ansible 的核心设计目标。写 playbook 时要考虑「再跑一遍会怎样」——
> `command` / `shell` 模块**不幂等**（每次都执行），能用对应模块（`dnf`、`service`、`copy`）就别用裸命令。

### 3.3 执行与验证

```bash
ansible-playbook site.yml                     # 正常执行
ansible-playbook site.yml --syntax-check      # 只检查语法，不执行
ansible-playbook site.yml -C                  # 检查模式（dry-run），看会改什么但不真改
ansible-playbook site.yml -C -D               # -D 同时显示改动差异（diff）
ansible-playbook site.yml --start-at-task="安装 httpd"   # 从指定任务开始跑
ansible-playbook site.yml --step              # 每步询问是否执行（排障用）
ansible-playbook site.yml -v / -vv / -vvv     # 详细程度递增，-vvv 连 SSH 过程都打出来
ansible-playbook site.yml --list-tasks        # 只列出会执行哪些任务
ansible-playbook site.yml --list-hosts        # 只列出会影响哪些主机
```

> 验证顺序：`--syntax-check` → `-C -D`（干跑看差异）→ 正式跑 → `-v` 看输出。
> `--check` 模式要求模块支持；不支持 check 的模块（如 `command`）会被跳过。

### 3.4 提权 become

| 写法 | 作用范围 |
| --- | --- |
| `ansible.cfg` 的 `[privilege_escalation]` | 全局默认 |
| play 级 `become: yes` | 整个 play |
| task 级 `become: yes` / `no` | 单个任务（最精细） |
| 命令行 `--become` / `-b` | 本次执行 |

```yaml
- name: 装包需要 root
  ansible.builtin.dnf:
    name: nginx
    state: present
  become: yes                # 只这个任务提权

- name: 读文件不需要 root
  ansible.builtin.command: cat /etc/os-release
  become: no
```

> 提权相关命令：`--ask-become-pass`（交互输入 sudo 密码）、`become_method=su`（用 su 而不是 sudo）、
> `become_user=www-data`（提到指定用户而不是 root）。

---

## 四、变量与事实

### 4.1 变量定义位置与优先级

Ansible 变量有 **22 个优先级层级**（官方文档），考试和实际用只需记住核心几层（**越高越优先**）：

| 层级 | 位置 | 例子 |
| --- | --- | --- |
| 高 | 命令行 `-e` | `ansible-playbook site.yml -e "http_port=9090"` |
|  | playbook `vars:` / `vars_files:` | play 里的 vars 块 |
|  | role `vars/main.yml` | role 内部变量 |
|  | inventory 组变量（`[web:vars]`） | 组级别 |
|  | inventory 主机变量（内联） | 单台级别 |
| 低 | role `defaults/main.yml` | 默认值，最低优先级 |

```yaml
# playbook 内定义
- hosts: web
  vars:
    http_port: 80
  vars_files:
    - vars/web-vars.yml
```

```bash
# 命令行覆盖一切（优先级最高）
ansible-playbook site.yml -e "http_port=9090"
```

**记忆口诀**：`命令行 > playbook > inventory > role defaults`。

> inventory 里定义变量应该用 `group_vars/` 和 `host_vars/` 目录，而不是直接写在 hosts 文件里——更清晰、可复用（见 §8.1）。

### 4.2 register：捕获任务结果

把任务的输出存进变量，后续任务判断使用：

```yaml
- name: 检查网页是否可访问
  hosts: web
  tasks:
    - name: 请求测试页
      ansible.builtin.uri:
        url: "http://{{ inventory_hostname }}/"
        return_content: yes          # 把响应体也带回来
        status_code: 200             # 期望的状态码，不是这个就失败
      register: web_check            # 结果存进 web_check

    - name: 打印结果
      ansible.builtin.debug:
        var: web_check

    - name: 失败才执行后续
      ansible.builtin.debug:
        msg: "页面异常"
      when: web_check.failed         # 用 register 的结果做条件
```

> `register` 先执行、后注册变量，所以在同一 task 里不能引用自己——要在下一个 task 用。

### 4.3 facts：事实变量

受管节点的系统信息（IP、内存、操作系统、内核…），Ansible 自动收集，**不需要定义**：

```bash
ansible web -m setup                          # 全量（信息量大）
ansible web -m setup -a "filter=ansible_distribution*"    # 只看发行版
ansible web -m setup -a "filter=ansible_memtotal_mb"      # 看总内存
```

playbook 里直接用：

```yaml
- name: 按发行版选择包名
  ansible.builtin.dnf:
    name: "{{ 'httpd' if ansible_distribution == 'RedHat' else 'apache2' }}"
    state: present

- name: 输出主机信息
  ansible.builtin.debug:
    msg: "{{ inventory_hostname }} 内存 {{ ansible_memtotal_mb }}MB，IP {{ ansible_default_ipv4.address }}"
```

常用 facts：

| 变量 | 含义 |
| --- | --- |
| `ansible_hostname` | 短主机名 |
| `ansible_fqdn` | 完整域名 |
| `ansible_distribution` | 发行版（RedHat / CentOS / Ubuntu） |
| `ansible_distribution_version` | 版本号 |
| `ansible_memtotal_mb` | 总内存（MB） |
| `ansible_default_ipv4.address` | 默认 IPv4 地址 |
| `ansible_devices` | 磁盘设备列表 |
| `ansible_mounts` | 挂载点列表 |

> 用不到 facts 时在 play 级写 `gather_facts: no` 能显著加速（省了 SSH 收集的时间）。
> `filter` 参数支持 shell 通配符：`filter=ansible_eth*`、`filter=*mount*`。

### 4.4 magic variables（魔术变量）

Ansible 内置、不用定义、反映当前执行状态：

| 变量 | 含义 |
| --- | --- |
| `inventory_hostname` | 当前主机在 inventory 里的名字 |
| `groups` | 所有组及其成员（`groups['web']` 拿到 web 组所有主机） |
| `group_names` | 当前主机属于哪些组 |
| `hostvars` | 其他主机的变量（`hostvars['node1']['ansible_hostname']`） |
| `play_hosts` | 当前 play 涉及的主机列表 |

### 4.5 Ansible Vault：加密敏感数据

密码、密钥这类不能直接写进 playbook 明文，用 Vault 加密：

```bash
# 创建加密文件
ansible-vault create secret.yml
# 编辑加密文件
ansible-vault edit secret.yml
# 加密已有文件
ansible-vault encrypt vars/passwords.yml
# 解密
ansible-vault decrypt vars/passwords.yml
# 只查看不修改
ansible-vault view secret.yml
# 改密码
ansible-vault rekey secret.yml
```

playbook 里引用加密文件，执行时要提供密码：

```yaml
- hosts: web
  vars_files:
    - vars/passwords.yml          # 已加密的文件
  tasks:
    - name: 用密码创建用户
      ansible.builtin.user:
        name: "{{ db_user }}"
        password: "{{ db_password | password_hash('sha512') }}"
```

```bash
# 执行时提供 Vault 密码的三种方式
ansible-playbook site.yml --ask-vault-pass                  # 交互输入
ansible-playbook site.yml --vault-password-file ~/.vault_pass   # 密码文件（注意权限 600）
ansible-playbook site.yml --vault-id prod@~/.vault_pass     # 多 Vault 场景指定 ID
```

> 密码文件权限必须 `600`（`chmod 600 ~/.vault_pass`），且**不要提交进 Git**（加进 `.gitignore`）。
> 只加密部分变量也行：`!vault` 标签加密单个值，不用整文件加密。

---

## 五、控制流

### 5.1 循环 loop

```yaml
# 列表循环
- name: 启动多个服务
  ansible.builtin.service:
    name: "{{ item }}"
    state: started
    enabled: yes
  loop:
    - firewalld
    - httpd

# 字典 + 列表循环（每个 item 是字典，可带多个字段）
- name: 创建多个用户
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    state: present
  loop:
    - name: apple
      groups: wheel
    - name: orange
      groups: root

# 循环 + register（收集每次的结果）
- name: 批量检查服务状态
  ansible.builtin.service:
    name: "{{ item }}"
    state: started
  loop:
    - httpd
    - sshd
    - chronyd
  register: service_results

- name: 打印每个结果
  ansible.builtin.debug:
    msg: "{{ item.item }}: {{ item.status }}"
  loop: "{{ service_results.results }}"
```

> `loop` 是现行写法，旧的 `with_items` / `with_list` 还能用但不推荐（Ansible 2.5+ 起建议用 `loop`）。
> 循环里变量名固定是 `item`（可用 `loop_control.loop_var` 改名）。

### 5.2 条件 when

```yaml
- name: 只在 RHEL 上装 httpd
  ansible.builtin.dnf:
    name: httpd
    state: present
  when: ansible_distribution == "RedHat"

- name: 只在 web 组执行
  ansible.builtin.debug:
    msg: "这是 web 节点"
  when: "'web' in group_names"

- name: 内存大于 2G 才装
  ansible.builtin.dnf:
    name: bigdata-tool
    state: present
  when: ansible_memtotal_mb > 2048

- name: 多条件（与）
  ansible.builtin.service:
    name: nginx
    state: started
  when:
    - ansible_distribution == "RedHat"
    - ansible_memtotal_mb > 1024
    - "'web' in group_names"

- name: 结合 register
  ansible.builtin.debug:
    msg: "上次执行有改动"
  when: my_task.changed
```

> `when` 的条件**不需要加 `{{ }}`**（它本身就是 Jinja2 表达式上下文）；写成 `when: "{{ var }}"` 是旧习惯，现在直接 `when: var`。

### 5.3 handlers：事件触发

handlers 是「**任务有改动时才执行**」的特殊任务，典型用途：改了配置才重启服务，没改就不重启（幂等的关键一环）。

```yaml
- hosts: web
  tasks:
    - name: 部署配置文件
      ansible.builtin.template:
        src: httpd.conf.j2
        dest: /etc/httpd/conf/httpd.conf
      notify: 重启 httpd           # 文件变了就触发（notify 按名字匹配 handler）

    - name: 同时改端口（同一个 notify 不重复触发）
      ansible.builtin.lineinfile:
        path: /etc/httpd/conf/httpd.conf
        regexp: "^Listen"
        line: "Listen {{ http_port }}"
      notify: 重启 httpd

  handlers:                        # handlers 与 tasks 平级
    - name: 重启 httpd
      ansible.builtin.service:
        name: httpd
        state: restarted
```

要点：

- handler 在**所有 tasks 执行完后**才运行（即使有多个 notify 也只跑一次）；
- notify 的名字必须与 handler 的 `name` **完全一致**；
- handler 只在任务报告 `changed` 时触发，没变（`ok`）就不跑——这是幂等的体现；
- 要立即触发（不等全部跑完）：`- meta: flush_handlers`。

### 5.4 错误处理

| 关键字 | 作用 |
| --- | --- |
| `ignore_errors: yes` | 任务失败也继续（**不推荐大面积用**，会掩盖真问题） |
| `failed_when` | 自定义什么算失败（比如某命令输出含特定文本才算失败） |
| `changed_when` | 自定义什么算「有改动」（`command`/`shell` 常用，配合幂等） |
| `block / rescue / always` | 结构化错误处理（类似 try / catch / finally） |

```yaml
- name: 结构化错误处理
  block:
    - name: 主任务
      ansible.builtin.command: /usr/local/bin/backup.sh
  rescue:
    - name: 失败了发告警
      ansible.builtin.uri:
        url: https://hooks.example.com/alert
        method: POST
        body: "backup failed on {{ inventory_hostname }}"
        body_format: json
  always:
    - name: 无论如何清理临时文件
      ansible.builtin.file:
        path: /tmp/backup.lock
        state: absent
```

```yaml
# failed_when / changed_when 示例
- name: 检查磁盘使用率
  ansible.builtin.command: df -h /
  register: disk
  changed_when: false                     # 查询类命令永远不该标 changed
  failed_when: "'100%' in disk.stdout"    # 满了才算失败

# force_handlers：即使 play 中途失败，已 notify 的 handler 也要执行
- hosts: web
  force_handlers: yes
  tasks: ...
```

> `ignore_errors` 是最后的手段——能用 `failed_when` 精确判断就别用 ignore。
> `block/rescue/always` 是官方推荐的错误处理方式（Ansible 2.x 起支持）。

---

## 六、常用模块（按系统管理任务分类）

考纲明确要求掌握以下 10 类模块。**同一任务优先用专用模块，而不是 `command` / `shell`**。

### 6.1 软件包与仓库

```yaml
- name: 安装软件包
  ansible.builtin.dnf:               # RHEL 9/10；旧版本用 yum
    name: httpd
    state: present                   # present / absent / latest

- name: 装多个包
  ansible.builtin.dnf:
    name:
      - httpd
      - mariadb-server
      - php
    state: present

- name: 添加仓库
  ansible.builtin.dnf:
    name: "https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm"
    state: present
    disable_gpg_check: yes

- name: 配置仓库（写 .repo 文件）
  ansible.builtin.yum_repository:
    name: epel
    description: EPEL
    baseurl: https://download.fedoraproject.org/pub/epel/10/Everything/x86_64/
    enabled: yes
    gpgcheck: yes
```

### 6.2 服务

```yaml
- name: 启动并开机自启
  ansible.builtin.service:
    name: httpd
    state: started                   # started / stopped / restarted / reloaded
    enabled: yes

- name: systemd 专属操作（daemon-reload 等）
  ansible.builtin.systemd:
    name: httpd
    daemon_reload: yes
    enabled: yes
    state: started
```

> `service` 模块兼容 SysV 与 systemd；需要 `daemon_reload`、`mask` 时用 `systemd` 模块。

### 6.3 防火墙

```yaml
- name: 放行 http 服务
  ansible.posix.firewalld:
    service: http
    permanent: yes
    state: enabled
    immediate: yes                   # 等价于 --permanent + --reload

- name: 放行端口
  ansible.posix.firewalld:
    port: 8080/tcp
    permanent: yes
    state: enabled
```

### 6.4 文件与内容

```yaml
- name: 创建目录
  ansible.builtin.file:
    path: /data/www
    state: directory                 # directory / touch / absent / link
    owner: nginx
    group: nginx
    mode: '0755'

- name: 拷贝文件（不改内容）
  ansible.builtin.copy:
    src: ./files/index.html
    dest: /var/www/html/index.html
    owner: root
    mode: '0644'
    backup: yes                      # 覆盖前备份原文件

- name: 直接写内容
  ansible.builtin.copy:
    content: "server_name {{ ansible_fqdn }};\n"
    dest: /etc/nginx/conf.d/server.conf

- name: 修改文件某一行
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: "^PermitRootLogin"
    line: "PermitRootLogin no"
    backup: yes

- name: 确保某行不存在
  ansible.builtin.lineinfile:
    path: /etc/fstab
    regexp: "^/dev/sdb1"
    state: absent

- name: 下载文件
  ansible.builtin.get_url:
    url: https://example.com/app.tar.gz
    dest: /tmp/app.tar.gz
    mode: '0644'
```

### 6.5 存储

```yaml
- name: 创建文件系统
  ansible.builtin.filesystem:
    fstype: xfs
    dev: /dev/sdb1

- name: 挂载
  ansible.posix.mount:
    path: /data
    src: /dev/sdb1
    fstype: xfs
    state: mounted                   # mounted / unmounted / present / absent
    opts: defaults

- name: LVM 逻辑卷
  ansible.builtin.lvol:
    vg: vgdata
    lv: lvapp
    size: 2g
```

### 6.6 用户与组

```yaml
- name: 创建用户
  ansible.builtin.user:
    name: king
    groups: wheel                    # 附加组
    shell: /bin/bash
    create_home: yes
    state: present

- name: 设置密码（先哈希）
  ansible.builtin.user:
    name: king
    password: "{{ 'mypass' | password_hash('sha512') }}"

- name: 创建组
  ansible.builtin.group:
    name: developers
    gid: 2000
    state: present
```

### 6.7 计划任务

```yaml
- name: 定时备份
  ansible.builtin.cron:
    name: "daily backup"
    minute: "30"
    hour: "2"
    job: "/usr/local/bin/backup.sh"
    user: root
    state: present
```

### 6.8 安全（SELinux）

```yaml
- name: 设置 SELinux 模式
  ansible.posix.selinux:
    policy: targeted
    state: enforcing                 # enforcing / permissive / disabled

- name: 设置布尔值
  ansible.posix.seboolean:
    name: httpd_can_network_connect
    state: yes
    persistent: yes

- name: 放行非标准端口
  ansible.posix.seport:
    ports: "8080"
    proto: tcp
    setype: http_port_t
    state: present

- name: 修复文件上下文
  ansible.builtin.command: restorecon -Rv /var/www/html
  changed_when: false
```

### 6.9 归档

```yaml
- name: 打包目录
  ansible.builtin.archive:
    path: /var/www/html
    dest: /backup/www.tar.gz
    format: gz

- name: 解压到目标
  ansible.builtin.unarchive:
    src: /backup/www.tar.gz
    dest: /restore/
    remote_src: yes                  # 压缩包在受管节点上（默认在控制节点）
```

### 6.10 模板 template

把 Jinja2 模板渲染成受管节点上的实际文件——**模板写在控制节点，变量在渲染时替换**：

```yaml
- name: 用模板生成配置
  ansible.builtin.template:
    src: templates/nginx.conf.j2     # 控制节点上的模板
    dest: /etc/nginx/nginx.conf      # 受管节点上的目标
    owner: root
    mode: '0644'
    validate: nginx -t -c %s         # 生成后先验证语法，不通过就不写入
    backup: yes
  notify: 重启 nginx
```

> `copy` vs `template`：`copy` 原样拷贝；`template` 会先把 `{{ }}` / `{% %}` 渲染成实际值再写入。
> `validate` 是考试加分点——生成配置先跑 `nginx -t` / `httpd -t` 验证，避免把坏配置写上去导致服务起不来。

#### Jinja2 模板语法

模板文件以 `.j2` 结尾，三种界定符：

```jinja2
{# 这是注释 #}

{# 变量替换 #}
Listen {{ http_port }}
ServerName {{ ansible_fqdn }}

{# 逻辑控制：条件 #}
{% if enable_ssl %}
Listen 443 ssl
{% endif %}

{# 逻辑控制：循环 #}
{% for host in groups['web'] %}
server {{ hostvars[host]['ansible_default_ipv4']['address'] }}:80;
{% endfor %}
```

> 三种界定符：`{{ }}` 输出变量、`{% %}` 逻辑控制、`{# #}` 注释。
> 模板里的变量就是 playbook 里的变量——facts、register、inventory 变量都能直接用。

---

## 七、Roles 与 Collections

### 7.1 role 目录结构

一个 role 是一套**可复用的完整单元**（任务 + 变量 + 文件 + 模板 + handler）：

<svg viewBox="0 0 680 395" width="100%" role="img" xmlns="http://www.w3.org/2000/svg">
  <title>Ansible Role 标准目录结构</title>
  <desc>展示一个标准 ansible role 的目录组成及各部分作用</desc>
  <text x="40" y="28" font-family="var(--font-sans)" font-size="15" font-weight="500" fill="var(--color-text-primary)">Ansible Role 标准目录结构</text>
  <rect x="40" y="45" width="600" height="330" rx="20" fill="var(--color-background-secondary)" stroke="var(--color-border-tertiary)" stroke-width="0.5"/>
  <text x="64" y="72" style="font-family:var(--font-mono);font-size:14px;font-weight:500" fill="var(--color-text-primary)">myrole/</text>
  <text x="152" y="72" style="font-size:12px" fill="var(--color-text-secondary)">role 根目录，一个 role = 一套可复用单元</text>
  <path d="M56 86 L624 86" stroke="var(--color-border-tertiary)" stroke-width="0.5" fill="none"/>
  <rect x="54" y="100" width="3" height="16" rx="1.5" fill="#378ADD"/>
  <text x="72" y="108" style="font-family:var(--font-mono);font-size:12px" fill="var(--color-text-tertiary)">&#9508;&#9472; </text>
  <text x="96" y="108" style="font-family:var(--font-mono);font-size:13px;font-weight:500" fill="#185FA5">tasks/main.yml</text>
  <text x="300" y="108" style="font-size:12px" fill="var(--color-text-secondary)">主任务入口（唯一必需，缺失则 role 空转）</text>
  <text x="72" y="138" style="font-family:var(--font-mono);font-size:12px" fill="var(--color-text-tertiary)">&#9508;&#9472; </text>
  <text x="96" y="138" style="font-family:var(--font-mono);font-size:13px" fill="var(--color-text-primary)">handlers/main.yml</text>
  <text x="300" y="138" style="font-size:12px" fill="var(--color-text-secondary)">notify 触发动作（重启服务等）</text>
  <text x="72" y="168" style="font-family:var(--font-mono);font-size:12px" fill="var(--color-text-tertiary)">&#9508;&#9472; </text>
  <text x="96" y="168" style="font-family:var(--font-mono);font-size:13px" fill="var(--color-text-primary)">defaults/main.yml</text>
  <text x="300" y="168" style="font-size:12px" fill="var(--color-text-secondary)">默认变量（低优先级，可被外部覆盖）</text>
  <text x="72" y="198" style="font-family:var(--font-mono);font-size:12px" fill="var(--color-text-tertiary)">&#9508;&#9472; </text>
  <text x="96" y="198" style="font-family:var(--font-mono);font-size:13px" fill="var(--color-text-primary)">vars/main.yml</text>
  <text x="300" y="198" style="font-size:12px" fill="var(--color-text-secondary)">内部变量（高优先级，不建议被覆盖）</text>
  <text x="72" y="228" style="font-family:var(--font-mono);font-size:12px" fill="var(--color-text-tertiary)">&#9508;&#9472; </text>
  <text x="96" y="228" style="font-family:var(--font-mono);font-size:13px" fill="var(--color-text-primary)">files/</text>
  <text x="300" y="228" style="font-size:12px" fill="var(--color-text-secondary)">静态文件，copy 模块直接引用</text>
  <text x="72" y="258" style="font-family:var(--font-mono);font-size:12px" fill="var(--color-text-tertiary)">&#9508;&#9472; </text>
  <text x="96" y="258" style="font-family:var(--font-mono);font-size:13px" fill="var(--color-text-primary)">templates/</text>
  <text x="300" y="258" style="font-size:12px" fill="var(--color-text-secondary)">Jinja2 模板（*.j2），template 模块用</text>
  <text x="72" y="288" style="font-family:var(--font-mono);font-size:12px" fill="var(--color-text-tertiary)">&#9508;&#9472; </text>
  <text x="96" y="288" style="font-family:var(--font-mono);font-size:13px" fill="var(--color-text-primary)">meta/main.yml</text>
  <text x="300" y="288" style="font-size:12px" fill="var(--color-text-secondary)">元数据 + role 依赖声明</text>
  <text x="72" y="318" style="font-family:var(--font-mono);font-size:12px" fill="var(--color-text-tertiary)">&#9492;&#9472; </text>
  <text x="96" y="318" style="font-family:var(--font-mono);font-size:13px" fill="var(--color-text-primary)">tests/</text>
  <text x="300" y="318" style="font-size:12px" fill="var(--color-text-secondary)">测试目录（可选，考试用不到）</text>
  <path d="M56 340 L624 340" stroke="var(--color-border-tertiary)" stroke-width="0.5" fill="none"/>
  <text x="64" y="360" style="font-size:12px" fill="var(--color-text-secondary)">defaults 可被 group_vars / -e 覆盖；vars 是 role 私有高优先级变量</text>
</svg>

> **defaults 与 vars 的区别是常考点**：`defaults/main.yml` 里的变量优先级**最低**，是给人覆盖用的；
> `vars/main.yml` 优先级很高（仅次于 `-e`），是 role 内部强依赖的值。**想把默认值开放给外部覆盖，放 defaults；不想被覆盖，放 vars。**

### 7.2 创建与引用 role

```bash
# 生成标准目录骨架
ansible-galaxy init roles/nginx
tree roles/nginx/                # 看一眼生成的结构

# 手动建也行（目录 + main.yml）
mkdir -p roles/nginx/{tasks,handlers,defaults,vars,files,templates,meta}
touch roles/nginx/tasks/main.yml
```

playbook 里引用 role 的三种方式：

```yaml
# 方式一：play 级 roles 列表（最常用）
- hosts: web
  roles:
    - nginx
    - firewall

# 方式二：import_role 静态导入（编译时展开）
tasks:
  - name: 静态导入 nginx
    ansible.builtin.import_role:
      name: nginx

# 方式三：include_role 动态加载（支持 when / loop）
tasks:
  - name: 动态加载 nginx
    ansible.builtin.include_role:
      name: nginx
    when: ansible_hostname == "node1"
```

| 方式 | 时机 | 支持 when / loop |
| --- | --- | --- |
| `roles:` | play 编译时 | 不支持（在 play 级） |
| `import_role` | 编译时展开 | 不支持 |
| `include_role` | 运行时才解析 | **支持** |

> 三种方式的执行顺序：先跑 `roles:` 列表（按顺序），再跑 tasks 里的 `import_role` / `include_role`（按位置）。
> 考试里最常用的是 `roles:` 列表 + `include_role` 带条件。

### 7.3 Ansible Galaxy

从社区下载现成 role：

```bash
ansible-galaxy search nginx                 # 搜索
ansible-galaxy info geerlingguy.nginx       # 看详情
ansible-galaxy install geerlingguy.nginx    # 安装到 ~/.ansible/roles/
ansible-galaxy list                         # 看已安装的
ansible-galaxy remove geerlingguy.nginx     # 删除
```

用 requirements 文件批量装（推荐，项目可复现）：

```yaml
# requirements.yml
roles:
  - name: geerlingguy.nginx
    version: "3.1.4"
  - name: geerlingguy.mysql

collections:
  - name: community.general
  - name: ansible.posix
```

```bash
ansible-galaxy install -r requirements.yml             # 装 roles
ansible-galaxy collection install -r requirements.yml  # 装 collections
```

> Galaxy 上的 role 质量参差——生产环境优先用 Red Hat 官方的 **RHEL System Roles**（见 7.5），社区 role 看下载量与最近更新时间。

### 7.4 Collections

Collections 是 Ansible 2.9+ 的内容打包格式（模块 + role + 插件 + 文档一体），比单个 role 粒度更大：

```bash
ansible-galaxy collection install community.general      # 装 collection
ansible-galaxy collection list                           # 看已装的
ansible-galaxy collection install -r requirements.yml    # 批量装
```

playbook 里引用 collection 中的模块要写**全限定名（FQCN）**：

```yaml
- name: 用 FQCN 写模块（推荐）
  ansible.builtin.copy:                  # 内置 collection
    src: a.txt
    dest: /tmp/

- name: 社区 collection 的模块
  community.general.archive:
    path: /var/www
    dest: /backup/www.tar.gz

- name: posix collection 的防火墙模块
  ansible.posix.firewalld:
    service: http
    permanent: yes
    state: enabled
```

> 考试里所有模块都建议写 FQCN（如 `ansible.builtin.copy`），一来避免同名模块冲突，二来 `ansible-doc` 查文档时也要用全名。

### 7.5 RHEL System Roles

Red Hat 官方维护的一组 role，**开箱即用、覆盖常见系统管理任务**，考试和生产都推荐优先用：

```bash
dnf install -y rhel-system-roles        # 安装
ls /usr/share/ansible/roles/             # 看装了哪些（rhel-system-roles.*）
ls /usr/share/doc/rhel-system-roles/     # 每个 role 都有 README + 示例 playbook
```

常用 System Roles：

| role | 用途 |
| --- | --- |
| `rhel-system-roles.network` | 网络配置（等价于 nmcli 自动化） |
| `rhel-system-roles.storage` | 磁盘 / LVM / 文件系统 |
| `rhel-system-roles.selinux` | SELinux 上下文与布尔值 |
| `rhel-system-roles.firewall` | firewalld 规则 |
| `rhel-system-roles.timesync` | NTP / chronyd 时间同步 |
| `rhel-system-roles.kdump` | kdump 崩溃转储 |
| `rhel-system-roles.logging` | rsyslog 配置 |
| `rhel-system-roles.podman` | 容器管理 |

使用示例：

```yaml
- hosts: web
  vars:
    network_connections:
      - name: eth0
        type: ethernet
        ip:
          address:
            - 192.168.128.20/24
          gateway4: 192.168.128.1
          dns:
            - 223.5.5.5
  roles:
    - rhel-system-roles.network
```

> 每个 System Role 的用法在 `/usr/share/doc/rhel-system-roles/<role>/README.md` 里——**先读 README 再写变量**，这是最省事的方式。

---

## 八、项目组织

### 8.1 推荐目录结构

```text
myproject/
├── ansible.cfg                  # 项目级配置
├── inventory/
│   ├── hosts.ini                # 清单
│   ├── group_vars/              # 组变量（按组名分文件）
│   │   ├── web.yml
│   │   └── db.yml
│   └── host_vars/               # 主机变量（按主机名分文件）
│       └── node1.example.com.yml
├── playbooks/
│   ├── site.yml                 # 主入口
│   └── web.yml
├── roles/
│   ├── nginx/
│   └── mysql/
├── files/                       # 静态文件
├── templates/                   # Jinja2 模板
└── vault/
    └── passwords.yml            # ansible-vault 加密
```

> 组变量与主机变量**不要写进 hosts 文件**，放 `group_vars/` 和 `host_vars/`——文件分开管理，改起来不用动 inventory。

### 8.2 import vs include

两者都是「把别的文件引进来」，区别在于**何时解析**：

| 特性 | `import_*`（静态） | `include_*`（动态） |
| --- | --- | --- |
| 解析时机 | playbook 编译时就展开 | 运行到这一步才读 |
| 支持 `when` / `loop` | 不支持 | 支持 |
| 支持变量化的文件名 | 不支持 | 支持（`include_tasks: "{{ os }}.yml"`） |
| `--list-tasks` 能看到 | 能看到 | 看不到（运行时才知道） |

```yaml
# import：预处理，按导入顺序展开
tasks:
  - name: 导入任务文件
    ansible.builtin.import_tasks: tasks/server.yml
    vars:
      package: httpd
      service: httpd

# include：运行时处理
tasks:
  - name: 按发行版动态包含
    ansible.builtin.include_tasks: "tasks/{{ ansible_distribution | lower }}.yml"

- name: 带条件的包含
  ansible.builtin.include_tasks: tasks/ssl.yml
  when: enable_ssl | bool
```

> 经验规则：**静态知道的用 import，依赖运行时判断的用 include**。
> 写 `--list-tasks` 时看不到 include 的内容，需要完整任务清单的话用 import。

### 8.3 并行与 forks

Ansible 默认一次只连 5 台主机，大集群要调并发：

```ini
# ansible.cfg
[defaults]
forks = 20                    # 同时连 20 台
```

```bash
ansible-playbook site.yml -f 50        # 命令行覆盖
```

```yaml
# playbook 级控制滚动更新
- hosts: web
  serial: 5                   # 一批 5 台，跑完再下一批（滚动更新场景）
  tasks: ...
```

> `forks` 控制**同时**连几台；`serial` 控制**分批**执行（每批 N 台）。两者独立，可以一起用。

---

## 九、故障处理

### 9.1 调试

```bash
ansible-playbook site.yml -v          # 一级详细：显示任务结果
ansible-playbook site.yml -vv         # 二级：显示更多内部信息
ansible-playbook site.yml -vvv        # 三级：连 SSH 交互过程都打出来（**终极排障**）
ansible-playbook site.yml -vvvv       # 四级：极少用，太啰嗦

# 查看日志（ansible-navigator 环境下）
cat ansible-navigator.log
cat artifact.json                     # 每个 play 的执行详情
```

### 9.2 常见错误

| 现象 | 根因 | 处理 |
| --- | --- | --- |
| `UNREACHABLE!` | SSH 不通 / 密钥不对 / 主机不存在 | `ansible all -m ping` 先测；检查 inventory 主机名 |
| `FAILED! ... Missing sudo password` | 没配 NOPASSWD 也没传密码 | `--ask-become-pass` 或 sudoers 加 NOPASSWD |
| `MODULE FAILURE` / 找不到 Python | 受管节点没装 python3 | `dnf install python3` 或指定 `ansible_python_interpreter` |
| 任务报 `changed` 但什么都没改 | 用了 `command` / `shell`（不幂等） | 换对应专用模块，或加 `changed_when: false` |
| handler 没触发 | notify 的名字与 handler 名不一致 | 两者必须**逐字一致**（包括大小写） |
| 变量没生效 | 优先级被更高层覆盖 | `ansible-inventory --list` + `-e` 排查，记住「命令行 > playbook > inventory」 |
| YAML 语法错误 | Tab 缩进 / 冒号后没空格 | `--syntax-check` 先跑一遍；编辑器设成空格缩进 |

### 9.3 文档查询

考纲明确要求「会用官方文档查模块用法」：

```bash
ansible-doc -l                          # 列出所有模块
ansible-doc -l | grep firewalld         # 找某类模块
ansible-doc ansible.builtin.copy        # 看模块的完整文档（参数、示例、注意事项）
ansible-doc -s ansible.builtin.copy     # 只看参数片段（snippet，适合快速查）
ansible-doc -t keyword -l               # 看所有关键字（when、loop 等）
man ansible-playbook                    # 传统 man
```

> `ansible-doc <FQCN>` 是考场排障的第一工具——模块的参数名、默认值、注意事项都在里面，比搜索引擎快。

---

## 十、考试要点清单

考前过一遍，确保每个都能不看文档写出来：

- [ ] 安装 ansible / ansible-core
- [ ] 写 ansible.cfg（inventory、remote_user、become、forks）
- [ ] 写静态 inventory（分组、`[group:vars]`、嵌套组）
- [ ] `ssh-copy-id` 分发密钥 + sudoers 配 NOPASSWD
- [ ] `ansible all -m ping` 验证连通性
- [ ] ad-hoc 命令（`-m` 指定模块、`-a` 传参、`-b` 提权）
- [ ] playbook 骨架（`---`、play、tasks、缩进）
- [ ] 幂等性验证（同一 playbook 跑两遍）
- [ ] `register` + `when` 条件执行
- [ ] `loop` 循环（列表、字典）
- [ ] handler + notify（名字逐字一致）
- [ ] `block / rescue / always`
- [ ] `failed_when` / `changed_when`
- [ ] 常用模块：dnf、service/systemd、copy、file、lineinfile、template、get_url、user、group、cron、mount、filesystem、lvol、firewalld、selinux、seboolean、seport
- [ ] Jinja2 模板（`{{ }}` 变量、`{% if %}`、`{% for %}`）
- [ ] 变量优先级（命令行 > playbook > inventory > defaults）
- [ ] facts（`ansible_distribution`、`ansible_memtotal_mb`、`ansible_default_ipv4`）
- [ ] magic variables（`inventory_hostname`、`groups`、`hostvars`）
- [ ] Ansible Vault（create / edit / encrypt / view / `--ask-vault-pass`）
- [ ] role 目录结构 + defaults vs vars 区别
- [ ] `roles:` / `import_role` / `include_role` 三种引用
- [ ] `ansible-galaxy install -r requirements.yml`
- [ ] `ansible-galaxy collection install`
- [ ] RHEL System Roles（`rhel-system-roles.*`，先读 README）
- [ ] `group_vars/` / `host_vars/` 目录分离
- [ ] import vs include 区别
- [ ] `forks` 并发 + `serial` 滚动
- [ ] `-vvv` 调试 + `ansible-doc` 查模块
- [ ] **`--syntax-check` → `-C -D` → 正式跑 → 验证重启后仍生效**

> 所有配置**重启后必须自动生效**（服务 `enabled: yes`、挂载写 fstab、防火墙加 `permanent`）——这是考试评分的第一道门槛。
