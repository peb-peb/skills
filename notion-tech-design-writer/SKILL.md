---
name: notion-tech-design-writer
description: Use when creating, completing, refactoring, or formatting a Notion technical design doc, DB schema doc, API definitions page, implementation plan, or engineering spec from PRDs, local code, schema drafts, comments, or reference tech docs.
metadata:
  short-description: Write polished Notion tech design docs
---

# Notion Tech Design Writer

## Style Rule

No jargon, no blabbering. Keep the doc short, concise, and to the point.

## Workflow

1. Fetch the target Notion page.
2. Fetch the related PRD and reference tech doc if provided.
3. Fetch linked child pages such as DB Schema and API Definitions.
4. Read unresolved comments and incorporate decisions.
5. Read local source docs, Prisma schema, services, DTOs, and controllers when relevant.
6. Separate product behavior from implementation detail.
7. Update stale or empty child pages when the tech doc depends on them.
8. Preserve child pages/databases when replacing Notion content.
9. Fetch updated pages and fix formatting issues.

## Tech Design Template

# 1. Overview
State what is being built and which systems it touches.

# 2. Goals and Non-goals
Separate implementation goals from excluded work.

# 3. Current System Context
Summarize existing code, services, schema, and gaps.

# 4. Requirements Summary
Split into functional and non-functional requirements.

# 5. Proposed Architecture
List modules, services, dependencies, and ownership.

# 6. Data Model
Summarize models, relations, constraints, and link DB Schema child page.

# 7. API Design
Summarize API groups and link API Definitions child page.

# 8. Lifecycle Flows
Document core flows step by step.

# 9. Access Control
List public, authenticated, owner/member/leader, and admin permissions.

# 10. Operational Details
Cover transactions, capacity, payment, storage, visibility, notifications, or background jobs.

# 11. Edge Cases
List realistic failure cases and expected behavior.

# 12. Observability
List logs, metrics, and audit events.

# 13. Testing Plan
Cover unit, integration, permission, migration, and concurrency tests.

# 14. Migration and Rollout Plan
Give ordered rollout steps.

# 15. Risks and Open Questions
Separate known risks from unresolved decisions.

# 16. Implementation Phases
Break work into buildable phases.

## Quality Bar

A good tech design should let engineering implement without guessing models, APIs, permissions, or rollout risks.
