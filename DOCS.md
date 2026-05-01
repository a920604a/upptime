# Upptime 專案文件

## 目錄

- [專案概述](#專案概述)
- [架構說明](#架構說明)
- [監控目標](#監控目標)
- [Workflow 詳解](#workflow-詳解)
- [資料儲存結構](#資料儲存結構)
- [設定說明](#設定說明)
- [初始化流程](#初始化流程)
- [常見問題排查](#常見問題排查)

---

## 專案概述

本專案基於 [Upptime](https://github.com/upptime/upptime)，是一套完全跑在 GitHub 上的網站監控與狀態頁面系統。

- **不需要伺服器**：所有運算皆使用 GitHub Actions 免費額度
- **不需要資料庫**：所有狀態資料以 JSON 格式儲存在 repo 本身
- **自動 Incident 管理**：網站掛掉時自動開 GitHub Issue，恢復後自動關閉
- **對外狀態頁面**：透過 GitHub Pages 提供公開狀態網址

對外網址：`https://a920604a.github.io/upptime`

---

## 架構說明

```
┌─────────────────────────────────────────────┐
│              GitHub Actions                  │
│                                             │
│  每 5 分鐘: Uptime CI ──────────────────┐  │
│  每天 23:00: Response Time CI ──────────┤  │
│  每天 00:00: Summary CI ────────────────┤  │
│  每天 00:00: Graphs CI ─────────────────┤  │
│  每天 01:00: Static Site CI ────────────┤  │
│  每週: Update Template CI ──────────────┘  │
└──────────┬──────────────────────────────────┘
           │ git commit/push
           ▼
┌─────────────────────────────────────────────┐
│              GitHub Repository               │
│                                             │
│  master branch:                             │
│    api/          ← 監控數據 (JSON)           │
│    graphs/       ← 圖表 (PNG)               │
│    history/      ← 歷史紀錄 (YAML)           │
│    README.md     ← 自動更新狀態表格           │
│                                             │
│  gh-pages branch:                           │
│    靜態狀態頁面 (HTML/CSS/JS)               │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│  GitHub Pages → 對外狀態頁面                 │
│  GitHub Issues → Incident 紀錄              │
└─────────────────────────────────────────────┘
```

---

## 監控目標

目前在 `.upptimerc.yml` 設定的監控項目：

| 名稱 | URL | 協定 |
|------|-----|------|
| Google | https://www.google.com | HTTP |
| Engineer News | https://engineer-news.pages.dev/ | HTTP |
| My Resume | https://a920604a.github.io/self-reusme-website/ | HTTP |
| Assets | https://asset-scope-frontend.pages.dev/ | HTTP |
| My Labs | https://a920604a-labs.pages.dev/ | HTTP |
| Nutrition | https://nutrition-risk-engine.pages.dev/ | HTTP |

---

## Workflow 詳解

### 1. Uptime CI（`uptime.yml`）

**觸發頻率：** 每 5 分鐘

**功能：**
- 對每個監控目標發送 HTTP/TCP 請求
- 若回應正常：更新 `api/{site}/uptime*.json`
- 若回應失敗：自動在 GitHub Issues 開一筆 incident report
- 若網站從失敗恢復：自動關閉對應的 Issue

```
每 5 分鐘
  └── ping 所有網站
        ├── 正常 → 更新 uptime JSON
        └── 失敗 → 開 GitHub Issue (incident)
              └── 恢復 → 關閉 Issue
```

---

### 2. Response Time CI（`response-time.yml`）

**觸發頻率：** 每天 23:00 UTC

**功能：**
- 記錄每個網站的 HTTP 回應時間（ms）
- 儲存日 / 週 / 月 / 年的回應時間資料到 `api/{site}/response-time*.json`

---

### 3. Summary CI（`summary.yml`）

**觸發頻率：** 每天 00:00 UTC

**功能：**
- 讀取 `api/` 的最新數據
- 重新產生 `README.md` 中的狀態表格（`<!--start: status pages-->` 區塊）

---

### 4. Graphs CI（`graphs.yml`）

**觸發頻率：** 每天 00:00 UTC

**功能：**
- 把 `api/` 的歷史數據繪製成 PNG 圖表
- 儲存到 `graphs/{site}/response-time-*.png`

---

### 5. Static Site CI（`site.yml`）

**觸發頻率：** 每天 01:00 UTC

**功能：**
- 用 Svelte/Sapper 把所有監控數據 build 成靜態網頁
- 把產出的靜態檔案 push 到 `gh-pages` branch
- GitHub Pages 自動部署到對外網址

---

### 6. Update Template CI（`update-template.yml`）

**觸發頻率：** 每天 00:00 UTC

**功能：**
- 從 upptime 官方模板同步最新版本
- 自動更新 `.github/workflows/` 下所有 workflow 檔案
- **注意：** 這會覆蓋所有對 workflow 檔案的手動修改

---

### 7. Setup CI（`setup.yml`）

**觸發條件：** `.upptimerc.yml` 有 push，或手動觸發

**功能：** 一次性執行所有初始化步驟，適用於第一次設定或變更監控設定後：

```
1. Update template     → 同步最新 workflow 版本
2. Update response time → 立即記錄一次回應時間
3. Update summary       → 更新 README 表格
4. Generate graphs      → 產生圖表（trigger Graphs CI）
5. Generate site        → Build 靜態頁面
6. GitHub Pages Deploy  → 部署到 gh-pages
```

---

## 資料儲存結構

```
upptime/
├── .upptimerc.yml              ← 唯一需要手動維護的設定檔
├── api/
│   └── {site-slug}/
│       ├── uptime.json         ← 當前 uptime 百分比
│       ├── uptime-day.json     ← 24 小時 uptime
│       ├── uptime-week.json    ← 7 天 uptime
│       ├── uptime-month.json   ← 30 天 uptime
│       ├── uptime-year.json    ← 365 天 uptime
│       ├── response-time.json  ← 最新回應時間 (ms)
│       ├── response-time-day.json
│       ├── response-time-week.json
│       ├── response-time-month.json
│       └── response-time-year.json
├── graphs/
│   └── {site-slug}/
│       ├── response-time.png
│       ├── response-time-day.png
│       ├── response-time-week.png
│       ├── response-time-month.png
│       └── response-time-year.png
├── history/
│   └── {site-slug}.yml         ← 每次 check 的完整歷史紀錄
├── README.md                   ← 自動產生，勿手動編輯狀態區塊
└── .github/
    └── workflows/              ← 自動產生，勿直接修改
```

---

## 設定說明

所有設定都在 `.upptimerc.yml`，這是**唯一需要手動維護**的設定檔。

### 基本設定

```yaml
owner: a920604a   # GitHub 帳號
repo: upptime     # Repository 名稱
```

### 新增監控目標

```yaml
sites:
  - name: 我的網站          # 顯示名稱
    url: https://example.com # 監控網址

  # TCP Ping 範例
  - name: Mail Server
    url: mail.example.com
    port: 25
    check: "tcp-ping"

  # IPv6 範例
  - name: IPv6 Service
    url: example.com
    port: 80
    check: "tcp-ping"
    ipv6: true
```

### 狀態頁面設定

```yaml
status-website:
  baseUrl: /upptime           # 若無自訂域名，填入 repo 名稱
  # cname: status.example.com  # 自訂域名（有的話取消注釋）
  name: Upptime
  navbar:
    - title: Status
      href: /
    - title: GitHub
      href: https://github.com/$OWNER/$REPO
```

### 變更設定後

修改 `.upptimerc.yml` 並 push → 自動觸發 **Setup CI** 執行完整更新。

---

## 初始化流程

第一次設定或重新設定時需要：

### 1. 建立 GitHub Personal Access Token (PAT)

前往 GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)

必要的 scope：
- `repo`（完整 repo 存取）
- `workflow`（觸發 Actions）

### 2. 設定 Repository Secret

前往 repo → Settings → Secrets and variables → Actions → New repository secret

- Name: `GH_PAT`
- Value: 步驟 1 建立的 token

### 3. 設定 Actions 權限

前往 repo → Settings → Actions → General → Workflow permissions

選擇：**Read and write permissions**

### 4. 啟用 GitHub Pages

前往 repo → Settings → Pages → Source

選擇：**Deploy from a branch** → Branch: `gh-pages`

### 5. 觸發 Setup CI

修改 `.upptimerc.yml` 任意內容後 push，或前往 Actions → Setup CI → Run workflow。

---

## 常見問題排查

### Setup CI 失敗：`Unable to find workflow 'Graphs CI'`

**原因：** `update-template` 步驟 push 新 commit 後，GitHub API 還沒來得及更新 workflow 清單，下一步的 `workflow-dispatch` 就找不到 `Graphs CI`。

**解法：**
1. 手動前往 `https://github.com/a920604a/upptime/actions/workflows/graphs.yml` 觸發一次 Graphs CI
2. 或忽略此錯誤，Graphs CI 每天 00:00 UTC 會自動執行

**注意：** 不要直接修改 `setup.yml`，`Update Template CI` 每天都會覆蓋回去。

---

### GitHub Pages Deploy 失敗：`403 Permission denied`

**原因：** `GH_PAT` 權限不足，無法 push 到 `gh-pages` branch。

**解法：** 重新建立 Classic PAT，確認勾選 `repo` 和 `workflow` scope，並更新 `GH_PAT` secret。

---

### 網站明明正常但被標記為 Down

**可能原因：**
- 網站封鎖了 GitHub Actions 的 IP（部分 CDN 會這樣）
- 網站需要特定 User-Agent

**解法：** 在 `.upptimerc.yml` 的監控項目加上：
```yaml
- name: My Site
  url: https://example.com
  headers:
    User-Agent: "Upptime/1.0"
```

---

### 如何手動觸發各 Workflow

前往 Actions 分頁，點選對應 workflow → Run workflow：

| Workflow | 用途 |
|----------|------|
| Uptime CI | 立即執行一次 uptime check |
| Response Time CI | 立即記錄一次回應時間 |
| Graphs CI | 立即重新產生圖表 |
| Static Site CI | 立即重新 build 並部署狀態頁面 |
| Summary CI | 立即更新 README 表格 |
| Setup CI | 執行完整初始化（設定變更後用） |
