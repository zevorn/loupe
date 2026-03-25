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

Clone the repo and use `--plugin-dir` to register it as a local plugin:

```bash
git clone git@github.com:zevorn/loupe.git ~/loupe
claude mcp add --plugin-dir ~/loupe
```

Or install the skill file directly:

```bash
git clone git@github.com:zevorn/loupe.git
cd loupe
./install.sh --claude
```

After installation, use in Claude Code:

```
/loupe-review <lore-url-or-msgid-or-commit> [base_branch] [+zh] [--ci] [--output-json <path>]
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
| `--output-json <path>` | JSON output path (directory or file) |

### Examples

```
/loupe-review https://lore.kernel.org/qemu-devel/20240101120000.12345-1-author@example.com/t.mbox.gz
/loupe-review master..HEAD
/loupe-review abc1234
/loupe-review riscv iommu fix
/loupe-review <msgid@domain> master --ci --output-json /tmp/review/
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
- `curl` (fallback patch download)
- `codex` (optional, enables multi-agent modes)
- QEMU source tree with `scripts/checkpatch.pl` (for style checking)

## License

MIT License. See [LICENSE](LICENSE) for details.
