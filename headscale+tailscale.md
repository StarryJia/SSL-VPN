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

### 2.2 修改关键配置 (`config.yaml`)

使用 `nano ~/headscale/config/config.yaml` 打开文件，找到并修改以下关键行：

> **⚠️ 避坑提示**：YAML 格式要求极其严格，**冒号后面必须有一个英文空格**。

```yaml
# 填入您虚拟机的局域网 IP 和映射端口
server_url: [http://服务端IP:8080](http://服务端IP:8080)

# 监听所有网卡的 8080 端口
listen_addr: 0.0.0.0:8080
```

### 2.3 启动 Headscale 容器

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

### 3.1 创建用户 (User)

Headscale 网络中的节点必须归属于某个用户。

```bash
docker exec headscale headscale users create devteam
```

### 3.2 获取用户 ID 并生成接入密钥 (AuthKey)

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

### 4.1 纯离线/暗网模式 (禁用官方 DERP)

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

### 4.2 开启 Headscale 内置的本地 DERP 服务

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

### 4.3 DERP 自定义端口私有化部署

当默认的 3478 端口发生冲突，或需要自定义中继节点配置时，需分离服务端的监听配置与客户端的路由下发配置。本节以 STUN 打洞端口修改为 `3479`、HTTPS 中继端口复用 `8081` 为例说明部署标准流程。

#### 1. 创建 DERP 路由配置文件

该文件用于定义客户端连接中继节点的规则。在宿主机配置目录中新建 `derp.yaml` 文件：

```bash
nano /root/headscale/config/derp.yaml
```

写入以下配置。注意：YAML 语法严格要求使用纯空格缩进，禁止使用 Tab 键。

```yaml
regions:
  900:
    regionid: 900
    regioncode: myderp
    regionname: My True 3479 DERP
    nodes:
      - name: 900a
        regionid: 900
        hostname: head.emolu.cn
        derpport: 8081
        stunport: 3479
        stunonly: false
```

#### 2. 修改 Headscale 主配置文件

打开 Headscale 主配置文件 `config.yaml`：

```bash
nano /root/headscale/config/config.yaml
```

定位至 `derp` 模块，修改并确认以下三个参数：

1. `stun_listen_addr` 更改为目标端口 `0.0.0.0:3479`。
2. `urls` 列表留空，以禁用官方默认节点。
3. `paths` 列表中添加自定义路由文件在**容器内部**的绝对路径。

修改后的标准结构如下：

```YAML
derp:
  server:
    enabled: true
    region_id: 999
    region_code: "headscale"
    region_name: "Headscale Embedded DERP"
    stun_listen_addr: "0.0.0.0:3479"
  urls: []
  paths:
    - /etc/headscale/derp.yaml
```

#### 3. 更新防火墙策略与重建容器

服务端端口变更涉及物理网络的修改，需同步更新云服务器防火墙规则与 Docker 端口映射。

1. **配置安全组规则**：在云服务商控制台，添加允许入方向 `UDP 3479` 端口的放行规则，授权对象设置为 `0.0.0.0/0`。
2. **重建 Docker 容器**：删除旧容器，并使用绝对路径挂载及新的 UDP 端口映射重新启动：

```bash
docker rm -f headscale

docker run -d \
  --name headscale \
  --restart always \
  -v /root/headscale/config:/etc/headscale \
  -v /root/headscale/data:/var/lib/headscale \
  -p 8080:8080 \
  -p 3479:3479/udp \
  ghcr.io/juanfont/headscale:latest \
  serve
```

#### 4. 客户端缓存清理与重连验证

服务端修改完成后，必须强制清理客户端的本地路由缓存，否则客户端将因使用错误旧端口导致连接失败（表现为 `unknown`）。

在 Windows 系统中，使用管理员权限运行命令提示符（CMD），依次执行以下命令：

```powershell
# 注销当前会话
tailscale logout

# 重启后台服务，清空内存路由状态
net stop Tailscale
net start Tailscale

# 携带强制认证参数请求新的节点地图
tailscale up --login-server=[https://head.emolu.cn:8081](https://head.emolu.cn:8081) --authkey=YOUR_API_KEY --force-reauth
```

连接建立后，执行网络状态检查：

```powershell
tailscale netcheck
```

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

## 7. 虚拟机部署 Web 可视化面板 (Headscale-UI) 及跨域修复

为了彻底告别命令行，我们可以部署官方社区推荐的 `headscale-ui` 面板。由于 Headscale 后端自身存在对 `OPTIONS` 跨域预检请求处理缺陷（导致 401 报错），我们需要引入一个轻量级的 Caddy 容器作为反向代理桥梁。

### 7.1 申请长期 API Key

在服务端（运行 Headscale 的宿主机）终端生成一个给 Web 面板专用的控制钥匙：

```Bash
# 生成一个有效期为 365 天的 API Key（运行后请务必复制保存打印出的那串字符）
docker exec headscale headscale apikeys create -e 365d
```

### 7.2 虚拟机部署 Caddy 反向代理 (跨域修复桥梁)

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

### 7.3 部署 Headscale-UI 前端容器

UI 容器内部服务运行在 `8080` 端口，为避免与宿主机上的 Headscale 冲突，我们将其映射到宿主机的 `8081` 端口：

```bash
docker run -d \
  --name headscale-ui \
  --restart always \
  -p 8082:8080 \
  ghcr.io/gurucomputing/headscale-ui:latest
```

### 7.4 浏览器登录与绑定配置

1. 打开浏览器，访问前端 UI 页面：`http://<服务端IP>:8081`。
2. 点击页面中的 **Settings (设置)**。
3. 填写以下两项关键信息：
   - **Server URL**: `http://<服务端IP>:9090` *(⚠️ 注意：这里的端口是 Caddy 桥梁的 9090，且末尾切勿带有斜杠 `/`)*
   - **API Key**: 填入在 7.1 步骤中生成的长串密钥。
4. 点击 **Save**，页面即可瞬间获取后端的 Nodes 和 Users 数据，可视化管理配置完成！

## 8 云上部署headscale-ui 和 Caddy 反向代理

本节将介绍如何通过 Caddy 反向代理，将 Headscale 后端大脑与 Headscale-UI 前端面板完美融合在**同一个域名和端口**下。此架构能彻底消除前后端分离带来的 CORS 跨域问题，并最大限度收敛公网暴露端口。

### 8.1 架构概览

* **公网入口 (Caddy)**：监听 `8081` 端口，负责 HTTPS 卸载与流量智能分发。
* **后端大脑 (Headscale)**：监听内网 `8080` 端口，处理机器注册与核心 API。
* **前端面板 (Headscale-UI)**：监听内网 `8082` 端口，提供纯静态的 Web 图形界面。

### 8.2 拉取并部署 Headscale-UI 容器

确保您的 Headscale 服务已在 `8080` 端口正常运行后，使用 Docker 启动 UI 容器，并将其映射到宿主机的 `8082` 端口。这里绑定 `127.0.0.1` 是为了防止 UI 界面直接暴露在公网，强制所有流量必须经过 Caddy 洗礼：

```bash
docker run -d \
  --name headscale-ui \
  --restart always \
  -p 127.0.0.1:8082:80 \
  ghcr.io/gurucomputing/headscale-ui:latest
```

## 8.3 配置 Caddyfile (单端口路由分发)

这是整套架构的灵魂所在。打开您的 `Caddyfile` 配置文件，写入以下内容。Caddy 将通过“路径识别（Path）”来决定流量去向：

```
head.emolu.cn:8081 {
    # 1. 自动 HTTPS 证书配置 (使用 Cloudflare DNS 验证)
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }

    # 2. 贴心小魔法：访问根目录自动跳转到 UI 面板
    handle_path / {
        redir /web/
    }

    # 3. 前端 UI 通道
    # 匹配规则：所有以 /web/ 开头的请求，一律发给 8082 的 UI 容器
    handle /web/* {
        reverse_proxy 127.0.0.1:8082
    }

    # 4. 后端大脑通道 (终极兜底)
    # 匹配规则：除了 UI 网页外，剩下所有的请求（包括客户端打洞、设备注册、API 调用）
    # 全部无条件交给 8080 端口的 Headscale 核心
    handle {
        reverse_proxy 127.0.0.1:8080
    }
}
```

配置完成后，重载 Caddy 服务使新规则生效：

```Bash
# 如果 Caddy 是通过 Docker 运行：
docker restart caddy

# 如果 Caddy 是宿主机直装：
sudo systemctl reload caddy
sudo systemctl status caddy
```

## 8.4 获取管理凭证并登录 Web 界面

Headscale-UI 采用纯前端渲染，数据安全不落地，登录必须使用底层生成的 API Key 进行鉴权。

1. **生成 API Key**： 在云主机终端执行以下命令，生成一串专属密钥：

   ```bash
   # 如果 Headscale 是容器运行，请使用以下命令：
   docker exec -it headscale headscale apikeys create
   
   # 如果是宿主机直装，直接执行：
   headscale apikeys create
   ```

   *注意：请务必复制这串极长的随机字符串，它仅在生成时显示一次。*

2. **登录面板并绑定**：

   - 打开浏览器，访问：`https://head.emolu.cn:8081/web/`
   - 在弹出的 Settings (设置) 页面中进行如下配置：
     - **Headscale URL**: 填入 `https://head.emolu.cn:8081` *(切记：这里是主网关 API 地址，结尾不要带 /web 也不要填错端口)*
     - **Headscale API Key**: 粘贴刚才生成的密钥。
   - 点击 **Save (保存)**。

配置无误后，您将成功进入主控制台并看到 Devices (设备列表)，此时即可在纯图形化界面中高效管理您的私有虚拟局域网。

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

## 9. 联动飞书 OIDC 与 Casdoor 认证联邦网关部署实战

## 9.1 核心阻碍与架构溯源

- **核心阻碍**：Headscale 底层依赖极其严格的标准 OIDC 库（强制要求 `/.well-known/openid-configuration` 发现端点），而免费版飞书开放平台提供的 API 为非标准魔改版本。若直接对接，Headscale 会因解析元数据失败而在启动时 Crash。
- **破局链路**：引入 **Casdoor** 作为身份联邦网关（Identity Broker）。架构变更为：`Tailscale 客户端 -> Headscale (严格标准 OIDC) -> Casdoor (协议翻译官) -> 飞书开放平台 (非标鉴权)`。

## 9.2 基础设施编排 (环境变量隔离与镜像加速)

采用 Docker Compose 声明式编排拉起 Casdoor 与 MySQL，并强制剥离敏感凭证。

**1. 创建项目目录栈与沙盒环境变量**

```Bash
mkdir -p /root/casdoor/conf
mkdir -p /root/casdoor/db_data
cd /root/casdoor

# 创建并锁定环境变量文件
nano .env
```

在 `.env` 中写入（替换为你的强密码）：

```Ini, TOML
CASDOOR_DB_PASSWORD=RootPassword_强密码
```

保存后修改权限：`chmod 600 .env`

**2. 编写 `docker-compose.yaml`**

*注：因国内公共镜像源阻断，此处已强制硬编码注入 `docker.m.daocloud.io` 加速前缀，并剔除过时的 `version` 声明。*

```YAML
services:
  db:
    image: docker.m.daocloud.io/library/mysql:8.0
    container_name: casdoor-mysql
    restart: always
    environment:
      # 动态读取 .env 防止明文泄露
      MYSQL_ROOT_PASSWORD: ${CASDOOR_DB_PASSWORD}
      MYSQL_DATABASE: casdoor
    volumes:
      - ./db_data:/var/lib/mysql
    command: --default-authentication-plugin=mysql_native_password

  casdoor:
    image: docker.m.daocloud.io/casbin/casdoor:latest
    container_name: casdoor
    restart: always
    ports:
      - "127.0.0.1:8000:8000"
    environment:
      RUN_MODE: dev
    volumes:
      - ./conf/app.conf:/conf/app.conf
    depends_on:
      - db
```

## 9.3 核心配置注入 (`app.conf` DSN 词法闭环)

创建 Casdoor 的核心配置文件：

```bash
nano /root/casdoor/conf/app.conf
```

写入以下参数。**致命级避坑提示**：`dataSourceName` 末尾**必须保留斜杠 `/`**，绝不能包含数据库名，否则底层 XORM 引擎二次拼接时会引发 `Unknown database 'casdoorcasdoor'` 崩溃！

```Ini, TOML
appname = casdoor
httpport = 8000
runmode = dev
SessionOn = true
copyrequestbody = true
driverName = mysql
# 【关键】密码必须与 .env 一致；域名使用 db 利用内部 DNS；末尾强制保留 /
dataSourceName = root:RootPassword_强密码@tcp(db:3306)/
dbName = casdoor

# 【关键】OIDC 发现接口的绝对公网地址
origin = "https://auth.emolu.cn:8081"
```

拉起协议栈：

```Bash
cd /root/casdoor
docker compose up -d
```

## 9.4 L7 反向代理网关重构 (极简透传)

针对同源架构，剔除冗余的 CORS 跨域拦截，使用纯透传逻辑重构 `Caddyfile`：

```
# 1. Headscale 核心与 UI 面板
head.emolu.cn:8081 {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }
    handle_path / {
        redir /web/
    }
    handle /web/* {
        reverse_proxy 127.0.0.1:8082
    }
    handle {
        reverse_proxy 127.0.0.1:8080
    }
}

# 2. Casdoor 认证联邦网关
auth.emolu.cn:8081 {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }
    # 极简透传，交由 Casdoor 底层引擎处理跨域
    reverse_proxy 127.0.0.1:8000
}
```

重载网关：`docker restart caddy`或`systemctl restart caddy`

> **⚠️ 网络边界排障警告 (Cloudflare 阻断)**：
>
> Cloudflare 的 CDN 代理（橙色云朵）默认封杀非标端口 `8081`。如果浏览器报 `ERR_CONNECTION_CLOSED`，必须登录 CF 后台，将 `auth.emolu.cn` 与 `head.emolu.cn` 的代理状态**强制改为“仅 DNS (灰色云朵)”**。

## 9.5 Casdoor 控制面联邦握手配置

访问 `https://auth.emolu.cn:8081`，默认账号 `admin`，密码 `123`。**登入后第一时间在“用户(Users)”列表中修改 `admin` 的默认弱口令，以防权限被全网爆破。**

**Step 1: 握手上游 (对接飞书)**

- 左侧菜单 $\rightarrow$ 身份提供商 (Providers) $\rightarrow$ 添加。
- 类型选择 **`Feishu`**，填入飞书应用的 `Client ID` 和 `Client Secret`。
- 将 Casdoor 生成的 **回调 URL** 完整复制，填入飞书开放平台后台的“安全设置 -> 重定向 URL”中。

**Step 2: 握手下游 (对接 Headscale)**

- 左侧菜单 $\rightarrow$ 应用 (Applications) $\rightarrow$ 添加。
- 提供商 (Providers) 勾选刚建的 `Feishu-SSO`，**取消勾选内置 Password**。
- 重定向 URL 强制填入：`https://head.emolu.cn:8081/oidc/callback`
- **保存此应用顶部生成的 `Client ID` 和 `Client Secret`，以备下一步注入 Headscale 使用。**

## 10. 身份联邦中枢：Casdoor 与飞书 SSO 深度集成实战

### 10.1 双重握手与租户隔离架构

将非标的飞书 OAuth 接入严格的 Headscale OIDC 引擎，Casdoor 在此扮演了“协议翻译官”与“权限防火墙”的双重角色。整个数据链路被严格切割为两次独立握手，且受到底层租户沙盒的物理隔离：

1. **获取身份 (上游)**：Casdoor (提供商) $\leftarrow$ 飞书开放平台 (身份源)
2. **下发凭证 (下游)**：Headscale (消费者) $\leftarrow$ Casdoor (应用)

------

### 10.2 飞书企业自建应用初始化 (身份源基座)

> **⚠️ 跨租户物理阻断警告**：
>
> 飞书底层实行极其严格的 Tenant（租户/企业团队）隔离。**个人账号之间无法直接共享自建应用。**
>
> 必须在飞书客户端通过“邀请成员”，将需要连入 VPN 的朋友拉入你所在的同一个“飞书团队”内，否则对方将面临 `你没有使用权限` 的绝对物理阻断。

1. 登录 **飞书开放平台开发者后台** (open.feishu.cn)，创建“企业自建应用”。
2. 进入 **凭证与基础信息**，记录下 `App ID` (对应 Client ID) 和 `App Secret` (对应 Client Secret)。
3. 进入 **权限管理**，申请开通获取用户基本信息相关的最小化权限。

------

### 10.3 租户权限隔离沙盒构建 (核心防御闭环)

> **⚠️ 越权提权熔断防御**：
>
> 绝不可将业务应用挂载于 Casdoor 默认的 `built-in` 组织下。飞书用户首次登录时会触发自动注册，若处于 `built-in` 组织，系统会判定其试图获取全局超级管理员权限并执行物理熔断报错。

1. 登录 Casdoor 后台，导航至 **身份认证 (Identity)** $\rightarrow$ **组织 (Organizations)**。
2. 添加全新的平民沙盒组织：
   - **名称 (Name)**：`vpn-users`
   - **显示名称 (Display name)**：`VPN 接入网络`
3. 保存创建，确立没有任何后台特权的业务隔离区。

------

### 10.4 握手上游：创建 Casdoor 提供商 (Provider)

此步骤旨在告诉 Casdoor 去哪里核验飞书用户的身份。

1. 导航至 **身份认证 (Identity)** $\rightarrow$ **提供商 (Providers)** $\rightarrow$ 添加。
2. 触发级联菜单：
   - **分类 (Category)**：强制选择 `OAuth`。
   - **类型 (Type)**：选择 `Lark` (或 `Feishu`)。
3. 注入飞书凭证：填入刚才获取的飞书 `App ID` 和 `App Secret`。
4. **企业级零信任管控策略 (ACL Matrix)**：
   - **可用于注册 (Can sign up)**：**开启** (允许飞书用户首次免密建档)。
   - **可用于登录 (Can sign in)**：**开启** (渲染登录入口)。
   - **可解绑定 (Can unbind)**：**强制关闭** (防止离职员工解绑飞书后，利用 Casdoor 残留账号继续潜入 VPN，实现身份终身物理锚定)。
5. **获取上游收货地址**：保存后，滚动至页面最底部，复制灰色的 **回调 URL (Callback URL)** (例如：`https://auth.emolu.cn:8081/callback`，具体以你面板显示为准，注意是否带有 `/api` 前缀)。

------

### 10.5 握手下游：创建 Casdoor 应用 (Application)

此步骤旨在将标准化的 OIDC 凭证暴露给 Headscale 大脑。

1. 导航至 **身份认证 (Identity)** $\rightarrow$ **应用 (Applications)** $\rightarrow$ 添加。
2. 基础信息配置：
   - **名称 (Name)**：`headscale`
   - **组织 (Organization)**：**强制选择刚才创建的 `vpn-users`** (完成权限降级)。
3. 提供商挂载：
   - 在提供商列表中，添加刚才建好的 `Lark`。
   - **强制删除/取消勾选内置的 `built-in` 密码提供商**，切断所有本地弱口令绕过通道。
4. **下游收货地址注入**：
   - **重定向 URL (Redirect URLs)**：强制填入 Headscale 的绝对回调地址：`https://head.emolu.cn:8081/oidc/callback`。
5. **提取最终凭证**：保存后，在页面顶部复制 Casdoor 为该应用生成的专属 **`客户端 ID (Client ID)`** 和 **`客户端密钥 (Client Secret)`**，准备写入 Headscale 的 `config.yaml`。

------

### 10.6 飞书端点覆盖与可用域发布 (最终闭环)

携带从 10.4 步骤获取的 Casdoor 上游回调地址，返回飞书开放平台完成最后拼图：

1. **词法级 URL 对齐**：

   进入应用-**安全设置** -**重定向 URL**，填入 Casdoor 底部的确切地址（如 `https://auth.emolu.cn:8081/callback`）。**此项若差一个斜杠，均会导致 `20029` 报错熔断。**

2. **扩容应用可用范围 (解除授权拦截)**：

   进入 **版本管理与发布** $\rightarrow$ 创建新版本 $\rightarrow$ 将 **可用范围** 修改为“部分员工”或“全部员工”（确保已将目标用户拉入该飞书团队内）。提交发布并等待生效。

------

### 10.7 批判评估与排障测度矩阵

如果在 `tailscale up` 唤起浏览器后遭遇拦截，请严格对照下表进行物理定界：

| **故障表象 / 错误码**              | **逻辑置信度** | **底层定界与破局点**                                         |
| ---------------------------------- | -------------- | ------------------------------------------------------------ |
| **飞书界面报错 `20029`**           | **绝对**       | **上游回调 URL 词法不匹配**。飞书后台填写的重定向地址，与 Casdoor Provider 页面底部生成的 URL 在字符层面不完全一致（漏了端口或路径不对）。 |
| **向 built-in 组织添加用户被禁用** | **绝对**       | **越权拦截触发**。Casdoor 的应用（Application）错误地挂载在了 `built-in` 组织下，请立刻将其移动到无特权的 `vpn-users` 组织。 |
| **飞书提示：你没有使用权限**       | **极高**       | **跨租户或可用域阻断**。发起请求的个人账号不在该应用的飞书白名单内。需将该用户拉入飞书团队，并在开放平台后台发布新版本扩大“可用范围”。 |
| **Headscale 容器无限重启**         | **高**         | **OIDC 字段废弃**。`config.yaml` 中包含旧版本语法（如 `strip_email_domain: true`），或 `issuer` 末尾多写了 `/`。需检查日志排除 FATAL 报错。 |

### 10.8 tailscale客户端跳转配置

该配置可使tailscale自动跳转到单点登录界面。

```powershell
# 1. 物理级屠戮：强制杀死前端托盘进程 (解决缓存污染的绝对核心)
Get-Process tailscale-ipn -ErrorAction SilentlyContinue | Stop-Process -Force

# 2. 停止底层路由服务
Stop-Service Tailscale -Force

# 3. 确保注册表劫持处于绝对正确状态 (覆盖写入)
New-ItemProperty -Path 'HKLM:\SOFTWARE\Tailscale IPN' -Name 'LoginURL' -PropertyType String -Value 'https://head.emolu.cn:8081' -Force

# 4. 唤醒系统底层服务
Start-Service Tailscale
```

