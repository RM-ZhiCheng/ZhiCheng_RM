# Ubuntu 开启 SSH（远程登录）教程

> 目标：让 Ubuntu 能被**别的电脑**（Windows / 另一台 Linux）通过命令行远程登录，
> 也是用 VS Code Remote-SSH、传文件、跑远程命令的前提。

---

## 一、SSH 是什么

- **SSH** = 安全的远程登录协议。开了 SSH 服务端（`sshd`），别的电脑就能
  `ssh 用户名@IP` 登进这台 Ubuntu。
- Ubuntu **默认不装 SSH 服务端**，所以要**手动装并启动**，否则连它一定报
  `Connection refused`。
- 名词区分：
  - **openssh-server（服务端）**：装在**被连接的那台 Ubuntu** 上；
  - **ssh 客户端**：在**你连它的那台电脑**上（Windows 自带）。

---

## 二、开启步骤（在 Ubuntu 终端里）

```bash
# 1) 更新源（可选，但推荐）
sudo apt update

# 2) 安装 SSH 服务端
sudo apt install openssh-server -y

# 3) 设为开机自启 + 立即启动
sudo systemctl enable --now ssh

# 4) 确认服务在跑（看到 active (running) 就对）
systemctl status ssh

# 5) 确认 22 端口在监听
ss -tlnp | grep :22
```

第 5 步正常应显示：

```
LISTEN 0  128  0.0.0.0:22  0.0.0.0:*  users:(("sshd",pid=...,fd=3))
```

> 提示：`netstat` 属于 `net-tools`，Ubuntu 默认没有（`sudo apt install net-tools` 才有）。
> 平时用系统自带的 **`ss -tlnp`** 就够了，不用装 net-tools。

---

## 三、查本机 IP

在 Ubuntu 里：

```bash
hostname -I
```

例如 `192.168.113.128`。这个 IP 就是别人连你的地址。

> ⚠️ 虚拟机（VMware NAT）的 IP **重启后可能变**，连不上时先回来重新查。

---

## 四、从另一台电脑连接

在 **Windows 的 PowerShell / CMD** 里：

```bash
ssh 用户名@IP
# 例：ssh qingyun@192.168.113.128
```

- 第一次会问：
  ```
  Are you sure you want to continue connecting (yes/no/[fingerprint])?
  ```
  → **必须输完整的 `yes`（三个字母），不能只输 `y`**，否则会一直提示
  `Please type 'yes' or 'no'`。
- 然后输 **Ubuntu 的登录密码**（输入时不显示，正常）。
- 看到命令行提示符 → 登录成功。

---

## 五、可选：免密登录（用密钥，省得每次输密码）

在**你连它的那台电脑**（如 Windows）上：

```bash
# 1) 生成密钥（一路回车）
ssh-keygen

# 2) 把公钥传到 Ubuntu（会要输一次密码）
ssh-copy-id 用户名@IP
```

以后 `ssh 用户名@IP` 就不用密码了。

---

## 六、防火墙（如果开了 ufw 才需要）

虚拟机里一般没开 ufw，可跳过。若开了：

```bash
sudo ufw allow ssh
# 或指定端口
sudo ufw allow 22/tcp
```

---

## 七、常见问题排查

| 现象 | 原因 | 解决 |
|---|---|---|
| `Connection refused` | Ubuntu **没装/没启** sshd（22 没人听） | 按本文第二节装 + 启动 |
| 一直提示 `Please type 'yes' or 'no'` | 指纹确认处输的是 `y` 不是 `yes` | 输完整的 **`yes`** |
| 连接超时（无响应） | IP 不对 / 不在同一网络 / VM 网络模式问题 | `hostname -I` 重查 IP；确认网络（NAT/桥接） |
| 密码总不对 | 记错密码 / 键盘大小写 | 在 Ubuntu 里 `sudo passwd 用户名` 重设 |
| 想换端口 | 改配置 | 编辑 `/etc/ssh/sshd_config` 的 `Port`，再 `sudo systemctl restart ssh` |

---

## 八、一句话流程

```
sudo apt install openssh-server -y          # 装服务端
sudo systemctl enable --now ssh             # 启动 + 自启
systemctl status ssh                        # 确认在跑
ss -tlnp | grep :22                         # 确认 22 端口
hostname -I                                 # 查 IP
# 另一台电脑：
ssh 用户名@IP                                # 连（第一次输 yes + 密码）
```

> 连上之后再上 **VS Code Remote-SSH**，见《VS Code 远程连接 Ubuntu 虚拟机 教程》。

