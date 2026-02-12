---
title: "Idempotent Scripts"
order: 9
description: "Every script is safe to re-run with OK/SKIP/TODO/!! feedback."
---

## Idempotent Scripts

### 35. Every Script Is Safe to Re-Run

Every script is safe to re-run. Use `[OK]`/`[SKIP]`/`[TODO]`/`[!!]` feedback pattern. Detect existing state before acting — check first, skip if done, help if broken. Never blow away existing work without checking. A script run twice should produce the same result as a script run once. This applies to setup scripts, deploy scripts, migration scripts, and pipeline scripts. The pattern:

- `[OK]` — action completed successfully
- `[SKIP]` — already done, nothing to do
- `[TODO]` — needs manual intervention, here's how
- `[!!]` — something went wrong, here's what to check

End with a summary: steps completed, steps skipped, issues found.
