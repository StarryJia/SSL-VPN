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
mkdir -p ~/headscale/data

# 2. 下载官方示例配置文件
wget https://raw.githubusercontent.com/juanfont/headscale/main/config-example.yaml -O ./headscale/config/config.yaml
```

## 2.2 修改关键配置 (`config.yaml`)

使用 `nano ~/headscale/config/config.yaml` 打开文件，找到并修改以下关键行：

> **⚠️ 避坑提示**：YAML 格式要求极其严格，**冒号后面必须有一个英文空格**。

```yaml
# 填入您虚拟机的局域网 IP 和映射端口
server_url: [http://服务端IP:8080](http://服务端IP:8080)

# 监听所有网卡的 8080 端口
listen_addr: 0.0.0.0:8080
```

## 2.3 启动 Headscale 容器

> **⚠️ 避坑提示**：务必加上 `--restart always` 参数。这相当于“免死金牌”，虚拟机重启后容器会自动拉起，防止数据未加载或丢失。

```bash
#拉起容器（补齐 derp（端口换为3479） 映射）
docker run -d \
  --name headscale \
  --restart always \
  -v ~/headscale/config:/etc/headscale \
  -v ~/headscale/data:/var/lib/headscale \
  -p 8080:8080 \
  -p 3479:3479/udp \
  ghcr.io/juanfont/headscale:latest \
  serve
```

------

## 3. 用户与密钥管理

## 3.1 创建用户 (User)

Headscale 网络中的节点必须归属于某个用户。

```bash
docker exec headscale headscale users create devteam
```

## 3.2 获取用户 ID 并生成接入密钥 (AuthKey)

> **⚠️ 避坑提示**：Headscale 最新版本不支持在命令中直接使用用户名，必须先查询用户的**数字 ID**（如 `1`）。

```bash
# 1. 查询用户列表，记住 devteam 对应的 ID
docker exec headscale headscale users list

# 2. 为该 ID (假设为 1) 生成 24 小时有效的可重复使用密钥
docker exec headscale headscale preauthkeys create -e 24h -u 1 --reusable
```

*(将生成的这串 `hskey-...` 妥善保存，用于客户端接入)*

------

## 4. DERP (中继服务器) 配置与私有化部署

**什么是 DERP？** 当由于严格的网络防火墙（如手机热点、对称 NAT）导致两台设备无法直接 P2P “打洞”相连时，流量会通过 DERP 中继服务器进行 HTTPS 伪装转发。Headscale 默认会去拉取 Tailscale 官方提供的全球 DERP 节点列表。

## 4.1 纯离线/暗网模式 (禁用官方 DERP)

**适用场景**：物理机完全没有外网连接（如保密机房、纯局域网环境）。如果不禁用官方列表，Headscale 启动时会因为连不上官方服务器而触发 `context deadline exceeded` 报错并闪退。

**操作步骤**：

1. 编辑配置文件：

   ```bash
   nano ~/headscale/config/config.yaml
   ```

2. 找到 `derp:` 配置块下的 `urls:`，将其设为空列表（或者用 `#` 注释掉官方网址）：

   ```YAML
   derp:
     server:
       # ...省略自带 server 配置...
     urls: []  # 设为空列表，强制不联网拉取官方节点
   ```

3. 重启服务生效：`docker restart headscale`

## 4.2 开启 Headscale 内置的本地 DERP 服务

**适用场景**：您的 Headscale 部署在具有公网 IP 的云服务器上，或者您确信所有设备都能连通该 Headscale 所在的局域网 IP。您可以开启它自带的 DERP 服务，实现完全私有化的流量中转。

**操作步骤**：

1. 编辑配置文件：

   ```Bash
   vim ~/headscale/config/config.yaml
   ```

2. 找到 `derp: server:` 节点，按如下修改开启内置中继：

   ```yaml
   derp:
     server:
       # 开启内置 DERP 服务
       enabled: true
   
       # 节点所在区域（仅作显示用）
       region_id: 999
       region_code: "local"
       region_name: "My Local DERP"
   
       # STUN 服务监听端口 (UDP，用于辅助打洞，需要在宿主机/防火墙放行该端口)
       stun_listen_addr: "0.0.0.0:3478"
   ```

3. **防火墙放行**：如果开启了内置 DERP，必须在宿主机和网络防火墙上额外放行 **UDP 3478** 端口。

4. 重启服务生效：`docker restart headscale`

> **⚠️ 进阶架构提示**： 如果 Headscale 控制端没有公网 IP，但您又想在异地网络极差的情况下实现稳定中转，建议在阿里云/腾讯云等具有公网 IP 的服务器上，单独部署一个开源的 `derper` (Tailscale 自建中继程序)，并在 `config.yaml` 的 `paths:` 中引入该私有节点的配置文件。

## 5. Windows 客户端接入

1. 去 Tailscale 官网下载并安装 Windows 客户端。
2. 安装后**不要**点击界面上的 Log in（会连到官方服务器）。
3. 以**管理员身份**打开 PowerShell，执行接入命令：

```PowerShell
tailscale up --login-server=[http://服务端IP:8080](http://服务端IP:8080) --authkey=你的密钥 --force-reauth
```

------

## 6. 日常运维必备命令

| **功能**           | **命令**                                                     |
| ------------------ | ------------------------------------------------------------ |
| **创建新用户**     | `docker exec headscale headscale users create <用户名>`      |
| **生成接入密钥**   | `docker exec headscale headscale preauthkeys create -e 24h -u <用户ID> --reusable` |
| **查看当前用户**   | `docker exec headscale headscale users list`                 |
| **查看有效密钥**   | `docker exec headscale headscale preauthkeys list`           |
| **使密钥过期**     | `docker exec headscale headscale preauthkeys expire --id <密钥ID>` |
| **删除密钥**       | `docker exec headscale headscale preauthkeys delete --id <密钥ID>` |
| **查看在线节点**   | `docker exec headscale headscale nodes list`                 |
| **踢出离线节点**   | `docker exec headscale headscale nodes delete -i <节点ID>`   |
| **查看宣告路由**   | `docker exec headscale headscale routes list`                |
| **审批/启用路由**  | `docker exec headscale headscale routes enable -r <路由ID>`  |
| **查看实时日志**   | `docker logs -f headscale`                                   |
| **查看客户端状态** | 在 Windows/Mac 客户端终端执行：`tailscale status`            |

## 7. 部署 Web 可视化面板 (Headscale-UI) 及跨域修复

为了彻底告别命令行，我们可以部署官方社区推荐的 `headscale-ui` 面板。由于 Headscale 后端自身存在对 `OPTIONS` 跨域预检请求处理缺陷（导致 401 报错），我们需要引入一个轻量级的 Caddy 容器作为反向代理桥梁。

## 7.1 申请长期 API Key

在服务端（运行 Headscale 的宿主机）终端生成一个给 Web 面板专用的控制钥匙：

```Bash
# 生成一个有效期为 365 天的 API Key（运行后请务必复制保存打印出的那串字符）
docker exec headscale headscale apikeys create -e 365d
```

## 7.2 部署 Caddy 反向代理 (跨域修复桥梁)

该步骤用于拦截浏览器的 `OPTIONS` 跨域预检请求并放行，同时将真实 API 请求转发给 Headscale 后端 (8080端口)。

**1. 创建 Caddy 配置文件：**

```bash
nano ~/headscale/Caddyfile
```

将以下内容完整粘贴进去（请将 `192.168.x.x` 替换为您服务端的真实局域网 IP）：

```
:9090 {
    # 1. 拦截浏览器的 OPTIONS 预检请求，直接返回 204 成功
    @options {
        method OPTIONS
    }
    handle @options {
        header Access-Control-Allow-Origin "*"
        header Access-Control-Allow-Methods "GET, POST, PUT, PATCH, DELETE, OPTIONS"
        header Access-Control-Allow-Headers "*"
        respond 204
    }

    # 2. 正常的请求转发给 Headscale 后端，并在返回时带上允许跨域的头
    handle {
        reverse_proxy 192.168.x.x:8080 {
            header_down Access-Control-Allow-Origin "*"
        }
    }
}
```

*(按 `Ctrl+O` 保存，回车确认，`Ctrl+X` 退出)*

**2. 启动 Caddy 代理容器：**

> **⚠️ 避坑提示**：若拉取官方镜像遇到 `connection refused` (被墙)，请在镜像名前添加国内加速源前缀，如 `docker.m.daocloud.io/library/caddy:latest`。

```Bash
docker run -d \
  --name headscale-cors-fix \
  --restart always \
  -p 9090:9090 \
  -v ~/headscale/Caddyfile:/etc/caddy/Caddyfile \
  caddy:latest
```

## 7.3 部署 Headscale-UI 前端容器

UI 容器内部服务运行在 `8080` 端口，为避免与宿主机上的 Headscale 冲突，我们将其映射到宿主机的 `8081` 端口：

```bash
docker run -d \
  --name headscale-ui \
  --restart always \
  -p 8081:8080 \
  ghcr.io/gurucomputing/headscale-ui:latest
```

## 7.4 浏览器登录与绑定配置

1. 打开浏览器，访问前端 UI 页面：`http://<服务端IP>:8081`。
2. 点击页面中的 **Settings (设置)**。
3. 填写以下两项关键信息：
   - **Server URL**: `http://<服务端IP>:9090` *(⚠️ 注意：这里的端口是 Caddy 桥梁的 9090，且末尾切勿带有斜杠 `/`)*
   - **API Key**: 填入在 8.1 步骤中生成的长串密钥。
4. 点击 **Save**，页面即可瞬间获取后端的 Nodes 和 Users 数据，可视化管理配置完成！

## 核心排障记录 (Troubleshooting)

### 1：多网卡路由黑洞 (命令卡死无响应)

- **现象**：浏览器能访问控制端，但 `tailscale up` 命令卡死。日志报错 `connectex: A connection attempt failed`。
- **原因**：Windows 物理网卡与虚拟机网卡共存时，Tailscale 底层引擎误将物理网卡（如 WLAN）作为默认路由发包，导致去往虚拟机的包被外网路由器丢弃。
- **解决**：
  - **临时法**：暂时断开物理 Wi-Fi，强迫系统走虚拟网卡，连接成功后再恢复。
  - **彻底法**：在 VMware 中配置 NAT 端口转发（将 `127.0.0.1:8080` 转发到虚拟机 `8080`），客户端使用 `http://127.0.0.1:8080` 登录。

### 2：手动执行 `tailscaled.exe` 导致 NoState 秒断

- **现象**：使用命令行强行执行 `& "...\tailscaled.exe"`，提示 `unexpected state: NoState`，且瞬间断开连接。
- **原因**：Tailscale 是前后端分离架构，`tailscaled.exe` 必须作为 Windows **系统最高权限服务 (System Service)** 在后台静默运行。手动在前台强行拉起，会导致权限错位并与原有后台服务发生管道抢占 (Access is denied)。
- **解决**：永远不要手动运行引擎程序。如遇卡死，去任务管理器结束所有 Tailscale 进程，并通过 Windows 服务管理器重启 `Tailscale` 服务。

### 3：手机热点导致 `no-derp-connection` 报错

- **现象**：日志疯狂报错连不上官方的 `Hong Kong` 中继服务器 (DERP)，且后台显示节点 Offline。
- **原因**：国内三大运营商在蜂窝网络（手机热点）下，由于严格的 NAT 类型和防火墙策略，经常会屏蔽或干扰 Tailscale 官方的海外中继节点。
- **解决**：
  - 若双方在**同一局域网/同一热点**下，该报错可无视（局域网直连已生效）。
  - 若在异地且必须使用，需更换为宽带 Wi-Fi，或在云服务器上自建私有 DERP 中继节点。