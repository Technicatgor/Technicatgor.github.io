---
layout: post
title: Migrate SQLite to Cloudflare D1 & Flask to FastAPI
date: 2026-08-19 16:13:00 +800
categories: [DevOps,D1]
tags: [d1,cloudflare,fastapi,flask,sqlite]
image:
  path: /assets/img/headers/d1-migration-cover.png
---

將本地 SQLite `inventory.db`（IT 資產管理系統）遷移到 Cloudflare D1，並將後端由 Flask 重寫做 FastAPI。

## D1 限制速查

| 限制 | Workers Free | Workers Paid |
|---|---|---|
| Databases per account | 10 | 50,000 |
| 單個 database 大小 | 500 MB | 10 GB（hard limit，唔可以加） |
| 每個 account 總儲存 | 5 GB | 1 TB |
| Rows read | 5M / 日 | 25B / 月包底，之後 $0.001 / M |
| Rows written | 100K / 日 | 50M / 月包底，之後 $1.00 / M |
| 每次 Worker invocation 查詢數 | 50 | 1,000 |
| Time Travel 還原窗口 | 7 日 | 30 日 |

其他 SQL 限制：最多 100 columns / table、row 上限 2 MB、SQL statement 上限 100 KB、bound parameters 上限 100、查詢最長 30 秒。

> [!tip] 設計取向
> D1 唔係用嚟做一個大 DB，而係 horizontal scale-out（例如 per-tenant 一個 DB）。Database 數量唔收錢，只計 reads / writes / storage。

## 遷移流程（.db → D1）

```bash
# 1. 建立 D1 database
npx wrangler login
npx wrangler d1 create my-app-db

# 2. Dump 本地 .db 做 SQL
sqlite3 inventory.db .dump > dump.sql

# 3. 清理 D1 唔支援嘅語句
sed -i '' '/^PRAGMA /d; /sqlite_sequence/d' dump.sql   # macOS
sed -i '/^PRAGMA /d; /sqlite_sequence/d' dump.sql      # Linux

# 4. 先本地試行，冇問題先上 remote
npx wrangler d1 execute <db名> --local --file=dump.sql
npx wrangler d1 execute <db名> --remote --file=dump.sql

# 5. 驗證
npx wrangler d1 execute <db名> --remote --command "SELECT COUNT(*) FROM assets"
```

> [!warning] Import 失敗後要清表先好重試
> Wrangler 逐句執行，中間 error 會停低，但之前成功嘅 rows 已經入咗。直接重跑會撞 PRIMARY KEY / UNIQUE 衝突：
> ```bash
> npx wrangler d1 execute <db名> --remote --command "DROP TABLE IF EXISTS assets"
> ```

## 我嘅 dump.sql 實際遇到嘅問題

| 問題 | 嚴重性 | 解法 |
|---|---|---|
| 尾行 `INSERT INTO sqlite_sequence` | 🔴 必死 | 刪除（AUTOINCREMENT 會自動跟 max id 更新） |
| 首行 `PRAGMA foreign_keys=OFF` | 🔴 建議刪 | 單 table 冇 FK，直接刪 |
| id=44（NOTE163）有 20 個值 vs 19 columns | 🔴 Import 報錯 | 刪走多餘嘅 `''`；係 export 工具整出嚟嘅，唔係原裝 data |
| Row 66 用 `unistr()` | 🟡 相容性 | 改用 `'...' \|\| char(10) \|\| '...'` 寫法，邊度都行到 |
| id 29 / 57 / 87 / 115 有可疑 `... ` 值 | 🟡 待核實 | 同原裝 .db 對照呢 4 行 |

最終修正版：`dump_d1_ready_v2.sql`，130 行 data 全部驗證通過，AUTOINCREMENT 正確接續（下一個 id = 138）。

## D1 備份方案

D1 冇 direct connection，備份要靠 REST API export：

```
POST /accounts/{account_id}/d1/database/{database_id}/export
Body: {"output_format": "polling"}
→ 不斷 poll（唔 poll 會自動 cancel）
→ status=complete 後拎 signed_url（1 小時內有效）下載 .sql
```

> [!warning] Export 期間 DB 會鎖住
> Export 進行時 D1 唔可以serve query，要安排喺低流量時段行。

可以用 FastAPI + APScheduler 每日定時觸發，或者更簡單用 GitHub Actions cron 行 `wrangler d1 export <db> --remote --output backup.sql`。

## Python 後端接 D1

核心做法：寫一個 `d1_client.py` shim，扮晒 `sqlite3` 嘅 API，入面其實係 call Cloudflare REST API `/query` endpoint。咁樣成個 app 嘅 query code 唔使改。

```python
# d1_client.py 核心概念
POST https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/d1/database/{DATABASE_ID}/query
Headers: Authorization: Bearer ***
Body: {"sql": "SELECT ... WHERE id = ?", "params": [123]}
# 回傳 result[0].results（list of dict）+ result[0].meta（last_row_id, changes）
```

要 mimic 嘅 sqlite3 行為：

- `conn.execute(sql, params).fetchone() / .fetchall()`
- `row[0]`（位置）同 `row["column"]`（名稱）都用到 → 自製 `D1Row` class
- `cursor.lastrowid` ← `meta.last_row_id`
- `commit()` / `close()` 變 no-op（REST 係 stateless）

Setup：API token 權限要 **Account → D1 → Edit**，scope 落指定 account；database ID 用 `npx wrangler d1 list` 拎。

> [!note] Latency 考量
> 每個 query = 1 次 HTTP round-trip。Dashboard 一頁 8 個 query 大約 0.5–1.5s。之後可以用 `;` 隔開多個 statement 做 batch 優化。

## Flask → FastAPI 移植重點

完整版喺 `main.py`，所有 route 邏輯已用 mock D1 API 測試通過。

| Flask | FastAPI |
|---|---|
| `@app.route("/x", methods=[...])` | `@app.get/post/put/delete("/x")` |
| `render_template("x.html", **ctx)` | `Jinja2Templates(directory="templates")` + `TemplateResponse(request, "x.html", ctx)` |
| `request.args.get("q")` | 函數參數 `q: str = ""` |
| `request.form["x"]` | `await request.form()`（要裝 `python-multipart`） |
| `request.get_json()` | Pydantic `BaseModel` |
| `jsonify(...)` | 直接 return dict |
| `redirect(url_for(...))` | `RedirectResponse(url=..., status_code=303)` |
| `flash()` | Starlette `SessionMiddleware` + 自製 flash helper（要裝 `itsdangerous`） |
| `app.run(port=5001)` | `uvicorn main:app --host 0.0.0.0 --port 5001 --reload` |

> [!warning] Route 次序
> `/asset/new` 一定要 declare 喺 `/asset/{asset_id}` **之前**，否則 "new" 會被當 int 解析變 422。

Template 要手改：`url_for('assets_list')` → `/assets` 直接寫死路徑；`get_flashed_messages()` → 改用 context 入面嘅 `flashes` 變數。

Bonus：FastAPI 自動有 `/docs`（Swagger UI）可以即場試 API。

## 環境變數（.env）

```bash
pip install python-dotenv
```

```bash
# .env
CF_ACCOUNT_ID=...
CF_D1_DATABASE_ID=...
CF_D1_API_TOKEN=...
SECRET_KEY=隨機長字串
```

> [!danger] 次序陷阱
> `d1_client.py` 喺 import 嗰刻已經讀 `os.environ`，所以 `load_dotenv()` 必須喺 import 之前行。最穩陣係直接放喺 `d1_client.py` 最頂：
> ```python
> from dotenv import load_dotenv
> load_dotenv()  # 必須喺讀 os.environ 之前
> ```

- `load_dotenv()` 唔會覆寫已存在嘅 env var，production 用真環境變數時自動讓路
- 佢係由 working directory 開始搵 `.env`，唔係 .py 檔案位置
- 或者用 CLI 起 app 可以唔改 code：`uvicorn main:app --env-file .env`
- **`.env` 必須入 `.gitignore`**，token 有 D1 write 權限
- Docker 部署用 compose 嘅 `env_file:` 注入，唔好抄 `.env` 入 image

## 待辦事項

- [ ] 核對 dump 入面 4 行可疑值（id 29、57、87、115 嘅 `... `）同原裝 .db 有冇出入
- [ ] 核對 id=44（NOTE163）喺原裝 .db 嘅真實欄位值
- [ ] 改 ASUS 帳號密碼，將 credentials 搬離 asset table
- [ ] 正式 import：`DROP TABLE` → `dump_d1_ready_v2.sql` → `COUNT(*)` 應為 130
- [ ] 改 4 個 Jinja template（`url_for` → 路徑、`get_flashed_messages()` → `flashes`）
- [ ] Production 前將 CORS `allow_origins=["*"]` 收窄做 frontend origin
- [ ] 設定 D1 每日自動備份（GitHub Actions cron 或 FastAPI + APScheduler）

## 相關檔案

- `dump_d1_ready_v2.sql` — 清理後嘅 import 檔（130 rows）
- `d1_client.py` — sqlite3-compatible D1 REST API shim
- `main.py` — FastAPI 版後端（HTML + JSON API + Excel export）
