# Skills

> **Purpose:** This repo is an **archive** of [Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills) for **safe-keeping and quick access**. Each skill is a self-contained `SKILL.md` (name + description + instructions) that can be dropped into a Claude Code / Codex skills directory and used directly.

Skills here are copied verbatim from local working setups (e.g. `~/.codex/skills`). The directory name is preserved as-is so a skill can be restored to its original location without rename surprises.

## Index

### 📝 Notion Documentation

| Skill | What it does | Use when |
| --- | --- | --- |
| [`notion-prd-writer`](./notion-prd-writer/SKILL.md) | Writes polished, concise Notion PRDs from rough product notes, code context, and reference docs. | Creating, completing, refactoring, or formatting a Notion PRD. |
| [`notion-tech-design-writer`](./notion-tech-design-writer/SKILL.md) | Writes Notion technical design docs, DB schema docs, API definition pages, and implementation plans. | Drafting or refactoring an engineering spec / tech design from a PRD, local code, or schema drafts. |

### ✉️ Copywriting / Notifications

| Skill | What it does | Use when |
| --- | --- | --- |
| [`oscode_notification_template_copywriter`](./oscode_notification_template_copywriter/SKILL.md) | Drafts and reviews transactional email/WhatsApp notification templates and returns implementation-ready metadata (snake_case names, variables, subject, body_text, body_html, notes). | Reviewing, drafting, or converting lifecycle/transactional notification copy (e.g. mentor/student onboarding). |

## Usage

To use a skill, place (or symlink) its directory into your agent's skills folder:

```bash
# Claude Code (project- or user-level)
cp -R notion-prd-writer ~/.claude/skills/

# Codex
cp -R notion-prd-writer ~/.codex/skills/
```

The agent picks it up by the `name` and `description` in the `SKILL.md` frontmatter.

## Conventions

- One skill per directory; the entry point is always `SKILL.md`.
- Frontmatter requires `name` and `description`; `metadata.short-description` is optional.
- Directory names are kept identical to their source for faithful restore.

## License

[MIT](./LICENSE)
