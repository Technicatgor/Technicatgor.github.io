---
layout: post
title: /grill-me 實用教學 — 讓 AI 動手前先拷問你
date: 2026-08-25 17:53:00 +800
categories: [DevOps,AI]
tags: [ai-agent,claude-code,cursor,codex,prompt]
image:
  path: /assets/img/headers/grill-me-cover.png
---

適用對象：使用 Claude Code / Cursor / Codex 等 AI coding agent，厭倦「寫一版、改一版」返工循環的人。

**核心一句話**：把方向反轉 —— 不是你餵需求給 AI，而是 AI 沿著決策樹一題一題拷問你，直到雙方對計劃有共識才准動手。

---

## 一、/grill-me 是什麼

由 Matt Pocock（TypeScript 教育家）開源，收在 [mattpocock/skills](https://github.com/mattpocock/skills) 倉庫。

**解決的痛點**：需求「方向明確、但邊界未對齊」時，agent 直接開工，寫到一半才發現全是預設假設 → 返工、燒 token。

它會把你的計劃拆解成決策樹，一層層釐清，讓你不會漏掉該注意的細節和決定。

### 架構設計

其實是兩個 skill 的分層設計：

| Skill | 角色 | 內容 |
|-------|------|------|
| `/grill-me` | 用戶入口（wrapper） | 正文只有一句：`Run a /grilling session`；只能人手敲命令觸發 |
| `/grilling` | 訪談原語（primitive） | 12 行正文，承載五條硬規則；其他 skill 也可調用 |

**為什麼拆開？** 讓「啟動入口」和「訪談紀律」不揉在一起，`grilling` 成為單一事實來源，任何需要訪談的 skill 都可以調用它，而不用各自重造。

---

## 二、核心規則（grilling 全文）

```yaml
---
name: grilling
description: Interview the user relentlessly about a plan or design.
  Use when the user wants to stress-test a plan before building,
  or uses any 'grill' trigger phrases.
---

Interview me relentlessly about every aspect of this plan until we reach
a shared understanding. Walk down each branch of the design tree,
resolving dependencies between decisions one-by-one.
For each question, provide your recommended answer.

Ask the questions one at a time, waiting for feedback on each question
before continuing. Asking multiple questions at once is bewildering.

If a question can be answered by exploring the codebase,
explore the codebase instead.
```

### 五條硬規則

| # | 規則 | 防什麼跑偏 |
|---|------|-----------|
| 1 | 沿決策樹逐分支推進，依賴關係一個一個解 | 一次過拋 30 條平行問題，依賴被壓平 |
| 2 | 每條問題必須附推薦答案（連理由和備選） | 用戶對著空白 prompt 自己猜，體驗極差 |
| 3 | 一次只問一題，等答覆才繼續 | 問題互相干擾，用戶答不清 |
| 4 | 事實 agent 自己查（codebase / 搜尋），決策由你拍板 | 把「項目用 Postgres 還是 MySQL」這種查得到的事也拿來問你 |
| 5 | 共享理解確認門禁：你說「都對」之前，不准動手 | Agent 自己腦補「應該就是這個意思」然後寫 code |

---

## 三、安裝方法

### 方法 A：skills.sh 安裝器（複製到本地 repo，方便 fork 修改）

```bash
# 裝整個 skills 倉庫
npx skills@latest add mattpocock/skills
npx skills update grill-me

# 或只裝單個 skill
npx skills add https://github.com/mattpocock/skills --skill grill-me
```

### 方法 B：Claude Code 原生 plugin（託管、唯讀、跟官方升級）

```
/plugin marketplace add mattpocock/skills
/plugin install mattpocock-skills@mattpocock
```

### 方法 C：DIY 自訂 slash command（任何 harness 都適用）

不想裝整個倉庫的話，直接把 grilling 的規則寫成自訂 command。以 Claude Code 為例，建立 `~/.claude/commands/grill-me.md`：

```yaml
---
description: Interview me relentlessly about a plan before building
---

針對我接下來提出的計畫，毫不留情地訪問我，直到我們達成共識為止。

規則：
1. 沿著決策樹一層一層往下走，把每個決定之間的依賴關係逐一解開
2. 每個問題都要附上你建議的答案（附理由和備選方案）
3. 一次只問一題，等我回答後再問下一題
4. 如果某件「事實」你自己查得到（搜尋、讀檔案、看 codebase），就自己查，不要問我；所有「決策」都必須由我拍板
5. 在我確認我們達成共識之前，不准動手執行任何東西

我的計畫：$ARGUMENTS
```

之後在任何對話輸入 `/grill-me 你的計畫` 即可使用。這個方法也適用於其他支援自訂 prompt / command 的 agent harness。

---

## 四、實戰流程（home lab 場景示範）

假設你想為自架服務加監控告警：

```
/grill-me 幫我的 Docker homelab 加一套監控 + 告警系統，現在有 20 多個 container 在跑
```

### 第 1 步：Agent 自己查事實（不會問你）

- 看 `docker-compose.yml` 有哪些服務
- 找有沒有現成的 Prometheus / Grafana / Uptime Kuma 配置
- 檢查 `.env` 有沒有 notification 相關變數（SMTP、Discord webhook 等）

> 「項目用什麼技術棧」這類問題屬於事實，agent 應該自己查，不該問你。

### 第 2 步：一題一題問，每題帶推薦答案

**Agent**：先定監控堆疊。我建議用 Prometheus + Grafana + Alertmanager：你已經在跑 Docker，這套是事實標準，長遠擴展性好。如果想輕量一點，Uptime Kuma 五分鐘就能跑，但只有 up/down 檢查，沒有 metrics 歷史。我的建議很明確：Prometheus 堆疊。能接受嗎？

**你**：接受。

**Agent**：接著定告警通道。我看到你的 `.env` 已經有 `DISCORD_WEBHOOK_URL`（自己查的事實），我建議告警直接推到 Discord。要不要加多一條 email 通道做後備？

### 第 3 步：共享理解確認門禁

**Agent**：到這裡我把所有決策捋一遍：

- 堆疊：Prometheus + Grafana + Alertmanager
- 告警通道：Discord webhook（主），email（後備）
- 告警規則：container down 超過 2 分鐘、CPU > 90% 持續 5 分鐘、磁碟 > 85%
- Retention：15 天
- 沿用現有 monitoring network，不開新 Docker network

全部都對嗎？確認後我才開始寫 compose file。

> 你確認之前，它不會動手。這個確認門禁是 v1.1.0 明確加上的硬約束。

---

## 五、使用時機判斷

| 場景 | 建議入口 | 原因 |
|------|---------|------|
| 臨時方案壓力測試 / 輕量需求對齊 | `/grill-me` | 純對話收斂，不寫檔案（stateless） |
| 項目級決策需要留底（ADR / CONTEXT.md） | `/grill-with-docs` | 邊問邊把術語和決策寫進文件 |
| 共享理解已收斂，想合成正式 spec | `/to-spec` | 不會再發起訪談 |
| 工作量超過單個 session / 路徑模糊 | `/wayfinder` | 在 issue tracker 建 decision map |
| 小 bug fix / 單檔案改動 | **直接進實作** | 不需要 grilling |
| 需要 UX 原型才答得到的問題 | **不要 grill** | grill 只擅長低保真決策 |

> ⚠️ **注意**：Matt Pocock 本人已公開表示，coding 場景他現在首推 `/grill-with-docs` 或 `domain-model`（新工作流是 `domain-model` → `to-prd` → `to-issues` → `tdd`）；`/grill-me` 的定位收窄為「窄壓力測試」和非編碼場景（季報、簡報、換工作等決策）。

---

## 六、實戰貼士

### 把約束寫進 prompt 兜底

`/grill-me` 的 wrapper 太薄，如果 harness 沒正確載入 frontmatter，agent 很容易退回自由發揮。把五條規則直接貼進 prompt 最穩妥：

```
/grill-me {你方向明確但邊界不齊的需求}

約束：
- 你只能在確認共享理解之後才動手
- 事實問題你自己查，不准問
- 每個問題都要帶推薦答案
- 一次只問一個
```

### 配合 Plan Mode 使用

先 `/grill-me` 收斂需求，再進 Plan Mode 出實施計劃，最後才 implement，形成「對齊 → 計劃 → 實作」閉環。

### 問題數量是預警指標

正常 session 大約 16–50 條問題、約 45 分鐘。如果問題開始失控膨脹（Matt 提過極端情況可達約 540 條），代表：

- 需求本身太大 → 換 `/wayfinder` 拆 session
- Agent 沒分清 fact vs decision，把該自己查的事拿來問你 → 直接打斷：「這個你自己查」

### 重要決策手動留底

`/grill-me` 是 stateless，session 結束什麼都不留。收斂完記得把共識寫進 ADR 或 CONTEXT.md（或直接叫 agent 寫），否則下次開新對話要重跑。

### 答不了的決策就記下來找人確認

有些問題連你自己都拍不了板（例如業務口徑、營運規則），grill 的價值正是提前暴露它們，而不是逼你即場回答。

---

## 七、不適合使用的情況（反向清單）

- ❌ 需求本身就模糊（vibe coding）— grill 的前提是「方向明確但邊界未確認」
- ❌ 純執行任務，決策已全部對齊 — grill 是浪費時間，直接進實作
- ❌ High-fidelity 問題（UI 手感、視覺走查）— 先做原型，再 grill 驗收
- ❌ 超大工作量，單 session 裝不下 — 先用 `/wayfinder` 建 decision map
- ❌ 重要決策需要長期留底 — 改用 `/grill-with-docs`

---

## 參考連結

- 倉庫：[mattpocock/skills](https://github.com/mattpocock/skills)
- AI Hero 官方文件：[skills-grill-me](https://www.aihero.dev/skills-grill-me)
- Matt 本人影片：[I stopped using /grill-me for coding](https://www.youtube.com/watch?v=6BB6exR8Zd8)
