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
wget https://raw.githubusercontent.com/juanfont/headscale/main/config-example.yaml -O ./headscale/config/config.yaml
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

## 1：多网卡路由黑洞 (命令卡死无响应)

- **现象**：浏览器能访问控制端，但 `tailscale up` 命令卡死。日志报错 `connectex: A connection attempt failed`。
- **原因**：Windows 物理网卡与虚拟机网卡共存时，Tailscale 底层引擎误将物理网卡（如 WLAN）作为默认路由发包，导致去往虚拟机的包被外网路由器丢弃。
- **解决**：
  - **临时法**：暂时断开物理 Wi-Fi，强迫系统走虚拟网卡，连接成功后再恢复。
  - **彻底法**：在 VMware 中配置 NAT 端口转发（将 `127.0.0.1:8080` 转发到虚拟机 `8080`），客户端使用 `http://127.0.0.1:8080` 登录。

## 2：手动执行 `tailscaled.exe` 导致 NoState 秒断

- **现象**：使用命令行强行执行 `& "...\tailscaled.exe"`，提示 `unexpected state: NoState`，且瞬间断开连接。
- **原因**：Tailscale 是前后端分离架构，`tailscaled.exe` 必须作为 Windows **系统最高权限服务 (System Service)** 在后台静默运行。手动在前台强行拉起，会导致权限错位并与原有后台服务发生管道抢占 (Access is denied)。
- **解决**：永远不要手动运行引擎程序。如遇卡死，去任务管理器结束所有 Tailscale 进程，并通过 Windows 服务管理器重启 `Tailscale` 服务。

## 3：手机热点导致 `no-derp-connection` 报错

- **现象**：日志疯狂报错连不上官方的 `Hong Kong` 中继服务器 (DERP)，且后台显示节点 Offline。
- **原因**：国内三大运营商在蜂窝网络（手机热点）下，由于严格的 NAT 类型和防火墙策略，经常会屏蔽或干扰 Tailscale 官方的海外中继节点。
- **解决**：
  - 若双方在**同一局域网/同一热点**下，该报错可无视（局域网直连已生效）。
  - 若在异地且必须使用，需更换为宽带 Wi-Fi，或在云服务器上自建私有 DERP 中继节点。

------

## 6. 日常运维必备命令

| **功能**           | **命令**                                                     |
| ------------------ | ------------------------------------------------------------ |
| **创建新用户**     | `docker exec headscale headscale users create <用户名>`      |
| **生成接入密钥**   | `docker exec headscale headscale preauthkeys create -e 24h -u <用户ID> --reusable` |
| **查看当前用户**   | `docker exec headscale headscale users list`                 |
| **查看有效密钥**   | `docker exec headscale headscale preauthkeys list -u <用户ID>` |
| **查看在线节点**   | `docker exec headscale headscale nodes list`                 |
| **踢出离线节点**   | `docker exec headscale headscale nodes delete -i <节点ID>`   |
| **查看宣告路由**   | `docker exec headscale headscale routes list`                |
| **审批/启用路由**  | `docker exec headscale headscale routes enable -r <路由ID>`  |
| **查看实时日志**   | `docker logs -f headscale`                                   |
| **查看客户端状态** | 在 Windows/Mac 客户端终端执行：`tailscale status`            |