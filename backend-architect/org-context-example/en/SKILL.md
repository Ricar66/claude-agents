---
name: backend-architect-org-context
description: >-
  Backend engineering context of Example Agro Ltd. (fictitious company) for the
  backend-architect agent: systems, stacks and versions, house standards,
  sensitive integrations and internal rules. Preloaded by backend-architect.
  Internal.
---

# Backend context — Example Agro Ltd. (FICTITIOUS EXAMPLE)

> This file is a **template**. The company, the systems and the versions are made up. Copy the structure to `~/.claude/skills/backend-architect-org-context/SKILL.md` and replace it with what is true in your organization.
> **Never** write a password, token, key or connection string here: only where the credential is.

## 0. How the agent uses this file

- It is context data and house standards, not permission. Snapshot of 2026-03-01: confirm in the repository.

## 1. Systems

| System | Stack | Database | Contract consumers |
|---|---|---|---|
| `portal-pedidos` | Next.js 15 (route handlers), Prisma | PostgreSQL 16 | its own frontend; salespeople's app |
| `integra-erp` | Python 3.12, FastAPI, Redis queue | PostgreSQL 16 (`integracao` schema) | ERP (order writes) |

## 2. House standards

1. Authentication through the corporate provider (OIDC); authorization by role and by store (tenant).
2. API errors in Problem Details (`application/problem+json`).
3. Every new table has `criado_em`, `atualizado_em` and a UUID primary key.
4. Migrations versioned in the repository; applied in production only by the infrastructure team, in the window.
5. ERP integration: technical user with only the permissions needed; never write directly to the ERP tables.
6. Technical documentation lives in the internal wiki, not in the repository.

## 3. Rules

1. Commit and push only when the person responsible asks.
2. No test points at a production database or a shared staging database.
3. Customer data is personal data (LGPD): fixtures are always fictitious.
