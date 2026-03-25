# Review Markdown Output Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Plan review skills (plan-ceo-review, plan-eng-review) write structured markdown reports to `docs/reviews/` on completion, so review findings persist beyond the session.

**Architecture:** A new `{{REVIEW_REPORT_OUTPUT}}` placeholder in `gen-skill-docs.ts` generates the common report infrastructure (file path construction, frontmatter, failure handling). Each skill template adds it after the Review Log section and defines skill-specific report body sections inline.

**Tech Stack:** TypeScript (gen-skill-docs.ts), Markdown templates (SKILL.md.tmpl), Bun test runner

**Spec:** `docs/superpowers/specs/2026-03-24-review-markdown-output-design.md`

---

## File Structure

| File | Action | Responsibility |
|------|--------|----------------|
| `scripts/gen-skill-docs.ts` | Modify (~line 2813) | Add `REVIEW_REPORT_OUTPUT` resolver function + register in RESOLVERS map |
| `plan-ceo-review/SKILL.md.tmpl` | Modify (line 14, ~line 755) | Add `Write` to allowed-tools; add `{{REVIEW_REPORT_OUTPUT}}` + CEO-specific sections after Review Dashboard |
| `plan-eng-review/SKILL.md.tmpl` | Modify (~line 270) | Add `{{REVIEW_REPORT_OUTPUT}}` + eng-specific sections after Review Dashboard |
| `test/gen-skill-docs.test.ts` | Modify | Add tests for REVIEW_REPORT_OUTPUT placeholder resolution |
| All SKILL.md files | Regenerate | `bun run build` |

---

### Task 1: Add `generateReviewReportOutput` function to gen-skill-docs.ts

**Files:**
- Modify: `scripts/gen-skill-docs.ts:2813` (add function near other review functions)
- Modify: `scripts/gen-skill-docs.ts:2828` (register in RESOLVERS map)

- [ ] **Step 1: Write the `generateReviewReportOutput` function**

Add after the `generatePlanFileReviewReport` function (after line 1386). The function returns prompt instructions for writing the review report markdown file.

```typescript
function generateReviewReportOutput(ctx: TemplateContext): string {
  return `## Review Report Output

After completing the Review Readiness Dashboard and Plan File Review Report above,
write a structured markdown report file to persist the full review findings.

### Determine the report file path

1. **Feature slug**: Derive from the plan file being reviewed in this session
   (e.g. \\\`2026-03-24-api-rate-limiting-design.md\\\` → \\\`api-rate-limiting\\\`).
   If no plan file, sanitize the git branch name (strip \\\`feat/\\\`, \\\`fix/\\\`, etc. prefixes,
   replace non-alphanumeric with hyphens). If branch is \\\`main\\\` or \\\`master\\\`, use \\\`unknown-feature\\\`.

2. **Date**: Use today's date in YYYY-MM-DD format.

3. **File path**: \\\`docs/reviews/{date}-${ctx.skillName}-{feature-slug}.md\\\`

4. **Duplicate check**: If that file already exists, append \\\`-2\\\`, \\\`-3\\\`, etc.

5. **Create directory**: \\\`mkdir -p docs/reviews\\\` before writing.

### Determine status

Map from the review log status you just persisted:
- \\\`clean\\\` → \\\`approved\\\`
- \\\`issues_open\\\` → \\\`changes-requested\\\`

### Determine reviewer

- If running in Claude Code: \\\`claude\\\`
- If running via Codex: \\\`codex\\\`
- If an outside voice was used during the review, add \\\`outside_voice: true\\\` to frontmatter

### Write the report

Use the Write tool to create the file. If the write fails:
1. Try fallback path: \\\`~/.gstack/projects/$SLUG/reviews/{filename}.md\\\` (create dir with \\\`mkdir -p\\\`)
2. If fallback also fails, warn the user and print the full report to terminal
3. The review is NOT considered failed — this is non-blocking

**PLAN MODE NOTE:** This write happens after review completion (post-completion step).
If plan-mode restrictions block \\\`docs/reviews/\\\`, the fallback at \\\`~/.gstack/\\\` is always available.`;
}
```

- [ ] **Step 2: Register the resolver in the RESOLVERS map**

In the `RESOLVERS` object at line ~2800, add:

```typescript
  REVIEW_REPORT_OUTPUT: generateReviewReportOutput,
```

Add it after the `PLAN_FILE_REVIEW_REPORT` entry (line 2814).

- [ ] **Step 3: Verify it compiles**

Run: `bun run gen:skill-docs --dry-run 2>&1 | head -5`
Expected: No errors about unknown REVIEW_REPORT_OUTPUT (it's not used yet, so no output change)

- [ ] **Step 4: Commit**

```bash
git add scripts/gen-skill-docs.ts
git commit -m "feat: add REVIEW_REPORT_OUTPUT placeholder resolver in gen-skill-docs"
```

---

### Task 2: Add `Write` to plan-ceo-review allowed-tools

**Files:**
- Modify: `plan-ceo-review/SKILL.md.tmpl:14-20`

- [ ] **Step 1: Add Write to allowed-tools list**

In `plan-ceo-review/SKILL.md.tmpl`, the allowed-tools section is at lines 14-20:

```yaml
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - AskUserQuestion
  - WebSearch
```

Add `Write` after `Read`:

```yaml
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - Bash
  - AskUserQuestion
  - WebSearch
```

- [ ] **Step 2: Commit**

```bash
git add plan-ceo-review/SKILL.md.tmpl
git commit -m "feat: add Write to plan-ceo-review allowed-tools for report output"
```

---

### Task 3: Add review report output to plan-ceo-review template

**Files:**
- Modify: `plan-ceo-review/SKILL.md.tmpl:755` (after `{{PLAN_FILE_REVIEW_REPORT}}`)

- [ ] **Step 1: Add the placeholder and CEO-specific sections**

After line 755 (`{{PLAN_FILE_REVIEW_REPORT}}`), add:

```markdown
{{REVIEW_REPORT_OUTPUT}}

The report file MUST use this structure. Populate each section from your review findings:

```markdown
---
date: {YYYY-MM-DD}
skill: plan-ceo-review
branch: {branch}
project: {project-slug}
feature: {feature-slug}
status: {approved|changes-requested}
mode: {SCOPE_EXPANSION|SELECTIVE_EXPANSION|HOLD_SCOPE|SCOPE_REDUCTION}
reviewer: {claude|codex}
commit: {short-sha}
---

# CEO Review — {Feature Title} ({YYYY-MM-DD})

## Executive Summary
2-3 sentence verdict: what was reviewed, what mode was used, overall assessment.
Pull from the Completion Summary produced above.

## Scope Decisions
| Proposal | Decision | Rationale |
|----------|----------|-----------|
Fill from Step 0D scope proposals. For HOLD_SCOPE/SCOPE_REDUCTION, write "No scope changes — mode was {mode}."

## Vision & Strategy
Summarize the 10-star vision, dream state delta, and product direction findings.
For HOLD_SCOPE/SCOPE_REDUCTION, summarize the rigor/reduction rationale instead.

## Risks & Concerns
List premises challenged, assumptions questioned, blind spots identified.
Include any CRITICAL GAPs from the failure modes analysis.

## Recommendations
Prioritized list of concrete next steps from the review.

## Action Items
- [ ] List each actionable item from the review
```
```

- [ ] **Step 2: Commit**

```bash
git add plan-ceo-review/SKILL.md.tmpl
git commit -m "feat: add review report output step to plan-ceo-review template"
```

---

### Task 4: Add review report output to plan-eng-review template

**Files:**
- Modify: `plan-eng-review/SKILL.md.tmpl:270` (after `{{PLAN_FILE_REVIEW_REPORT}}`)

- [ ] **Step 1: Add the placeholder and eng-specific sections**

After line 270 (`{{PLAN_FILE_REVIEW_REPORT}}`), add:

```markdown
{{REVIEW_REPORT_OUTPUT}}

The report file MUST use this structure. Populate each section from your review findings:

```markdown
---
date: {YYYY-MM-DD}
skill: plan-eng-review
branch: {branch}
project: {project-slug}
feature: {feature-slug}
status: {approved|changes-requested}
unresolved: {count}
critical_gaps: {count}
reviewer: {claude|codex}
commit: {short-sha}
---

# Eng Review — {Feature Title} ({YYYY-MM-DD})

## Executive Summary
2-3 sentence verdict: architecture soundness, readiness to implement.
Pull from the Completion Summary produced above.

## Architecture Analysis
Summarize data flow, component boundaries, dependency assessment from the
architecture review section. Include any diagrams produced.

## Edge Cases & Gaps
| Issue | Severity | Description | Recommendation |
|-------|----------|-------------|----------------|
Fill from issues found across all review sections (architecture, code quality,
performance, test gaps). Use Critical/High/Medium severity levels.

## Test Coverage Assessment
Summarize the test diagram findings: what's covered, what's missing,
recommended coverage strategy.

## Performance Considerations
Bottlenecks identified, scaling concerns, optimization opportunities.
If no performance issues found, state "No performance concerns identified."

## Recommendations
Prioritized list of concrete changes needed, pulled from review findings.

## Action Items
- [ ] List each actionable item from the review
```
```

- [ ] **Step 2: Commit**

```bash
git add plan-eng-review/SKILL.md.tmpl
git commit -m "feat: add review report output step to plan-eng-review template"
```

---

### Task 5: Add tests for REVIEW_REPORT_OUTPUT placeholder

**Files:**
- Modify: `test/gen-skill-docs.test.ts`

- [ ] **Step 1: Add test for placeholder resolution in generated SKILL.md files**

Add to the `describe('gen-skill-docs', ...)` block:

```typescript
  test('plan-ceo-review SKILL.md contains review report output section', () => {
    const content = fs.readFileSync(path.join(ROOT, 'plan-ceo-review', 'SKILL.md'), 'utf-8');
    expect(content).toContain('## Review Report Output');
    expect(content).toContain('docs/reviews/');
    expect(content).toContain('plan-ceo-review');
    // Verify CEO-specific sections are present
    expect(content).toContain('## Scope Decisions');
    expect(content).toContain('## Vision & Strategy');
  });

  test('plan-eng-review SKILL.md contains review report output section', () => {
    const content = fs.readFileSync(path.join(ROOT, 'plan-eng-review', 'SKILL.md'), 'utf-8');
    expect(content).toContain('## Review Report Output');
    expect(content).toContain('docs/reviews/');
    expect(content).toContain('plan-eng-review');
    // Verify eng-specific sections are present
    expect(content).toContain('## Architecture Analysis');
    expect(content).toContain('## Test Coverage Assessment');
  });

  test('REVIEW_REPORT_OUTPUT placeholder uses skill name from context', () => {
    // Verify the placeholder resolves differently per skill
    const ceoContent = fs.readFileSync(path.join(ROOT, 'plan-ceo-review', 'SKILL.md'), 'utf-8');
    const engContent = fs.readFileSync(path.join(ROOT, 'plan-eng-review', 'SKILL.md'), 'utf-8');
    // CEO template should reference plan-ceo-review in the file path pattern
    expect(ceoContent).toContain('plan-ceo-review');
    // Eng template should reference plan-eng-review in the file path pattern
    expect(engContent).toContain('plan-eng-review');
  });

  test('review report output includes status mapping', () => {
    const content = fs.readFileSync(path.join(ROOT, 'plan-ceo-review', 'SKILL.md'), 'utf-8');
    expect(content).toContain('approved');
    expect(content).toContain('changes-requested');
    expect(content).toContain('clean');
    expect(content).toContain('issues_open');
  });

  test('review report output includes fallback path', () => {
    const content = fs.readFileSync(path.join(ROOT, 'plan-ceo-review', 'SKILL.md'), 'utf-8');
    expect(content).toContain('~/.gstack/projects/$SLUG/reviews/');
  });
```

- [ ] **Step 2: Run tests to verify they fail (templates not yet regenerated)**

Run: `bun test test/gen-skill-docs.test.ts`
Expected: New tests FAIL because SKILL.md hasn't been regenerated yet with the placeholder.

- [ ] **Step 3: Commit test**

```bash
git add test/gen-skill-docs.test.ts
git commit -m "test: add tests for REVIEW_REPORT_OUTPUT placeholder resolution"
```

---

### Task 6: Regenerate all SKILL.md files and verify

**Files:**
- All SKILL.md files (regenerated)

- [ ] **Step 1: Regenerate skill docs**

Run: `bun run build`

- [ ] **Step 2: Run all tests**

Run: `bun test`
Expected: All tests PASS, including the new REVIEW_REPORT_OUTPUT tests.

- [ ] **Step 3: Verify no unresolved placeholders**

Run: `grep -r '{{REVIEW_REPORT_OUTPUT}}' plan-ceo-review/SKILL.md plan-eng-review/SKILL.md`
Expected: No matches (placeholder should be resolved).

- [ ] **Step 4: Spot-check generated content**

Read `plan-ceo-review/SKILL.md` and `plan-eng-review/SKILL.md` to verify:
- "## Review Report Output" section appears after Review Dashboard and Plan File Review Report
- File path pattern includes the correct skill name
- Status mapping instructions are present
- Fallback path is present
- CEO-specific sections (Scope Decisions, Vision & Strategy) appear in CEO review
- Eng-specific sections (Architecture Analysis, Test Coverage) appear in eng review

- [ ] **Step 5: Commit generated files**

```bash
git add plan-ceo-review/SKILL.md plan-eng-review/SKILL.md
git commit -m "chore: regenerate SKILL.md files with review report output"
```

Note: `bun run build` regenerates all SKILL.md files. Only the two modified
templates will produce different output. Stage only the changed files.
