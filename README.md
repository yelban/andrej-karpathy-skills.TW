# Karpathy-Inspired Claude Code Guidelines

A small `CLAUDE.md` that complements Claude Code's built-in guidance, derived from [Andrej Karpathy's observations](https://x.com/karpathy/status/2015883857489522876) on LLM coding pitfalls.

English | [繁體中文（台灣）](./README.zh-TW.md)

## Status (May 2026)

Claude Code's system prompt (Opus 4.7 / Sonnet 4.6 era) now includes most of the generic "don't over-engineer / make surgical changes / don't add speculative features" guidance that earlier versions of this skill provided. **This version intentionally keeps only what the system prompt still does not cover, and reframes the "leverage" point as a user-side prompting guide.**

The earlier full-rules version lives in [`archived/v1/`](./archived/v1/) for reference.

## What the assistant gets

Three reminders, copied verbatim from [`CLAUDE.md`](./CLAUDE.md):

1. **Stop when confused** — if a request is ambiguous, name what is unclear and ask; do not pick an interpretation silently.
2. **Every changed line should trace to the request** — re-read your diff before reporting done; if a line does not serve the user's stated goal, remove it.
3. **Loop on declarative goals** — when a verifiable end state exists, drive toward it autonomously.

That is the entire instruction file. The other pitfalls Karpathy named (overcomplication, drive-by refactors, speculative features, dead-code creep, removing comments the model "doesn't like") are already addressed by Claude Code's default system prompt; duplicating them only dilutes signal.

## What the user does — the actual leverage

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

### Explicit reframing: the `dec` slash command

The command converts the request into success criteria + verification command + non-goals **without implementing anything**. You confirm, then it executes. Use when you want the declarative discipline applied to a single prompt without changing how you write the rest.

```
/dec fix the login flicker on first load
```

Returns success criteria (e.g. "Playwright screenshot diff < 2px across 10 runs"), verification command, and explicit non-goals. If the task is too subjective or too small, it replies "not applicable — just do it" instead of forcing a conversion.

> **Note on invocation:** when installed via the plugin (Option A below), Claude Code namespaces the command to `/andrej-karpathy-skills:dec`. For the short `/dec` form, install the command file manually (Option C).

## Install

The three reminders and the `/dec` command are independent — pick any combination.

| | Three reminders | `/dec` command | Mechanism |
|---|---|---|---|
| **A. Plugin** | as skill (triggered when relevant) | `/andrej-karpathy-skills:dec` (namespaced) | auto-updates via marketplace |
| **B. `CLAUDE.md`** | always-on in system prompt | — | per-project file |
| **C. Manual command** | — | `/dec` (short) | global or per-project file |

**Option A: Claude Code plugin** — skill + namespaced command, both auto-update.

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

### Recommended combinations

- **B + C** (no plugin) — `CLAUDE.md` always-on + short `/dec`. Smallest footprint, no plugin marketplace dependency. Manual updates by re-running `curl`.
- **A only** — single install command, auto-updates, but `/dec` invocation is namespaced and the rules only apply when the skill triggers (not always-on).
- **A + B** — plugin for `/dec` (namespaced) + `CLAUDE.md` for always-on rules. Some redundancy (skill content overlaps `CLAUDE.md`) but harmless.

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
