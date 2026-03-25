# loupe-review

LLM-assisted patch review with multi-agent cross-reference for mailing list patches.

## When to Use

Use this skill when the user asks to:
- Review a patch from lore.kernel.org
- Review a mailing list submission
- Apply and review patches from lore
- Review a patch series
- Check a QEMU or Linux kernel patch
- Analyze commit quality

## How to Use

When the user provides a patch source (lore URL, Message-Id, subject keywords, local commit, or commit range), follow the full review workflow defined in the loupe-review command file.

The user will specify the input in their message. Parse their request to determine:
- **source**: lore URL, Message-Id, subject search keywords, local commit SHA, or commit range
- **base_branch** (optional): branch to diff against, default `master`
- **flags**: `+zh` for Chinese output, `--ci` for JSON output, `--base <branch>`, `--output-json <path>`

Then execute the loupe-review workflow from `commands/loupe-review.md` in this skill's source repository.

## Input Examples

- `review https://lore.kernel.org/qemu-devel/20240101120000.12345-1-author@example.com/t.mbox.gz`
- `review <msgid@domain>`
- `review riscv iommu fix`
- `review HEAD`
- `review master..HEAD`
- `review abc1234 --base stable-9.1 --ci --output-json /tmp/out/`

## Review Modes

This skill automatically detects the environment:
- **Codex dual-agent mode**: launches two independent Codex agents for parallel review with cross-reference
- **CI single-agent mode** (with `--ci`): single agent, JSON output, no interaction

## Workflow Reference

The complete review workflow is defined in `commands/loupe-review.md` in the loupe repository. Follow all steps as documented there.
