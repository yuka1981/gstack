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
1. Plan file name if one exists (e.g. `2026-03-24-api-rate-limiting-design.md` → `api-rate-limiting`)
2. Git branch name as fallback (e.g. `feat/user-auth-redesign` → `user-auth-redesign`)
3. AskUserQuestion if neither is available: "What feature is this review for? (used for the report filename)"

**Duplicate detection**: If a same-day review file already exists for the same skill and feature, append an incrementing counter: `-2`, `-3`, etc.

### Trigger

Automatic on review completion — at the final summary step, after all interactive review is done. No user prompt needed.

### Template System

A new `{{REVIEW_REPORT_OUTPUT}}` placeholder in `gen-skill-docs.ts` handles the common structure:

- YAML frontmatter generation
- File path construction (slug detection, duplicate handling, directory creation)
- Header and footer
- Instructions to write the file via the Write tool

Each skill's `.tmpl` file places `{{REVIEW_REPORT_OUTPUT}}` at its completion step and defines skill-specific report sections inline.

### Common Structure

```markdown
---
date: {YYYY-MM-DD}
skill: {skill-name}
branch: {git-branch}
project: {project-slug}
feature: {feature-slug}
status: {approved|changes-requested|blocked}
reviewer: claude
commit: {short-sha}
---

# {Skill Display Name} — {Feature Title} ({YYYY-MM-DD})

{skill-specific sections here}

---
*Review log: ~/.gstack/projects/{slug}/{branch}-reviews.jsonl*
*Reviewed at commit: {full-sha}*
```

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
2. **`plan-ceo-review/SKILL.md.tmpl`** — Add report output step with CEO-specific sections
3. **`plan-eng-review/SKILL.md.tmpl`** — Add report output step with eng-specific sections
4. Regenerate `plan-ceo-review/SKILL.md` and `plan-eng-review/SKILL.md`

## Non-Goals

- No changes to the existing JSONL review log system (it continues as-is)
- No interactive "save to file?" prompt — always writes on completion
- No retroactive report generation from past review logs
