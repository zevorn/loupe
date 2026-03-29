# loupe

LLM-assisted patch review plugin for Claude Code and Codex with multi-agent cross-reference.

Supersedes [patch-review](https://github.com/zevorn/patch-review). Designed for open source projects that use mailing list workflows (QEMU, Linux kernel, etc.). Downloads, applies, and performs multi-stage code review with external context from Patchwork and lore.kernel.org.

## Features

- **Multiple input sources**: lore.kernel.org URLs, Message-Ids, subject keyword search, local commits, commit ranges
- **Automatic patch download**: Uses `b4` with graduated fallback to direct lore download
- **Multi-stage review**: Five-stage analysis covering correctness, security, resource management, and more
- **Multi-agent modes**:
  - **Claude + Codex**: Claude reviews with parallel Codex cross-reference
  - **Codex dual-agent**: Two independent Codex reviews with cross-reference
  - **CI single-agent**: Lightweight mode for automated pipelines
- **CI mode**: `--ci` flag for non-interactive JSON output, designed for GitHub Actions
- **External context**: Queries Patchwork and lore for prior reviews, version history, and CI status
- **Inline review generation**: Produces mailing-list-ready review replies

## Installation

### Claude Code (recommended)

Clone the repo and install the skill file:

```bash
git clone git@github.com:zevorn/loupe.git
cd loupe
./install.sh --claude
```

This copies `patch-review.md` to `~/.claude/commands/loupe/`, making
`/loupe:patch-review` available as a slash command.

After installation, use in Claude Code:

```
/loupe:patch-review <lore-url-or-msgid-or-commit> [base_branch] [+zh] [--ci] [--output-json <path>]
```

### Codex

```bash
git clone git@github.com:zevorn/loupe.git
cd loupe
./install.sh --codex
```

### Uninstall

```bash
# Manual install
./install.sh --uninstall
```

## Usage

### Input Formats

| Format | Example |
|--------|---------|
| lore URL | `https://lore.kernel.org/qemu-devel/<msgid>/t.mbox.gz` |
| Message-Id | `<msgid@domain>` or `msgid@domain` |
| Subject search | `virtio-net fix`, `riscv vector` |
| Local commit | `HEAD`, `abc1234` |
| Commit range | `master..HEAD`, `abc1234..def5678` |

### Flags

| Flag | Description |
|------|-------------|
| `+zh` | Include Chinese analysis section |
| `--ci` | CI mode: single agent, no interaction, JSON output |
| `--base <branch>` | Explicit base branch (useful for subject search) |
| `--search` | Force subject search mode (bypass ref detection) |
| `--output-json <path>` | JSON output path (directory or file) |

### Examples

```
/loupe:patch-review https://lore.kernel.org/qemu-devel/20240101120000.12345-1-author@example.com/t.mbox.gz
/loupe:patch-review master..HEAD
/loupe:patch-review abc1234
/loupe:patch-review riscv iommu fix
/loupe:patch-review <msgid@domain> master --ci --output-json /tmp/review/
```

## Review Modes

| Mode | Environment | Behavior |
|------|-------------|----------|
| Claude + Codex | Claude Code with codex installed | Claude five-stage + Codex parallel, cross-reference |
| Claude only | Claude Code without codex | Claude five-stage only |
| Codex dual | Codex CLI | Two independent Codex agents, cross-reference |
| CI single | `--ci` flag | Single agent, JSON output, no interaction |

## Dashboard

Review results from CI mode are published to [loupe-web](https://github.com/zevorn/loupe-web), a static site dashboard at [zevorn.github.io/loupe-web](https://zevorn.github.io/loupe-web/).

## Dependencies

- `git` (required)
- `b4` (recommended, falls back to `curl` if unavailable)
- `curl` (fallback patch download and API queries)
- `jq` (required for Codex environments; used in curl-based API parsing)
- `codex` (optional, enables multi-agent modes)
- QEMU source tree with `scripts/checkpatch.pl` (for style checking)

## Acknowledgments

The multi-stage review protocol (five-stage A-E analysis) and the static
site design are inspired by [Sashiko](https://sashiko.dev/), an agentic
Linux kernel code review system by the
[Linux Foundation](https://www.linuxfoundation.org/)
(Apache License 2.0, source: [github.com/sashiko-dev/sashiko](https://github.com/sashiko-dev/sashiko)).

The review prompt design draws from the open-source
[review-prompts](https://github.com/masoncl/review-prompts) by Chris Mason.

## License

MIT License. See [LICENSE](LICENSE) for details.
