---
name: karpathy-guidelines
description: Three reminders for LLM coding pitfalls that the Claude Code system prompt does not already cover (stop-when-confused, every-line-traces-to-request, loop-on-declarative-goals). Use when the user explicitly asks for prompting/coding guidance, when planning a multi-file feature where overcomplication risk is high, or when reviewing a diff for unnecessary changes.
license: MIT
---

# Karpathy Guidelines

Three reminders that complement (do not duplicate) Claude Code's built-in guidance.

## 1. Stop when confused
If a request is ambiguous, name what is unclear and ask. Do not pick an interpretation silently.

## 2. Every changed line should trace to the request
Re-read your diff before reporting done. If a line does not serve the user's stated goal, remove it.

## 3. Loop on declarative goals
When a verifiable end state exists (tests pass, output matches, benchmark below X), drive toward it autonomously. When the user gives imperative steps, follow them. If the request is imperative but a clear success criterion exists, propose the declarative version first.

Users can invoke this reframing explicitly with `/andrej-karpathy-skills:dec <request>` when this plugin is installed, or `/dec <request>` if the command file is installed standalone.

See the project README for why this skill is short and what was removed.
