# Headscale (WireGuard) 私有 VPN 部署与排障实战手册

## 1. 架构与环境概述
* **核心协议**：WireGuard (底层高性能加密隧道)
* **服务端控制面**：Headscale (部署于 Ubuntu 虚拟机，IP: `服务端IP`)
* **客户端**：Tailscale 官方客户端 (Windows)
* **网络环境**：VMware NAT 模式 (VMnet8)，存在多网卡共存情况。

---

## 2. 服务端部署流程 (Ubuntu + Docker)

### 2.1 准备目录与配置文件
> **⚠️ 避坑提示**：强烈建议使用主目录绝对路径（如 `~/headscale`），避免使用 `$(pwd)` 导致由于当前目录变化而让 Docker 创建空文件夹，从而“弄丢”数据。

```bash
# 1. 回到主目录并创建配置文件夹
cd ~
mkdir -p ~/headscale/config

# 2. 创建空白的 SQLite 数据库文件
touch ~/headscale/config/db.sqlite

# 3. 下载官方示例配置文件
wget [https://raw.githubusercontent.com/juanfont/headscale/main/config-example.yaml](https://raw.githubusercontent.com/juanfont/headscale/main/config-example.yaml) -O ~/headscale/config/config.yaml
```

## 2.2 修改关键配置 (`config.yaml`)

使用 `nano ~/headscale/config/config.yaml` 打开文件，找到并修改以下关键行：

> **⚠️ 避坑提示**：YAML 格式要求极其严格，**冒号后面必须有一个英文空格**。

YAML

```
# 填入您虚拟机的局域网 IP 和映射端口
server_url: [http://服务端IP:8080](http://服务端IP:8080)

# 监听所有网卡的 8080 端口
listen_addr: 0.0.0.0:8080

# 数据库路径（注意：这是容器内的路径，不要改成宿主机路径）
db_path: /etc/headscale/db.sqlite
```

## 2.3 启动 Headscale 容器

> **⚠️ 避坑提示**：务必加上 `--restart always` 参数。这相当于“免死金牌”，虚拟机重启后容器会自动拉起，防止数据未加载或丢失。

Bash

```
docker run -d \
  --name headscale \
  --restart always \
  -v ~/headscale/config:/etc/headscale \
  -p 8080:8080 \
  ghcr.io/juanfont/headscale:latest \
  serve
```

------

## 3. 用户与密钥管理

## 3.1 创建用户 (User)

Headscale 网络中的节点必须归属于某个用户。

Bash

```
docker exec headscale headscale users create devteam
```

## 3.2 获取用户 ID 并生成接入密钥 (AuthKey)

> **⚠️ 避坑提示**：Headscale 最新版本不支持在命令中直接使用用户名，必须先查询用户的**数字 ID**（如 `1`）。

Bash

```
# 1. 查询用户列表，记住 devteam 对应的 ID
docker exec headscale headscale users list

# 2. 为该 ID (假设为 1) 生成 24 小时有效的可重复使用密钥
docker exec headscale headscale preauthkeys create -e 24h -u 1 --reusable
```

*(将生成的这串 `hskey-...` 妥善保存，用于客户端接入)*

------

## 4. Windows 客户端接入

1. 去 Tailscale 官网下载并安装 Windows 客户端。
2. 安装后**不要**点击界面上的 Log in（会连到官方服务器）。
3. 以**管理员身份**打开 PowerShell，执行接入命令：

PowerShell

```
tailscale up --login-server=[http://服务端IP:8080](http://服务端IP:8080) --authkey=你的密钥 --force-reauth
```

------

## 5. 核心排障记录 (Troubleshooting)

在部署过程中，最棘手的问题是**客户端执行连接命令后“无响应 / 死等超时”**。以下是针对此现象的完整排查链路：

## 症状一：Tailscale 后台服务卡死

- **现象**：命令无响应，没有日志生成。
- **原因**：Windows 端的核心服务 `tailscaled.exe` 陷入死锁或状态冲突。
- **解决**：打开任务管理器，强制结束所有 `tailscale.exe` 和 `tailscaled.exe` 进程，然后重启服务。

## 症状二：多网卡路由黑洞（本次实战中的“真凶”）

- **现象**：浏览器能访问 `http://服务端IP:8080/windows`，但 Tailscale 命令依然无响应。强制前台运行引擎（`& "C:\Program Files\Tailscale\tailscaled.exe"`）后，日志报错 `dial tcp 服务端IP:8080... connectex: A connection attempt failed`。
- **原因**：Windows 存在多张网卡（物理 Wi-Fi、VMnet1、VMnet8）。Tailscale 底层的 Go 引擎强行绑定了系统的“默认路由网卡”（即物理 Wi-Fi）去发包，导致发往虚拟机 `137` 网段的数据包被路由器直接丢弃，形成路由黑洞。
- **解决（断网大法）**：
  1. 暂时禁用 Windows 宿主机的物理 Wi-Fi / 拔掉网线。
  2. 此时系统默认路由只剩下 VMnet 虚拟网卡。
  3. 重新执行 `tailscale up ...` 命令，瞬间秒连。
  4. 连上后，重新开启 Wi-Fi，VPN 隧道自动维持，不影响后续使用。

------

## 6. 日常运维必备命令

所有服务端命令需在 Ubuntu 终端下执行：

| **功能**                 | **命令**                                                   |
| ------------------------ | ---------------------------------------------------------- |
| **查看在线节点 (设备)**  | `docker exec headscale headscale nodes list`               |
| **查看当前用户**         | `docker exec headscale headscale users list`               |
| **查看有效密钥**         | `docker exec headscale headscale preauthkeys list -u 1`    |
| **查看实时运行日志**     | `docker logs -f headscale`                                 |
| **删除/踢出节点**        | `docker exec headscale headscale nodes delete -i <节点ID>` |
| **查看客户端状态 (Win)** | 在 Windows PowerShell 执行：`tailscale status`             |