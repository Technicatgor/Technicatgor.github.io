---
layout: post
title: Camofox Browser — Headless Browser for Dockerized Hermes Agent
date: 2026-08-28 18:00:00 +800
categories: [Docker,Browser,AIOps]
tags: [docker,camofox,hermes-agent,headless-browser,browser-automation]
image: /assets/camofox-banner.png
---

![banner](/assets/camofox-banner.png)

## 目標

使 Docker 容器內嘅 Hermes Agent 能夠使用 headless browser（navigate / click / snapshot / screenshot 等），無需在 host 安裝 Chrome。

## 架構

```
┌─────────────────┐        ┌──────────────────┐
│  hermes         │ ────→  │  camofox         │
│  nousresearch/  │ 9377   │  camofox-browser │
│  hermes-agent   │        │(Camoufox/Firefox)│
└─────────────────┘        └──────────────────┘
        │ mount                     │
   ~/.hermes → /opt/data     ~/.camofox-docker → /root/.camofox
        │                         │
   8642 Web UI              6080 noVNC / 5901 VNC（直播畫面）
```

兩個容器於同一個 compose network；9377 **無需 publish 至 host**，內部連線即可。

## 安裝步驟

### 1. Build Camofox image

```bash
git clone https://github.com/jo-inc/camofox-browser
cd camofox-browser
make build        # 自動偵測架構（aarch64 / x86_64），預先下載 Camoufox + yt-dlp
```

⚠️ 請勿直接執行 `docker build`——Dockerfile 依賴 bind mount 取得 `dist/` 內預先下載之 binary，必須透過 `make` 執行。

常用指令：`make up` / `make down` / `make reset` / `make fetch`

指定版本：`make up VERSION=135.0.1 RELEASE=beta.24`

### 2. docker-compose

```yaml
services:
  hermes:
    image: nousresearch/hermes-agent:latest
    volumes:
      - ~/.hermes:/opt/data
    environment:
      - CAMOFOX_URL=http://camofox:9377
      - CAMOFOX_API_KEY=*** camofox 嘅 ACCESS_KEY 相同>
    depends_on:
      - camofox

  camofox:
    image: camofox-browser:135.0.1-aarch64   # 請改為 make build 生成之實際 tag
    restart: unless-stopped
    ports:
      - "6080:6080"     # noVNC 網頁直播
      - "5901:5900"     # 原生 VNC client
      # 注意：9377 無需 publish
    environment:
      - CAMOFOX_PORT=9377
      - ENABLE_VNC=1
      - VNC_BIND=0.0.0.0
      - VNC_PASSWORD=***
      - MAX_OLD_SPACE_SIZE=2048          # 預設僅 128MB，務必調高
      - CAMOFOX_ACCESS_KEY=*** token> # 啟用 auth；純內網可省略
      - CAMOFOX_CRASH_REPORT_ENABLED=false
    volumes:
      - ~/.camofox-docker:/root/.camofox  # session profile 持久化
```

### 3. Hermes 設定

```bash
hermes config set toolsets '["hermes-cli", "browser"]'
hermes config set browser.cloud_provider camofox
# 或透過 wizard：hermes tools → Browser Automation → Camofox
```

最終 `~/.hermes/config.yaml`：

```yaml
toolsets:
  - hermes-cli
  - browser

browser:
  cloud_provider: camofox
  camofox:
    rewrite_loopback_urls: true          # 僅需使 Camofox 可存取 host 上之應用程式
    loopback_host_alias: host.docker.internal
    managed_persistence: true            # 跨 session 保留登入狀態
```

`~/.hermes/.env`：

```bash
CAMOFOX_URL=http://camofox:9377
CAMOFOX_API_KEY=*** camofox 嘅 CAMOFOX_ACCESS_KEY 相同>
```

修改任何 config 或 env 後，必須 **`docker restart <hermes容器>`**（gateway process 才會重新讀取配置）。

### 4. 驗證

```bash
# Camofox 健康檢查（/health 為唯一免 auth 之 route）
docker exec <hermes容器> curl -s http://camofox:9377/health
# 預期：{"ok":true,...}；idle 時 browserRunning:false 為正常行為（on-demand 啟動 browser）

# auth 測試（如有設定 key）
docker exec <hermes容器> curl -s -H "Authorization: Bearer ***" -X POST http://camofox:9377/tabs

# 實戰測試
hermes chat
> Navigate to https://example.com and tell me the heading

# Agent 操作期間再次 curl health，應見 browserRunning:true / activeSessions:1
# 開啟 VNC：http://localhost:6080 查看即時畫面
```

## 疑難排解紀錄（本次實際發生）

| 錯誤訊息 | 原因 | 解決方案 |
|---|---|---|
| `browser tool is not available (no Chrome running)` | Hermes 仍處於 local browser mode，官方 image 未內建 Chrome | 設定 `browser.cloud_provider: camofox`；僅設定 `CAMOFOX_URL` 不足以切換 |
| `web_extract` 報錯 port 3002 連線失敗 | 3002 = Firecrawl。配置中遺留 `FIRECRAWL_API_URL=http://localhost:3002`，但容器內 localhost 無任何服務 | 無 Firecrawl 則移除該 env var 及執行 `web.extract_backend ''`；有則使用 service 名稱 `http://firecrawl-api:3002` |
| Browser 導航返回 `401 Unauthorized` | Camofox 已設定 `CAMOFOX_ACCESS_KEY` 但 Hermes 未附帶 Bearer token | Hermes 端設定 `CAMOFOX_API_KEY`（名稱不同但值必須一致） |

## 注意事項總結

- **容器內之 localhost 指容器本身**：`CAMOFOX_URL`、`FIRECRAWL_API_URL` 等全部須使用 compose service 名稱，而非 `localhost`。
- **Auth 命名差異**：伺服器端變數名為 `CAMOFOX_ACCESS_KEY`，Hermes 客戶端為 `CAMOFOX_API_KEY`，兩者須設為相同值。
- **`/health` 免 auth**：curl 能連通僅代表網路暢通，不代表 auth 設定正確。
- **`cloud_provider` 必須明確設定**：否則將始終 fallback 至 local mode 尋找 Chrome。
- **修改配置後須重啟整個容器**：僅重啟 CLI 對 gateway（Web UI 8642 / Telegram）無效。
- **`hermes config set` 須寫入有 mount 之 volume**：使用 `--rm` 臨時容器執行 set 等同未設定。
- **舊版 Hermes Bug**：設定 `CAMOFOX_API_KEY` 後，部分 browser 操作未附帶 auth header 而返回 403（issue #20476），執行 `docker pull` 更新 image 即可解決。
- **Camofox 記憶體限制**：`MAX_OLD_SPACE_SIZE` 預設值 128MB，必須調高（建議 2048）。
- **配置巢狀位置**：`rewrite_loopback_urls`、`managed_persistence` 必須放置於 `browser.camofox` 區塊下，放置於 top-level 將被靜默忽略。

## 延伸應用

- Cookie 匯入（例如 X/Twitter 登入狀態）：使用 `CAMOFOX_API_KEY` 作為 Bearer token 呼叫 `POST /sessions/{userId}/cookies`，搭配 cookies.txt 瀏覽器擴展。
- 遭封鎖之網站：Camofox 支援 proxy 環境變數（`PROXY_HOST` 等），會自動依 exit IP 調整 locale 與 timezone。
- 其他 browser backend 選項：local agent-browser（`npm i -g agent-browser && agent-browser install --with-deps`，約 200MB，需嵌入自訂 image）、CDP `/browser connect`（僅限 interactive CLI）、Browserbase / Browser Use cloud。

## 參考資料

- Hermes Browser Automation 文件：https://hermes-agent.nousresearch.com/docs/user-guide/features/browser
- Hermes 環境變數參考：https://hermes-agent.nousresearch.com/docs/reference/environment-variables
- Camofox 專案：https://github.com/jo-inc/camofox-browser
- Docker image 未內建 Chrome 已知問題：hermes-agent issue #15697
- Camofox auth 403 Bug：hermes-agent issue #20476
