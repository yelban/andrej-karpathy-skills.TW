---
description: Reframe an imperative request as declarative (success criteria + verification + non-goals). Do not implement yet.
---

`/dec` is short for **declarative**. Convert the request below into a declarative contract. **Do not implement.**

Output four blocks:

1. **成功條件 (Success criteria)**：可驗證的端狀態（測試通過 / 輸出比對 / 效能門檻 / lint clean，擇一或多）
2. **驗證指令 (Verification command)**：具體可執行的命令或檢查方式（例：`bun test foo.spec.ts`、`scripts/bench.sh`）
3. **非目標 (Non-goals)**：本次明確不做什麼，避免 scope creep
4. **Ready-to-use `/goal` condition**：把 #1 與 #3 合成一條 Claude Code 內建 `/goal` 可吃的字串，複合條件用 AND 串接。例：
   ```
   /goal "npx playwright test login.spec.ts passes AND login component file unchanged from baseline"
   ```

輸出後等使用者確認。確認後使用者可選：

- 依契約自己 / 讓 Claude 直接實作（standalone 用法、適合 Cursor 或舊版 Claude Code）
- 複製 #4 的指令呼叫 `/goal`，讓 Claude Code 自動 loop 到達標（需 Claude Code v2.1.139+ 內建 `/goal`）

若任務本質主觀（UI 微調、文案、命名）或太小（typo、單行 rename），
回覆「不適用，建議直接做」、不要硬轉換。

---

Request: $ARGUMENTS
