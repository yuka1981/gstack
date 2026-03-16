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

## Update Check (run first)

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || .claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
```

If output shows `UPGRADE_AVAILABLE <old> <new>`: read `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` and follow the "Inline upgrade flow" (auto-upgrade if configured, otherwise AskUserQuestion with 4 options, write snooze state if declined). If `JUST_UPGRADED <from> <to>`: tell user "Running gstack v{to} (just updated!)" and continue.

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
