# 受 Karpathy 啟發的 Claude Code 指引

一份精簡的 `CLAUDE.md`，補強 Claude Code 內建系統提示詞未涵蓋的部分；內容衍生自 [Andrej Karpathy 對 LLM 編碼陷阱的觀察](https://x.com/karpathy/status/2015883857489522876)。

[English](./README.md) | 繁體中文（台灣）

## 現況（2026 年 5 月）

Claude Code 在 Opus 4.7 / Sonnet 4.6 時代的預設系統提示詞已經內建了大部分通用的「不要過度設計／外科手術式修改／不要加沒人要的功能」指引 — 這些原本是本 skill 第一版的主要內容。**新版刻意只保留系統提示詞還未涵蓋的部分，並把 Karpathy 真正的「leverage」洞見重新定位為使用者端的 prompting 指引。**

舊版完整四原則保留在 [`archived/v1/`](./archived/v1/) 供參考。

## 給 LLM 看的內容

三條 reminder，與 [`CLAUDE.md`](./CLAUDE.md) 完全一致：

1. **困惑時停下** — 請求語意不清時，明確指出哪裡不清楚並提問；不要默默挑一個解讀就動手。
2. **每一行改動都要對應到請求** — 回報完成前，重看自己的 diff；任何沒有直接服務使用者目標的行就刪掉。
3. **以宣告式目標跑 loop** — 當存在可驗證的終態時，自主驅動直到達成。

整個指令檔就這樣。Karpathy 列出的其他陷阱（過度複雜化、順手重構、推測性功能、死碼累積、刪掉模型「看不順眼」的註解⋯⋯）都已經被 Claude Code 預設系統提示詞涵蓋；在這裡重述只會稀釋訊號。

## 給使用者的真正槓桿

Karpathy 最強的洞見其實是**使用者端的紀律**，不是 LLM 自我約束的事：

> 「LLM 非常擅長循環執行直到達成特定目標⋯⋯不要告訴它該做什麼，給它成功標準然後看著它跑完。」

要在自己的工作流中解鎖這個槓桿：

### 把命令式請求轉成宣告式

| 命令式（槓桿弱） | 宣告式（槓桿強） |
|---|---|
| 「加上輸入驗證」 | 「為這些無效輸入寫失敗測試，然後讓它們通過」 |
| 「修這個 bug」 | 「寫一個能重現 bug 的測試，然後讓它通過 — 其他測試必須還能過」 |
| 「讓它更快」 | 「把這個負載下的 p95 延遲壓到 X ms 以下；用 `scripts/bench.sh` 量」 |
| 「重構 X」 | 「重構 X 但不能改變可觀察行為；既有測試必須還能過」 |

### 同時交付驗證工具

連同目標一起給它驗證手段：測試指令、benchmark 腳本、lint 指令、給視覺檢查的 browser MCP。然後放手讓它迭代。

### 何時用哪一種

- **宣告式**：有可觀察結果的功能、bug 修復、效能工作、有測試覆蓋的重構。
- **命令式**：探索性編輯、UI 微調、文字創作、任何「完成」標準主觀的工作。

### 顯式 reframing：`dec` 指令

這個指令會請 Claude 把需求轉成「成功條件 + 驗證指令 + 非目標」**但不動手實作**。你確認後它才開始做。適合你想對單一 prompt 套用宣告式紀律、又不想改變其他寫法時使用。

```
/dec 修掉登入頁第一次載入時的閃爍
```

回覆會包含成功條件（例：「Playwright 截圖比對 10 次，位移 < 2px」）、驗證指令、明確非目標。若任務太主觀或太小，會回「不適用，建議直接做」，不會硬轉換。

> **呼叫名稱注意**：透過外掛（選項 A）安裝時，Claude Code 會把指令 namespace 成 `/andrej-karpathy-skills:dec`。想要短的 `/dec`，請用選項 C 手動安裝。

## 安裝

三條規則跟 `/dec` 指令各自獨立——任意組合都可以。

| | 三條規則 | `/dec` 指令 | 機制 |
|---|---|---|---|
| **A. 外掛** | 以 skill 形式（按情境觸發） | `/andrej-karpathy-skills:dec`（含 namespace） | 透過 marketplace 自動更新 |
| **B. `CLAUDE.md`** | 永遠在 system prompt | — | 依專案放一份 |
| **C. 手動指令檔** | — | `/dec`（短） | 全域或依專案 |

**選項 A：Claude Code 外掛** — skill + namespaced 指令，自動更新。

```
/plugin marketplace add yelban/andrej-karpathy-skills.TW
/plugin install andrej-karpathy-skills@karpathy-skills
```

**選項 B：依專案使用 `CLAUDE.md`** — 三條規則在該專案永遠載入。

```bash
curl -o CLAUDE.md https://raw.githubusercontent.com/yelban/andrej-karpathy-skills.TW/main/CLAUDE.md
```

**選項 C：手動安裝 `/dec` 指令** — 不經過外掛 namespace，直接打短的 `/dec`。

```bash
# 全域（所有專案可用）
mkdir -p ~/.claude/commands
curl -o ~/.claude/commands/dec.md https://raw.githubusercontent.com/yelban/andrej-karpathy-skills.TW/main/commands/dec.md

# 或依專案
mkdir -p .claude/commands
curl -o .claude/commands/dec.md https://raw.githubusercontent.com/yelban/andrej-karpathy-skills.TW/main/commands/dec.md
```

### 推薦組合

- **B + C**（不裝外掛）— `CLAUDE.md` 永遠在 + 短 `/dec`。最輕量、不依賴 plugin marketplace；要更新時重跑 `curl` 即可。
- **只裝 A** — 一條指令搞定、自動更新；但 `/dec` 會帶 namespace，規則也只在 skill 觸發時生效（非永遠在）。
- **A + B** — 外掛拿 `/dec`（namespaced）+ `CLAUDE.md` 拿永遠在的規則。skill 內容與 CLAUDE.md 略有重疊但無害。

## 搭配 Cursor 使用

本倉庫附 [`.cursor/rules/karpathy-guidelines.mdc`](.cursor/rules/karpathy-guidelines.mdc)，設定為 `alwaysApply: true`。詳情見 [`CURSOR.md`](./CURSOR.md)。

## 為什麼新版這麼短 — 而 A/B 實證告訴我們什麼

最初的論證（基於讀 Opus 4.7 系統提示詞）：v1 大部分規則已被內建提示詞涵蓋、重述會稀釋訊號。完整 v1 → v2 逐行對照見 [`archived/v1/NOTE.md`](./archived/v1/NOTE.md#detailed-v1--v2-diff-for-claudemd)（英文）。

這是觀察判斷、不是量測。所以 2026 年 5 月我們跑了小型 A/B 實證：

- 3 cells：無 CLAUDE.md / v1 upstream (65 行) / v2 我們版 (19 行)
- 4 個誘發 Karpathy 痛點的 toy task + 最區分維度 T1 ambiguous-bug 加碼 N=10
- Opus 4.7 受測、Sonnet 4.6 blind judge

**結果：三個 cell 沒有統計顯著差異。** T1 加碼到 N=10 後三 cell 全部 7/10 正確、Fisher exact p = 1.000。30 個 runs 中 **0 個**在編輯前問 clarification——不論哪版規則都沒能可靠觸發「停下來問」。

誠實版結論：在這個 toy task 規模上、CLAUDE.md（不論哪版）對 Opus 4.7 行為的邊際效應**小到 N=10 測不出來**。**任選一版皆可；user-side declarative framing（`/dec`）的槓桿可能比規則檔本身大。**

完整資料、scripts、caveats、以及 Phase 1 (N=3) 一度看起來「v1 顯著優於 v2」最後被 Phase 2 (N=10) 攤平的過程：[`EXPERIMENT.md`](./EXPERIMENT.md)（英文）。

## 與上游的關係

本倉庫是 [`forrestchang/andrej-karpathy-skills`](https://github.com/forrestchang/andrej-karpathy-skills) 的繁體中文（台灣）在地化 fork，為 Claude Code Opus 4.7 時代更新內容。Plugin / marketplace 名稱刻意與上游一致；README 為雙語（英文 + 繁體中文）。

## 授權

MIT
