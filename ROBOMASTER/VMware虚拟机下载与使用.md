# VMware 虚拟机（Workstation）下载与使用 教程

> 目标：从零下载 VMware Workstation → 安装 → 建一台 Ubuntu 虚拟机 → 会用。

---

## 一、下载 VMware Workstation Pro

> 现状：VMware 已被 **Broadcom** 收购，**Workstation Pro 现在个人和商用都免费**，
> 但只能从 **Broadcom 官方门户**下载（要注册一个免费账号）。
> 老的 CDN `download3.vmware.com` 已失效，**不要**去第三方站下（有捆绑/中毒风险）。

### 下载步骤
1. 打开 **Support Portal**（注意：不是 Broadcom 公司首页 broadcom.com）：
   ```
   https://support.broadcom.com/group/ecx/downloads
   ```
2. **注册 / 登录**一个免费 Broadcom 账号。
3. 在搜索框搜 **`VMware Workstation Pro`**。
4. 选**最新版本**（如 17.x / 2x.x）→ 选 **Windows** → 下载安装包
   （文件名类似 `VMware-workstation-full-XX.x.x-xxxxx.exe`）。

### 安装
- 双击 `.exe` → 一路 **Next**（默认即可）→ 免费版无需输入许可证 → **Install** → Finish。
- 装完重启一次更稳。

### 前置检查（很重要）
- 进 BIOS/固件，确认 **CPU 虚拟化（Intel VT-x / AMD-V）已开启**，否则 VM 开不了机。
- 预留磁盘空间：一个 Ubuntu VM 建议至少 **40–60 GB**。

---

## 二、创建 Ubuntu 虚拟机

1. VMware → **"创建新的虚拟机"（Create a New Virtual Machine）**。
2. 选 **典型（Typical）**（新手够用）或 **自定义（Custom）**。
3. **安装来源**：选 **"安装程序光盘映像文件 (ISO)"**，指向你的 **Ubuntu 22.04 Desktop ISO**。
   - 没有 ISO 就去 https://ubuntu.com/download/desktop 下 **22.04 LTS**。
4. 填 **虚拟机名称** + **存放位置**（选空间大的分区，如 `D:\`）。
5. **磁盘容量**：建议 **40–60 GB**（选"将虚拟磁盘存储为单个文件"或拆分都行）。
6. **硬件**：内存建议 **≥ 8 GB**、CPU **≥ 4 核**（按你机器调整）。
7. **网络类型**：
   - **NAT**（默认，推荐）：VM 能上网，宿主机也能连它；
   - **桥接（Bridged）**：VM 相当于局域网里一台独立机器，会拿路由器分配的 IP。
8. 完成 → 开机 → 走 Ubuntu 安装向导（语言、键盘、磁盘"清除整个磁盘并安装"、设**用户名+密码**、等安装完重启）。

> 装 Ubuntu 时设的 **用户名/密码**，就是以后 `ssh 用户名@IP` 用的那套。

---

## 三、基本使用

| 操作 | 怎么做 |
|---|---|
| 开机 / 关机 | 顶部工具栏 ▶ 电源按钮；或在 Ubuntu 里正常关机 |
| 暂停 / 挂起 | 建议少用（VM 里挂起易出问题），要停就用"关机/挂起"|
| **快照（Snapshot）** | 「虚拟机 → 快照 → 拍摄快照」——**强烈建议**：装好系统、配好环境后各拍一次，出问题一键回滚 |
| 全屏 | `Ctrl + Alt + Enter` |
| 鼠标/键盘被"吸"进 VM | 按 **`Ctrl + Alt`** 释放 |
| 宿主机→VM 复制粘贴/拖拽 | 装 **VMware Tools**（Ubuntu 自带 open-vm-tools 通常已够）|
| 共享文件夹 | 「虚拟机设置 → 选项 → 共享文件夹」|

---

## 四、注意事项

- **IP 会变**：NAT 模式下 VM 的 IP（`192.168.x.x`）重启后可能变。
  在 Ubuntu 里用 `hostname -I` 查当前 IP。
- **NAT vs 桥接**：
  - 只想宿主机连 VM / VM 上网 → **NAT** 够用；
  - 想让同网段其它设备也能连 VM → **桥接**。
- **磁盘别放满**：VM 会随使用变大，宿主机分区要留余量。
- **别和 WSL 搞混**：如果 Windows 还装了 WSL，那是另一套 Linux，和 VMware 里的 VM 是两回事。

---

## 五、一句话流程

```
官方门户下 VMware Workstation Pro（免费、需注册）
   → 安装（确认 BIOS 开了虚拟化）
   → 新建 VM，挂 Ubuntu 22.04 ISO
   → 设内存/CPU/磁盘/网络（NAT）
   → 装 Ubuntu，记住用户名+密码
   → 拍快照
```

> 装好 Ubuntu 后，下一步如果要用 VS Code 远程连它，见《VS Code 远程连接 Ubuntu 虚拟机 教程》。

