# Git 与 GitHub 协同开发笔记

## 目录
- [1. Git 基础配置与指令](#1-git-基础配置与指令)
  - [1.1 SSH 密钥配置](#11-ssh-密钥配置)
  - [1.2 仓库基本操作](#12-仓库基本操作)
  - [1.3 分支管理](#13-分支管理)
- [2. GitHub 多设备多账号协同开发指南](#2-github-多设备多账号协同开发指南)
  - [2.1 场景描述](#21-场景描述)
  - [2.2 权限配置 (Collaborator)](#22-权限配置-collaborator)
  - [2.3 多账号 SSH 配置 (核心方案)](#23-多账号-ssh-配置-核心方案)
  - [2.4 协同工作流 (Workflow)](#24-协同工作流-workflow)
  - [2.5 常见问题](#25-常见问题)
- [3. SSH 密钥维护清单](#3-ssh-密钥维护清单)
  - [3.1 检查现有密钥](#31-检查现有密钥)
  - [3.2 新建密钥（Ed25519）](#32-新建密钥ed25519)
  - [3.3 清理旧密钥](#33-清理旧密钥)
  - [3.4 Linux 间免密登录实战：A ↔ B](#34-linux-间免密登录实战a--b)
    - [3.4.1 环境概览](#341-环境概览)
    - [3.4.2 本机 IP 查询与 `ping` 详解](#342-本机-ip-查询与-ping-详解)
    - [3.4.3 前置准备](#343-前置准备)
    - [3.4.4 正式配置流程](#344-正式配置流程)
    - [3.4.5 调试过程与故障清单](#345-调试过程与故障清单)
    - [3.4.6 最终验证与日常操作](#346-最终验证与日常操作)
    - [3.4.7 后续可选增强](#347-后续可选增强)
- [4. 常见推送场景脚本](#4-常见推送场景脚本)
  - [4.1 全新仓库从 0 到 1](#41-全新仓库从-0-到-1)
  - [4.2 推送到已存在的远程分支](#42-推送到已存在的远程分支)
- [5. 分支与远程管理进阶](#5-分支与远程管理进阶)
  - [5.1 本地分支改名与删除](#51-本地分支改名与删除)
  - [5.2 远程别名管理](#52-远程别名管理)
  - [5.3 删除远程分支并解除跟踪](#53-删除远程分支并解除跟踪)
  - [5.4 建立与调整上游关系](#54-建立与调整上游关系)
  - [5.5 状态与引用查询](#55-状态与引用查询)
- [6. 其他速查](#6-其他速查)

## 1. Git 基础配置与指令

### 1.1 SSH 密钥配置
**作用**：实现本地与 GitHub 的免密安全通信。

1.  **生成密钥 (Ed25519 算法)**:
    ```bash
    ssh-keygen -t ed25519 -C "your_email@example.com"
    ```
2.  **添加 SSH Agent**:
    ```bash
    eval "$(ssh-agent -s)"
    ssh-add ~/.ssh/id_ed25519
    ```
3.  **部署到 GitHub**:
    - 复制公钥内容: `cat ~/.ssh/id_ed25519.pub`
    - GitHub -> Settings -> SSH and GPG keys -> New SSH key。
4.  **测试连接**:
    ```bash
    ssh -T git@github.com
    # 成功回显: Hi username! You've successfully authenticated...
    ```

### 1.2 仓库基本操作
- **初始化**: `git init`
- **关联远程**: `git remote add origin git@github.com:User/Repo.git`
- **查看状态**: `git status`
- **添加文件**: `git add .`
- **提交更改**: `git commit -m "message"`
- **推送代码**: `git push -u origin main` (首次), `git push` (后续)
- **拉取代码**: `git pull origin main`

### 1.3 分支管理
- **查看分支**: `git branch -a`
- **创建并切换**: `git checkout -b new_branch`
- **删除本地分支**: `git branch -d branch_name`
- **删除远程分支**: `git push origin --delete branch_name`
- **重命名分支**: `git branch -m old new`

---

## 2. GitHub 多设备多账号协同开发指南

### 2.1 场景描述
- **设备**：多台电脑 (A, B, C)
- **账号**：多个 GitHub 账号 (主账号 `Main`, 副账号 `Sub`)
- **目标**：在同一台电脑上，针对不同项目使用不同账号进行 Push/Pull。

### 2.2 权限配置 (Collaborator)
若 `Main` 账号想向 `Sub` 账号的仓库推送代码，必须先获得权限：
1. 登录 `Sub` 账号。
2. 仓库 Settings -> Collaborators -> Add people。
3. 邀请 `Main` 账号。
4. `Main` 账号接受邀请。

### 2.3 多账号 SSH 配置 (核心方案)
通过 `~/.ssh/config` 文件，根据不同的 Host 别名自动使用不同的密钥。

#### 步骤 1: 生成多对密钥
```bash
# 主账号密钥
ssh-keygen -t ed25519 -C "main@ex.com" -f ~/.ssh/id_ed25519_main

# 副账号密钥
ssh-keygen -t ed25519 -C "sub@ex.com" -f ~/.ssh/id_ed25519_sub
```

#### 步骤 2: 配置 Config 文件
编辑 `~/.ssh/config`:
```ssh
# 默认 GitHub (主账号)
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_main

# 副账号 GitHub (别名 github-sub)
Host github-sub
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_sub
```

#### 步骤 3: 使用别名克隆/关联
- **主账号项目**:
  `git clone git@github.com:MainUser/Repo.git`
- **副账号项目**:
  `git clone git@github-sub:SubUser/Repo.git`
  *(注意：这里将域名 `github.com` 替换为了配置文件中的 `github-sub`)*

#### 实战案例：解决 "Permission denied"
**现象**: 推送时提示 `Permission to xxx denied to user yyy`.
**原因**: 使用了错误的密钥（身份）。
**解决**:
1. `git remote -v` 查看当前远程地址。
2. 修改远程地址，使用正确的 Host 别名：
   ```bash
   git remote set-url origin git@github-sub:SubUser/Repo.git
   ```

### 2.4 协同工作流 (Workflow)
**原则**: **Pull before Push** (先拉后推)。

1.  **电脑 A** (例如主账号):
    ```bash
    git add .
    git commit -m "Feature A done"
    git push origin main
    ```
2.  **电脑 B** (例如副账号):
    ```bash
    git pull origin main   # 开工前同步
    # ... 进行修改 ...
    git add .
    git commit -m "Feature B done"
    git pull origin main   # 推送前再次确认
    git push origin main
    ```
3.  **其他设备**：复用上述流程，始终保持“先拉后推”。

### 2.5 常见问题
- **冲突 (Conflict)**: `git pull` 报冲突 → 手动解决 → `git add` → `git commit`。
- **暂存 (Stash)**: 临时保存未提交修改 `git stash`，恢复 `git stash pop`。
- **清理远程分支**: 远程已删除、但本地仍保留 → `git remote prune origin`。
- **身份标识配置**：针对不同账号或设备，在项目目录下使用局部配置区分提交者：
  ```bash
  git config user.name "Your Name"
  git config user.email "you@example.com"
  ```

---

## 3. SSH 密钥维护清单

### 3.1 检查现有密钥

```bash
ls -al ~/.ssh
``` 

- 私钥权限需为 `600 (-rw-------)`，公钥权限 `644 (-rw-r--r--)`。

### 3.2 新建密钥（Ed25519）

```bash
ssh-keygen -t ed25519 -C "注释备注"
```

1. 默认保存路径（回车即可），也可指定新文件名。
2. 建议设置 passphrase 以增强安全性。
3. 启动代理并添加私钥：
   ```bash
   eval "$(ssh-agent -s)"
   ssh-add ~/.ssh/id_ed25519
   ```
4. 复制公钥：`cat ~/.ssh/id_ed25519.pub` 上传到 GitHub。
5. 测试连接：`ssh -T git@github.com`。

> 如果使用多个 Host 别名，建议分别验证：
> ```bash
> ssh -T git@github.com
> ssh -T git@github-sub
> ```

### 3.3 清理旧密钥

1. 在 GitHub 删除旧公钥。
2. 本地删除：`rm ~/.ssh/旧私钥文件`。

### 3.4 Linux 间免密登录实战：A ↔ B

> 目标：在 A 机（客户端）与 B 机（服务器）之间建立稳定的 SSH 免密登录，并记录调试过程中遇到的所有问题与解决方案。

#### 3.4.1 环境概览

| 角色 | 系统 / 版本 | OpenSSH 版本 | IP 地址 | 备注 |
| :--- | :--- | :--- | :--- | :--- |
| A 机 | Ubuntu 22.04.5 LTS | OpenSSH_8.9p1 Ubuntu-3ubuntu0.13 | `xxx.xxx.xxx.xxx` | 客户端，已存在 `~/.ssh/id_ed25519*` 密钥对 |
| B 机 | Ubuntu 24.04.3 LTS | OpenSSH_9.6p1 Ubuntu-3ubuntu13.14 | `xxx.xxx.xxx.xxx` | 服务器，需要安装 `openssh-server` |

#### 3.4.2 本机 IP 查询与 `ping` 详解

- 查看本机 IP：
  ```bash
  ip addr show
  ip -brief addr
  hostname -I
  ip route get 8.8.8.8
  ```
  说明：
  - `ip addr show` 输出所有网卡及 IPv4/IPv6；关注非 `lo` 的接口（如 `enp*`, `wlp*`）。
  - `ip -brief addr` 为简表形式，字段更精炼。
  - `hostname -I` 直接列出主机分配到的地址（不含 `127.0.0.1`）。
  - `ip route get 8.8.8.8` 可快速得出访问公网时使用的源地址，特别适合多网卡场景。
- `ping <目标IP>`：向目标发送 ICMP Echo 请求，验证网络连通性与时延。
- 常用参数：
  - `-c <次数>`：限定发送次数，避免一直运行（例如 `ping -c 4 xxx.xxx.xxx.xxx`）。
  - `-i <秒>`：两次请求之间的间隔，默认 1 秒。
  - `-W <秒>`：等待回复的超时时间，默认 1 秒。
  - `-4` / `-6`：强制使用 IPv4 / IPv6。
- 典型输出解析：
  ```bash
  ping -c 4 xxx.xxx.xxx.xxx
  ```
  返回示例：
  ```
  PING xxx.xxx.xxx.xxx (xxx.xxx.xxx.xxx) 56(84) bytes of data.
  64 bytes from xxx.xxx.xxx.xxx: icmp_seq=1 ttl=63 time=3.69 ms
  ...
  --- xxx.xxx.xxx.xxx ping statistics ---
  4 packets transmitted, 4 received, 0% packet loss, time 3005ms
  rtt min/avg/max/mdev = 3.317/6.744/15.928/3.174 ms
  ```
  - `icmp_seq` 表示序号，`ttl`（Time-To-Live）过小可能意味着经过过多路由，`time` 为往返延迟。
  - 总结区的 `packet loss` 为丢包率，应保持 0%；`avg` 为平均往返时间。

#### 3.4.3 前置准备

1. **A 机确认密钥与工具**
  ```bash
  ls ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
  ssh -V
  ```
  可选：更新密钥口令 `ssh-keygen -p -f ~/.ssh/id_ed25519`，或载入 agent：
  ```bash
  eval "$(ssh-agent -s)"
  ssh-add ~/.ssh/id_ed25519
  ```
  期望输出：存在密钥文件时 `ls` 正常列出路径，`ssh -V` 显示 `OpenSSH_8.9p1 …`。
2. **B 机安装并启用 SSH 服务**
  ```bash
  sudo apt update
  sudo apt install openssh-server -y
  sudo systemctl enable --now ssh
  sudo systemctl status ssh
  sudo ss -tlnp | grep ssh
  ```
  期望输出：`systemctl status` 中 `Active: active (running)`；`ss -tlnp` 至少包含 `LISTEN ... :22`。
3. **B 机防火墙放行（默认端口 22）**
  ```bash
  sudo ufw allow OpenSSH
  sudo ufw status
  ```
  期望输出：`Allow OpenSSH` 规则显示为 `ALLOW`。
4. **确认双方网络互通**
  ```bash
  # A 机执行
  ping xxx.xxx.xxx.xxx
  ```
  期望输出：持续收到 `64 bytes from ... time=... ms`，丢包率为 0%。

#### 3.4.4 正式配置流程

1. **确定目标账号**：在 B 机上运行 `whoami`，确认需要免密登录的用户名（本次为 `l`）。
2. **部署公钥（默认端口 22）**
  ```bash
  ssh-copy-id -i ~/.ssh/id_ed25519.pub l@xxx.xxx.xxx.xxx
  ```
  首次会询问是否信任主机指纹，输入 `yes`，随后输入 B 机账户密码。
  期望输出：
  ```
  Number of key(s) added: 1
  Now try logging into the machine, with:   "ssh 'l@xxx.xxx.xxx.xxx'"
  ```
3. **验证免密登录**
  ```bash
  ssh l@xxx.xxx.xxx.xxx
  ```
  登录成功后使用 `exit` 或 `Ctrl+D` 返回 A 机终端。
4. **可选：为 A 机配置别名**
  编辑 `~/.ssh/config`，新增：
  ```
  Host b-server
     HostName xxx.xxx.xxx.xxx
     User l
     Port 22
     IdentityFile ~/.ssh/id_ed25519
     IdentitiesOnly yes
  ```
  字段含义速查：
  - `Host`：**客户端自定义别名**，输入 `ssh b-server` 时会套用该块配置，可取任意便于记忆的英文名称。
  - `HostName`：远端主机的实际地址，可以是 IP 或域名。
  - `User`：登录远端时使用的用户名，省得每次在命令里写 `l@` 前缀。
  - `Port`：连接端口，默认 22；若服务端改了端口，这里需要同步更新。
  - `IdentityFile`：指定使用哪把私钥，常见为 `~/.ssh/id_ed25519` 或 `id_rsa`。
  - `IdentitiesOnly`：设为 `yes` 时只尝试上述私钥，避免因 `ssh-agent` 中其它密钥导致认证失败。

  之后使用 `ssh b-server` 即可连接。

#### 3.4.5 调试过程与故障清单

| 序号 | 现象 / 命令 | 根因分析 | 解决方案 |
| :--- | :--- | :--- | :--- |
| 1 | `ssh-copy-id user@B主机` → `Could not resolve hostname b…` | 使用了中文别名 “B主机”，DNS 无法解析 | 改用实际 IP 或在 `~/.ssh/config` 中定义英文 Host 别名 |
| 2 | `ssh -p <自定义端口> …` → `Connection refused`，`ss -tlnp` 仅见 `:22` | 仅修改了客户端连接端口，但服务端未正确监听自定义端口 | 回到默认端口 22 完成部署；若未来要启用新端口，需要在 `sshd_config` 中正确添加并放行防火墙后再测试 |
| 3 | `ssh-copy-id …` → `Host key verification failed` | `~/.ssh/known_hosts` 保存了旧指纹，与当前 B 机不一致 | 在 A 机运行 `ssh-keygen -R xxx.xxx.xxx.xxx` 后重新连接并接受新指纹 |
| 4 | `ssh-copy-id -i … user@…` → 多次提示密码错误 | B 机账号并非 `user`，实际用户名是 `l` | 在 B 机使用 `whoami` 查明用户名，改用 `ssh-copy-id -i … l@xxx.xxx.xxx.xxx` |

#### 3.4.6 最终验证与日常操作

- 再次确认密钥生效：
  ```bash
  ssh l@xxx.xxx.xxx.xxx
  ```
- 退出会话：`exit` 或 `Ctrl+D`。
- 若需清理主机指纹（例如 B 机重装）：
  ```bash
  ssh-keygen -R xxx.xxx.xxx.xxx
  ```
- 查看当前登录会话与网络：`who`, `w`, `ss -tnp`。

#### 3.4.7 后续可选增强

- 若要启用自定义端口，推荐 **保留 22 端口** 作为应急通道，并在确认服务端监听新端口且防火墙放行后，再更新 A 机的连接命令。
- 使用 VS Code Remote – SSH，可以在 A 机 VS Code 中直接开发 B 机上的代码。
- 若需 GUI 程序，可结合 X11 转发（`ssh -Y l@xxx.xxx.xxx.xxx`）或远程桌面方案；注意开放必要端口并评估网络带宽。

---

## 4. 常见推送场景脚本

### 4.1 全新仓库从 0 到 1

1. GitHub 创建仓库 `name1`。
2. 本地执行：
   ```bash
   git init
   git remote add name2 git@github.com:lzqsyc/name1.git
   git add .
   git commit -m "initial commit"
   git branch -M name3
   git push -u name2 name3  # 将本地 name3 与远程同名分支建立跟踪
   ```

### 4.2 推送到已存在的远程分支

```bash
git init
git remote add name2 git@github.com:lzqsyc/name5.git
git add .
git commit -m "initial commit"
git branch -M name3
git push -u name2 name3:name6  # 本地 name3 → 远程 name6
git branch --set-upstream-to=name2/name6 name3
```

---

## 5. 分支与远程管理进阶

### 5.1 本地分支改名与删除

- 改名：`git branch -m <旧> <新>`（当前分支可省略旧名）
- 删除：
  - 安全删除（已合并）：`git branch -d <分支>`
  - 强制删除：`git branch -D <分支>`

### 5.2 远程别名管理

- 重命名远程：`git remote rename <旧> <新>`
- 删除远程：`git remote remove <别名>`

### 5.3 删除远程分支并解除跟踪

```bash
git push <远程别名> --delete <分支>
git checkout <本地分支>
git branch --unset-upstream
```

### 5.4 建立与调整上游关系

```bash
git branch --set-upstream-to=<远程>/<分支> <本地分支>
git push -u <远程> <本地分支>:<远程分支>
```

### 5.5 状态与引用查询

```bash
git status
git branch -v
git branch -a
git remote -v
git ls-remote <远程>
git fetch <远程>
git checkout -b 分支 远程/分支
```

---
