# loupe Implementation Plan

## Goal Description

Implement loupe — a Claude Code / Codex plugin for LLM-assisted patch review with a static site dashboard, across two repositories:

- **loupe** (`~/loupe`): Pure plugin containing a single skill file (`loupe-review.md`) based on patch-review with multi-agent and CI extensions.
- **loupe-web** (`~/loupe-web`): Review JSON data storage, zero-build single-page SPA, CI scripts, and GitHub Actions daily cron workflow.

The skill supports three review modes (Claude+Codex cross-review, Codex dual-agent, CI single-agent) and outputs structured JSON for the static site dashboard hosted on GitHub Pages.

## Acceptance Criteria

Following TDD philosophy, each criterion includes positive and negative tests for deterministic verification.

- AC-1: loupe plugin installs correctly via `claude plugin add`
  - Positive Tests (expected to PASS):
    - `.claude-plugin/plugin.json` exists and is valid JSON with `name: "loupe"`
    - `commands/loupe-review.md` exists with correct frontmatter (`name: loupe-review`)
    - `install.sh --help` exits 0 and prints usage text
    - `install.sh --claude` copies skill to `~/.claude/commands/loupe-review.md`
    - `install.sh --codex` copies skill to `~/.codex/skills/loupe/SKILL.md`
  - Negative Tests (expected to FAIL):
    - `install.sh --invalid-flag` exits non-zero
    - `install.sh --uninstall` removes installed files; subsequent check confirms they are gone

- AC-2: loupe-review.md skill file contains all required sections
  - Positive Tests (expected to PASS):
    - Frontmatter contains `name: loupe-review` and updated description
    - Arguments section documents `--ci` and `--output-json <path>`
    - Step 1.1 contains environment detection logic for all four modes (claude+codex, claude-only, codex-dual, ci)
    - Steps 8.6, 9.5, 10, 11, 12 contain mode-specific conditional logic
    - Step 12 contains the complete JSON schema template
    - All 15 workflow steps from spec Section 4.4 are present
  - Negative Tests (expected to FAIL):
    - `grep -c 'patch-review' commands/loupe-review.md` in title/header lines returns 0 (no stale references)
    - Searching for `$ARGUMENTS` placeholder in non-argument sections returns 0

- AC-3: loupe-web SPA renders landing page correctly
  - Positive Tests (expected to PASS):
    - `docs/index.html` exists as a single self-contained HTML file (no external dependencies)
    - Opening in browser with sample data shows a table with correct columns (Date, Subsystem, Series, Author, Ver, #, Verdict, Findings)
    - Findings column renders 4 colored circle-number dots per row
    - Zero-count findings display at 25% opacity
    - Search input filters table rows by subject/author/message-id
    - Subsystem dropdown filters by subsystem
    - Verdict and Severity dropdowns filter correctly
  - Negative Tests (expected to FAIL):
    - With empty `index.json` (`reviews: []`), table shows no rows (no crash)
    - Searching for a non-existent term shows 0 results (count updates)

- AC-4: loupe-web SPA renders detail page correctly
  - Positive Tests (expected to PASS):
    - Clicking a table row navigates to detail view via URL hash
    - Detail view shows: series title, metadata, verdict badge, version history, ML context, review stages (A-E), findings
    - Each finding card shows: patch_context with diff highlighting, review body, confidence footer
    - Back button returns to landing page
    - `Escape` key returns to landing page
  - Negative Tests (expected to FAIL):
    - Navigating to a non-existent review hash shows error state (not blank page)

- AC-5: CI scripts function correctly
  - AC-5.1: `update-index.sh` builds correct index
    - Positive Tests (expected to PASS):
      - With 2 sample JSON files, outputs `index.json` with `reviews` array of length 2
      - Each entry contains: id, path, title, author, date, subsystem, version, patch_count, verdict, findings counts
      - `updated_at` field is a valid ISO timestamp
    - Negative Tests (expected to FAIL):
      - With no review JSON files (only index.json), outputs `reviews: []` (no error)
  - AC-5.2: `fetch-new-patches.sh` queries and filters correctly
    - Positive Tests (expected to PASS):
      - Script exits 0 and creates `/tmp/new-patches.json` as a valid JSON array
      - On first run (no existing reviews), all fetched patches appear in output
    - Negative Tests (expected to FAIL):
      - Already-reviewed message-ids (present in existing review JSONs) are excluded from output

- AC-6: GitHub Actions workflow is valid
  - Positive Tests (expected to PASS):
    - `.github/workflows/daily-review.yml` is valid YAML
    - Workflow triggers on `schedule` (cron) and `workflow_dispatch`
    - Workflow has `contents: write` permission
    - Steps execute in order: checkout → fetch → review → update-index → commit+push
  - Negative Tests (expected to FAIL):
    - When no new patches found (COUNT=0), the review step exits early without error

- AC-7: Design system matches specification
  - Positive Tests (expected to PASS):
    - Font family is `ui-monospace, SFMono-Regular, "SF Mono", Menlo, Consolas, monospace`
    - Severity colors: Critical `#d32f2f`, Major `#1565c0`, Minor `#f59e0b`, Nit `#94a3b8`
    - Background colors: page `#f5f5f5`, cards `#ffffff`, code `#f6f8fa`
    - Diff highlighting: added `rgba(46, 160, 67, 0.15)`, removed `rgba(248, 81, 73, 0.15)`
  - Negative Tests (expected to FAIL):
    - No external CSS/JS files are referenced (everything inline)

## Path Boundaries

Path boundaries define the acceptable range of implementation quality and choices.

### Upper Bound (Maximum Acceptable Scope)

The implementation includes both repositories fully functional: loupe plugin with complete skill file supporting all three review modes (Claude+Codex, Codex dual-agent, CI single-agent), install script for both platforms, and loupe-web with a polished SPA featuring all filter/search capabilities, keyboard shortcuts (`/`, `j`/`k`, `Enter`, `Escape`), sample test data, working CI scripts, and a valid GitHub Actions workflow. The SPA follows sashiko.dev design conventions precisely.

### Lower Bound (Minimum Acceptable Scope)

The implementation includes: loupe plugin with working skill file (at minimum supporting CI mode and claude-only mode), basic install script, and loupe-web with a functional SPA that renders the landing table and detail view from JSON data, working `update-index.sh`, and a valid GitHub Actions workflow YAML. Keyboard shortcuts and advanced filtering may be simplified.

### Allowed Choices

- Can use: vanilla HTML/CSS/JS (no frameworks), bash/jq for scripts, standard GitHub Actions
- Cannot use: npm/node dependencies, CSS/JS frameworks (React, Vue, Tailwind), build tools (webpack, vite), static site generators (Hugo, Jekyll)
- Fixed choices per spec: JSON data format, sashiko.dev UI style, severity color scheme (#d32f2f/#1565c0/#f59e0b/#94a3b8), monospace font stack

## Feasibility Hints and Suggestions

> **Note**: This section is for reference and understanding only. These are conceptual suggestions, not prescriptive requirements.

### Conceptual Approach

1. Start with the loupe repo scaffold (plugin manifest, install script) — these are small, self-contained files.
2. Create loupe-review.md by copying patch-review.md and surgically adding the new sections (mode detection, `--ci` flag, JSON output). This minimizes risk of breaking the existing 920-line workflow.
3. For loupe-web, create sample JSON data first, then build the SPA against it. This ensures the SPA works with real data structures before CI generates actual reviews.
4. The SPA uses hash-based routing (`#/review/<path>`) — no server-side logic needed. `fetch()` loads JSON relative to the HTML file location.
5. CI scripts should be tested locally with the sample data before committing the GitHub Actions workflow.

### Relevant References

- `~/patch-review/commands/patch-review.md` — Base skill file to extend (920 lines)
- `~/patch-review/install.sh` — Install script pattern to follow
- `~/patch-review/.claude-plugin/plugin.json` — Plugin manifest pattern
- `docs/superpowers/specs/2026-03-25-loupe-design.md` — Complete design specification
- Visual mockups in `~/loupe/.superpowers/brainstorm/` — Approved UI designs

## Dependencies and Sequence

### Milestones

1. **Milestone 1: loupe plugin complete** — Plugin can be installed and the skill file is ready
   - Phase A: Scaffold (plugin.json, LICENSE, README, .gitignore)
   - Phase B: install.sh
   - Phase C: loupe-review.md skill file (the largest component)

2. **Milestone 2: loupe-web data layer** — Review JSON storage and CI scripts work
   - Phase A: Repository scaffold
   - Phase B: Sample test data (2 review JSONs + index.json)
   - Phase C: CI scripts (update-index.sh, fetch-new-patches.sh)

3. **Milestone 3: loupe-web presentation** — SPA renders reviews correctly
   - Phase A: HTML/CSS structure (landing + detail views)
   - Phase B: JavaScript (data loading, rendering, filtering, routing)
   - Phase C: Keyboard shortcuts and polish

4. **Milestone 4: CI pipeline** — GitHub Actions workflow automated
   - Phase A: daily-review.yml workflow
   - Phase B: End-to-end verification

Dependencies: Milestone 2 Phase B (sample data) should be completed before Milestone 3 (SPA needs data to test against). Milestones 1 and 2A are independent and can be parallelized.

## Implementation Notes

### Code Style Requirements
- Implementation code and comments must NOT contain plan-specific terminology such as "AC-", "Milestone", "Step", "Phase", or similar workflow markers
- These terms are for plan documentation only, not for the resulting codebase
- Use descriptive, domain-appropriate naming in code instead
- HTML/CSS/JS follows sashiko.dev conventions: monospace font, minimal UI, information-dense
- Shell scripts use `set -euo pipefail`, consistent variable naming
- All commits signed with: `Signed-off-by: Chao Liu <chao.liu.zevorn@gmail.com>`
- No AI-related co-author signatures in commits

--- Original Design Draft Start ---

# loupe Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement loupe — a Claude Code / Codex plugin for LLM-assisted patch review with a static site dashboard, across two repositories (loupe + loupe-web).

**Architecture:** loupe repo is a pure Claude/Codex plugin containing a single skill file (`loupe-review.md`) based on patch-review with multi-agent and CI extensions. loupe-web repo holds review JSON data, a zero-build single-page SPA, CI scripts, and a GitHub Actions daily cron workflow.

**Tech Stack:** Markdown (skill), HTML/CSS/JS (SPA), Bash/jq (CI scripts), GitHub Actions (CI), GitHub Pages (hosting)

**Spec:** `docs/superpowers/specs/2026-03-25-loupe-design.md`

**Source material:** `~/patch-review/commands/patch-review.md` (920-line skill to extend)

---

## File Map

### loupe repo (`~/loupe`)

| File | Action | Responsibility |
|------|--------|---------------|
| `.claude-plugin/plugin.json` | Create | Plugin manifest |
| `commands/loupe-review.md` | Create | Core skill — review workflow + multi-agent + JSON output |
| `install.sh` | Create | Manual install for Claude/Codex |
| `LICENSE` | Create | MIT license |
| `README.md` | Create | Usage docs |
| `.gitignore` | Modify | Add `.superpowers/`, editor temps |

### loupe-web repo (`~/loupe-web`)

| File | Action | Responsibility |
|------|--------|---------------|
| `docs/index.html` | Create | Single-page app — landing + detail views |
| `docs/reviews/index.json` | Create | Review index (initially empty) |
| `scripts/fetch-new-patches.sh` | Create | Query Patchwork API for new riscv patches |
| `scripts/update-index.sh` | Create | Rebuild index.json from review JSON files |
| `.github/workflows/daily-review.yml` | Create | Daily cron → codex review → push |
| `README.md` | Create | Repo docs |
| `.gitignore` | Create | Ignore temps |

### Test data (`~/loupe-web`, temporary for verification)

| File | Action | Responsibility |
|------|--------|---------------|
| `docs/reviews/2026-03-25/sample-review.json` | Create | Sample review JSON for SPA testing |
| `docs/reviews/2026-03-24/sample-review-2.json` | Create | Second sample for table rendering |

---

## Task 1: loupe repo — scaffold

**Files:**
- Create: `~/loupe/.claude-plugin/plugin.json`
- Create: `~/loupe/LICENSE`
- Create: `~/loupe/README.md`
- Modify: `~/loupe/.gitignore`

- [ ] **Step 1: Create plugin.json**

```json
{
  "name": "loupe",
  "description": "LLM-assisted patch review with multi-agent cross-reference. Supports lore URLs, Message-Ids, subject search, local commits, and commit ranges.",
  "version": "1.0.0",
  "author": { "name": "zevorn" },
  "repository": "https://github.com/zevorn/loupe",
  "homepage": "https://github.com/zevorn/loupe#readme",
  "license": "MIT",
  "keywords": [
    "loupe", "patch-review", "code-review", "mailing-list",
    "qemu", "linux-kernel", "lore", "patchwork", "codex", "multi-agent"
  ]
}
```

Write to `~/loupe/.claude-plugin/plugin.json`.

- [ ] **Step 2: Create LICENSE**

Write MIT license to `~/loupe/LICENSE` with `Copyright (c) 2026 Chao Liu`.

- [ ] **Step 3: Create README.md**

Write `~/loupe/README.md` covering:
- Project description (supersedes patch-review)
- Installation (`claude plugin add github:zevorn/loupe`)
- Usage (`/loupe-review <source> [base_branch] [+zh] [--ci] [--output-json <path>]`)
- Input formats table (lore URL, Message-Id, subject search, commit, range)
- Review modes (Claude+Codex, Codex dual, CI single)
- Dependencies
- Link to loupe-web dashboard
- License

- [ ] **Step 4: Update .gitignore**

Append to `~/loupe/.gitignore`:
```
.superpowers/
*.swp
*.swo
*~
.DS_Store
```

- [ ] **Step 5: Verify structure**

Run: `find ~/loupe -not -path '*/.git/*' -not -path '*/.claude/*' -not -path '*/.superpowers/*' -not -path '*/.humanize/*' | sort`

Expected: `.claude-plugin/plugin.json`, `commands/` (empty for now), `LICENSE`, `README.md`, `.gitignore`, `docs/` (spec + plan)

- [ ] **Step 6: Commit**

```bash
cd ~/loupe
git add .claude-plugin/plugin.json LICENSE README.md .gitignore
git commit -m "feat: add loupe plugin scaffold

Plugin manifest, MIT license, README with usage docs, and gitignore.

Signed-off-by: Chao Liu <chao.liu.zevorn@gmail.com>"
```

---

## Task 2: loupe repo — install.sh

**Files:**
- Create: `~/loupe/install.sh`

- [ ] **Step 1: Write install.sh**

Based on `~/patch-review/install.sh` structure. Script must:
- Set `SKILL_FILE` to `commands/loupe-review.md`
- `--claude`: copy to `~/.claude/commands/loupe-review.md`
- `--codex`: copy to `${CODEX_HOME:-~/.codex}/skills/loupe/SKILL.md`
- `--all`: both
- `--uninstall`: remove both
- No args: install all
- `chmod +x`

Write to `~/loupe/install.sh`.

- [ ] **Step 2: Verify help output**

Run: `bash ~/loupe/install.sh --help`

Expected: Usage text showing all options.

- [ ] **Step 3: Commit**

```bash
cd ~/loupe
git add install.sh
git commit -m "feat: add install script for Claude and Codex

Supports --claude, --codex, --all, --uninstall flags.
Preferred method: claude plugin add github:zevorn/loupe

Signed-off-by: Chao Liu <chao.liu.zevorn@gmail.com>"
```

---

## Task 3: loupe repo — loupe-review.md skill file

**Files:**
- Create: `~/loupe/commands/loupe-review.md`
- Reference: `~/patch-review/commands/patch-review.md`

This is the largest task. The skill file is based on patch-review.md (~920 lines) with the following extensions per spec Section 4.5.

- [ ] **Step 1: Copy patch-review.md as base**

Copy `~/patch-review/commands/patch-review.md` to `~/loupe/commands/loupe-review.md`.

- [ ] **Step 2: Update frontmatter**

Change name from `patch-review` to `loupe-review`. Update description to mention multi-agent and CI mode.

```yaml
---
name: loupe-review
description: Download, apply, and review mailing list patches with multi-agent cross-reference (Claude+Codex or dual-Codex). Supports lore URLs, Message-Ids, subject search, local commits, commit ranges. CI mode outputs structured JSON for automated pipelines.
---
```

- [ ] **Step 3: Update title and arguments section**

Change `# patch-review:` to `# loupe-review:`. Update arguments format to:

```
Format: `<source> [base_branch] [+zh] [--ci] [--output-json <path>]`
```

Add documentation for `--ci` and `--output-json` parameters after the existing `+zh` docs. See spec Section 4.1 for exact wording.

- [ ] **Step 4: Extend Step 1 — parse new flags**

In the "Step 1: Parse arguments" section, add parsing for:
- `--ci` flag → set `$CI_MODE=true`
- `--output-json <path>` → set `$OUTPUT_JSON_PATH` (handle directory vs file path per spec)
- Strip these flags before parsing source/base_branch

Add after the existing argument extraction block.

- [ ] **Step 5: Add Step 1.1 — Environment detection and mode selection**

Insert a new section after Step 1 (before Step 1.5). Title: `### Step 1.1: Detect environment and select review mode`

Content per spec Section 4.2:
```
if $CI_MODE:
    $REVIEW_MODE = "ci"
elif running in Claude Code (check for Claude-specific env vars or context):
    if `command -v codex` succeeds:
        $REVIEW_MODE = "claude+codex"
    else:
        $REVIEW_MODE = "claude-only"
elif running in Codex:
    $REVIEW_MODE = "codex-dual"

Print: "Review mode: $REVIEW_MODE"
```

For CI mode, also enforce: source must be a Message-Id or lore URL (not subject search). If subject search detected, error and exit.

- [ ] **Step 6: Update Step 8.6 — dual-Codex mode support**

Current Step 8.6 launches a single Codex in background. Extend to handle Mode 2 (codex-dual):

When `$REVIEW_MODE = "codex-dual"`:
- Launch Agent-2 in background (same as current codex background launch)
- Agent-1 (current agent) proceeds with five-stage review as normal
- Both agents independently perform the full review
- Cross-reference happens in Step 9.5 (same as current)

When `$REVIEW_MODE = "ci"`:
- Skip this step entirely (no parallel review)

When `$REVIEW_MODE = "claude-only"`:
- Skip this step entirely

- [ ] **Step 7: Update Step 9.5 — unified cross-reference**

The current cross-reference logic works for Mode 1. Extend to also work for Mode 2:
- Mode 1: Claude verifies Codex-only findings (existing behavior)
- Mode 2: Each agent verifies the other's unique findings
- CI mode: Skip cross-reference entirely
- Claude-only: Skip cross-reference entirely

Add `$REVIEW_MODE` checks at the beginning of Step 9.5.

- [ ] **Step 8: Update Step 10 — CI mode skips interactive display**

Add at the beginning of Step 10:
```
If $CI_MODE: skip this step (no interactive summary display).
```

- [ ] **Step 9: Update Step 11 — conditional output**

Add at the beginning of Step 11:
```
If $CI_MODE: skip reply.md generation, proceed to Step 12 (JSON output).
```

- [ ] **Step 10: Add Step 12 — CI JSON output**

Add a new section after Step 11. Title: `### Step 12: Generate structured JSON output (CI mode only)`

```
If not $CI_MODE: skip this step.

Construct the review JSON per the schema in spec Section 5.

Determine output path:
- If $OUTPUT_JSON_PATH ends with `/` or is a directory: auto-name as `<id>.json`
- Otherwise: use as full file path
- Default: `/tmp/loupe-review-<timestamp>/review.json`

mkdir -p for the output directory.
Write the JSON file.
Print: "Review JSON written to: <path>"
```

Include the complete JSON structure template from spec Section 5 in the skill file, so the LLM knows exactly what to output.

- [ ] **Step 11: Update reply.md output path**

Change output path from `~/qemu-patch/reply/` to `~/loupe/reply/` (or keep configurable). Update the file naming: `<series-short-name>-<version>-reply.md`.

- [ ] **Step 12: Update Reviewed-by and sign-off references**

Ensure all `Reviewed-by:` and `Chao Liu` references remain consistent. No changes needed if patch-review already uses `Chao Liu <chao.liu.zevorn@gmail.com>`.

- [ ] **Step 13: Read full file and verify completeness**

Read the entire `~/loupe/commands/loupe-review.md` and verify:
- Frontmatter is correct
- All 12 steps are present and numbered correctly
- Mode detection logic is in Step 1.1
- `--ci` and `--output-json` are documented
- JSON schema template is in Step 12
- No leftover `patch-review` references in titles/headers

- [ ] **Step 14: Commit**

```bash
cd ~/loupe
git add commands/loupe-review.md
git commit -m "feat: add loupe-review skill with multi-agent and CI mode

Based on patch-review with extensions:
- Environment detection (Claude+Codex / Codex dual / CI single)
- --ci flag for non-interactive JSON output
- --output-json path with directory auto-naming
- Dual-Codex independent review mode
- Unified cross-reference logic
- Structured JSON output per loupe schema v1

Signed-off-by: Chao Liu <chao.liu.zevorn@gmail.com>"
```

---

## Task 4: loupe-web repo — scaffold

**Files:**
- Create: `~/loupe-web/README.md`
- Create: `~/loupe-web/.gitignore`
- Create: `~/loupe-web/docs/reviews/index.json` (empty)
- Directories: `docs/reviews/`, `scripts/`, `.github/workflows/`

- [ ] **Step 1: Initialize git repo**

```bash
cd ~/loupe-web
git init
git remote add origin git@github.com:zevorn/loupe-web.git
```

- [ ] **Step 2: Create .gitignore**

Write `~/loupe-web/.gitignore`:
```
*.swp
*.swo
*~
.DS_Store
/tmp/
```

- [ ] **Step 3: Create directory structure**

```bash
mkdir -p ~/loupe-web/docs/reviews
mkdir -p ~/loupe-web/scripts
mkdir -p ~/loupe-web/.github/workflows
```

- [ ] **Step 4: Create empty index.json**

Write `~/loupe-web/docs/reviews/index.json`:
```json
{
  "updated_at": "",
  "reviews": []
}
```

- [ ] **Step 5: Create README.md**

Write `~/loupe-web/README.md` covering:
- Project description (review data + static site for loupe)
- Link to loupe plugin repo
- Link to live site (https://zevorn.github.io/loupe-web/)
- How CI works (daily cron, codex exec, JSON output)
- How to run locally (open docs/index.html)
- Directory structure
- License (MIT)

- [ ] **Step 5b: Create LICENSE**

Write MIT license to `~/loupe-web/LICENSE` with `Copyright (c) 2026 Chao Liu`.

- [ ] **Step 6: Commit**

```bash
cd ~/loupe-web
git add .
git commit -m "feat: initialize loupe-web repository

Directory structure for review data, static site, CI scripts,
and GitHub Actions workflow.

Signed-off-by: Chao Liu <chao.liu.zevorn@gmail.com>"
```

---

## Task 5: loupe-web — static site SPA (index.html)

**Files:**
- Create: `~/loupe-web/docs/index.html`

This is the single-page app. All CSS inline, all JS inline, no dependencies. Follows sashiko.dev style per spec Section 7.

- [ ] **Step 1: Write the HTML skeleton + CSS**

Create `~/loupe-web/docs/index.html` with:
- Full CSS from spec Section 7.2-7.6 (design system, severity colors, diff highlighting)
- HTML structure: `<div id="app">` with landing and detail view containers
- Responsive layout, max-width 1200px
- All component styles: filter bar, table, badges, finding cards, stages, disclaimer

CSS variables (from spec):
```css
--bg: #f5f5f5; --bg-card: #ffffff; --bg-code: #f6f8fa;
--text: #24292f; --text-muted: #666666; --text-dim: #999999;
--border: #d0d7de; --border-light: #eee; --link: #007bff;
--sev-critical: #d32f2f; --sev-major: #1565c0;
--sev-minor: #f59e0b; --sev-nit: #94a3b8;
--diff-add: rgba(46, 160, 67, 0.15);
--diff-del: rgba(248, 81, 73, 0.15);
```

Font: `ui-monospace, SFMono-Regular, "SF Mono", Menlo, Consolas, monospace`

- [ ] **Step 2: Write landing page HTML**

Inside the landing view container:
- Header: `<h1>loupe</h1>` + subtitle + disclaimer
- Stats bar: 4 stat boxes (populated by JS)
- Filter bar: search input, subsystem dropdown, verdict dropdown, severity dropdown, date dropdown
- Table with headers: Date, Subsystem, Series, Author, Ver, #, Verdict, Findings
- Table body: `<tbody id="review-table-body"></tbody>` (populated by JS)
- Legend: 4 colored dots with labels

- [ ] **Step 3: Write detail page HTML**

Inside the detail view container (hidden by default):
- Back nav link
- Title + metadata row
- Verdict badge
- Version history table: `<tbody id="version-history-body"></tbody>`
- ML context section: `<div id="ml-context"></div>`
- Review stages: `<div id="review-stages"></div>`
- Findings summary bar
- Findings container: `<div id="findings-container"></div>`
- Footer disclaimer

- [ ] **Step 4: Write JS — data loading and state**

```javascript
const state = { reviews: [], current: null, filters: {} };

async function loadIndex() {
    const resp = await fetch('reviews/index.json');
    const data = await resp.json();
    state.reviews = data.reviews;
    renderLanding();
}

async function loadReview(path) {
    const resp = await fetch(path);
    state.current = await resp.json();
    renderDetail();
}

function showLanding() { /* toggle views */ }
function showDetail() { /* toggle views */ }
```

- [ ] **Step 5: Write JS — landing page rendering**

```javascript
function renderLanding() {
    renderStats();
    renderSubsystemDropdown();
    renderTable(applyFilters(state.reviews));
}

function renderStats() { /* compute totals, this-week, critical, ready */ }

function renderTable(reviews) {
    /* For each review: date, subsystem, series link, author, ver, #,
       verdict badge, 4 colored finding dots (0=dimmed) */
}

function renderFindingDots(findings) {
    /* Returns HTML: 4 circles with counts, .dot-zero class for 0 */
}
```

- [ ] **Step 6: Write JS — filtering and search**

```javascript
function applyFilters(reviews) {
    return reviews.filter(r => {
        if (filters.search && !matchesSearch(r, filters.search)) return false;
        if (filters.subsystem && r.subsystem !== filters.subsystem) return false;
        if (filters.verdict && r.verdict !== filters.verdict) return false;
        if (filters.severity && !matchesSeverity(r, filters.severity)) return false;
        if (filters.date && !matchesDate(r, filters.date)) return false;
        return true;
    });
}

/* Bind event listeners to filter inputs */
document.querySelector('.search-input').addEventListener('input', ...);
document.querySelectorAll('.filter-select').forEach(el => el.addEventListener('change', ...));
```

- [ ] **Step 7: Write JS — detail page rendering**

```javascript
function renderDetail() {
    const r = state.current;
    /* Set title, metadata, verdict badge */
    renderVersionHistory(r.version_history);
    renderMLContext(r.ml_context);
    renderStages(r.review.stages);
    renderFindingsSummary(r.review.findings);
    renderFindings(r.review.findings);
}

function renderFinding(f) {
    /* Header: severity + stage + source badges, file:line, patch index
       Patch context: diff lines with green/red highlighting
       Body: title, description, suggestion code block
       Footer: confidence + reason */
}
```

- [ ] **Step 8: Write JS — keyboard shortcuts**

```javascript
document.addEventListener('keydown', (e) => {
    if (e.key === '/') { /* focus search, prevent default */ }
    if (e.key === 'j') { /* next row */ }
    if (e.key === 'k') { /* prev row */ }
    if (e.key === 'Enter') { /* open selected row */ }
    if (e.key === 'Escape') { /* back to landing */ }
});
```

- [ ] **Step 9: Write JS — init**

```javascript
window.addEventListener('DOMContentLoaded', () => {
    loadIndex();
    window.addEventListener('hashchange', handleRoute);
});

function handleRoute() {
    const hash = window.location.hash;
    if (hash.startsWith('#/review/')) {
        const path = decodeURIComponent(hash.slice(9));
        loadReview(path);
    } else {
        showLanding();
    }
}
```

- [ ] **Step 10: Commit**

```bash
cd ~/loupe-web
git add docs/index.html
git commit -m "feat: add static site SPA

Single-page app with landing page (table, filters, search)
and detail view (findings, stages, diff context).
Zero build step, pure HTML/CSS/JS, sashiko.dev style.

Signed-off-by: Chao Liu <chao.liu.zevorn@gmail.com>"
```

---

## Task 6: loupe-web — sample test data

**Files:**
- Create: `~/loupe-web/docs/reviews/2026-03-25/riscv-iommu-ipsr-v2.json`
- Create: `~/loupe-web/docs/reviews/2026-03-24/riscv-sbi-cppc-v1.json`
- Modify: `~/loupe-web/docs/reviews/index.json`

Create sample data to verify the SPA renders correctly.

- [ ] **Step 1: Create sample review JSON #1**

Write `~/loupe-web/docs/reviews/2026-03-25/riscv-iommu-ipsr-v2.json` with a complete review JSON per spec Section 5. Include:
- 3 findings (1 major, 2 nits)
- Version history (v1 + v2)
- ML context populated
- Verdict: needs_revision

- [ ] **Step 2: Create sample review JSON #2**

Write `~/loupe-web/docs/reviews/2026-03-24/riscv-sbi-cppc-v1.json` with:
- 1 finding (1 nit)
- Version history (v1 only)
- Verdict: ready_to_merge

- [ ] **Step 3: Update index.json**

Update `~/loupe-web/docs/reviews/index.json` with entries for both sample reviews.

- [ ] **Step 4: Verify SPA renders**

Open `~/loupe-web/docs/index.html` in browser (or use `python3 -m http.server` from `docs/`). Verify:
- Landing page shows 2 rows with correct data
- Filters work (subsystem, verdict, search)
- Clicking a row shows detail view
- Finding cards render with diff highlighting
- Back button works
- `0 0 0 0` dots display correctly for zero-finding reviews

Run: `cd ~/loupe-web/docs && python3 -m http.server 8080 &`

Then: `curl -s http://localhost:8080/ | head -5` to verify HTML is served.

- [ ] **Step 5: Commit**

```bash
cd ~/loupe-web
git add docs/reviews/
git commit -m "feat: add sample review data for SPA testing

Two sample reviews demonstrating the JSON schema and
verifying the SPA renders correctly.

Signed-off-by: Chao Liu <chao.liu.zevorn@gmail.com>"
```

---

## Task 7: loupe-web — CI scripts

**Files:**
- Create: `~/loupe-web/scripts/fetch-new-patches.sh`
- Create: `~/loupe-web/scripts/update-index.sh`

- [ ] **Step 1: Write update-index.sh**

Write `~/loupe-web/scripts/update-index.sh`:

```bash
#!/usr/bin/env bash
# Rebuild docs/reviews/index.json from individual review JSON files.
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_ROOT="$(cd "${SCRIPT_DIR}/.." && pwd)"
REVIEWS_DIR="${REPO_ROOT}/docs/reviews"
INDEX_FILE="${REVIEWS_DIR}/index.json"

# Build index using jq
# Walk all .json files (excluding index.json itself)
# Extract summary fields from each review

entries=$(find "${REVIEWS_DIR}" -name '*.json' -not -name 'index.json' -print0 \
    | xargs -0 -I{} jq -c '{
        id: .id,
        path: (input_filename | ltrimstr("'"${REPO_ROOT}/docs/"'")),
        title: .series.title,
        author: .series.author.name,
        date: .series.date,
        subsystem: .series.subsystem,
        version: .series.version,
        patch_count: .series.patch_count,
        verdict: .review.verdict,
        findings: {
            critical: ([.review.findings[] | select(.severity == "critical")] | length),
            major: ([.review.findings[] | select(.severity == "major")] | length),
            minor: ([.review.findings[] | select(.severity == "minor")] | length),
            nit: ([.review.findings[] | select(.severity == "nit")] | length)
        }
    }' {} \
    | jq -s 'sort_by(.date) | reverse')

jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    --argjson reviews "${entries}" \
    '{ updated_at: $ts, reviews: $reviews }' > "${INDEX_FILE}"

echo "Updated ${INDEX_FILE} with $(echo "${entries}" | jq length) reviews."
```

`chmod +x`.

- [ ] **Step 2: Test update-index.sh with sample data**

Run: `bash ~/loupe-web/scripts/update-index.sh`

Expected: `Updated .../index.json with 2 reviews.`

Verify: `jq '.reviews | length' ~/loupe-web/docs/reviews/index.json` → `2`

- [ ] **Step 3: Write fetch-new-patches.sh**

Write `~/loupe-web/scripts/fetch-new-patches.sh`:

```bash
#!/usr/bin/env bash
# Query Patchwork API for new QEMU RISC-V patches in the last 24 hours.
# Filters out already-reviewed patches (checks index.json).
# Outputs: /tmp/new-patches.json (JSON array of message-ids)
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_ROOT="$(cd "${SCRIPT_DIR}/.." && pwd)"
INDEX_FILE="${REPO_ROOT}/docs/reviews/index.json"

SINCE=$(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%S 2>/dev/null \
    || date -u -v-24H +%Y-%m-%dT%H:%M:%S)

API_URL="https://patchwork.ozlabs.org/api/patches/"
PARAMS="project=qemu-devel&q=riscv&since=${SINCE}&order=-date&per_page=50"

echo "Fetching patches since ${SINCE}..."

curl -sL "${API_URL}?${PARAMS}" -o /tmp/patchwork-response.json

# Extract unique message-ids, grouped by series (prefer cover letter)
# Filter out already-reviewed (match by message_id stored in each review JSON)
EXISTING_MSGIDS_FILE=$(mktemp)
find "${REPO_ROOT}/docs/reviews" -name '*.json' -not -name 'index.json' \
    -exec jq -r '.series.message_id // empty' {} \; \
    > "${EXISTING_MSGIDS_FILE}" 2>/dev/null || true

NEW_MSGIDS=$(jq -r '
    [.[] | {
        msgid: .msgid,
        series_id: (.series[0].id // null),
        date: .date
    }]
    | group_by(.series_id)
    | map(.[0].msgid)
    | .[]
' /tmp/patchwork-response.json)

# Filter out already-reviewed (skip grep if no existing reviews)
if [ -s "${EXISTING_MSGIDS_FILE}" ]; then
    echo "${NEW_MSGIDS}" \
        | grep -v -F -f "${EXISTING_MSGIDS_FILE}" \
        | jq -R -s 'split("\n") | map(select(. != ""))' \
        > /tmp/new-patches.json
else
    echo "${NEW_MSGIDS}" \
        | jq -R -s 'split("\n") | map(select(. != ""))' \
        > /tmp/new-patches.json
fi
rm -f "${EXISTING_MSGIDS_FILE}"

COUNT=$(jq length /tmp/new-patches.json)
echo "Found ${COUNT} new patch series to review."
```

`chmod +x`.

- [ ] **Step 4: Verify fetch script parses correctly**

Run: `bash ~/loupe-web/scripts/fetch-new-patches.sh`

Expected: Outputs count of new patches (may be 0 if none in last 24h). No errors. `/tmp/new-patches.json` exists and contains a JSON array.

Verify: `jq type /tmp/new-patches.json` → `"array"`

- [ ] **Step 5: Commit**

```bash
cd ~/loupe-web
git add scripts/
git commit -m "feat: add CI scripts for patch fetching and index generation

- fetch-new-patches.sh: queries Patchwork API, filters already-reviewed
- update-index.sh: rebuilds index.json from review JSON files

Signed-off-by: Chao Liu <chao.liu.zevorn@gmail.com>"
```

---

## Task 8: loupe-web — GitHub Actions workflow

**Files:**
- Create: `~/loupe-web/.github/workflows/daily-review.yml`

- [ ] **Step 1: Write daily-review.yml**

Write `~/loupe-web/.github/workflows/daily-review.yml` per spec Section 6.2:

```yaml
name: Daily Patch Review

on:
  schedule:
    - cron: '0 8 * * *'
  workflow_dispatch:

permissions:
  contents: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          sudo apt-get update -qq
          sudo apt-get install -y -qq jq

      - name: Fetch new patches from Patchwork
        run: bash scripts/fetch-new-patches.sh

      - name: Review patches
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          COUNT=$(jq length /tmp/new-patches.json)
          if [ "$COUNT" -eq 0 ]; then
            echo "No new patches to review."
            exit 0
          fi
          DATE_DIR="docs/reviews/$(date +%Y-%m-%d)"
          mkdir -p "$DATE_DIR"
          for msgid in $(jq -r '.[]' /tmp/new-patches.json); do
            echo "Reviewing: ${msgid}"
            codex exec "loupe-review ${msgid} --ci --output-json ${DATE_DIR}/"
          done

      - name: Update index
        run: bash scripts/update-index.sh

      - name: Commit and push
        run: |
          git config user.name "loupe-bot"
          git config user.email "loupe-bot@users.noreply.github.com"
          git add docs/reviews/
          git diff --cached --quiet || \
            git commit -m "review: $(date +%Y-%m-%d) daily patch review"
          git push
```

- [ ] **Step 2: Validate YAML syntax**

Run: `python3 -c "import yaml; yaml.safe_load(open('$HOME/loupe-web/.github/workflows/daily-review.yml'))"`

Expected: No errors.

- [ ] **Step 3: Commit**

```bash
cd ~/loupe-web
git add .github/workflows/daily-review.yml
git commit -m "feat: add daily patch review GitHub Actions workflow

Cron job runs daily at UTC 08:00. Fetches new RISC-V patches
from Patchwork, reviews via codex + loupe-review, commits JSON
results, and pushes for GitHub Pages deployment.

Signed-off-by: Chao Liu <chao.liu.zevorn@gmail.com>"
```

---

## Task 9: Final verification and cleanup

- [ ] **Step 1: Verify loupe repo structure**

```bash
find ~/loupe -not -path '*/.git/*' -not -path '*/.claude/*' \
    -not -path '*/.superpowers/*' -not -path '*/.humanize/*' \
    -not -path '*/docs/superpowers/*' -type f | sort
```

Expected files:
```
.claude-plugin/plugin.json
.gitignore
commands/loupe-review.md
install.sh
LICENSE
README.md
```

- [ ] **Step 2: Verify loupe-web repo structure**

```bash
find ~/loupe-web -not -path '*/.git/*' -type f | sort
```

Expected files:
```
.github/workflows/daily-review.yml
.gitignore
docs/index.html
docs/reviews/index.json
docs/reviews/2026-03-25/riscv-iommu-ipsr-v2.json
docs/reviews/2026-03-24/riscv-sbi-cppc-v1.json
README.md
scripts/fetch-new-patches.sh
scripts/update-index.sh
```

- [ ] **Step 3: Test SPA end-to-end**

```bash
cd ~/loupe-web/docs
python3 -m http.server 8080 &
curl -s http://localhost:8080/ | grep -c 'loupe'
curl -s http://localhost:8080/reviews/index.json | jq '.reviews | length'
kill %1
```

Expected: HTML contains "loupe", index has 2 reviews.

- [ ] **Step 4: Remove sample data (optional)**

If sample data should not ship in the repo, remove the test JSON files and reset index.json to empty. Otherwise keep as demo data.

- [ ] **Step 5: Final commits if any cleanup needed**

Only if changes were made in steps 1-4.

--- Original Design Draft End ---
