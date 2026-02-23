# dotclaude — Portable AI Engineering Workflow

This repository contains user-level Claude Code commands and agents for
AI-assisted software engineering.

## Local Tool Configuration

Commands and agents in this repo are tool-agnostic. Specific tool configuration
(ticket system, etc.) is defined in:

@local/tools.md

If `local/tools.md` does not exist, copy `local/tools.md.template` and fill in
your tool details.

## Workspace Directory

Commands use `{WORKSPACE_DIR}` to refer to the ticket workspace root.

**Default**: `.ai-workspace`

To override, set "Directory" under the "Workspace" section in `local/tools.md`.

## Git Provider Detection

The git provider CLI is auto-detected from the current repository's remote URL:

```bash
git remote get-url origin
```

**Default mappings**:
- `github.com` → `gh`
- `gitlab.com` → `glab`

If `local/tools.md` contains a "Git Provider Overrides" section, use those
hostname-to-CLI mappings for self-hosted or non-standard setups (e.g.,
`git.mycompany.com = gh`).

If the provider cannot be detected or its CLI is not installed, inform the user.
