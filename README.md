# gh-repozip

gh CLI extension —— 一鍵下載指定 GitHub user / org 的所有 repo（含完整 git 歷史），打包成單一 zip 存到指定本地路徑。

## 安裝

```bash
gh extension install gsinvest017-ai/gh-repozip
```

## 用法

```
gh repozip <user-or-org> [-o <輸出目錄>] [--limit N]
```

### 參數

| 參數 | 說明 |
|------|------|
| `<user-or-org>` | GitHub username 或 org 名稱（必填）。若為登入帳號本人且 token 有權限，自動包含 private repo。 |
| `-o, --output <dir>` | zip 輸出目錄（預設：當前目錄；不存在自動建立） |
| `--limit N` | 最多抓取 N 個 repo（預設：1000，正整數） |
| `-h, --help` | 顯示說明 |

### 範例

```bash
gh repozip torvalds -o ~/backups
# → ~/backups/torvalds-repos-20260607.zip
```

## 行為說明

- **完整 clone**：每個 repo 保留 working tree 與 `.git` 完整歷史；zip 內路徑為 `<user>/<repo名>/...`
- **失敗容錯**：單一 repo clone 失敗不中斷整體流程；結尾列出失敗清單，exit code 2（zip 照常產出）
- **中斷清理**：Ctrl-C 或訊號中斷時自動刪除暫存目錄
- **重跑覆寫**：同日重跑同一 user/org 會覆寫同名 zip

## 依賴

- `gh`（需先執行 `gh auth login`）
- `git`
- `zip`
- bash 4+（Linux / WSL / macOS 需較新版 bash）

## License

MIT
