---
layout: post
title: Deploy Hermes Agent & Web UI with Docker Compose
date: 2026-08-19 16:30:00 +800
categories: [DevOps,Docker]
tags: [docker,hermes,agent,webui,compose,homelab]
image:
  path: /assets/img/headers/hermes-docker-compose-cover.png
---

用 Docker Compose 一站式部署 Hermes Agent 同 Hermes Web UI，適合 home server / VPS 長時間運行。

## 架構概覽

```
+------------------+     +------------------+
|  Hermes Agent    |     |  Hermes Web UI   |
|  (Gateway)       |     |  (Dashboard)     |
|  :8642 (API)     |     |  :8787 (UI)      |
+--------+---------+     +--------+---------+
         |                         |
         +----------+--------------+
                    |
            +-------v--------+
            |  ~/.hermes     |
            |  (persistent)  |
            |  config.yaml   |
            |  .env          |
            |  sessions/     |
            |  skills/       |
            |  memories/     |
            +----------------+
```

兩個 container：
- **hermes** — Hermes Agent gateway（跑 agent 邏輯）
- **hermes-webui** — Web dashboard（瀏覽器介面）

共用同一個 `~/.hermes` host directory，保持 config、memory、sessions 一致。

## 環境要求

- Docker 24+
- Docker Compose v2+
- 最少 2GB RAM（Agent 4GB limit）

## 步驟

### 1. 建立專案目錄

```bash
mkdir -p ~/hermes-docker && cd ~/hermes-docker
```

### 2. 設定 UID / GID

確認你嘅 user ID 同 group ID（通常係 1000）：

```bash
id
# uid=1000(heston) gid=1000(heston) ...
```

### 3. 建立 `.env` 檔案

```bash
# .env
HERMES_UID=1000
HERMES_GID=1000
```

> [!warning] 安全提醒
> `.env` 必須入 `.gitignore`。

### 4. 建立 `docker-compose.yml`

```yaml
services:
  hermes:
    image: nousresearch/hermes-agent:latest
    container_name: hermes
    restart: unless-stopped
    environment:
      - HERMES_UID=${HERMES_UID:-1000}
      - HERMES_GID=${HERMES_GID:-1000}
      - HERMES_WRITE_SAFE_ROOT=/opt/data:/workspace
    command: gateway run
    ports:
      - "8642:8642"
    volumes:
      - ~/.hermes:/opt/data
      - ~/hermes-workspace:/workspace
      - /var/run/docker.sock:/var/run/docker.sock
      - hermes-agent-src:/opt/hermes     # 新增:畀 WebUI 讀 agent source
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: "2.0"
    shm_size: 1g
    networks:
      - hermes-net

  hermes-webui:
    image: ghcr.io/nesquena/hermes-webui:latest
    container_name: hermes-webui
    restart: unless-stopped
    depends_on:
      - hermes
    ports:
      - "127.0.0.1:8787:8787"    # 要 LAN/Tailscale 存取先好拎走 127.0.0.1
    volumes:
      - ~/.hermes:/home/hermeswebui/.hermes # 同 agent 同一個 host dir
      - hermes-agent-src:/home/hermeswebui/.hermes/hermes-agent:ro
      - ~/hermes-workspace:/workspace
    environment:
      - WANTED_UID=${HERMES_UID:-1000}
      - WANTED_GID=${HERMES_GID:-1000}
      - HERMES_WEBUI_HOST=0.0.0.0
      - HERMES_WEBUI_PORT=8787
      - HERMES_WEBUI_STATE_DIR=/home/hermeswebui/.hermes/webui
      # - HERMES_WEBUI_PASSWORD=your-secret   # expose 出網必設
      # - HERMES_SKIP_CHMOD=1          # 若 host ~/.hermes/.env 係 0640
    networks:
      - hermes-net

networks:
  hermes-net:
    driver: bridge

volumes:
  hermes-agent-src:      # 首次 up 會自動從 image /opt/hermes 填入
```

### 5. 啟動

```bash
docker compose up -d
```

### 6. 驗證

```bash
# 檢查 container 狀態
docker compose ps

# 檢查 logs
docker compose logs -f hermes

# Smoke test
docker compose exec hermes hermes doctor

# 發送一個測試訊息
docker compose exec hermes hermes chat -q "Say hello"
```

## 目錄結構

```
~/hermes-docker/
├── docker-compose.yml
├── .env
├── .gitignore
├── ~/.hermes/              # host 端，container 掛載到 /opt/data
│   ├── config.yaml
│   ├── .env
│   ├── skills/
│   ├── memories/
│   ├── sessions/
│   └── logs/
└── ~/hermes-workspace/     # 工作區，agent + webui 共用
```

## 常見操作

### 查看 logs

```bash
docker compose logs -f hermes
```

### 更新版本

```bash
docker compose pull
docker compose up -d   # ~/.hermes 不變，只會重建 image
```

### 進入 shell

```bash
docker compose exec hermes sh
```

### 重啟 gateway

```bash
docker compose restart hermes
```

### 停止服務

```bash
docker compose down
```

## 關鍵設定說明

### `HERMES_WRITE_SAFE_ROOT`

將 Agent 嘅 safe write root 映射到你嘅 workspace，令到 container 內寫文件會落返 host `~/hermes-workspace/`。

### `/var/run/docker.sock`

畀 Hermes Agent 有權限控制 Docker（例如跑 `docker compose` 命令）。如果唔需要可移除。

### Web UI 綁定 `127.0.0.1`

`hermes-webui` 嘅 port 8787 綁定 `127.0.0.1`，即係只可從 server 本機訪問。如果要 LAN / Tailscale 外部訪問：

```yaml
ports:
  - "8787:8787"
```

並且建議設定密碼：

```yaml
environment:
  - HERMES_WEBUI_PASSWORD=your-secret
```

## LXC 注意事項

如果跑喺 Proxmox LXC 度，可能缺 `iptable_nat` kernel module 導致 port mapping 失效。解決方案：

**方案 A**：用 `network_mode: host` 代替 `ports:`

```yaml
services:
  hermes:
    network_mode: host
    # 去掉 'ports:' 配置
```

**方案 B**：用 Nginx reverse proxy 喺 host 度转发，container 只 listen `127.0.0.1:8642`

## 問題排查

| 問題 | 解法 |
|---|---|
| `hermes doctor` 報 provider 錯誤 | 檢查 `~/.hermes/.env` 入面 API key 是否正確 |
| Gateway 無法連接 | 確認 port 8642 無衝突，`docker compose logs` 睇 error |
| Web UI 打唔開 | 檢查 port 8787 是否被 bind，`127.0.0.1` 代表只能本機訪問 |
| Permission denied 寫 files | 檢查 UID/GID 是否同 host user 一致，`docker compose exec hermes id` |
| Volume `hermes-agent-src` 空 | 首次 build 會由 image 複製，`docker compose up --build` 重試 |
| Memory 不足 | 調整 `deploy.resources.limits.memory`，最少 2G |

## 相關連結

- [Hermes Agent 官方文檔](https://hermes-agent.nousresearch.com/docs/)
- [Environment Variables 完整列表](https://hermes-agent.nousresearch.com/docs/reference/environment-variables)
- [Provider 設定指南](https://hermes-agent.nousresearch.com/docs/integrations/providers)
