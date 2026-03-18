---
name: salt-review
version: 1.0.0
description: |
  Review Salt states for idempotency, targeting correctness, Pillar/Grain
  usage, Vault secret integration, and Fluent-Bit conflict detection.
  Infrascope-specific infrastructure review.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - AskUserQuestion
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

## Preamble (run first)

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || .claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
mkdir -p ~/.gstack/sessions
touch ~/.gstack/sessions/"$PPID"
_SESSIONS=$(find ~/.gstack/sessions -mmin -120 -type f 2>/dev/null | wc -l | tr -d ' ')
find ~/.gstack/sessions -mmin +120 -type f -delete 2>/dev/null || true
_CONTRIB=$(~/.claude/skills/gstack/bin/gstack-config get gstack_contributor 2>/dev/null || true)
_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "BRANCH: $_BRANCH"
_LAKE_SEEN=$([ -f ~/.gstack/.completeness-intro-seen ] && echo "yes" || echo "no")
echo "LAKE_INTRO: $_LAKE_SEEN"
```

If output shows `UPGRADE_AVAILABLE <old> <new>`: read `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` and follow the "Inline upgrade flow" (auto-upgrade if configured, otherwise AskUserQuestion with 4 options, write snooze state if declined). If `JUST_UPGRADED <from> <to>`: tell user "Running gstack v{to} (just updated!)" and continue.

If `LAKE_INTRO` is `no`: Before continuing, introduce the Completeness Principle.
Tell the user: "gstack follows the **Boil the Lake** principle — always do the complete
thing when AI makes the marginal cost near-zero. Read more: https://garryslist.org/posts/boil-the-ocean"
Then offer to open the essay in their default browser:

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

Only run `open` if the user says yes. Always run `touch` to mark as seen. This only happens once.

## AskUserQuestion Format

**ALWAYS follow this structure for every AskUserQuestion call:**
1. **Re-ground:** State the project, the current branch (use the `_BRANCH` value printed by the preamble — NOT any branch from conversation history or gitStatus), and the current plan/task. (1-2 sentences)
2. **Simplify:** Explain the problem in plain English a smart 16-year-old could follow. No raw function names, no internal jargon, no implementation details. Use concrete examples and analogies. Say what it DOES, not what it's called.
3. **Recommend:** `RECOMMENDATION: Choose [X] because [one-line reason]` — always prefer the complete option over shortcuts (see Completeness Principle). Include `Completeness: X/10` for each option. Calibration: 10 = complete implementation (all edge cases, full coverage), 7 = covers happy path but skips some edges, 3 = shortcut that defers significant work. If both options are 8+, pick the higher; if one is ≤5, flag it.
4. **Options:** Lettered options: `A) ... B) ... C) ...` — when an option involves effort, show both scales: `(human: ~X / CC: ~Y)`

Assume the user hasn't looked at this window in 20 minutes and doesn't have the code open. If you'd need to read the source to understand your own explanation, it's too complex.

Per-skill instructions may add additional formatting rules on top of this baseline.

## Completeness Principle — Boil the Lake

AI-assisted coding makes the marginal cost of completeness near-zero. When you present options:

- If Option A is the complete implementation (full parity, all edge cases, 100% coverage) and Option B is a shortcut that saves modest effort — **always recommend A**. The delta between 80 lines and 150 lines is meaningless with CC+gstack. "Good enough" is the wrong instinct when "complete" costs minutes more.
- **Lake vs. ocean:** A "lake" is boilable — 100% test coverage for a module, full feature implementation, handling all edge cases, complete error paths. An "ocean" is not — rewriting an entire system from scratch, adding features to dependencies you don't control, multi-quarter platform migrations. Recommend boiling lakes. Flag oceans as out of scope.
- **When estimating effort**, always show both scales: human team time and CC+gstack time. The compression ratio varies by task type — use this reference:

| Task type | Human team | CC+gstack | Compression |
|-----------|-----------|-----------|-------------|
| Boilerplate / scaffolding | 2 days | 15 min | ~100x |
| Test writing | 1 day | 15 min | ~50x |
| Feature implementation | 1 week | 30 min | ~30x |
| Bug fix + regression test | 4 hours | 15 min | ~20x |
| Architecture / design | 2 days | 4 hours | ~5x |
| Research / exploration | 1 day | 3 hours | ~3x |

- This principle applies to test coverage, error handling, documentation, edge cases, and feature completeness. Don't skip the last 10% to "save time" — with AI, that 10% costs seconds.

**Anti-patterns — DON'T do this:**
- BAD: "Choose B — it covers 90% of the value with less code." (If A is only 70 lines more, choose A.)
- BAD: "We can skip edge case handling to save time." (Edge case handling costs minutes with CC.)
- BAD: "Let's defer test coverage to a follow-up PR." (Tests are the cheapest lake to boil.)
- BAD: Quoting only human-team effort: "This would take 2 weeks." (Say: "2 weeks human / ~1 hour CC.")

## Contributor Mode

If `_CONTRIB` is `true`: you are in **contributor mode**. You're a gstack user who also helps make it better.

**At the end of each major workflow step** (not after every single command), reflect on the gstack tooling you used. Rate your experience 0 to 10. If it wasn't a 10, think about why. If there is an obvious, actionable bug OR an insightful, interesting thing that could have been done better by gstack code or skill markdown — file a field report. Maybe our contributor will help make us better!

**Calibration — this is the bar:** For example, `$B js "await fetch(...)"` used to fail with `SyntaxError: await is only valid in async functions` because gstack didn't wrap expressions in async context. Small, but the input was reasonable and gstack should have handled it — that's the kind of thing worth filing. Things less consequential than this, ignore.

**NOT worth filing:** user's app bugs, network errors to user's URL, auth failures on user's site, user's own JS logic bugs.

**To file:** write `~/.gstack/contributor-logs/{slug}.md` with **all sections below** (do not truncate — include every section through the Date/Version footer):

```
# {Title}

Hey gstack team — ran into this while using /{skill-name}:

**What I was trying to do:** {what the user/agent was attempting}
**What happened instead:** {what actually happened}
**My rating:** {0-10} — {one sentence on why it wasn't a 10}

## Steps to reproduce
1. {step}

## Raw output
```
{paste the actual error or unexpected output here}
```

## What would make this a 10
{one sentence: what gstack should have done differently}

**Date:** {YYYY-MM-DD} | **Version:** {gstack version} | **Skill:** /{skill}
```

Slug: lowercase, hyphens, max 60 chars (e.g. `browse-js-no-await`). Skip if file already exists. Max 3 reports per session. File inline and continue — don't stop the workflow. Tell user: "Filed gstack field report: {title}"

# Salt State Review

You are reviewing Salt states for the Infrascope platform — a single-tenant bare metal/VM/HPC environment on Rocky Linux 9.

---

## Step 1: Identify changed Salt files

```bash
git diff origin/develop --name-only | grep -E '\.(sls|jinja|yml)'
```

If no Salt-related files changed, output: **"No Salt files in this diff — nothing to review."** and stop.

---

## Step 2: Read all changed files

Read each changed Salt file completely. Also read:
- `salt/top.sls` — to understand state assignment
- `salt/pillar/top.sls` — to understand pillar assignment
- Any referenced pillar files

---

## Step 3: Idempotency audit

For each changed .sls file, check:

1. **cmd.run / cmd.script** — must have `creates`, `unless`, or `onlyif` guard. Flag any without.
2. **file.managed** — must have `source`, correct `user`/`group`/`mode`. Check `replace: False` where needed.
3. **pkg.installed** — verify package names exist in Rocky Linux 9 repos.
4. **service.running** — must have `enable: True` and `watch` or `require` for config changes.
5. **State ordering** — verify `require`, `watch`, `onchanges` chains. Draw the dependency order.

Output format:
```
IDEMPOTENCY AUDIT: N files checked, M issues found

CRITICAL:
- [file:state_id] cmd.run without guard — runs every highstate
  Fix: add `unless: test -f /path/to/expected/output`

INFORMATIONAL:
- [file:state_id] file.managed missing explicit mode
  Fix: add `mode: '0644'`
```

---

## Step 4: Targeting audit

For each state:
1. Check if targeting uses Grains or Pillars (good) vs hardcoded minion IDs (bad)
2. Verify the targeting grain/pillar actually exists in pillar data
3. Check for overly broad targeting (`'*'`) on destructive operations

---

## Step 5: Security audit

Check:
- Secrets (passwords, tokens, API keys, certificates) must come from Vault via `vault.read_secret`, not from Pillar data or hardcoded values
- SSH keys must have `mode: '0600'`
- Certificate files must have `mode: '0644'` (public) or `mode: '0600'` (private)
- Certificate paths should use Vault PKI integration, not static files
- Fluent-Bit config changes must not conflict with existing SSL/SSH setup (known Infrascope pain point — check for port conflicts, certificate path overlaps)

---

## Step 6: Pillar/Grain consistency

1. Verify every `pillar.get()` or `salt['pillar.get']()` reference has a corresponding entry in the pillar data
2. Check for pillar keys that exist but are never referenced (dead config)
3. Verify grain-based conditionals reference grains that actually exist on target minions

---

## Step 7: Output findings

Format same as `/review`:

```
Salt State Review: N issues (X critical, Y informational)

**CRITICAL** (blocking):
- [file:state_id] Problem description
  Fix: suggested fix

**Issues** (non-blocking):
- [file:state_id] Problem description
  Fix: suggested fix
```

If no issues found: `Salt State Review: No issues found. N files reviewed.`

For each CRITICAL issue, use AskUserQuestion with options:
A) Fix it now
B) Acknowledge risk
C) False positive — skip
