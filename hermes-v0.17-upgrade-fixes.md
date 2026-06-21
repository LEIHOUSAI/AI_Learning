# Hermes Agent v0.17.0 Upgrade Fixes

> Released 2026-06-19. If you upgraded to v0.17.0 and hit issues, this doc has you covered.

---

## Issue 1: Dashboard Refuses to Bind (ERR_CONNECTION_REFUSED)

### Symptom

```
Refusing to bind dashboard to 0.0.0.0 — the OAuth auth gate engages on non-loopback binds, but no auth providers are registered...
```

Dashboard port 9119 refuses all connections.

### Root Cause

v0.17.0 added a mandatory OAuth authentication gate for the dashboard. When bound to `0.0.0.0` without an auth provider configured, it refuses to start.

### Fix

Add this environment variable to your Docker Compose / run command:

```yaml
environment:
  - HERMES_DASHBOARD_INSECURE=1
```

Full compose example:
```yaml
services:
  hermes:
    image: nousresearch/hermes-agent:latest
    command: gateway start -p default
    environment:
      - HERMES_DASHBOARD=1
      - HERMES_DASHBOARD_INSECURE=1
```

⚠️ `HERMES_DASHBOARD_INSECURE=1` skips the auth gate. Only use on trusted networks (home LAN behind NAT is usually fine).

---

## Issue 2: Feishu / Lark Platform Broken After Upgrade

### Symptom

```
WARNING gateway.platform_registry: Platform 'Feishu / Lark' requirements not met (pip install 'hermes-agent[feishu]')
ERROR gateway.run: Platform 'feishu' is registered but adapter creation failed (check dependencies and config)
WARNING gateway.run: No adapter available for feishu
```

Feishu bot stops responding, and the `[Lark] connected` line never appears in logs.

### Root Cause

In v0.17.0, Hermes **moved all platform-specific dependencies out of the core package** into optional extras. This was an intentional architectural change for:

- Supply chain security (smaller core = smaller blast radius)
- Smaller Docker image size
- Faster installs

The Docker image no longer includes `lark-oapi` and related feishu/lark dependencies.

### Fix

#### Option A: One-time fix (lost on image update)

```bash
docker exec hermes /usr/local/bin/uv pip install --reinstall --python /opt/hermes/.venv/bin/python 'hermes-agent[feishu]'
```

Then restart the container.

#### Option B: Permanent fix (survives image updates)

1. Create the auto-install script on your NAS:

```bash
mkdir -p /root/.hermes/cont-init.d
cat > /root/.hermes/cont-init.d/99-feishu-deps.sh << 'EOF'
#!/bin/sh
for i in 1 2 3 4 5; do
  echo "Attempt $i: installing hermes-agent[feishu]..."
  /usr/local/bin/uv pip install --python /opt/hermes/.venv/bin/python 'hermes-agent[feishu]' 2>&1 && exit 0
  echo "Retrying in 5s..."
  sleep 5
done
echo "FAILED after 5 attempts"
exit 1
EOF
chmod +x /root/.hermes/cont-init.d/99-feishu-deps.sh
```

2. Mount it in your compose `volumes:`:

```yaml
volumes:
  - /root/.hermes:/opt/data
  - /root/.hermes/cont-init.d:/etc/cont-init.d:ro
```

This runs the install at container boot every time, so future image updates won't break feishu again.

---

## Working v0.17.0 Compose Reference

```yaml
services:
  hermes:
    image: nousresearch/hermes-agent:latest
    container_name: hermes
    restart: unless-stopped
    command: gateway start -p default
    ports:
      - 8642:8642  # gateway API
      - 9119:9119  # dashboard
    volumes:
      - /root/.hermes:/opt/data
      - /root/.hermes/cont-init.d:/etc/cont-init.d:ro
    environment:
      - HERMES_DASHBOARD=1
      - HERMES_DASHBOARD_INSECURE=1
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: "2.0"
```

---

## Verification

After applying both fixes, the startup logs should show:

```
HERMES_DASHBOARD_READY port=9119
  Hermes Web UI → http://0.0.0.0:9119
[Lark] [2026-06-21 ...] [INFO] connected to wss://msg-frontier.feishu.cn/...
```

No `Refusing to bind dashboard` errors. No `Platform 'Feishu / Lark' requirements not met` warnings.
