---
name: loupe-review
description: Download, apply, and review mailing list patches with multi-agent cross-reference (Claude+Codex or dual-Codex). Supports lore URLs, Message-Ids, subject search, local commits, commit ranges. CI mode outputs structured JSON for automated pipelines. Use when the user asks to review a patch, review a mailing list submission, apply patches from lore, review a patch series, check a QEMU/Linux kernel patch, or analyze commit quality.
---

# loupe-review: Download, apply, and review mailing list patches

## Arguments

$ARGUMENTS

Format: `<source> [base_branch] [+zh] [--ci] [--base <branch>] [--output-json <path>]`

- `source` (required): One of the following input forms:
  - **lore URL**: `https://lore.kernel.org/qemu-devel/<msgid>/t.mbox.gz`
  - **Message-Id**: `<msgid@domain>` or bare `msgid@domain`
  - **Subject search**: Keywords from patch subject (e.g., `virtio-net fix`,
    `riscv vector`). Triggers interactive search on Patchwork and lore.
  - **Local commit**: A single git SHA or ref (e.g., `HEAD`, `abc1234`)
  - **Local commit range**: `<base>..<tip>` (e.g., `master..HEAD`,
    `abc1234..def5678`)
- `base_branch` (optional): Branch to base on. Default: `master`.
  Ignored when source is a local commit range (base is derived from range).
- `+zh` (optional): Include Chinese analysis section in the review file.
  Default: off (English-only output). When present, a `## 中文分析` section
  is added to the review file header with bullet-point findings in Chinese.
- `--ci` (optional): CI mode. Single agent, no interactive prompts, outputs
  structured JSON instead of reply.md. Requires source to be a lore URL or
  Message-Id (subject search is not supported in CI mode).
- `--base <branch>` (optional): Explicit base branch. Useful for subject
  search mode where the positional `base_branch` is ambiguous with search
  keywords. Overrides the positional second arg. Default: `master`.
- `--output-json <path>` (optional): JSON output path for CI mode. If path
  ends with `/` or is an existing directory, the file is auto-named
  `<id>.json` inside it. Otherwise treated as a full file path.
  Default: `/tmp/loupe-review-<timestamp>/review.json`.

## Workflow

### Step 1: Parse arguments and detect input mode

Extract from the user-provided arguments:

First, scan for and extract all flags (before parsing positional args):
- If `+zh` is present, set `$ZH_MODE=true` and remove it from args.
- If `--ci` is present, set `$CI_MODE=true` and remove it from args.
- If `--output-json <path>` is present, set `$OUTPUT_JSON_PATH=<path>` and
  remove both tokens from args.
- If `--base <branch>` is present, set `$BASE_BRANCH=<branch>` and remove
  both tokens from args. This allows specifying the base branch explicitly,
  which is especially useful in subject search mode where positional
  `base_branch` is ambiguous with search keywords.

After flag extraction, parse remaining positional args. The parsing is
mode-dependent — first attempt to detect the input mode from the first
arg, then decide how to consume the rest:

- If `$BASE_BRANCH` was already set by `--base`, use it for all modes.
- If the first arg is a lore URL, Message-Id, commit range, or local
  commit → it is the `source`. The second arg (if any) is `base_branch`
  (default: `$BASE_BRANCH` or `master`).
- If the first arg does not match any of the above → **all remaining
  positional args are joined as subject search keywords** (e.g.,
  `riscv iommu fix` → search for `"riscv iommu fix"`). The base
  branch comes from `--base` flag or defaults to `master`.

Detect the input mode (test the first positional arg):

1. **lore URL**: Starts with `https://lore.kernel.org/`. Extract Message-Id
   and mailing list name from the URL path.
   Example: `https://lore.kernel.org/qemu-devel/<msgid>/t.mbox.gz`
   → `<msgid>`, `$MAILING_LIST = "qemu-devel"`
   The list name is the first path segment after the hostname.
2. **Local commit range**: Contains `..` (e.g., `master..HEAD`). Split into
   base and tip refs.
3. **Local commit**: Verify the first arg with `git rev-parse <ref>`.
   If it succeeds, apply these rules to decide commit vs search:
   - **1 arg**: always commit mode.
   - **2 args**: commit mode if EITHER the first arg is SHA-like
     (7-40 hex chars) / contains ref modifiers (`^`, `~`, `@{`),
     OR the second arg is also a valid git ref (`git rev-parse`
     succeeds on second arg). Otherwise fall through to search.
   - **3+ args**: always fall through to search.
   (This check MUST come before Message-Id detection, because refs
   like `HEAD@{1}` or `main@{upstream}` contain `@` but are valid
   git refs, not Message-Ids.)
   If `git rev-parse` fails on the first arg, fall through to mode 4.
   **Rationale**: `HEAD master` → commit+base (2nd is valid ref);
   `master regression` → search (2nd is NOT a ref);
   `abc1234 release-9.0` → commit+base (1st is SHA-like);
   `HEAD` → commit (single arg); `riscv iommu fix` → search (3 args).
4. **Message-Id**: Contains `@` but is not a URL and not a valid git ref.
   Strip angle brackets if present. **URL-encode** the Message-Id before
   using it in any URL (percent-encode `/`, `+`, `=`, and other reserved
   characters). To determine `$MAILING_LIST`, query lore's cross-list
   search:
   ```
   https://lore.kernel.org/all/<url-encoded-message-id>/
   ```
   The redirect or response will reveal the actual list (e.g.,
   `lore.kernel.org/linux-riscv/...`). Extract the list name from the
   resolved URL. If the query fails or the list cannot be determined:
   - **Interactive mode**: ask the user which mailing list this
     Message-Id belongs to.
   - **CI mode**: exit with error — a bare Message-Id without a
     resolvable list is not actionable in non-interactive mode.
5. **Subject search**: Any input that does not match modes 1–4. Join
   **all remaining positional args** as the search string (e.g.,
   `riscv iommu fix` → search keywords `"riscv iommu fix"`).

If no arguments provided, ask the user for the source.

Set `$INPUT_MODE` to one of: `lore`, `msgid`, `range`, `commit`, `search`.
Set `$MAILING_LIST` to the detected list name (default: `qemu-devel`).

All subsequent lore.kernel.org URLs and Patchwork API queries MUST use
`$MAILING_LIST` instead of hardcoded `qemu-devel`. **Always URL-encode
Message-Ids** before embedding in URLs (percent-encode `/`, `+`, `=`,
and other reserved chars). For example:
- lore: `https://lore.kernel.org/$MAILING_LIST/<url-encoded-msgid>/t.mbox.gz`
- Patchwork: `project=$MAILING_LIST`

Set `$PATCHWORK_BASE` based on `$MAILING_LIST`:
- `qemu-devel` → `https://patchwork.ozlabs.org/api`
- `linux-*`, `kvm`, `kvmarm` → `https://patchwork.kernel.org/api`
- Other → `https://patchwork.kernel.org/api` (fall back to ozlabs if empty)

This MUST be set here (not deferred to Step 7) because Step 1.5
(subject search) needs it immediately.

### Step 1.1: Detect environment and select review mode

Determine the review mode based on environment and flags:

```
if $CI_MODE:
    $REVIEW_MODE = "ci"
    # Enforce: source must be lore URL or Message-Id in CI mode
    if $INPUT_MODE not in ("lore", "msgid"):
        Error: "CI mode requires a lore URL or Message-Id. Got: $INPUT_MODE"
        Exit.
elif running in Claude Code:
    if `command -v codex` succeeds:
        $REVIEW_MODE = "claude+codex"
    else:
        $REVIEW_MODE = "claude-only"
elif running in Codex:
    $REVIEW_MODE = "codex-dual"
```

Print: `Review mode: $REVIEW_MODE`

For modes `lore` and `msgid`, proceed to Step 2 (remote fetch).
For modes `commit` and `range`, skip to Step 2b (local export).
For mode `search`, proceed to Step 1.5 (subject search).

**CI mode behavior**: In CI mode, skip all interactive steps (user prompts,
search selection). If a step says "ask the user", skip it and proceed with
defaults.

### Step 1.5: Subject search (mode: `search`)

Search for patches by subject keywords on Patchwork and lore, present results
to the user, and extract the Message-Id for the selected patch.

#### 1.5a: Search Patchwork API

Use WebFetch for API queries. If WebFetch is unavailable (Codex), use
`curl -sL "<url>" | jq '...'` via the Bash tool instead.

Query the Patchwork REST API with the keywords:

```
WebFetch: $PATCHWORK_BASE/patches/?project=$MAILING_LIST&q=<keywords>&order=-date&per_page=20
```

URL-encode the keywords (spaces → `%20` or `+`).

From the JSON response, extract for each result:
- `name` — patch subject line
- `msgid` — Message-Id
- `date` — submission date
- `submitter.name` — author name
- `state` — patch status (New / Under Review / Accepted / …)
- `series[0].name` — series name (if part of a series)
- `series[0].id` — series ID

#### 1.5b: Fallback — search lore

If Patchwork returns no results or is unreachable, fall back to lore search:

```
WebFetch: https://lore.kernel.org/$MAILING_LIST/?q=<keywords>&x=A
```

Parse the HTML response to extract matching threads:
- Subject lines
- Message-Ids (from `href` attributes linking to messages)
- Dates and authors

#### 1.5c: Present results to user

Display the search results as a numbered list:

```
Search results for "<keywords>":

  #  | Date       | Subject                                    | Author         | Status
  ---+------------+--------------------------------------------+----------------+-----------
  1  | 2026-03-20 | [PATCH v3 0/5] virtio-net: Fix RSC ...     | Alice Smith    | Under Review
  2  | 2026-03-18 | [PATCH v2 0/5] virtio-net: Fix RSC ...     | Alice Smith    | Superseded
  3  | 2026-03-15 | [PATCH 1/2] virtio-net: Add feature X      | Bob Jones      | New
  ...
```

If results span multiple series, group by series where possible (use
`series[0].id` to group). Show the cover letter (`0/N`) entry when available,
otherwise show the first patch of the series.

#### 1.5d: User selection

Ask the user to select one result by number. If the user wants to refine the
search, allow them to provide new keywords and repeat from 1.5a.

#### 1.5e: Extract Message-Id and continue

From the selected result, extract the `msgid` field and the `series[0].id`.

Record two separate identifiers:
- `$FETCH_MSGID` — the Message-Id used for downloading the series via b4/curl.
  For series with a cover letter, prefer the cover letter's Message-Id.
- `$SERIES_ID` — the Patchwork series ID (from `series[0].id`), used for
  querying Patchwork in Step 7a/7b. This works regardless of whether we use
  the cover letter or patch Message-Id for fetching.

If the selected result is part of a series (has `series[0].id`), query
the series endpoint to find the cover letter:

```
WebFetch: $PATCHWORK_BASE/series/<series_id>/
```

Extract `cover_letter.msgid` from the response. If available, set
`$FETCH_MSGID` to the cover letter's msgid. Keep `$SERIES_ID` as-is.

Set `$INPUT_MODE` to `msgid` and proceed to Step 2 (remote fetch) with
`$FETCH_MSGID`. Step 7a should use `$SERIES_ID` for Patchwork queries
when available (falling back to `$FETCH_MSGID` if `$SERIES_ID` is not set).

### Step 2: Download patches with b4 (modes: `lore`, `msgid`)

```bash
mkdir -p /tmp/loupe-review-<timestamp>
cd /tmp/loupe-review-<timestamp>
b4 am <message_id>
```

b4 will produce:
- A `.mbx` file (mbox ready for `git am`)
- A `.cover` file (cover letter, if present)

#### b4 failure fallback

If b4 fails (not installed, network error, series not found), apply a
graduated fallback strategy instead of immediately asking the user:

**Fallback 1 — Direct lore mbox download**:
```bash
# Download the entire thread as mbox
curl -sL "https://lore.kernel.org/$MAILING_LIST/<message_id>/t.mbox.gz" \
    | gunzip > /tmp/loupe-review-<timestamp>/thread.mbox

# If the above fails (e.g., message-id encoding), try URL-encoded form
curl -sL "https://lore.kernel.org/$MAILING_LIST/<url_encoded_msgid>/t.mbox.gz" \
    | gunzip > /tmp/loupe-review-<timestamp>/thread.mbox
```

When using the raw mbox fallback, **you MUST filter the thread mbox
before it can be used by Step 5**. The raw thread contains replies,
cover letters, and non-patch messages that will break `git am`.

Filtering steps (execute immediately after download):
1. Split `thread.mbox` into individual messages
2. Keep only messages whose Subject contains `[PATCH` or `[RFC`
3. Discard messages that have no diff body (pure text replies)
4. Sort remaining messages by subject index (`[PATCH n/m]`)
5. Concatenate the sorted, filtered messages into a new file:
   `filtered.mbox` in the same directory
6. Remove or rename the original `thread.mbox` so Step 5 picks up
   only `filtered.mbox`

The cover letter (`0/m` index or no diff body) should be saved
separately as `.cover` for Step 3 metadata extraction but NOT
included in the file passed to `git am`.

**Fallback 2 — Single patch via lore raw endpoint**:
```bash
# For a single patch (not a series)
curl -sL "https://lore.kernel.org/$MAILING_LIST/<message_id>/raw" \
    > /tmp/loupe-review-<timestamp>/patch.mbox
```

**Fallback 3 — Ask user or exit (CI)**:

- **Interactive mode**: If all automated approaches fail, show the
  errors and ask for guidance. Suggest the user manually download the
  mbox or provide a local file path.
- **CI mode** (`$CI_MODE`): If all automated approaches fail, resolve
  the output path first (same logic as Step 12):
  ```bash
  # Resolve output path before writing error JSON
  if $OUTPUT_JSON_PATH ends with "/" or is a directory:
      OUTPUT_FILE="${OUTPUT_JSON_PATH}/error-<message-id-slug>.json"
  else:
      OUTPUT_FILE="${OUTPUT_JSON_PATH}"
  # Default if not set:
  OUTPUT_FILE="${OUTPUT_FILE:-/tmp/loupe-review-<timestamp>/error.json}"
  mkdir -p "$(dirname "$OUTPUT_FILE")"
  ```
  Write error JSON to `$OUTPUT_FILE` and exit non-zero:
  ```json
  {
    "schema_version": "1",
    "error": "Failed to download patches for <message-id>",
    "message_id": "<message-id>",
    "generated_at": "<timestamp>",
    "generator": "loupe-review v1.0"
  }
  ```
  This ensures the CI pipeline can detect and handle failures without
  hanging on interactive prompts.

### Step 2b: Export local commits as patches (modes: `commit`, `range`)

For local commit input, export patches using `git format-patch`:

```bash
mkdir -p /tmp/loupe-review-<timestamp>

# For a commit range (e.g., master..HEAD):
git format-patch <base>..<tip> -o /tmp/loupe-review-<timestamp>/

# For a single commit:
git format-patch -1 <sha> -o /tmp/loupe-review-<timestamp>/
```

This produces numbered `.patch` files ready for review. There is no `.mbx`
or `.cover` file in this mode.

First, set `$REVIEW_TIP` and `$REVIEW_BASE` (needed by metadata extraction
and all subsequent steps):
- For commit range: `$REVIEW_BASE = <base>`, `$REVIEW_TIP = <tip>`
- For single commit: `$REVIEW_TIP = <sha>`. For `$REVIEW_BASE`:
  - If `<sha>` has a parent: `$REVIEW_BASE = <sha>^`
  - If `<sha>` is a root commit (no parent): set `$ROOT_COMMIT = true`
    and leave `$REVIEW_BASE` unset. Root commits require special
    handling — see below.

Then proceed to **Step 3 (metadata extraction)** to populate
`$SERIES_VERSION`, `$SUBJECT_STEM`, title, author, and patch count.
For local mode, extract these from the `.patch` files and commit messages:
- **Title**: first commit's subject line (or range description)
- **Author**: from `git log --format='%an <%ae>' -1 $REVIEW_TIP`
- **Version**: `v1` (local commits have no version tag unless subject
  contains `[PATCH vN]`)
- **Patch count**: number of `.patch` files produced

Then skip Step 3.5 (prerequisite detection) and Step 4 (branch
creation) — the commits are already in the local repo. Proceed directly
to Step 6 (intent summary).

All subsequent steps that use `<base_branch>..$REVIEW_TIP` MUST use
`$REVIEW_BASE..$REVIEW_TIP` instead for local modes.

**Root commit handling**: When `$ROOT_COMMIT = true`, the `A..B` range
syntax cannot be used (there is no ancestor commit). Instead:
- `git log`: use `git log --oneline $REVIEW_TIP` (just the one commit)
- `git diff`: use `git diff-tree -p --root $REVIEW_TIP`
- `git format-patch`: use `git format-patch --root $REVIEW_TIP`
- `codex review`: use `git diff-tree -p --root $REVIEW_TIP | codex exec ...`

**IMPORTANT**: All subsequent steps that use diff/review commands MUST
use `$REVIEW_BASE..$REVIEW_TIP` instead of hardcoded refs.

For remote modes (lore, msgid), after branch creation and patch
application in Steps 4-5:
- `$REVIEW_TIP = HEAD` (patches are applied to the current branch tip)
- `$REVIEW_BASE`: set to `<base_branch>` ONLY if Step 4.5 did not
  already update it (i.e., no prerequisites were applied). If Step 4.5
  set `$REVIEW_BASE` to the post-prerequisite HEAD, do NOT overwrite it.
  This ensures prerequisite commits are excluded from the review range.

### Step 3: Analyze the patch series (version-aware)

Read the `.cover` file (if present) and the patch file (`.mbx` from b4,
or `.mbox`/`filtered.mbox` from curl fallback, or `.patch` from local
`format-patch`) to get:
- Series title, purpose, patch count, author

#### 3a: Extract version information

Parse the series subject line to extract version metadata:

- **Version number**: Extract `vN` from `[PATCH vN ...]` or `[RFC vN ...]`.
  If no version tag, assume v1.
- **Subject stem**: Strip version tag, index (`n/m`), and prefixes
  (`PATCH`, `RFC`, `RESEND`) to get the base subject. This is used later
  for finding prior versions (Step 7d).
  Example: `[PATCH v3 2/5] hw/i386: Add foo support` → stem: `hw/i386: Add foo support`
- **Changelog**: If the cover letter or individual patches contain a
  changelog section (text between `---` and the diffstat, or after
  `Changes in vN:`), extract it. This shows what changed from the
  prior version.

Record: `$SERIES_VERSION`, `$SUBJECT_STEM`, `$CHANGELOG` (if any).

#### 3b: Derive branch name

Derive branch name: `review/<short-description>` from series title.

### Step 3.5: Detect prerequisite patches

Check for prerequisite dependencies in the `.cover` and `.mbx` files.

Search for dependency indicators:
- `Based-on:` tag (usually contains message-id or lore URL)
- `Depends-on:` tag
- Text patterns: "depends on", "based on", "on top of", "prerequisite", "requires"

**Do NOT** use `In-Reply-To:` or `References:` headers as dependency
indicators — these are normal threading headers present in every patch
series and would cause false positives.

Common formats:
```
Based-on: <message-id@domain>
Based-on: https://lore.kernel.org/qemu-devel/<msgid>/
Depends-on: [PATCH v2 0/3] Add foo support
This series depends on the "Add bar" series posted earlier.
```

If dependencies are found, extract the dependency references and resolve
them to message-ids for Step 4.5:

- **Message-Id or lore URL**: use directly.
- **Subject text only** (e.g., `Depends-on: [PATCH v2 0/3] Add foo`):
  search Patchwork for the subject to find its message-id:
  ```
  $PATCHWORK_BASE/patches/?project=$MAILING_LIST&q=<subject keywords>&order=-date&per_page=5
  ```
  Pick the best match by subject similarity and extract its `msgid`.
  If search fails, inform the user and ask for the message-id manually.

Record resolved message-ids for Step 4.5. If no dependencies, proceed
to Step 4.

### Step 4: Create review worktree

Use `git worktree` to create an isolated review environment, avoiding
conflicts with the user's working tree (which may have uncommitted
changes):

```bash
cd <repo_root>
REVIEW_WORKTREE="/tmp/loupe-review-<timestamp>/worktree"
git worktree add -b <branch_name> "${REVIEW_WORKTREE}" <base_branch>
cd "${REVIEW_WORKTREE}"
```

If the branch already exists, ask user to delete/recreate or rename.
All subsequent steps (4.5, 5, 8, 9, etc.) operate inside
`${REVIEW_WORKTREE}`, not the user's original working tree.

After creating the worktree, update submodules to match the branch state:

```bash
git submodule update
```

This ensures submodules are synchronized with the current branch.

### Step 4.5: Apply prerequisite patches (on review branch)

**IMPORTANT**: Prerequisites MUST be applied on the review branch created
in Step 4, not on the user's original branch.

If Step 3.5 found dependencies:

1. Inform the user: "Found prerequisite: <description>"
2. Download and apply prerequisites on the review branch:
   ```bash
   # Download prerequisite (same fallback strategy as Step 2)
   mkdir -p /tmp/loupe-review-<timestamp>/prereq-<n>
   cd /tmp/loupe-review-<timestamp>/prereq-<n>
   # Determine the prereq's mailing list:
   # 1. If dependency was a lore URL, extract list from URL path
   # 2. If bare Message-Id, resolve via lore.kernel.org/all/<msgid>/
   #    (same as Step 1 mode 4 detection)
   # 3. Fall back to $MAILING_LIST only if resolution fails
   if [ -n "${PREREQ_LORE_LIST}" ]; then
       PREREQ_LIST="${PREREQ_LORE_LIST}"
   else
       # Query lore cross-list to detect prereq's actual list
       PREREQ_LIST=$(curl -sL -o /dev/null -w '%{url_effective}' \
           "https://lore.kernel.org/all/<url-encoded-prereq-msgid>/" \
           | sed 's|https://lore.kernel.org/||;s|/.*||')
       PREREQ_LIST="${PREREQ_LIST:-$MAILING_LIST}"
   fi
   b4 am <prerequisite-message-id> || \
     curl -sL "https://lore.kernel.org/${PREREQ_LIST}/<prerequisite-message-id>/t.mbox.gz" \
       | gunzip > thread.mbox
   # If thread.mbox was used, filter it (same as Step 2 fallback):
   # keep only [PATCH]/[RFC] subjects, sort by index, save as filtered.mbox
   # Then remove the original thread.mbox to prevent find from picking it up:
   if [ -f thread.mbox ]; then rm thread.mbox; fi

   # Apply prerequisite patches ON THE REVIEW BRANCH
   cd <repo-root>
   PREREQ_DIR="/tmp/loupe-review-<timestamp>/prereq-<n>"
   PREREQ_FILES=$(find "${PREREQ_DIR}" -maxdepth 1 \( -name '*.mbx' -o -name '*.mbox' \) -print | sort)
   echo "${PREREQ_FILES}" | xargs git am
   ```
3. If prerequisite application fails:
   - Try `--3way`
   - If still fails, inform user and ask whether to:
     - Skip prerequisite and try current series anyway
     - Abort review
     - Manually resolve conflicts
4. After all prerequisites are applied, **update `$REVIEW_BASE`** to
   the current HEAD so that the review range excludes prerequisite
   commits:
   ```bash
   $REVIEW_BASE = $(git rev-parse HEAD)
   ```
   This ensures `$REVIEW_BASE..$REVIEW_TIP` in Steps 9/checkpatch
   covers only the current series, not the prerequisites.
5. Proceed to Step 5

**Note**: Some implicit dependencies may not be declared. If patch application
fails in Step 5, suggest checking lore.kernel.org for recent related series
that might be prerequisites.

### Step 5: Apply patches

Apply patches from the download directory. Collect all patch files first
to avoid unmatched globs being passed literally to `git am`:

```bash
PATCH_DIR="/tmp/loupe-review-<timestamp>"
PATCH_FILES=$(find "${PATCH_DIR}" -maxdepth 1 \( -name '*.mbx' -o -name '*.mbox' -o -name '*.patch' \) -print | sort)
if [ -z "${PATCH_FILES}" ]; then
    Error: "No patch files found in ${PATCH_DIR}"
    Exit.
fi
echo "${PATCH_FILES}" | xargs git am
```

On failure: show error, `git am --abort`, retry with `--3way`.
Still failing: ask user.

### Step 6: Patch intent summary

**IMPORTANT**: Before diving into detailed code review, provide a clear summary
of what the patch series intends to do. This helps the user understand the
overall goal and context.

Present to the user:

1. **Series Overview**:
   - Title and author
   - Number of patches
   - Base branch and any prerequisites applied

2. **What Problem Does It Solve?**:
   - Extract from cover letter and commit messages
   - What bug is being fixed or feature being added?
   - Why is this change needed?

3. **How Does It Solve It?**:
   - High-level approach taken
   - Key architectural or design decisions
   - Which subsystems/files are affected

4. **Scope of Changes**:
   - Number of files modified
   - Which architectures/targets are affected
   - Is this a refactor, bug fix, new feature, or API change?

5. **Dependencies and Context**:
   - Any prerequisite patches that were applied
   - Related patch series or issues mentioned
   - Links to relevant documentation or specifications

**Format**: If `+zh` flag is set, present in Chinese. Otherwise present in
English. Keep it concise — a few bullet points, not paragraphs.

After presenting the summary, proceed directly to code review (do NOT ask
the user for permission to continue).

### Step 7: Patchwork & mailing list context collection

**Local mode** (`commit`/`range`): **Skip this entire step.** Local
commits have no message-id, so Patchwork and lore queries are not
applicable. Proceed directly to Step 8.

For remote modes (`lore`/`msgid`/`search`):

Before reviewing the code, gather external context from Patchwork and the
mailing list archive. This surfaces prior reviewer feedback, version history,
and related work that pure code reading cannot reveal.

Use the **WebFetch** tool for all API calls and page fetches below. If
WebFetch is not available (e.g., in Codex environments), use `curl` via
the Bash tool instead:
```bash
curl -sL "<url>" | jq '...'   # For JSON APIs
curl -sL "<url>"               # For HTML pages
```
If a query fails or returns empty, skip it and move on — this step is
best-effort.

#### 7a: Find the patch on Patchwork

`$PATCHWORK_BASE` was already set in Step 1 based on `$MAILING_LIST`.

If `$SERIES_ID` is set (from Step 1.5e subject search), query by series
directly:
```
WebFetch: $PATCHWORK_BASE/series/$SERIES_ID/
```

Otherwise, query by message-id:
```
WebFetch: $PATCHWORK_BASE/patches/?project=$MAILING_LIST&msgid=<message-id>
```

From the response, extract:
- `id` — needed for comment queries
- `state` — patch status (New / Under Review / Accepted / Rejected / …)
- `delegate` — assigned maintainer (if any)
- `series[0].id` — series ID for further queries
- `check` — CI check status

If the patch is not found on the primary instance, try the fallback:
- For `qemu-devel`: use Patchew (`https://patchew.org/api/v1/projects/qemu/series/?message_id=<message-id>`)
- For other lists: try `https://patchwork.kernel.org/api/patches/?project=$MAILING_LIST&msgid=<message-id>`

Record which backend returned data: set `$CONTEXT_BACKEND` to
`"patchwork"` or `"patchew"`. This determines which API to use in
subsequent substeps.

#### 7b: Collect review comments across the whole series

Comments, R-b tags, and blockers may be on any patch in the series,
not just the one found in 7a. Query all patches in the series.

If `$CONTEXT_BACKEND = "patchwork"`:
1. First, list all patches in the series:
   ```
   WebFetch: $PATCHWORK_BASE/series/<series_id>/
   ```
   Extract the list of patch IDs from the `patches` array.
2. For **each** patch ID, fetch comments:
   ```
   WebFetch: $PATCHWORK_BASE/patches/<patch_id>/comments/
   ```
3. Aggregate all comments across patches.

If `$CONTEXT_BACKEND = "patchew"`:
```
WebFetch: https://patchew.org/api/v1/series/<series_id>/messages/
```
(Patchew returns all messages in the series in one call.)

Extract:
- Reviewer name & email
- Comment content (inline review feedback)
- Any Reviewed-by / Acked-by / Tested-by tags
- Concerns or blockers raised

#### 7c: Fetch mailing list thread discussion from lore

Retrieve the full discussion thread from lore.kernel.org:
```
WebFetch: https://lore.kernel.org/$MAILING_LIST/<message-id>/t/
```

Look for:
- Replies from maintainers (look for `@redhat.com`, `@linaro.org`, or
  known QEMU maintainers)
- Requests for changes or outstanding objections
- Positive signals (Reviewed-by, LGTM, "queued", "applied")

#### 7d: Version history — find prior versions

Use `$SERIES_VERSION` and `$SUBJECT_STEM` from Step 3a. If version > 1:

1. Use the subject stem (already cleaned in Step 3a) to search for earlier
   versions on Patchwork:
   ```
   WebFetch: $PATCHWORK_BASE/patches/?project=$MAILING_LIST&q=<subject_stem>&order=-date&per_page=15
   ```

2. For each earlier version found, fetch its comments (7b) to understand:
   - What feedback was given on v1, v2, etc.
   - Whether the current version addresses those concerns
   - Recurring issues across versions

3. Cross-reference the `$CHANGELOG` (from Step 3a) against prior version
   feedback: does the changelog claim to address the issues reviewers
   raised? Are there concerns from prior versions that the changelog does
   not mention?

Also search lore for the earlier thread:
```
WebFetch: https://lore.kernel.org/$MAILING_LIST/?q=<subject-stem-keywords>&o=-1
```

#### 7e: Related patches in the same subsystem

Identify the subsystem from modified file paths (e.g., `hw/i386/` →
`intel_iommu`, `hw/virtio/` → `virtio`).

Search Patchwork for recent activity in the same area:
```
WebFetch: $PATCHWORK_BASE/patches/?project=$MAILING_LIST&q=<subsystem-keyword>&state=*&order=-date&per_page=10
```

Focus on:
- Patches touching the same files in the last 3 months
- Ongoing refactors or cleanup series that might conflict
- Recently accepted patches that establish new patterns

#### 7f: Context synthesis

Compile the collected context into a brief summary for reference in later
steps:

1. **Review status**: patch state, assigned maintainer, CI results
2. **Existing feedback**: key points from reviewer comments (both Patchwork
   and lore), any Reviewed-by/Acked-by tags already given
3. **Version evolution** (if applicable): what changed between versions, which
   prior concerns were addressed, which remain open
4. **Subsystem activity**: related recent patches, potential conflicts, new
   patterns to follow

Present this summary briefly (use Chinese if `+zh`, otherwise English)
before proceeding to Git history analysis.

### Step 8: Git history analysis

Before reviewing the code changes themselves, check the git history of each
modified file to gather context. This provides crucial review evidence that
pure code reading cannot:

```bash
# For each file touched by the series:
# Use $REVIEW_BASE for local modes, <base_branch> for remote modes
git log --oneline -20 $REVIEW_BASE -- <file_path>
```

Focus on:
- Recent refactors or bug fixes in the same area — does the patch conflict with
  or duplicate recent work?
- Original author and reviewers of the code being modified — are they CC'd?
- Patterns and conventions established by prior commits (naming, error handling
  style, memory management idioms).
- Whether the patch reverts or contradicts a previous intentional change.

Use `git log -p` or `git show <hash>` on specific historical commits when a
deeper look is needed (e.g., to understand why a particular API was chosen).

### Step 8.5: Code context prefetching

Before starting the multi-stage code review, gather deep context about the
code being modified. This step provides the reviewer with comprehensive
understanding beyond what the diff alone shows.

#### 8.5a: Read full modified files

For each file touched by the series, read the **complete file content** at
the reviewed revision (not just the diff hunks).

**IMPORTANT**: For local `commit`/`range` modes, the worktree may not
match `$REVIEW_TIP`. Use `git show $REVIEW_TIP:<file_path>` to read
files at the correct revision instead of reading from the filesystem.
For remote modes (where patches were applied to the current branch),
reading the worktree directly is acceptable.

This allows understanding:
- The surrounding code structure
- How the modified function fits into the file
- Existing patterns and conventions in the file
- Other related code that may be affected by the change

#### 8.5b: Identify and read related definitions

From the diff, extract key identifiers (struct names, function names, macros,
type definitions) and locate their definitions.

**IMPORTANT**: For local `commit`/`range` modes, use `git grep` at the
reviewed revision instead of worktree `grep`, since the worktree may
not match `$REVIEW_TIP`:

```bash
# For each important identifier in the diff:
# Local modes: use git grep at $REVIEW_TIP
git grep -n "typedef.*<type_name>" $REVIEW_TIP -- '*.h' '*.c'
git grep -n "struct <struct_name> {" $REVIEW_TIP -- '*.h'
git grep -n "<function_name>(" $REVIEW_TIP -- '*.h' '*.c' | head -10

# Remote modes (worktree matches): grep or git grep both work
```

Read the header files that define types, macros, and APIs used in the patch.

#### 8.5c: Trace callers and callees

For modified or newly added functions:
- Find call sites (who calls this function?)
- Find callees (what does this function call?)
- Identify the ownership and lifecycle patterns in the call chain

```bash
# Local modes: git grep at $REVIEW_TIP
git grep -n "<function_name>" $REVIEW_TIP -- '*.c' '*.h' | head -20

# Remote modes: worktree grep is also acceptable
```

#### 8.5d: Context budget

Keep total prefetched context reasonable:
- Read at most 10 files fully
- For large files (>500 lines), focus on the sections relevant to the diff
- Prioritize: modified files > header files > caller/callee files

### Step 8.6: Launch parallel review (mode-dependent)

This step launches a parallel review if the current mode supports it.

**Mode check**:
- `$REVIEW_MODE = "ci"` → **Skip** this step entirely (single agent).
- `$REVIEW_MODE = "claude-only"` → **Skip** this step entirely.
- `$REVIEW_MODE = "claude+codex"` → Launch **one** Codex review in background.
- `$REVIEW_MODE = "codex-dual"` → Launch **Agent-2** (second Codex) in
  background. Agent-1 (current agent) continues with Step 9.

**For claude+codex and codex-dual modes**, launch codex review in background.

Always use an explicit diff to ensure Codex reviews exactly the same
commit range as the primary reviewer, without including uncommitted
changes or failing on root commits.

```bash
# Generate the diff matching $REVIEW_BASE..$REVIEW_TIP
cd <repo_root>
if [ "$ROOT_COMMIT" = "true" ]; then
    git diff-tree -p --root $REVIEW_TIP > /tmp/loupe-review-<timestamp>/review.diff
else
    git diff $REVIEW_BASE..$REVIEW_TIP > /tmp/loupe-review-<timestamp>/review.diff
fi

# Use Bash tool with run_in_background=true
codex exec "Review the following code diff for bugs, security issues, \
    and correctness problems. Output findings with [P0-9] severity markers." \
    < /tmp/loupe-review-<timestamp>/review.diff \
    > /tmp/loupe-review-<timestamp>/codex-review.log 2>&1
```

This approach ensures: (1) root commits work (via `diff-tree --root`),
(2) uncommitted changes are excluded, and (3) the diff scope matches
the primary reviewer's `$REVIEW_BASE..$REVIEW_TIP` exactly.

The codex review runs in background while the current agent proceeds with
Step 9. Its output will be collected in Step 9.5.

**For codex-dual mode specifically**: both Agent-1 and Agent-2 independently
execute the full five-stage review. Agent-2 is the background codex process;
Agent-1 is the current agent running Step 9.

### Step 9: Multi-stage code review protocol

**IMPORTANT**: Instead of a single monolithic review pass, perform the review
in multiple focused stages. Each stage targets a specific category of issues.
This multi-stage approach (inspired by the sashiko kernel review system) reduces
blind spots and improves issue detection.

First, gather the basic diff information. If `$ROOT_COMMIT` is true,
use the root-commit alternatives from Step 2b. Otherwise:

1. `git log --oneline $REVIEW_BASE..$REVIEW_TIP`
2. `git diff $REVIEW_BASE..$REVIEW_TIP --stat`
3. `git show <hash>` for each commit

For root commits:
1. `git log --oneline $REVIEW_TIP` (single commit)
2. `git diff-tree -p --root --stat $REVIEW_TIP`
3. `git show $REVIEW_TIP`

Then execute the following review stages sequentially:

#### Stage A: Conceptual & Implementation Verification

Focus **only** on whether the patch does what it claims:

- Does the code change match the commit message description?
- Is the architectural approach sound for this problem?
- Are API contracts and interfaces respected?
- Does the implementation align with QEMU's design patterns for this subsystem?
- Are there simpler alternatives that achieve the same goal?

For each concern, cite the specific commit message claim and the code that
contradicts or insufficiently implements it.

#### Stage B: Correctness & Logic Analysis

Focus **only** on logic bugs and correctness issues:

- Trace execution flow through the modified code paths
- Off-by-one errors, boundary conditions, integer overflow/underflow
- Null pointer dereferences, uninitialized variables
- Error handling: are all error paths handled? Do they clean up properly?
- Edge cases: empty input, maximum values, concurrent access
- Type correctness: signedness, truncation, implicit conversions
- Return value checking: are error returns from called functions checked?

For each finding, provide the **exact execution path** that triggers the bug.

#### Stage C: Resource Management & Concurrency

Focus **only** on resource lifecycle and thread safety:

- Memory management: leaks, use-after-free, double-free
- Reference counting: `object_ref`/`object_unref` balance
- Lock ordering and potential deadlocks (BQL, per-device locks)
- Thread-safety of shared state across vCPU threads
- QEMU Object Model lifecycle: realize/unrealize, instance_init/finalize
- QOM property getters/setters during migration
- `Error **errp` propagation: is `error_propagate` used correctly?
  Are errors set before returning failure?

#### Stage D: Security & Device Emulation Review

Focus **only** on security boundaries and hardware emulation correctness:

- **Guest input validation**: All values from guest (MMIO reads/writes, PCI
  config, DMA descriptors) MUST be validated. QEMU is a security boundary
  between guest and host.
- Buffer overflows from guest-controlled sizes or indices
- Integer overflows in address calculations
- DMA: `dma_memory_read`/`dma_memory_write` with guest-controlled addresses
- MMIO handlers: proper range checking, alignment handling
- PCI BAR and config space access validation
- Side-channel considerations in crypto or security-relevant code
- Migration compatibility: field versioning, subsection guards

#### Stage E: Finding Verification & Deduplication

Review ALL findings from stages A–D and apply strict quality control:

1. **Eliminate duplicates**: Merge findings that describe the same root issue
2. **Require concrete evidence**: Each finding MUST include:
   - Exact file path and line number(s)
   - The problematic code snippet
   - A clear explanation of WHY it is wrong (reference to spec, convention,
     or demonstrable logic flaw)
   - A concrete scenario or execution path that triggers the issue
3. **Dismiss speculative findings**: If you cannot construct a specific scenario
   that triggers the issue, demote it to a "note" or remove it entirely.
   Speculation wastes reviewer and author time.
4. **Classify severity**:
   - **Critical**: Data corruption, security vulnerability, crash, or
     migration breakage
   - **Major**: Functional bug, resource leak, incorrect behavior
   - **Minor**: Suboptimal code, missing edge case handling, style violation
     with functional impact
   - **Nit**: Pure style, naming, or documentation issues
5. **Cross-reference Patchwork context** (from Step 7): check whether the
   patch addresses feedback from prior versions, follows patterns established
   by recent subsystem patches, and aligns with maintainer requests

#### Checkpatch

Run checkpatch on each patch:
```bash
# For root commits, use --root; otherwise use range
if [ "$ROOT_COMMIT" = "true" ]; then
    git format-patch --root $REVIEW_TIP -o /tmp/loupe-review-<timestamp>/checkpatch/
else
    git format-patch $REVIEW_BASE..$REVIEW_TIP -o /tmp/loupe-review-<timestamp>/checkpatch/
fi
./scripts/checkpatch.pl /tmp/loupe-review-<timestamp>/checkpatch/*.patch
```

### Step 9.5: Collect and cross-reference parallel review

**Mode check**:
- `$REVIEW_MODE = "ci"` → **Skip** this step entirely (no parallel review).
- `$REVIEW_MODE = "claude-only"` → **Skip** this step entirely.
- `$REVIEW_MODE = "claude+codex"` or `"codex-dual"` → Proceed.

If a parallel review was launched in Step 8.6, collect its results now.

#### 9.5a: Read Codex output

Read the codex review log file:
```
/tmp/loupe-review-<timestamp>/codex-review.log
```

If the background command has not finished yet, wait for it to complete (it was
launched with `run_in_background`).

If codex exited with non-zero or produced empty output, note the failure and
proceed with Claude-only findings. Do not block the review.

#### 9.5b: Parse Codex findings

Codex review outputs issues with `[P0-9]` severity markers:
```
- [P0] Critical issue description - /path/to/file.c:line-range
  Detailed explanation.
- [P1] High priority issue - /path/to/file.c:line-range
  Detailed explanation.
```

Extract all findings with their severity, file path, line range, and
description.

#### 9.5c: Cross-reference with Claude findings

Compare Codex findings against Claude's findings from Stages A–E:

1. **Both agree** (`[Claude+Codex]`): Same issue found by both reviewers.
   Match by file + line range + issue category. These have highest confidence.
2. **Claude only** (`[Claude]`): Found by Claude but not by Codex. Present
   as normal.
3. **Codex only** (`[Codex]`): Found by Codex but not by Claude. Claude MUST
   verify each Codex-only finding before including it:
   - Read the relevant code and trace the execution path
   - If the finding is valid, include it with `[Codex]` attribution
   - If the finding is a false positive, discard it with a brief note in
     the internal cross-reference log

Record the cross-reference results for use in Steps 10 and 11.

### Step 10: Summary

**CI mode**: If `$CI_MODE`, skip this step (no interactive display). Proceed
directly to Step 11.

Present the **`## Summary`** section content (as defined in Step 11a) to the
user in the conversation. This is the same content that will go into the
review file header — no need to produce it twice.

The summary covers: verdict, version history, patchwork/ML context, findings
table, and checkpatch. See Step 11a for the exact format.

If Codex was available, tag findings: `[C+X]` both, `[X]` codex-only.
Keep the entire summary under 40 lines.

### Step 11: Generate inline review reply

**CI mode**: If `$CI_MODE`, skip this step entirely. Proceed to Step 12
(JSON output).

Generate a review reply file saved to `~/loupe/reply/`.

```bash
mkdir -p ~/loupe/reply
```

File naming: `~/loupe/reply/<series-short-name>-<version>-reply.md`

#### 11a: File structure

The file always starts with an **English executive summary** (outside code
blocks, for the reviewer's own reference), followed by **English reply code
blocks** (ready to paste to the mailing list). A Chinese section is appended
only when `+zh` is passed.

```markdown
# Review: [PATCH vN 0/M] <title>
<!-- metadata: author, date, msgid, patchwork ID, base -->

## Summary

**Verdict**: Needs revision / Ready to merge / Blocked
**Author**: <name>, **Patches**: <N>, **Base**: <branch>

### Version history
| Ver | Date | # | Key change |
|-----|------|---|------------|
| v1  | ... | 8 | Initial post |
| v2  | ... | 5 | Dropped foo per reviewer X |
| vN  | ... | 7 | Current — added tests |

### Patchwork & ML context
- **State**: new / under review / accepted
- **R-b tags**: <who> on <which patches>
- **CI**: pass / fail / pending
- **Prior feedback**: <1-2 bullet points of key unresolved
  concerns from earlier versions>
- **Maintainer activity**: <queued? / silent? / asked for changes?>

### Findings (<N>C / <N>M / <N>m / <N>n)
| # | Sev | Patch | Issue (one line) |
|---|-----|-------|------------------|
| 1 | C   | 2/7   | ... |
...

### checkpatch
<clean / list issues>

## 中文分析              ← only if +zh flag is set
<!-- Same content as Summary, translated to Chinese. -->

---

## Reply to [PATCH vN 0/M] cover letter
` ` `
<English inline reply — ready to send>
` ` `

## Reply to [PATCH vN 1/M] <subject>
` ` `
<English inline reply>
` ` `
...
```

Key rules:
- The `## Summary` section is **always present** (English).
- The `## 中文分析` section is **only included when `+zh` is in the
  arguments**.
- Everything inside fenced code blocks (the replies) is **always pure
  English, mailing-list ready**.
- The Summary section should be **≤40 lines** total. Version history
  and findings tables are the core — skip subsections that add no info
  (e.g., skip "Version history" for v1 patches).

#### 11b: Mailing list vocabulary — use freely

| Shorthand | Meaning |
|-----------|---------|
| LGTM | Looks Good To Me |
| nit: | Nitpick (trivial style issue) |
| s/foo/bar/ | Replace "foo" with "bar" |
| FWIW | For What It's Worth |
| IIUC | If I Understand Correctly |
| AFAICT | As Far As I Can Tell |
| IOW | In Other Words |
| WRT | With Regard To |
| FYI | For Your Information |
| NB | Nota Bene (important note) |
| w/ / w/o | with / without |
| OOB | Out Of Bounds |
| UAF | Use After Free |
| DTRT | Do The Right Thing |
| IWBN | It Would Be Nice |
| R-b: / A-b: / T-b: | Reviewed-by / Acked-by / Tested-by |
| NAK | Negative Acknowledgment (reject) |

Before generating any reply, determine the reviewer's identity:
```bash
$REVIEWER_NAME = $(git config user.name)
$REVIEWER_EMAIL = $(git config user.email)
```
If not set, ask the user for their name and email.

Also use conventions like:
- `Reviewed-by: $REVIEWER_NAME <$REVIEWER_EMAIL>` for clean patches
- Bare `LGTM.` when a patch is obviously correct
- `s/ELFDATA2LSB/info->is_big_endian ? ELFDATA2MSB : ELFDATA2LSB/`
  instead of "please change ELFDATA2LSB to a conditional expression"

#### 11c: Brevity rules — CRITICAL

**Write like a seasoned kernel reviewer, not a tutorial.**

1. **One-liners for nits**: `nit: s/foo/bar/` — done.
2. **2-3 lines for bugs**: state the bug, cite the spec/code, suggest fix.
3. **Skip clean patches** entirely, or just: `LGTM.` / give R-b.
4. **Skip clean hunks** — don't quote code just to say "looks good".
   Use `[...]` liberally.
5. **Don't re-explain** the code being reviewed — the author wrote it.
6. **Don't repeat** the commit message description.
7. **Don't pad** with "Thanks for this patch", "This is a nice
   improvement", etc. Get to the point.
8. **Cover letter reply**: one short paragraph of overall impression,
   then bullet-list the blockers. Nothing else.
9. **Max lengths**:
   - Cover letter reply: ≤10 lines
   - Per-patch reply: ≤15 lines (excluding quoted context)
   - Individual comment: ≤3 lines (unless explaining a complex bug)
10. **If nothing to say about a patch, omit its section entirely.**

Anti-patterns (NEVER do these):
- ❌ "The split between instruction entries (0-5, always LE) and data
      entries (6-9, endianness-dependent) is correct and well-commented."
  → Just skip it. Clean code doesn't need praise.
- ❌ "The firmware dynamic info encoding looks correct. OpenSBI will
      need matching big-endian support to consume this."
  → Skip. The author knows their downstream.
- ❌ "Good — cpu_synchronize_state() ensures mstatus is current when
      running under KVM."
  → Skip. Restating obvious correct code is noise.
- ❌ Quoting 10 lines of diff just to say "LGTM"
  → Don't quote code you have no comment on.

Good patterns:
- ✅ `nit: s/.virtio_is_big_endian/.internal_is_big_endian/ in subject`
- ✅ `This won't compile — MSTATUS_SBE is undefined. Restore the
      #define from v4 patch 1.`
- ✅ `Per priv spec §3.1.6.1, endianness w/ MPRV=1 should use MPP,
      not env->priv. Masked today since all bits are identical, but
      worth fixing now.`
- ✅ `ELFDATA2LSB is still hardcoded — BE ELFs will be rejected.
      s/ELFDATA2LSB/info->is_big_endian ? ELFDATA2MSB : ELFDATA2LSB/`

#### 11d: Formatting rules

- Line width: target 72 chars, hard limit 78 (excluding quoted lines).
- `On <date>, <author> wrote:` header for each patch.
- Quote with `> ` prefix. Use `[...]` to skip irrelevant hunks.
- Place comments directly below the relevant quoted line(s).
- End each per-patch reply with `Thanks,\n$REVIEWER_NAME`.
- Wrap reply body in markdown fenced code block (` ``` `).

#### 11e: Codex attribution

When Codex was available, prefix cross-referenced comments:
- `[C+X]` — both found it
- `[X]` — codex-only, verified by Claude
- No tag for Claude-only (default)

If Codex unavailable, omit tags. Add at file end:
`Note: Claude-only review (Codex unavailable).`

#### 11f: Example

```
## Reply to [PATCH v5 3/7] target/riscv: Implement runtime data endianness

` ` `
On Tue, 24 Mar 2026 16:40:16 +0000, Djordje Todorovic wrote:
> - Update mo_endian_env() in op_helper.c to call the
>   new helper

Commit message says op_helper.c is updated, but the
diff doesn't touch it. Stale description from rebase?

[...]
> +static inline MemOp mo_endian_env(CPURISCVState *env)
> +{
> +    return riscv_cpu_data_is_big_endian(env)
>            ? MO_BE : MO_LE;
> +}

Build failure: op_helper.c:32 already defines
mo_endian_env() (returns MO_TE), and it #includes
internals.h. Drop the op_helper.c copy.

[...]
> +    switch (env->priv) {
> +    case PRV_M:
> +        return env->mstatus & MSTATUS_MBE;

Per priv spec §3.1.6.1, when MPRV=1 endianness
should follow MPP, not current priv. Masked today
(all bits identical at reset) but worth fixing.

Thanks,
$REVIEWER_NAME
` ` `
```

### Step 12: Generate structured JSON output (CI mode only)

**If not `$CI_MODE`**: skip this step.

Construct the review JSON using all data gathered in previous steps. The JSON
must conform to the loupe review schema v1:

```json
{
  "schema_version": "1",
  "id": "<YYYYMMDD>-<short-description>-v<version>",

  "series": {
    "title": "<full series title from Step 3>",
    "version": "<$SERIES_VERSION>",
    "patch_count": "<number of patches>",
    "author": { "name": "<author name>", "email": "<author email>" },
    "date": "<submission date YYYY-MM-DD>",
    "subsystem": "<auto-inferred from patch file paths>",
    "message_id": "<original message-id>",
    "lore_url": "https://lore.kernel.org/$MAILING_LIST/<message-id>/",
    "patchwork_url": "<patchwork URL if found>",
    "base_branch": "<base branch>"
  },

  "version_history": [
    {
      "version": 1,
      "date": "<date>",
      "message_id": "<msgid>",
      "key_change": "<description>"
    }
  ],

  "ml_context": {
    "patchwork_state": "<state from Step 7a>",
    "reviewed_by": ["<R-b tags from Step 7b>"],
    "acked_by": ["<A-b tags from Step 7b>"],
    "ci_status": "<pass/fail/pending from Step 7a>",
    "prior_feedback": ["<key feedback from Step 7d>"],
    "maintainer_activity": "<queued/silent/asked for changes>"
  },

  "review": {
    "verdict": "<needs_revision/ready_to_merge/blocked>",
    "summary": "<one-line summary from Step 10>",
    "mode": "<$REVIEW_MODE>",
    "stages": {
      "A": "<Stage A one-line summary>",
      "B": "<Stage B one-line summary>",
      "C": "<Stage C one-line summary>",
      "D": "<Stage D one-line summary>",
      "E": "<Stage E one-line summary>"
    },
    "findings": [
      {
        "id": 1,
        "severity": "<critical/major/minor/nit>",
        "stage": "<A/B/C/D or combined e.g. B+D>",
        "patch_index": "<n/m>",
        "file": "<file path>",
        "line": "<line or line-range>",
        "patch_context": [
          "<diff lines relevant to this finding>"
        ],
        "title": "<short title>",
        "description": "<explanation>",
        "suggestion": "<fix suggestion or null>",
        "confidence": "<high/medium/low>",
        "confidence_reason": "<why this confidence level>",
        "source": "<claude/codex/claude+codex/ci-single>"
      }
    ],
    "checkpatch": {
      "status": "<clean/issues>",
      "issues": ["<checkpatch issues if any>"]
    }
  },

  "generated_at": "<ISO 8601 timestamp>",
  "generator": "loupe-review v1.0",
  "disclaimer": "LLM-generated draft. Not an authoritative review."
}
```

#### Output path resolution

Determine the output file path:
1. If `$OUTPUT_JSON_PATH` is set:
   - If it ends with `/` or is an existing directory: write to
     `$OUTPUT_JSON_PATH/<id>.json`
   - Otherwise: use as full file path
2. If `$OUTPUT_JSON_PATH` is not set:
   - Default: `/tmp/loupe-review-<timestamp>/review.json`

```bash
mkdir -p "$(dirname "$OUTPUT_PATH")"
```

Write the JSON to `$OUTPUT_PATH`.

Print: `Review JSON written to: $OUTPUT_PATH`

#### Subsystem inference

To populate `series.subsystem`, scan the file paths modified by the patches:
- `hw/riscv/` or `target/riscv/` → `riscv`
- `hw/riscv/riscv-iommu` → `riscv-iommu`
- `target/riscv/vector` → `riscv-vector`
- `target/riscv/kvm/` → `riscv-kvm`
- `hw/arm/` or `target/arm/` → `arm`
- `hw/i386/` or `target/i386/` → `x86`
- `hw/virtio/` → `virtio`
- `hw/block/` → `block`
- `hw/net/` → `net`
- Use the most specific match. If multiple subsystems, use the most common one.
