# 受 Karpathy 啟發的 Claude Code 指引

一份精簡的 `CLAUDE.md`，補強 Claude Code 內建系統提示詞未涵蓋的部分；內容衍生自 [Andrej Karpathy 對 LLM 編碼陷阱的觀察](https://x.com/karpathy/status/2015883857489522876)。

[English](./README.md) | 繁體中文（台灣）

## 現況（2026 年 5 月）

Claude Code 在 Opus 4.7 / Sonnet 4.6 時代的預設系統提示詞已經內建了大部分通用的「不要過度設計／外科手術式修改／不要加沒人要的功能」指引 — 這些原本是本 skill 第一版的主要內容。**新版刻意只保留系統提示詞還未涵蓋的部分，並把 Karpathy 真正的「leverage」洞見重新定位為使用者端的 prompting 指引。**

舊版完整四原則保留在 [`archived/v1/`](./archived/v1/) 供參考。

## 給使用者的真正槓桿

**這是整份 README 最重要的章節。** 我們的 [A/B 實證](./EXPERIMENT.md)（2026 年 5 月）發現下方那三條規則在 Opus 4.7 上沒有可測量效應（最區分維度每組 N=10、Fisher exact p=1.00）。但這節的「使用者端描述方式」（user-side framing）與模型無關——不論對面是哪個 LLM、它都會改變你拿回來的東西。

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

### `/dec`：「邊界設定器」，讓 `/goal` 真的會收斂

兩個 slash command 剛好對應 Karpathy 那句「給成功條件 + 看著它跑」的**兩個動詞**：

| | `/dec`（本倉庫） | `/goal`（Claude Code 內建，v2.1.139+） |
|---|---|---|
| 階段 | **動手前**：把模糊請求改寫成契約 | **動手中**：盯著 Claude 跑直到契約達成 |
| 動作 | 重寫使用者輸入、**不動手** | 每 turn 後讓小模型評估是否達標、沒達標自動進下一 turn |
| 持久度 | 一次性轉換、等你確認 | Session-scoped、直到 `/goal clear` |
| 評估者 | **你**（人工 review 契約） | **Haiku**（讀 transcript 自動判 yes/no） |
| Karpathy 動詞 | "give it success criteria" | "watch it go" |

#### 單獨用 `/dec`

`dec` 是 **declarative**（宣告式）的縮寫。指令把命令式需求改寫成契約；你確認後才會動工。

```
/dec 修登入頁第一次載入時的閃爍
```

回覆會給你成功條件（例如「Playwright 截圖比對 10 次、位移 < 2px」）、驗證指令、明確非目標——**外加一條可直接複製貼上的 `/goal` condition 字串**（用 AND 把成功條件與非目標串成複合條件）。若任務太主觀或太小，會回「不適用，建議直接做」、不硬轉換。適合單一 prompt 套用宣告式紀律、不需要自主迭代的場合（或在 Cursor / 舊版 Claude Code 沒有 `/goal` 時）。

#### `/dec` 當作 `/goal` 的「邊界設定器」

`/goal` 的效果完全取決於你給它的 condition 字串。寫不好的 condition 永遠不會收斂：

```
❌ /goal "登入頁不要閃爍"
   Haiku 怎麼判定「不閃爍」？看截圖？讀 console？
   結果 evaluator 一律回 yes 或一律回 no、loop 永遠不收斂。

✅ /dec 修登入頁第一次載入時的閃爍
   →  成功條件：Playwright 截圖比對 10 次、位移 < 2px
       驗證指令：npx playwright test login-flicker.spec.ts
       非目標：不重構登入元件、不動 auth 流程

✅ /goal "npx playwright test login-flicker.spec.ts passes AND
          登入元件檔案路徑與 baseline 一致"
   Haiku 讀 transcript 內的 pytest 輸出能精準判定。
   Loop 真的會收斂。
```

`/dec` 強制設定 `/goal` 自己給不出的三條邊界：

1. **可機器判定的成功條件**——「diff < 2px」「10 passed」「p95 < X ms」evaluator 看 transcript 就能 yes/no
2. **嵌進契約的驗證指令**——強迫 Claude 真的去跑檢查、而不是靜態推理然後回報「應該可以了」（這正是我們 T4 declarative-loop 測試看到的失敗模式）
3. **明確非目標**——`/goal` 的 condition 可以是複合的：「X passes AND test files unchanged AND no new files in src/legacy/」evaluator 一條一條檢

#### 完整 pipeline

```
1. /dec <模糊需求>                 ← 契約 + 一條編好的 /goal condition
2. 你 review 契約                  ← 人類確認方向
3. 複製 #1 那條 /goal 指令貼上     ← Haiku 接手當判官
4. Claude 自主 loop 到收斂          ← Karpathy 說的「watch it go」
```

#### 為什麼這才是真正的槓桿

不依賴模型的使用者端紀律。我們的 [A/B 實證](./EXPERIMENT.md) 顯示 CLAUDE.md 規則對 Opus 4.7 的程式碼行為沒有可測量影響——但「寫得好的契約 + 自主評估迴圈」是**你**能控制的槓桿、不是只能祈禱模型自己學會的東西。Opus 4.7 → 4.8 → 5.0 升級時這個槓桿也不會貶值；`/dec` 模板與 `/goal` evaluator 都還能用。

> **呼叫名稱注意**：透過外掛（選項 A）安裝時，Claude Code 會把指令 namespace 成 `/andrej-karpathy-skills:dec`。想要短的 `/dec`，請用選項 C 手動安裝。內建的 `/goal` 不受安裝方式影響、永遠可用。

> **`/goal` 評估者注意**：`/goal` 把每 turn 的 transcript 餵給 Claude Code 內建的「small fast model」slot、[預設是 Haiku](https://code.claude.com/docs/en/goal.md)。**沒有 `/goal` 專屬的 model 設定**；唯一替換方式是用 `ANTHROPIC_DEFAULT_HAIKU_MODEL` 環境變數整體 redirect 那個 slot（[model config 文件](https://code.claude.com/docs/en/model-config.md)）——但這會把 `haiku` alias 全部換掉、不只 `/goal`。一般使用不需動。

## 給 LLM 看的內容

三條 reminder，與 [`CLAUDE.md`](./CLAUDE.md) 完全一致。保留是因為成本低、不同模型或更長的上下文下或許有用；但在 Opus 4.7 上實證邊際效應不顯著（見 [`EXPERIMENT.md`](./EXPERIMENT.md)）。

1. **困惑時停下** — 請求語意不清時，明確指出哪裡不清楚並提問；不要默默挑一個解讀就動手。
2. **每一行改動都要對應到請求** — 回報完成前，重看自己的 diff；任何沒有直接服務使用者目標的行就刪掉。
3. **以宣告式目標跑 loop** — 當存在可驗證的終態時，自主驅動直到達成。

整個指令檔就這樣。Karpathy 列出的其他陷阱（過度複雜化、順手重構、推測性功能、死碼累積、刪掉模型「看不順眼」的註解⋯⋯）都已經被 Claude Code 預設系統提示詞涵蓋；在這裡重述只會稀釋訊號。

## 安裝

三條規則跟 `/dec` 指令各自獨立——任意組合都可以。

| | 三條規則 | `/dec` 指令 | 機制 |
|---|---|---|---|
| **A. 外掛** | — (v3.0.0 起 skill 已移除；CLAUDE.md 改用 B / C / D) | `/andrej-karpathy-skills:dec`（含 namespace） | 透過 marketplace 自動更新 |
| **B. `CLAUDE.md`** | 永遠在 system prompt | — | 依專案放一份、手動 `curl` |
| **C. 手動指令檔** | — | `/dec`（短） | 全域或依專案、手動 `curl` |
| **D. `git clone`** | 整檔 `cp` *或* `sed` 追加規則 | `/dec`（短、symlink 過去） | `git pull` 更新 `/dec`；`CLAUDE.md` 是你的可編輯副本 |

**選項 A：Claude Code 外掛** — 只安裝 `/dec` 指令（含 namespace）、透過 marketplace 自動更新。v3.0.0 起移除了那個包著三條規則的 skill——因為實證告訴我們它在 Opus 4.7 上沒有可測量效應（見 [`EXPERIMENT.md`](./EXPERIMENT.md)）。要永遠在的規則，請用下面的 B / C / D。

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

**選項 D：`git clone` + symlink** — `/dec` 透過 `git pull` 自動更新；`CLAUDE.md` 整檔 `cp` 作為起點、之後依專案自由修改。

```bash
# 1. Clone 一次（位置自選；範例放 ~/.claude/external/）
mkdir -p ~/.claude/external
git clone https://github.com/yelban/andrej-karpathy-skills.TW \
  ~/.claude/external/andrej-karpathy-skills.TW

# 2. 把短的 /dec 指令 symlink 到全域（命令本身 stateless、跟著上游走就好）
mkdir -p ~/.claude/commands
ln -sf ~/.claude/external/andrej-karpathy-skills.TW/commands/dec.md \
  ~/.claude/commands/dec.md

# 3. 依專案的 CLAUDE.md——擇一。
#    不用 symlink：CLAUDE.md 是專案自己的東西，所以是「cp 或追加之後自己改」。

# (a) 專案還沒 CLAUDE.md，整檔 cp 當起點：
cp ~/.claude/external/andrej-karpathy-skills.TW/CLAUDE.md ./CLAUDE.md

# (b) 專案已有自己的 CLAUDE.md，只追加三條規則：
sed -n '/^## Stop when confused/,$p' \
  ~/.claude/external/andrej-karpathy-skills.TW/CLAUDE.md >> ./CLAUDE.md

# 更新 /dec、拉新版 README / EXPERIMENT.md：
cd ~/.claude/external/andrej-karpathy-skills.TW && git pull
# CLAUDE.md 不會自動更新——想拿新規則時、自己再重跑 (a) 或 (b)。
```

> `sed` 從第一個 `## Stop when confused` 標題開始抓、跳過 README title 與前言段。末尾 `/dec` 呼叫說明會被一起追加——當作專案 CLAUDE.md 的 footer 也合用。

### 推薦組合

- **只裝 D**（推薦給 power user）— clone 一次、symlink `/dec`、`cp` CLAUDE.md 當起點。`git pull` 更新 `/dec`（與未來的 README / EXPERIMENT.md）；CLAUDE.md 保留專案內可編輯。不依賴 marketplace、保留短 `/dec`。
- **B + C**（不裝外掛、不 clone）— `CLAUDE.md` 永遠在 + 短 `/dec`，兩個都 `curl`。最輕量、但要更新時要手動重跑 `curl`。
- **只裝 A** — 一條指令搞定、自動更新。v3.0.0 起 plugin 只含 `/dec`（無 skill）、所以這個組合只給你斜線指令、沒有永遠在的規則。
- **A + B** — 外掛拿 `/dec`（namespaced）+ `CLAUDE.md` 拿永遠在的規則。v3.0.0 起職責清楚：plugin 管 `/dec`、`CLAUDE.md` 管規則、不再重疊。

## 搭配 Cursor 使用

本倉庫附 [`.cursor/rules/karpathy-guidelines.mdc`](.cursor/rules/karpathy-guidelines.mdc)，設定為 `alwaysApply: true`。詳情見 [`CURSOR.md`](./CURSOR.md)。

## 為什麼新版這麼短 — 而 A/B 實證告訴我們什麼

最初的論證（基於讀 Opus 4.7 系統提示詞）：v1 大部分規則已被內建提示詞涵蓋、重述會稀釋訊號。完整 v1 → v2 逐行對照見 [`archived/v1/NOTE.md`](./archived/v1/NOTE.md#detailed-v1--v2-diff-for-claudemd)（英文）。

這是觀察判斷、不是量測。所以 2026 年 5 月我們跑了小型 A/B 實證：

- 3 組：無 CLAUDE.md / v1 上游版（65 行）/ v2 我們版（19 行）
- 4 個誘發 Karpathy 痛點的 toy task + 最區分維度 T1 ambiguous-bug 加碼到每組 N=10
- 受測模型：Opus 4.7；盲判官（blind judge）：Sonnet 4.6

**結果：三組沒有統計顯著差異。** T1 加碼到每組 N=10 後三組全部 7/10 正確、Fisher exact p = 1.000。30 次 runs 中 **0 次**在編輯前問澄清（clarification）——不論哪版規則都沒能可靠觸發「停下來問」。

誠實版結論：在這個 toy task 規模上、CLAUDE.md（不論哪版）對 Opus 4.7 行為的邊際效應**小到 N=10 測不出來**。**任選一版皆可；使用者端的宣告式描述方式（user-side declarative framing，就是 `/dec` 在做的事）槓桿可能比規則檔本身大。**

完整資料、scripts、caveats、以及 Phase 1 (N=3) 一度看起來「v1 顯著優於 v2」最後被 Phase 2 (N=10) 攤平的過程：[`EXPERIMENT.md`](./EXPERIMENT.md)（英文）。

## 與上游的關係

本倉庫是 [`forrestchang/andrej-karpathy-skills`](https://github.com/forrestchang/andrej-karpathy-skills) 的繁體中文（台灣）在地化 fork，為 Claude Code Opus 4.7 時代更新內容。Plugin / marketplace 名稱刻意與上游一致；README 為雙語（英文 + 繁體中文）。

## 授權

MIT
