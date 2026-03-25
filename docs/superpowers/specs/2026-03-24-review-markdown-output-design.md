# Review Markdown Output — Design Spec

## Problem

Plan review skills (plan-ceo-review, plan-eng-review) output their full narrative to the terminal only. When the session ends, the review findings are lost. Only structured JSONL metadata is persisted via `gstack-review-log`. Users cannot reference review findings after the session closes.

## Scope

- **In scope**: plan-ceo-review, plan-eng-review
- **Out of scope**: review (PR review), retro, qa-only, and all other skills

## Design

### File Location & Naming

```
docs/reviews/{YYYY-MM-DD}-{skill}-{feature-slug}.md
```

Examples:
- `docs/reviews/2026-03-24-plan-ceo-review-user-auth-redesign.md`
- `docs/reviews/2026-03-24-plan-eng-review-api-rate-limiting.md`

**Feature slug derivation** (in priority order):
1. The plan file being reviewed in this session (e.g. `2026-03-24-api-rate-limiting-design.md` → `api-rate-limiting`). If multiple plan files exist, use the one explicitly referenced during the review — do not guess.
2. Git branch name sanitized via `gstack-slug` (e.g. `feat/user-auth-redesign` → `user-auth-redesign`)
3. Fallback: `unknown-feature` (deterministic, no user prompt — the review should not block on a filename question)

**Slug normalization**: All slug derivation (project, branch, feature) uses `gstack-slug` or equivalent sanitization to match existing review log path conventions.

**Duplicate detection**: If a same-day review file already exists for the same skill and feature, append an incrementing counter: `-2`, `-3`, etc.

### Trigger

Automatic on review completion — at the final summary step, after all interactive review is done.

### Failure Policy

File write is **non-blocking**. If the primary write to `docs/reviews/` fails (permission error, path conflict, tool error, plan-mode restriction):
1. The review itself completes normally — JSONL logging is unaffected
2. Fallback write to `~/.gstack/projects/{project-slug}/reviews/{filename}.md` (guaranteed writable, outside repo)
3. If fallback also fails, warn the user and print the full report content to the terminal
4. The review is not considered failed due to a file write error

**Plan-mode note**: The preamble restricts plan-mode writes to the plan file only. The report write happens after the review completes (post-completion step), which is outside the plan-mode scope. If plan-mode restrictions still block the write, the fallback path at `~/.gstack/` is always available.

### Template System

A new `{{REVIEW_REPORT_OUTPUT}}` placeholder in `gen-skill-docs.ts` handles the common structure:

- YAML frontmatter generation
- File path construction (slug detection via `gstack-slug`, duplicate handling, directory creation)
- Header and footer
- Instructions to write the file via the Write tool

Each skill's `.tmpl` file places `{{REVIEW_REPORT_OUTPUT}}` at its completion step and defines skill-specific report sections inline.

### Tool Permissions

- **plan-eng-review**: Already has `Write` in allowed-tools. No change needed.
- **plan-ceo-review**: Add `Write` to allowed-tools. (It already instructs writing CEO plan files at `~/.gstack/projects/` but was missing the formal permission.)

### Common Structure

```markdown
---
date: {YYYY-MM-DD}
skill: {skill-name}
branch: {sanitized-branch-via-gstack-slug}
project: {project-slug}
feature: {feature-slug}
status: {approved|changes-requested}
reviewer: {claude|codex|other}
commit: {short-sha}
---

# {Skill Display Name} — {Feature Title} ({YYYY-MM-DD})

{skill-specific sections here}

---
*Review log: ~/.gstack/projects/{project-slug}/{branch}-reviews.jsonl*
*Reviewed at commit: {full-sha}*
```

### Status Mapping

The markdown report reuses the existing JSONL status values with this mapping:

| JSONL status | Markdown status | When |
|---|---|---|
| `clean` | `approved` | 0 unresolved decisions AND 0 critical gaps |
| `issues_open` | `changes-requested` | Any unresolved decisions or critical gaps |

Note: No `blocked` status. Reports are only written on review completion, so an interrupted review produces no report file. The JSONL log also only writes on completion (it is appended in the same completion step), so an interrupted review produces neither artifact.

### Reviewer Field

The `reviewer` frontmatter field is set dynamically based on runtime context:
- `claude` when running in Claude Code
- `codex` when running via Codex integration
- If an outside-voice was used during the review, add `outside_voice: true` to frontmatter

### CEO Review Sections

```markdown
## Executive Summary
<!-- 2-3 sentence verdict: what was reviewed, what mode, overall assessment -->

## Scope Decisions
| Proposal | Decision | Rationale |
|----------|----------|-----------|
| ...      | Accepted / Deferred / Rejected | ... |

## Vision & Strategy
<!-- What's the 10-star version? How does this fit the product direction? -->

## Risks & Concerns
<!-- Premises challenged, assumptions questioned, blind spots identified -->

## Recommendations
<!-- Prioritized list of concrete next steps -->

## Action Items
- [ ] ...
```

Additional frontmatter field: `mode: {SCOPE_EXPANSION|SELECTIVE_EXPANSION|HOLD_SCOPE|SCOPE_REDUCTION}`

### Eng Review Sections

```markdown
## Executive Summary
<!-- 2-3 sentence verdict: architecture soundness, readiness to implement -->

## Architecture Analysis
<!-- Data flow, component boundaries, dependency assessment -->

## Edge Cases & Gaps
| Issue | Severity | Description | Recommendation |
|-------|----------|-------------|----------------|
| ...   | Critical/High/Medium | ... | ... |

## Test Coverage Assessment
<!-- What's covered, what's missing, coverage strategy -->

## Performance Considerations
<!-- Bottlenecks identified, scaling concerns, optimization opportunities -->

## Recommendations
<!-- Prioritized list of concrete changes needed -->

## Action Items
- [ ] ...
```

Additional frontmatter fields: `unresolved: {count}`, `critical_gaps: {count}`

## Files to Modify

1. **`scripts/gen-skill-docs.ts`** — Add `{{REVIEW_REPORT_OUTPUT}}` placeholder resolution
2. **`plan-ceo-review/SKILL.md.tmpl`** — Add `Write` to allowed-tools; add report output step with CEO-specific sections
3. **`plan-eng-review/SKILL.md.tmpl`** — Add report output step with eng-specific sections
4. Regenerate all skill docs via `bun run build` (covers both Claude SKILL.md and Codex `.agents/skills/` outputs)
5. **`test/gen-skill-docs.test.ts`** — Add tests for: `{{REVIEW_REPORT_OUTPUT}}` placeholder resolution, duplicate filename counter logic, status mapping correctness

## Non-Goals

- No changes to the existing JSONL review log system (it continues as-is)
- No interactive "save to file?" prompt — always writes on completion
- No retroactive report generation from past review logs
