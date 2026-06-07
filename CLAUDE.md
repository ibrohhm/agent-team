# agent-team

## Commit Rules

- Never commit without explicit user request
- Never use `git add .` or `git add -A` — stage specific files by name
- Never amend published commits
- Never skip `--no-verify`

## Version Bump

Whenever you modify any file under `agents/`, `skills/`, or `.claude-plugin/`, bump the patch version in `.claude-plugin/plugin.json` as part of the same change set.

Version format: `MAJOR.MINOR.PATCH` — increment PATCH only unless user specifies otherwise.

Example: `0.3.0` → `0.3.1`
