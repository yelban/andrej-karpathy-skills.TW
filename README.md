# Karpathy-Inspired Claude Code Guidelines

A small `CLAUDE.md` that complements Claude Code's built-in guidance, derived from [Andrej Karpathy's observations](https://x.com/karpathy/status/2015883857489522876) on LLM coding pitfalls.

English | [繁體中文（台灣）](./README.zh-TW.md)

## Status (May 2026)

Claude Code's system prompt (Opus 4.7 / Sonnet 4.6 era) now includes most of the generic "don't over-engineer / make surgical changes / don't add speculative features" guidance that earlier versions of this skill provided. **This version intentionally keeps only what the system prompt still does not cover, and reframes the "leverage" point as a user-side prompting guide.**

The earlier full-rules version lives in [`archived/v1/`](./archived/v1/) for reference.

## What the user does — the actual leverage

**This is the most important section in this README.** Our [empirical A/B test](./EXPERIMENT.md) (May 2026) found that the three reminders below had no measurable effect on Opus 4.7's behavior (Fisher exact p=1.00 at N=10 per cell on the most discriminating task). The user-side framing covered here is independent of the model — it shifts what you get back regardless of which LLM is on the other side.

Karpathy's strongest observation is **a user-side discipline**, not something the assistant self-enforces:

> "LLMs are exceptionally good at looping until they meet specific goals... Don't tell it what to do, give it success criteria and watch it go."

To unlock this in your own workflow:

### Convert imperative → declarative

| Imperative (weak leverage) | Declarative (strong leverage) |
|---|---|
| "Add input validation" | "Write failing tests for these invalid inputs, then make them pass" |
| "Fix the bug" | "Write a test that reproduces the bug, then make it pass — other tests must still pass" |
| "Make it faster" | "Reduce p95 latency under this load to <X ms; benchmark with `scripts/bench.sh`" |
| "Refactor X" | "Refactor X without changing observable behavior; existing tests must still pass" |

### Give the assistant the means to verify

Together with the goal, hand it the verification tool: a test command, a benchmark script, a lint command, a browser MCP for visual checks. Then leave it to iterate.

### When to use which

- **Declarative**: features with observable outcomes, bug fixes, performance work, refactors with test coverage.
- **Imperative**: exploratory edits, UI tweaks, prose, anything where "done" is subjective.

### `/dec`: the boundary-setter that makes `/goal` actually converge

Two slash commands map onto Karpathy's two verbs — "give it success criteria" and "watch it go":

| | `/dec` (this repo) | `/goal` (built into Claude Code v2.1.139+) |
|---|---|---|
| Phase | **Before action**: rewrites a vague request into a contract | **During action**: keeps Claude turning until the contract is met |
| Action | Rewrites your input; **does not implement yet** | After each turn a small fast model evaluates whether the contract holds; if not, Claude starts another turn automatically |
| Persistence | One-shot transformation; you confirm before execution | Session-scoped until `/goal clear` |
| Evaluator | **You** (review the contract before execution) | **Haiku** (yes/no judge reading the transcript) |
| Karpathy verb | "give it success criteria" | "watch it go" |

#### `/dec` alone

`dec` is short for **declarative**. The command reframes a command-style request into a contract; you confirm before anything is implemented.

```
/dec fix the login flicker on first load
```

Returns success criteria (e.g. "Playwright screenshot diff < 2px across 10 runs"), a verification command, and explicit non-goals — plus a ready-to-use `/goal` condition string that compiles the success criteria and non-goals into a single AND-joined expression you can paste directly. If the task is too subjective or too small, it replies "not applicable — just do it" instead of forcing a conversion. Good for one-shot prompts where you want the declarative discipline without committing to autonomous looping (or when you're on Cursor / an older Claude Code without `/goal`).

#### `/dec` as the boundary-setter for `/goal`

`/goal` is only as good as the condition string you feed it. Vague conditions never converge:

```
❌ /goal "make the login page not flicker"
   How does Haiku verify "no flicker"? Watch screenshots? Read console?
   The evaluator answers always-yes or always-no — the loop never converges.

✅ /dec fix the login flicker on first load
   →  Success:   Playwright screenshot diff < 2px across 10 runs
      Verify:    npx playwright test login-flicker.spec.ts
      Non-goals: do not refactor the login component; do not touch auth

✅ /goal "npx playwright test login-flicker.spec.ts passes AND
          the login component file is unchanged from baseline"
   Haiku reads pytest output from the transcript and judges deterministically.
   The loop actually converges.
```

`/dec` enforces three boundaries that `/goal` alone cannot:

1. **Machine-checkable success conditions** — "diff < 2px", "10 passed", "p95 < X ms" map cleanly to evaluator yes/no.
2. **A verification command embedded in the contract** — forces Claude to actually run the check, not statically reason "this should work now". (Patching-without-running was a real failure mode in our T4 declarative-loop test.)
3. **Explicit non-goals** — `/goal`'s condition string is compound: `"X passes AND test files unchanged AND no new files in src/legacy/"`. The evaluator checks each clause.

#### The full pipeline

```
1. /dec <vague request>            ← contract + a pre-compiled /goal condition
2. you review the contract         ← human confirms direction
3. paste the /goal line from #1    ← Haiku takes over as judge
4. Claude loops to convergence     ← Karpathy's "watch it go"
```

#### Why this is the real leverage

User-side discipline that does not depend on which model is on the other side. Our [empirical A/B test](./EXPERIMENT.md) found CLAUDE.md rules had no measurable effect on Opus 4.7's coding behavior — but a well-formed contract plus an autonomous evaluation loop is leverage **you** control, not leverage you hope the model picks up. It also doesn't depreciate when Opus 4.7 → 4.8 → 5.0; the `/dec` template and `/goal` evaluator stay the same.

> **Note on invocation:** when installed via the plugin (Option A below), Claude Code namespaces the command to `/andrej-karpathy-skills:dec`. For the short `/dec` form, install the command file manually (Option C). The built-in `/goal` is always available regardless of install method.

> **Note on the `/goal` evaluator:** `/goal` sends each turn's transcript to Claude Code's built-in "small fast model" slot, which [defaults to Haiku](https://code.claude.com/docs/en/goal.md). There is no `/goal`-specific model override; the only way to swap it is to redirect the slot globally with the `ANTHROPIC_DEFAULT_HAIKU_MODEL` environment variable ([model config docs](https://code.claude.com/docs/en/model-config.md)), which changes the `haiku` alias everywhere — most setups never need to touch this.

## What the assistant gets

Three reminders, copied verbatim from [`CLAUDE.md`](./CLAUDE.md). Kept because they're cheap and may help on different models or longer contexts, but the empirical marginal effect on Opus 4.7 is small (see [`EXPERIMENT.md`](./EXPERIMENT.md)).

1. **Stop when confused** — if a request is ambiguous, name what is unclear and ask; do not pick an interpretation silently.
2. **Every changed line should trace to the request** — re-read your diff before reporting done; if a line does not serve the user's stated goal, remove it.
3. **Loop on declarative goals** — when a verifiable end state exists, drive toward it autonomously.

That is the entire instruction file. The other pitfalls Karpathy named (overcomplication, drive-by refactors, speculative features, dead-code creep, removing comments the model "doesn't like") are already addressed by Claude Code's default system prompt; duplicating them only dilutes signal.

## Install

The three reminders and the `/dec` command are independent — pick any combination.

| | Three reminders | `/dec` command | Mechanism |
|---|---|---|---|
| **A. Plugin** | — (skill removed in v3.0.0; use B / C / D for `CLAUDE.md`) | `/andrej-karpathy-skills:dec` (namespaced) | auto-updates via marketplace |
| **B. `CLAUDE.md`** | always-on in system prompt | — | per-project file, manual `curl` |
| **C. Manual command** | — | `/dec` (short) | global or per-project file, manual `curl` |
| **D. `git clone`** | `cp` whole file *or* `sed`-append rules | `/dec` (short, symlinked) | `git pull` updates `/dec`; `CLAUDE.md` is your editable copy |

**Option A: Claude Code plugin** — installs only the `/dec` command (namespaced), auto-updates via marketplace. The skill that wrapped the three reminders was removed in v3.0.0 after the empirical A/B test showed it had no measurable effect (see [`EXPERIMENT.md`](./EXPERIMENT.md)). For the always-on rules, use Option B, C, or D below.

```
/plugin marketplace add yelban/andrej-karpathy-skills.TW
/plugin install andrej-karpathy-skills@karpathy-skills
```

**Option B: `CLAUDE.md` per-project** — three reminders always loaded for that project.

```bash
curl -o CLAUDE.md https://raw.githubusercontent.com/yelban/andrej-karpathy-skills.TW/main/CLAUDE.md
```

**Option C: Manual `/dec` command** — short invocation without the plugin namespace.

```bash
# Global (all projects)
mkdir -p ~/.claude/commands
curl -o ~/.claude/commands/dec.md https://raw.githubusercontent.com/yelban/andrej-karpathy-skills.TW/main/commands/dec.md

# Or per-project
mkdir -p .claude/commands
curl -o .claude/commands/dec.md https://raw.githubusercontent.com/yelban/andrej-karpathy-skills.TW/main/commands/dec.md
```

**Option D: `git clone` + symlink** — `/dec` auto-updates via `git pull`; `CLAUDE.md` is copied as a starting point you can freely edit per project.

```bash
# 1. Clone once (any location works; example uses ~/.claude/external/)
mkdir -p ~/.claude/external
git clone https://github.com/yelban/andrej-karpathy-skills.TW \
  ~/.claude/external/andrej-karpathy-skills.TW

# 2. Symlink the short /dec command globally (the command itself is stateless,
#    so a symlink that follows upstream is what you want)
mkdir -p ~/.claude/commands
ln -sf ~/.claude/external/andrej-karpathy-skills.TW/commands/dec.md \
  ~/.claude/commands/dec.md

# 3. Per-project CLAUDE.md — choose ONE.
#    NOT a symlink: a project's CLAUDE.md belongs to the project,
#    so you copy or append, then keep editing it yourself.

# (a) Project doesn't have a CLAUDE.md yet — copy the file as a starting point:
cp ~/.claude/external/andrej-karpathy-skills.TW/CLAUDE.md ./CLAUDE.md

# (b) Project already has its own CLAUDE.md — append just the three rules:
sed -n '/^## Stop when confused/,$p' \
  ~/.claude/external/andrej-karpathy-skills.TW/CLAUDE.md >> ./CLAUDE.md

# To update /dec and pull future README / EXPERIMENT.md updates:
cd ~/.claude/external/andrej-karpathy-skills.TW && git pull
# CLAUDE.md does NOT auto-update — re-run (a) or (b) only if you want to.
```

> The `sed` extraction starts at the first `## Stop when confused` heading, skipping the title and intro paragraph. The trailing `/dec` invocation note is included — useful as a footer in your project's CLAUDE.md.

### Recommended combinations

- **D alone** — clone once, symlink `/dec`, copy CLAUDE.md as a starting point. `git pull` updates `/dec` (and future README / EXPERIMENT.md); CLAUDE.md stays editable per project. No marketplace, short `/dec`. Recommended for power users.
- **B + C** (no plugin, no clone) — `CLAUDE.md` always-on + short `/dec`, both via `curl`. Smallest footprint, but updates are manual (re-run the `curl` commands).
- **A only** — single install command, auto-updates. Since v3.0.0 the plugin is `/dec`-only (no skill), so this combination gives you the slash command without any always-on rules.
- **A + B** — plugin for `/dec` (namespaced) + `CLAUDE.md` for always-on rules. Clean separation since v3.0.0: plugin owns `/dec`, `CLAUDE.md` owns rules, no overlap.

## Using with Cursor

The repository includes [`.cursor/rules/karpathy-guidelines.mdc`](.cursor/rules/karpathy-guidelines.mdc) with `alwaysApply: true`. See [`CURSOR.md`](./CURSOR.md) for setup details and how it differs from the Claude Code install.

## Why this version is shorter — and what an A/B test actually showed

The original rationale (based on reading Opus 4.7's system prompt) was: most of v1's rules now live in the built-in system prompt, so duplicating them dilutes signal. Full v1 → v2 diff in [`archived/v1/NOTE.md`](./archived/v1/NOTE.md#detailed-v1--v2-diff-for-claudemd).

That argument was a judgment, not a measurement. So in May 2026 we ran a small empirical A/B:

- 3 cells: no CLAUDE.md / v1 upstream (65 lines) / v2 ours (19 lines)
- 4 toy tasks targeting Karpathy's named pitfalls + N=10 follow-up on the most discriminating task (T1 ambiguous-bug)
- Opus 4.7 subject, Sonnet 4.6 blind judge

**Result: no statistically significant difference between any cells.** On T1 with N=10 per cell, all three landed at 7/10 correct (Fisher exact p = 1.000 pairwise). 0/30 runs asked clarification before editing — none of the rule sets reliably triggered "stop and ask" behavior on a task that looked superficially singular.

The honest takeaway: at the toy-task scale we tested, the marginal effect of CLAUDE.md (any flavor) on Opus 4.7's behavior is too small to measure with N=10. **Use whichever flavor you prefer; the user-side declarative framing (`/dec`) likely matters more than the rules file itself.**

Full data, scripts, caveats, and the Phase 1 (N=3) result that initially looked like "v1 wins" before Phase 2 (N=10) flattened it: [`EXPERIMENT.md`](./EXPERIMENT.md).

## Relationship to upstream

This repository is a Traditional Chinese (Taiwan) localization fork of [`forrestchang/andrej-karpathy-skills`](https://github.com/forrestchang/andrej-karpathy-skills), updated for the Claude Code Opus 4.7 era. Plugin / marketplace names intentionally match upstream; the README is bilingual (English + 繁體中文).

## License

MIT
