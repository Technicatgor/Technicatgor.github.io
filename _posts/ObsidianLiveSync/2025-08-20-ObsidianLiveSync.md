---
layout: post
title: Obsidian LiveSync
date: 2025-08-20 16:00:00 +800
categories: [Tool]
tags: [Obsidian]
image:
  path: /assets/img/headers/obd-livesync-cover.png
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---

### Deploy locally

1. Prepare

```bash
# Creating the save data & configuration directories.
mkdir couchdb-data
mkdir couchdb-etc
```

2. Create a `compose.yml` file with the following added to it

#### without proxy

```yaml
services:
  couchdb:
    image: couchdb:latest
    container_name: couchdb-for-ols
    environment:
      - COUCHDB_USER=<INSERT USERNAME HERE> #Please change as you like.
      - COUCHDB_PASSWORD=<INSERT PASSWORD HERE> #Please change as you like.
    volumes:
      - ./couchdb-data:/opt/couchdb/data
      - ./couchdb-etc:/opt/couchdb/etc/local.d
    ports:
      - 5984:5984
    restart: unless-stopped
```

#### with Traefik

```yaml
services:
  couchdb:
    image: couchdb:latest
    container_name: obsidian-livesync
    environment:
      - COUCHDB_USER=username
      - COUCHDB_PASSWORD=password
    volumes:
      - ./data:/opt/couchdb/data
      - ./local.ini:/opt/couchdb/etc/local.ini
    # Ports not needed when already passed to Traefik
    #ports:
    #  - 5984:5984
    restart: unless-stopped
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      # The Traefik Network
      - "traefik.docker.network=proxy"
      # Don't forget to replace 'obsidian-livesync.example.org' with your own domain
      - "traefik.http.routers.obsidian-livesync.rule=Host(`obsidian-livesync.example.org`)"
      # The 'websecure' entryPoint is basically your HTTPS entrypoint. Check the next code snippet if you are encountering problems only; you probably have a working traefik configuration if this is not your first container you are reverse proxying.
      - "traefik.http.routers.obsidian-livesync.entrypoints=websecure"
      - "traefik.http.routers.obsidian-livesync.service=obsidian-livesync"
      - "traefik.http.services.obsidian-livesync.loadbalancer.server.port=5984"
      - "traefik.http.routers.obsidian-livesync.tls=true"
      # Replace the string 'letsencrypt' with your own certificate resolver
      - "traefik.http.routers.obsidian-livesync.tls.certresolver=letsencrypt"
      - "traefik.http.routers.obsidian-livesync.middlewares=obsidiancors"
      # The part needed for CORS to work on Traefik 2.x starts here
      - "traefik.http.middlewares.obsidiancors.headers.accesscontrolallowmethods=GET,PUT,POST,HEAD,DELETE"
      - "traefik.http.middlewares.obsidiancors.headers.accesscontrolallowheaders=accept,authorization,content-type,origin,referer"
      - "traefik.http.middlewares.obsidiancors.headers.accesscontrolalloworiginlist=app://obsidian.md,capacitor://localhost,http://localhost"
      - "traefik.http.middlewares.obsidiancors.headers.accesscontrolmaxage=3600"
      - "traefik.http.middlewares.obsidiancors.headers.addvaryheader=true"
      - "traefik.http.middlewares.obsidiancors.headers.accessControlAllowCredentials=true"

networks:
  proxy:
    external: true
```

3. Run the Docker Compose file to boot check

```bash
docker compose up -d
```

### Go to CouchDB admin page

- Go to your server ip eg. `http://192.168.1.0:5984/_utils`, login with your credentials created in compose file.
  ![/assets/img/obd-01.png](/assets/img/obd-01.png)

- You will see your db in CouchDB
- Open Obsidian apps and browse the LiveSync plugin
  ![/assets/img/obd-02.png](/assets/img/obd-02.png)

- Click Option and setup your connection.
- Server URI, Username, Password, Database Name
  ![/assets/img/obd-03.png](/assets/img/obd-03.png)

- Test Database Connection, Encryption
  ![/assets/img/obd-04.png](/assets/img/obd-04.png)

- Synchronization Method
  i. Change Live Sync method
  ![/assets/img/obd-05.png](/assets/img/obd-05.png)

### Install Obsidian on mobile

- Create a new vault and install the LiveSync plugin and configure the connection. The same setting in desktop side, then click **Fetch** button Fetch Settings.
  ![/assets/img/obd-06.png](/assets/img/obd-06.png)

Done! It will sync the notes in live.

ref link: <https://github.com/vrtmrz/obsidian-livesync>

---

## 常見排錯紀錄

> 環境: CouchDB (Docker) 作為同步後端 + Obsidian Self-hosted LiveSync 插件，前面可掛 reverse proxy (Nginx / Apache / Caddy)。

### 問題一：CORS 錯誤

#### 錯誤訊息

```
the request was successful by API. But the native fetch API failed!
Please check CORS setting on the remote database!
While this condition, you cannot enable LiveSync
```

#### 原因

- 插件嘅 **Test** 按鈕走 Obsidian 內部 API，唔受瀏覽器 CORS 限制 → 所以會「測試成功」
- 實際同步 (PouchDB / LiveSync) 走原生 `fetch()`，受 CORS 限制 → 被擋
- Obsidian 嘅 origin 唔係標準網域：
  - 桌面版：`app://obsidian.md`
  - 手機版：`capacitor://localhost`
- 帶 credentials 嘅 request **唔可以用 wildcard `*`**，必須明確列出 origin

#### 解法 A：CouchDB `local.ini` 加 CORS 設定

```ini
[couchdb]
single_node = true
max_document_size = 50000000

[chttpd]
enable_cors = true
bind_address = 0.0.0.0
max_http_request_size = 4294967296
require_valid_user = true

[cors]
origins = app://obsidian.md, capacitor://localhost, http://localhost
credentials = true

[httpd]
WWW-Authenticate = Basic realm="couchdb"
```

改完重啟 CouchDB：

```bash
docker compose restart
```

#### 解法 B：由 reverse proxy 處理（與 CouchDB 二擇一）

> ⚠️ **CouchDB 同 proxy 只能由其中一層加 CORS header**，兩層都加會造成 `Access-Control-Allow-Origin` 重複，瀏覽器照樣拒絕。

**Nginx** 範例（此時 CouchDB 端 `enable_cors = false`）：

```nginx
add_header 'Access-Control-Allow-Methods' 'GET, PUT, POST, HEAD, DELETE';
add_header 'Access-Control-Allow-Headers' 'accept, authorization, content-type, origin, referer';
add_header 'Access-Control-Allow-Origin' 'app://obsidian.md, capacitor://localhost, http://localhost';
add_header 'Access-Control-Allow-Credentials' 'true';
add_header 'Vary' 'Origin';
```

```bash
nginx -t && systemctl reload nginx
```

#### 插件內建修復工具

- Settings → Self-hosted LiveSync → Setup → **Check database configuration** → 有問題按 **Fix**
- 新版精靈有 **Detect and Fix CouchDB Issues** 按鈕，可自動補齊 CORS / chttpd 設定

#### 最後手段（繞過 CORS 檢查）

- Settings → 開啟 power user 功能 → Power users → **Use Internal API**
- 走 Obsidian 自己嘅 request handler，完全繞過瀏覽器 CORS
- 定位：驗證帳密/DB 是否正確嘅臨時手段，唔建議長期使用

---

### 問題二：413 Request Entity Too Large

#### 錯誤訊息

```
the request may have failed the reason sent by the server
413 Request Entity Too Large
```

#### 原因

大型筆記 / 嵌入圖片 / 附件喺同步推送時，超過 **CouchDB 或 reverse proxy 嘅 request body 上限**（兩層各自獨立限制，任何一層超標都會 413）。

#### 解法 1：CouchDB 端（兩個設定都要調）

```ini
[couchdb]
max_document_size = 50000000

[chttpd]
max_http_request_size = 4294967296
```

- `max_document_size`：單一文件 JSON body 上限（唔含附件）
- `max_http_request_size`：整個 HTTP request body 上限（含附件）← 同步大檔最容易撞呢個

改完重啟：

```bash
docker compose restart
```

#### 解法 2：Nginx 端

```nginx
client_max_body_size 100M;
```

```bash
nginx -t && systemctl reload nginx
```

#### 解法 3：Apache 端

```apache
LimitRequestBody 104857600
```

```bash
systemctl restart apache2
```

#### 注意事項

- **Cloudflare free tier 有 100MB request body 硬上限**，無法調高；經 Cloudflare Tunnel / proxy 嘅話，單檔超過 100MB 點改本機設定都冇用
- Caddy 預設無 body size 上限，用 Caddy 嘅話優先檢查 CouchDB 端
- 插件端輔助：Settings → Sync Settings → **chunk size 調低至 50–100**，讓單次複寫嘅 payload 變細，從源頭避免撞上限

---

### 標準驗證流程（每次改完設定）

1. 重啟 CouchDB / reload proxy
2. Obsidian 插件內：**Test → Check database configuration → Apply**
3. 重新啟用 LiveSync
4. 觀察 sync status 回到 `↑ 0 ↓ 0`

---

### 排錯時間線總結

| 階段 | 症狀 | 根因 | 解法 |
|------|------|------|------|
| 部署 | — | — | Docker Compose 起 CouchDB + 插件連線 |
| 問題 1 | API 測試成功但 native fetch 失敗 | Obsidian 非標準 origin 被 CORS 擋；credentialed request 唔可以用 `*` | `local.ini` 明確列出 origins，或只由 proxy 單層處理 CORS |
| 問題 2 | 413 Request Entity Too Large | 文件/附件超過 CouchDB 或 proxy 嘅 body size 上限 | 調高 `max_http_request_size` / `client_max_body_size`，插件 chunk size 調低 |

---

### 常用檢查指令速查

```bash
# CouchDB 容器
docker compose up -d              # 啟動
docker compose restart            # 改完 local.ini 後重啟
docker compose logs -f            # 睇即時 log 排查連線問題

# 驗證 CouchDB 是否正常回應
curl -u admin_user:admin_password http://<server>:5984/_up

# 查看目前 CORS 設定
curl -u admin_user:admin_password http://<server>:5984/_node/_local/_config/cors

# Nginx
nginx -t                          # 測試設定語法
systemctl reload nginx            # 套用設定

# Apache
systemctl restart apache2
```
