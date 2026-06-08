---
name: notion-prd-writer
description: Use when creating, completing, refactoring, or formatting a Notion PRD from product notes, local docs, code context, comments, open questions, or a reference PRD format.
metadata:
  short-description: Write polished Notion PRDs
---

# Notion PRD Writer

## Style Rule

No jargon, no blabbering. Keep the doc short, concise, and to the point.

## Workflow

1. Fetch the target Notion page.
2. Fetch any linked PRD, tech doc, schema doc, or reference formatting page.
3. Read unresolved comments and Open Questions before editing.
4. Read local source docs/code when the user provides them.
5. Convert rough notes into decisions, flows, rules, risks, and acceptance criteria.
6. Remove duplicate points, vague TODOs, filler, and over-explaining.
7. Preserve child pages/databases when replacing Notion content.
8. Fetch the updated page after writing and fix formatting issues.

## PRD Template

# Summary
State what is being built and why in plain language.

# Scope
## In scope
List what V1 includes.

## Out of scope
List what V1 intentionally excludes.

# Key Decisions
Number the locked product decisions.

# Blockers
List required confirmations before implementation.

# User Flows
Describe the main user/admin flows step by step.

# Rules
Use domain-specific sections, for example:
- Registration Rules
- Team Rules
- Submission Rules
- Visibility Rules
- Invite Rules

# Admin Requirements
List what admins/organisers can do.

# Success Metrics
List measurable outcomes.

# Risks
List risks with short mitigation notes.

# Open Questions
Only include unresolved product questions. If none, say so.

# Acceptance Criteria
Use a table when helpful.

## Quality Bar

A good PRD should let engineering build without guessing the product behavior.
