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

## Git Provider

The git CLI tool is read from `local/tools.md` under "Git Provider" → **CLI**.

If no CLI is configured, inform the user and suggest adding it to `local/tools.md`.
