# VS Code 远程连接 Ubuntu 虚拟机 —— 完整教程

> 场景：Windows 主机上装了 VMware Workstation，里面跑一台 Ubuntu 虚拟机；
> 目标是从 Windows 用 **VS Code Remote-SSH** 直接连进这台 Ubuntu 写代码。
> 本文按"建 VM → 开 SSH → Windows 测试 → VS Code 连接 → 踩坑排查"完整记录。

---

## 0. 前置
- Windows 已装 **VMware Workstation**（免费版即可）
- 已在 VMware 里建好 **Ubuntu 22.04 虚拟机**并能正常开机进桌面
- Windows 上装了 **VS Code**

---

## 1. Ubuntu 里「开启 SSH」

Ubuntu 默认**不装 SSH 服务端**，所以一开始连不上（`Connection refused`）。在 **Ubuntu 终端**里：

```bash
# 1) 安装 SSH 服务端
sudo apt update
sudo apt install openssh-server -y

# 2) 开机自启 + 立即启动
sudo systemctl enable --now ssh

# 3) 确认在跑（看到 active (running) 就对）
systemctl status ssh

# 4) 确认 22 端口在监听（用系统自带的 ss，不用网上教程里的 netstat）
ss -tlnp | grep :22
```

第 4 步应看到类似：

```
LISTEN 0  128  0.0.0.0:22  0.0.0.0:*  users:(("sshd",pid=...,fd=3))
```

> 备注：`netstat` 属于 `net-tools`，Ubuntu 默认没装（要 `sudo apt install net-tools`）。
> 平时直接用 `ss -tlnp` 就行，不用装。

---

## 2. 查 Ubuntu 的 IP

在 **Ubuntu 终端**里：

```bash
hostname -I
```

例如得到 `192.168.113.128`（VMware NAT 网段通常是 `192.168.x.x`）。

> ⚠️ **VMware NAT 的 IP 重启后可能会变**。以后连不上，先回来 `hostname -I` 看新 IP。

---

## 3. 先做「纯命令 SSH」测试（关键一步）

在 **Windows 的 PowerShell / CMD** 里：

```bash
ssh 用户名@IP
# 例：ssh qingyun@192.168.113.128
```

- 第一次会问 `Are you sure you want to continue connecting (yes/no/[fingerprint])?`
  → **必须输完整的 `yes`（三个字母），不能只输 `y`**；输 `y` 会一直提示 `Please type 'yes' or 'no'`。
- 然后输 **Ubuntu 的登录密码**。
- 能看到 Ubuntu 的命令行提示符 → **SSH 通了**。

> 如果这一步就报 `Connection refused` → 回第 1 步检查 `sshd` 有没有装/启动。

---

## 4. VS Code Remote-SSH 配置

### 4.1 装扩展
VS Code 里装扩展 **`Remote - SSH`**（微软官方）。

### 4.2 添加主机
`Ctrl + Shift + P` → **`Remote-SSH: Add New SSH Host...`** → 输入：

```
ssh 用户名@IP
```

（如 `ssh qingyun@192.168.113.128`）→ 保存到默认的 `C:\Users\<用户名>\.ssh\config`。

### 4.3 【关键】改 `settings.json`（绕开"下载 server 卡死"）

打开：`Ctrl + Shift + P` → **`Preferences: Open User Settings (JSON)`**
文件路径：`C:\Users\<用户名>\AppData\Roaming\Code\User\settings.json`

加入这两条（**注意前面的逗号，JSON 少个逗号就会整份失效**）：

```json
"remote.SSH.localServerDownload": "always",
"remote.SSH.useLocalServer": false
```

含义：
- **`localServerDownload: "always"`**：先在 **Windows 本地**下载 "VS Code Server"，再传给 Ubuntu。
  → 因为 Ubuntu 直连微软 CDN 常被卡；在你 Windows 上下载（有代理/网速好）再推过去，绕过卡死。
- **`useLocalServer: false`**：关掉 VS Code 在 Windows 上的本地 SSH 助手进程（有时会导致连接卡住）。

### 4.4 连接
`Ctrl + Shift + P` → **`Remote-SSH: Connect to Host...`** → 选 `用户名@IP` → 输密码。

第一次会看到 **"正在下载 VS Code 服务器…"** —— 这就是容易卡住的一步；
按 4.3 配好后，它会**在本地下载**，通常能顺利完成。

---

## 5. 排查记录（本次实际踩的坑）

| 现象 | 原因 | 解决 |
|---|---|---|
| `ssh` 连接 `Connection refused` | Ubuntu 没装/没启 `sshd`（22 端口没人听） | `sudo apt install openssh-server -y` + `systemctl enable --now ssh` |
| 一直提示 `Please type 'yes' or 'no'` | 在主机指纹确认处输的是 `y` 不是 `yes` | 输完整 **`yes`** |
| VS Code 卡在「正在打开远程…／正在下载 VS Code 服务器」 | 远端从微软 CDN 下 server 卡住 | `remote.SSH.localServerDownload: "always"`（本地下载再传） |
| 加了设置却不生效 | `settings.json` **少了一个逗号**，整份 JSON 语法错误 | 补上逗号（JSON 必须合法） |
| 之后连不上 | VMware NAT 的 **IP 变了** | Ubuntu 里 `hostname -I` 看新 IP |

---

## 6. 日常使用速查

```bash
# Ubuntu 侧：确认 SSH 服务在跑
systemctl status ssh

# Ubuntu 侧：查当前 IP
hostname -I
```

```bash
# Windows 侧：纯命令行 SSH
ssh 用户名@IP
```

- VS Code 连接：`Remote-SSH: Connect to Host...` → 选主机。
- **别连错目标**：如果你同时装了 WSL（如 `WSL: Ubuntu-22.04`），注意区分是连 **VMware 里的 Ubuntu** 还是 **WSL**。

---

## 7. 小结（一条链路）

```
VMware 建 Ubuntu VM
   → Ubuntu 装并启动 openssh-server（22 端口）
   → hostname -I 拿到 IP
   → Windows 用 ssh 测试（输 yes + 密码）
   → VS Code 装 Remote-SSH，添加主机
   → settings.json 加 localServerDownload=always（本地下 server）
   → VS Code 连接成功
```

