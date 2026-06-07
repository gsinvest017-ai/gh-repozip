# gh-repozip Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 一個 `gh` CLI extension：`gh repozip <user>` 一鍵 clone 指定 user/org 的所有 repo（含完整 git 歷史）並打包成單一 zip 存到指定路徑。

**Architecture:** 單一 bash 入口檔（gh extension 慣例：repo 名 `gh-repozip`、可執行檔同名）。流程：依賴檢查 → `gh repo list` 取清單 → 逐一 clone 到 mktemp 暫存目錄 → `zip -r` → 移到輸出目錄 → trap 清理。

**Tech Stack:** bash、gh CLI、git、zip、shellcheck（靜態檢查）。

**Spec:** `docs/superpowers/specs/2026-06-07-gh-repozip-design.md`

---

### Task 1: 主程式 `gh-repozip`

**Files:**
- Create: `gh-repozip`（repo 根目錄，可執行）

- [ ] **Step 1: 寫入完整 script**

```bash
#!/usr/bin/env bash
# gh-repozip — 下載指定 user/org 的所有 repo（含完整 git 歷史）打包成單一 zip
set -euo pipefail

usage() {
    cat <<'EOF'
用法: gh repozip <user-or-org> [-o <輸出目錄>] [--limit N]

一鍵下載指定 GitHub user / org 的所有 repo（含完整 git 歷史），
打包成單一 zip：<user>-repos-<YYYYMMDD>.zip

參數:
  <user-or-org>        GitHub username 或 org 名稱（必填）。
                       若為登入帳號本人且 token 有權限，自動包含 private repo。
  -o, --output <dir>   zip 輸出目錄（預設：當前目錄；不存在自動建立）
  --limit N            最多抓取 N 個 repo（預設：1000）
  -h, --help           顯示本說明

範例:
  gh repozip torvalds -o ~/backups
  → ~/backups/torvalds-repos-20260607.zip
EOF
}

TARGET=""
OUTPUT_DIR="."
LIMIT=1000

while [[ $# -gt 0 ]]; do
    case "$1" in
        -o|--output)
            [[ $# -ge 2 ]] || { echo "錯誤：$1 需要參數" >&2; exit 1; }
            OUTPUT_DIR="$2"; shift 2 ;;
        --limit)
            [[ $# -ge 2 ]] || { echo "錯誤：--limit 需要參數" >&2; exit 1; }
            LIMIT="$2"; shift 2 ;;
        -h|--help)
            usage; exit 0 ;;
        -*)
            echo "錯誤：未知選項 $1" >&2; usage >&2; exit 1 ;;
        *)
            if [[ -z "$TARGET" ]]; then TARGET="$1"; shift
            else echo "錯誤：多餘的參數 $1" >&2; usage >&2; exit 1; fi ;;
    esac
done

[[ -n "$TARGET" ]] || { usage >&2; exit 1; }
[[ "$LIMIT" =~ ^[0-9]+$ ]] || { echo "錯誤：--limit 必須是正整數" >&2; exit 1; }

# --- 依賴檢查 ---
for cmd in gh git zip; do
    command -v "$cmd" >/dev/null 2>&1 || { echo "錯誤：找不到 $cmd，請先安裝" >&2; exit 1; }
done
gh auth status >/dev/null 2>&1 || { echo "錯誤：gh 未登入，請先執行 gh auth login" >&2; exit 1; }

# --- 取得 repo 清單 ---
echo "查詢 $TARGET 的 repo 清單（上限 $LIMIT）..."
mapfile -t repos < <(gh repo list "$TARGET" --limit "$LIMIT" --json nameWithOwner -q '.[].nameWithOwner')
total=${#repos[@]}
(( total > 0 )) || { echo "錯誤：$TARGET 沒有可見的 repo（或帳號不存在）" >&2; exit 1; }
echo "找到 $total 個 repo"

# --- 暫存目錄 + 清理 trap ---
WORKDIR=$(mktemp -d)
trap 'rm -rf "$WORKDIR"' EXIT INT TERM

# --- 逐一 clone（完整歷史）---
failed=()
i=0
for r in "${repos[@]}"; do
    i=$((i + 1))
    name=${r#*/}
    echo "[$i/$total] cloning $r ..."
    if ! gh repo clone "$r" "$WORKDIR/$TARGET/$name" -- --quiet; then
        echo "  ⚠ clone 失敗：$r" >&2
        failed+=("$r")
    fi
done

cloned=$(( total - ${#failed[@]} ))
(( cloned > 0 )) || { echo "錯誤：所有 repo 都 clone 失敗" >&2; exit 1; }

# --- 壓縮 ---
mkdir -p "$OUTPUT_DIR"
zipname="${TARGET}-repos-$(date +%Y%m%d).zip"
echo "壓縮 $cloned 個 repo 成 $zipname ..."
(cd "$WORKDIR" && zip -r -q "$zipname" "$TARGET")
mv "$WORKDIR/$zipname" "$OUTPUT_DIR/$zipname"

# --- 總結 ---
zip_path=$(cd "$OUTPUT_DIR" && pwd)/$zipname
zip_size=$(du -h "$zip_path" | cut -f1)
echo ""
echo "完成：$zip_path（$zip_size，$cloned/$total 個 repo）"
if (( ${#failed[@]} > 0 )); then
    echo "以下 repo clone 失敗：" >&2
    printf '  - %s\n' "${failed[@]}" >&2
    exit 2
fi
```

- [ ] **Step 2: 設為可執行**

Run: `chmod +x ~/gh-repozip/gh-repozip`

- [ ] **Step 3: shellcheck 靜態檢查**

Run: `shellcheck ~/gh-repozip/gh-repozip`（若未安裝：`sudo apt-get install -y shellcheck` 或改用 `docker run --rm -v ~/gh-repozip:/m koalaman/shellcheck /m/gh-repozip`；都不行則跳過並在總結註明）
Expected: 無 error（SC 提示若為 style/info 可接受，error 必須修）

- [ ] **Step 4: Commit**

```bash
cd ~/gh-repozip && git add gh-repozip
git commit -m "feat: 新增 gh-repozip 主程式（clone 全部 repo 並打包單一 zip）

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 2: README 與 LICENSE

**Files:**
- Create: `README.md`
- Create: `LICENSE`

- [ ] **Step 1: 寫 README.md**

內容必含（繁中）：
- 一句話簡介（同 spec 目的）
- 安裝：`gh extension install gsinvest017-ai/gh-repozip`
- 用法區塊：與 `gh-repozip` script 內 usage() 完全一致的參數說明與範例
- 行為說明：含完整 git 歷史、private repo 條件、失敗 repo 不中斷（exit 2）、Ctrl-C 自動清理暫存
- 依賴：gh（已登入）、git、zip

- [ ] **Step 2: 寫 LICENSE（MIT，著作權人 gsinvest017-ai，年份 2026）**

- [ ] **Step 3: Commit**

```bash
cd ~/gh-repozip && git add README.md LICENSE
git commit -m "docs: 新增 README 與 MIT LICENSE

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 3: 本地安裝 + 實跑驗證

**Files:** 無新檔（驗證任務）

- [ ] **Step 1: 本地安裝 extension**

Run: `cd ~/gh-repozip && gh extension install .`
Expected: 安裝成功；`gh extension list` 出現 `repozip`

- [ ] **Step 2: help 與錯誤路徑驗證**

Run: `gh repozip --help` → Expected: 顯示用法說明，exit 0
Run: `gh repozip` → Expected: 顯示用法到 stderr，exit 1
Run: `gh repozip this-user-should-not-exist-zzz9 2>&1` → Expected: 「沒有可見的 repo（或帳號不存在）」，exit 1

- [ ] **Step 3: 對小帳號實跑**

Run: `gh repozip octocat --limit 3 -o /tmp/repozip-test`
Expected: 顯示 `[1/3]`...`[3/3]` 進度、產出 `/tmp/repozip-test/octocat-repos-<今日>.zip`、exit 0

- [ ] **Step 4: 驗證 zip 內容**

Run: `unzip -l /tmp/repozip-test/octocat-repos-*.zip | head -20`
Expected: 路徑為 `octocat/<repo名>/...`，且包含 `.git/` 條目（完整歷史）

- [ ] **Step 5: 清理測試產物**

Run: `rm -rf /tmp/repozip-test`

---

### Task 4: 推上 GitHub + 遠端安裝驗證

**Files:** 無新檔（發佈任務）

- [ ] **Step 1: 建立 public repo 並推送**

```bash
cd ~/gh-repozip
gh repo create gh-repozip --public --source=. --remote=origin --push \
  --description "gh extension：一鍵下載 user/org 所有 repo（含 git 歷史）打包成單一 zip"
```

Expected: repo 建立於 `gsinvest017-ai/gh-repozip`，main 已推上

- [ ] **Step 2: 遠端安裝驗證**

```bash
gh extension remove repozip
gh extension install gsinvest017-ai/gh-repozip
gh repozip --help
```

Expected: 從遠端安裝成功，help 正常顯示

---

## Self-Review 紀錄

- Spec coverage：用法/flags（Task 1 usage + 參數解析）、依賴檢查、repo 清單、clone 進度、zip 命名與輸出、trap 清理、總結輸出、全部錯誤處理表（user 不存在、單 repo 失敗 exit 2、mkdir -p、Ctrl-C、未登入）、shellcheck + 實跑測試（Task 3）、public remote（Task 4）— 全數對應。
- Placeholder scan：無 TBD/TODO；README 內容以明確清單規定。
- Type consistency：flag 名稱（`-o/--output`、`--limit`）、zip 命名 `<user>-repos-<YYYYMMDD>.zip` 在 spec、usage、README 規定間一致。
