# gh-repozip 設計文件

日期：2026-06-07
狀態：已核准（方案 A）

## 目的

一個 `gh` CLI extension：一鍵下載指定 GitHub user / org 的**所有 repo**（含完整
git 歷史），打包成**單一 zip** 存到指定本地路徑。

## 使用方式

```bash
# 安裝
gh extension install gsinvest017-ai/gh-repozip

# 使用
gh repozip <user-or-org> [-o <輸出目錄>] [--limit N]

# 範例
gh repozip torvalds -o ~/backups
# 產出：~/backups/torvalds-repos-20260607.zip
```

- `<user-or-org>`：必填。GitHub username 或 org 名稱。若為登入帳號本人且 token
  有權限，自動包含 private repo（`gh repo list` 原生行為）。
- `-o, --output <dir>`：zip 輸出目錄，預設當前目錄；不存在時自動建立。
- `--limit N`：最多抓取 N 個 repo，預設 1000。
- `-h, --help`：使用說明。

## Repo 結構

```
gh-repozip/
├── gh-repozip      # bash 入口（gh extension 規定：與 repo 同名的可執行檔）
├── README.md       # 繁中說明 + 安裝/使用方式
├── LICENSE         # MIT
└── docs/superpowers/specs/   # 設計文件（本檔）
```

GitHub remote：`gsinvest017-ai/gh-repozip`（public，遠端安裝必要條件）。

## 執行流程（方案 A：一般 clone + 整包壓縮）

1. **依賴檢查**：`gh`、`git`、`zip` 是否在 PATH；`gh auth status` 是否已登入。
   缺任一項 → 明確錯誤訊息退出。
2. **取得 repo 清單**：
   `gh repo list <user> --limit <N> --json nameWithOwner -q '.[].nameWithOwner'`
3. **逐一 clone**：`mktemp -d` 建暫存目錄，依序 `gh repo clone <nameWithOwner>`
   （完整歷史，working tree + `.git`），輸出 `[i/total] cloning <name>...` 進度。
4. **壓縮**：在暫存目錄內 `zip -r -q` 成 `<user>-repos-<YYYYMMDD>.zip`，
   移到輸出目錄。
5. **清理**：`trap` 確保正常結束或中斷（INT/TERM/EXIT）都會刪除暫存目錄。
6. **總結輸出**：repo 數、zip 路徑與大小、失敗清單（若有）。

## 錯誤處理

| 情況 | 行為 |
|------|------|
| user/org 不存在或無可見 repo | 明確訊息，exit 1 |
| 單一 repo clone 失敗 | 記錄後繼續，結尾列出失敗清單；zip 照常產出，exit 非 0 |
| 輸出目錄不存在 | 自動 `mkdir -p` |
| 中途 Ctrl-C | trap 清理暫存目錄 |
| 未登入 gh | 提示 `gh auth login`，exit 1 |

## 已考慮但否決的替代方案

- **方案 B `git clone --mirror`**：zip 最小，但解壓後需再 clone 一次才有
  working tree，使用不直覺。
- **方案 C 並行 clone（`xargs -P`）**：較快但進度輸出混雜、錯誤處理複雜；
  repo 數通常不大，順序 clone 可接受。
- **GitHub zipball API**：最快但不含 git 歷史，不符需求。

## 測試

- `shellcheck gh-repozip` 無 error。
- 對小帳號實跑：驗證 zip 產出、內含各 repo 的 working tree 與 `.git`、
  失敗 repo 不中斷整體流程、Ctrl-C 後暫存目錄被清掉。
