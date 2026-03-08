# OpenClaw 在 TrueNAS 上的完整安装与排坑记录

本文档基于实际部署过程，记录了在 TrueNAS 系统上通过 Dockge 安装 OpenClaw，并配置硅基流动（SiliconFlow）模型的全历程。涵盖了从初始部署到最终成功访问所遇到的各种问题及其解决方法，希望能为遇到类似问题的朋友提供参考。

---

## 1. 环境与目标

- **NAS 系统**：TrueNAS （IP: `192.168.1.100`）
- **容器管理**：Dockge（已预装，支持 `docker-compose`）
- **目标软件**：OpenClaw（AI 智能体网关） + 硅基流动大模型
- **最终访问方式**：通过 HTTPS（Caddy 反向代理）在局域网内安全访问

---

## 2. 第一阶段：部署 OpenClaw 与首次访问尝试

### 2.1 在 Dockge 中创建 OpenClaw 堆栈

**docker-compose.yml 初始内容**（后经多次调整）：

```yaml
version: '3.8'
services:
  openclaw:
    image: ghcr.io/openclaw/openclaw:latest
    container_name: openclaw-gateway
    restart: unless-stopped
    ports:
      - "18789:18789"
    volumes:
      - /mnt/pool/apps/openclaw/data:/home/node/.openclaw
    environment:
      - NODE_ENV=production
      - OPENCLAW_GATEWAY_TOKEN=lzw_treeNas_openclaw   # 自定义令牌
```

部署后，容器启动正常，但发现无法通过 `http://192.168.1.100:18789` 访问。

### 2.2 端口映射问题

**现象**：访问 `http://192.168.1.100:18789` 无响应。

**原因**：初始配置中 `ports` 使用了 `"127.0.0.1:18789:18789"`，导致服务只监听本机，未暴露给局域网。

**解决**：修改为 `"18789:18789"` 后，可以从局域网访问，但出现新错误。

---

## 3. 第二阶段：安全上下文与设备身份错误

### 3.1 错误：`control ui requires device identity (use HTTPS or localhost secure context)`

**现象**：访问 `http://192.168.1.100:18789` 时，浏览器提示连接失败，OpenClaw 日志显示：

```
[ws] closed before connect ... reason=control ui requires device identity (use HTTPS or localhost secure context)
```

**原因**：OpenClaw 控制界面强制要求安全上下文（Secure Context），即 HTTPS 或 localhost。通过纯 HTTP 访问局域网 IP 不被允许，无法生成设备身份。

### 3.2 临时绕过：SSH 隧道（localhost 访问）

**命令**（在本地电脑执行）：

```bash
ssh -L 8888:localhost:18789 root@192.168.1.100
```

保持该 SSH 会话开启，然后本地浏览器访问 `http://localhost:8888`。此时浏览器认为访问的是 localhost，属于安全上下文，可以继续。

### 3.3 首次登录与令牌输入

访问 `http://localhost:8888` 后，页面提示 `unauthorized: gateway token missing`。

**解决**：在 URL 后添加 `?token=lzw_treeNas_openclaw` 直接进入，或通过页面输入令牌。

### 3.4 设备配对（pairing required）

登录后遇到 `pairing required` 错误，OpenClaw 要求手动批准设备。

**查看待批准设备**：

```bash
docker exec -it openclaw-gateway openclaw devices list
```

**批准设备**：

```bash
docker exec -it openclaw-gateway openclaw devices approve <设备ID>
```

批准后，通过 SSH 隧道可以正常使用 OpenClaw，但直接 IP 访问问题依旧。

---

## 4. 第三阶段：配置硅基流动模型

### 4.1 编辑 openclaw.json

配置文件位于宿主机 `/mnt/pool/apps/openclaw/data/openclaw.json`。

添加硅基流动 provider：

```json
"models": {
  "providers": {
    "siliconflow": {
      "baseUrl": "https://api.siliconflow.cn/v1",
      "apiKey": "你的硅基流动 API Key",
      "api": "openai-completions",
      "models": [
        {
          "id": "deepseek-ai/DeepSeek-V3",
          "name": "DeepSeek-V3",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 200000,
          "maxTokens": 8192
        }
      ]
    }
  }
}
```

并设置默认模型 `"agents.defaults.model.primary": "siliconflow/deepseek-ai/DeepSeek-V3"`。

重启 OpenClaw 容器使配置生效：

```bash
docker restart openclaw-gateway
```

---

## 5. 第四阶段：配置 Caddy 反向代理（永久 HTTPS 方案）

为了彻底解决安全上下文问题，决定使用 Caddy 提供 HTTPS 反向代理。

### 5.1 第一次尝试：Caddy 堆栈配置

**docker-compose.yml**（初始）：

```yaml
version: '3.8'
services:
  caddy:
    image: caddy:latest
    container_name: caddy-openclaw
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
volumes:
  caddy_data:
  caddy_config:
```

**Caddyfile**：

```caddy
https://192.168.1.100:8443 {
    tls internal
    reverse_proxy 127.0.0.1:18789
}
```

### 5.2 问题：端口冲突

启动后 Caddy 报错：

```
Error: listening on :80: bind: address already in use
```

**原因**：`host` 网络模式下，Caddy 默认会监听 80 端口（用于自动 HTTPS 重定向），而 TrueNAS 自身占用了 80 端口。

**尝试解决**：添加 `auto_https off` 到 Caddyfile 全局块，但依然报错，因为即使关闭自动 HTTPS，Caddy 在 `host` 模式下仍试图绑定 80 端口。

### 5.3 第二次尝试：改用 bridge 网络并映射端口

修改 `docker-compose.yml`，移除 `network_mode: host`，添加端口映射：

```yaml
ports:
  - "8443:8443"
```

但 Caddyfile 中仍使用 `https://192.168.1.100:8443`，导致容器内无法绑定到宿主机 IP，Caddy 回退到监听 80 端口。

### 5.4 第三次尝试：修正 Caddyfile 监听地址

将 Caddyfile 改为：

```caddy
https://:8443 {
    tls internal
    reverse_proxy 192.168.1.100:18789
}
```

但发现容器内 `/etc/caddy/Caddyfile` 仍然是默认配置，说明挂载未生效。

### 5.5 问题：Caddyfile 挂载失败

检查宿主机文件：

```bash
ls -l /mnt/AppStore/Dockage/caddy/Caddyfile
```

发现 `Caddyfile` 是一个**目录**而非文件！原因是之前误创建了目录。删除目录并创建正确文件：

```bash
rm -rf /mnt/AppStore/Dockage/caddy/Caddyfile
nano /mnt/AppStore/Dockage/caddy/Caddyfile   # 写入正确配置
```

重新部署后，容器内 Caddyfile 内容正确，日志显示监听 `:8443`。

### 5.6 问题：TLS 内部错误

本地测试：

```bash
curl -k https://127.0.0.1:8443
```

返回错误 `SSL_ERROR_SYSCALL`，`openssl s_client` 显示 `tlsv1 alert internal error`。

**原因**：Caddy 的 `tls internal` 自动生成证书时可能与系统环境或容器权限有关，导致 TLS 握手失败。

### 5.7 最终解决：使用预先生成的自签名证书

**生成证书**：

```bash
mkdir -p /mnt/AppStore/Dockage/caddy/certs
cd /mnt/AppStore/Dockage/caddy/certs
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes -subj "/CN=192.168.1.100"
```

**修改 Caddyfile**：

```caddy
https://:8443 {
    tls /certs/cert.pem /certs/key.pem
    reverse_proxy 192.168.1.100:18789
}
```

**修改 docker-compose.yml**，挂载证书目录：

```yaml
volumes:
  - ./Caddyfile:/etc/caddy/Caddyfile
  - ./certs:/certs
  - caddy_data:/data
  - caddy_config:/config
```

重新部署后，`curl -k https://127.0.0.1:8443` 成功返回 OpenClaw HTML。

---

## 6. 第五阶段：最终访问与设备授权

### 6.1 浏览器访问 HTTPS

在局域网电脑上打开 `https://192.168.1.100:8443`，接受自签名证书警告（点击“高级”->“继续前往”）。

页面提示 `gateway token missing`。

**解决**：在 URL 后添加 `?token=lzw_treeNas_openclaw` 直接进入，或通过页面输入令牌。

### 6.2 再次遇到设备配对

登录后提示 `pairing required`。重复之前的设备批准步骤：

```bash
docker exec -it openclaw-gateway openclaw devices list
docker exec -it openclaw-gateway openclaw devices approve <设备ID>
```

刷新页面，成功进入 OpenClaw 主界面。

### 6.3 后续优化：允许来源（origin）

如果之后出现 WebSocket 连接错误 `origin not allowed`，需要在 `openclaw.json` 的 `gateway.controlUi.allowedOrigins` 中添加 `"https://192.168.1.100:8443"`，然后重启 OpenClaw。

---

## 7. 总结与常用命令

### 7.1 关键配置总结

- **OpenClaw 数据卷**：`/mnt/pool/apps/openclaw/data`，包含 `openclaw.json`
- **Caddy 配置目录**：`/mnt/AppStore/Dockage/caddy`，包含 `Caddyfile`、`certs/`、`docker-compose.yml`
- **网关令牌**：`lzw_treeNas_openclaw`（可自行修改）
- **硅基流动 API Key**：需从平台获取并填入 `openclaw.json`

### 7.2 常用命令

| 操作 | 命令 |
|------|------|
| 查看 OpenClaw 日志 | `docker logs openclaw-gateway` |
| 重启 OpenClaw | `docker restart openclaw-gateway` |
| 查看待批准设备 | `docker exec -it openclaw-gateway openclaw devices list` |
| 批准设备 | `docker exec -it openclaw-gateway openclaw devices approve <设备ID>` |
| 查看 Caddy 日志 | `docker logs caddy-openclaw` |
| 重启 Caddy | `docker restart caddy-openclaw` |
| 本地测试 HTTPS | `curl -k https://127.0.0.1:8443` |
| 重新部署 Caddy | `cd /mnt/AppStore/Dockage/caddy && docker-compose down && docker-compose up -d` |

### 7.3 最终访问地址

```
https://192.168.1.100:8443
```

使用令牌 `lzw_treeNas_openclaw` 登录。

---

## 8. 心得体会

- OpenClaw 对安全要求较高，强制 HTTPS 或 localhost 是为了保护设备身份，虽增加了部署复杂度，但提升了安全性。
- 使用 SSH 隧道是一种快速绕过安全上下文的调试手段，但长期使用仍需配置 HTTPS。
- Caddy 配置中，文件挂载路径必须准确，特别注意宿主机上不要出现同名目录。
- 自签名证书在家庭内网完全可用，只需在浏览器中接受一次风险提示。
- 设备配对是 OpenClaw 的额外安全层，必须手动批准新设备，确保只有授权设备能连接。

希望这份详细记录能帮助更多人顺利部署 OpenClaw！