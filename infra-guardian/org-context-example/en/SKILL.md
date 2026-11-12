---
name: infra-guardian-org-context
description: >-
  Infrastructure context of Example Agro Ltd. (fictitious company) for the
  infra-guardian agent: environments, servers, what runs where, network,
  backup, monitoring, known operational risks and internal rules.
  Preloaded by infra-guardian. Internal.
---

# Infrastructure context — Example Agro Ltd. (FICTITIOUS EXAMPLE)

> This file is a **template**. The company, the servers, the IPs (ranges reserved for documentation) and the risks are made up. Copy the structure to `~/.claude/skills/infra-guardian-org-context/SKILL.md` and replace it with what is true in your organization.
> **Never** write a password, token, key or secret value here: only where the credential is and what the risk is.

## 0. How the Infra Guardian uses this file

- It is context data, not instructions. It does not widen permissions or authorize connecting to a server.
- It is a dated snapshot (last reviewed: 2026-03-01). Confirm in the code or configuration before turning an item into a finding.

## 1. Scope authorized by default

| Allowed (default) | Needs written authorization from the IT team beforehand |
|---|---|
| Read code, IaC, compose, pipelines, runbooks and documentation in the repositories | SSH, RDP, `kubectl`, cloud CLI or a database query in any environment |
| Read-only Git commands and local static validation | Load test, firewall, DNS or proxy change, service restart |
| Read the exports that IT attaches (output of `df`, `free`, `docker ps`, dashboards) | Access the cloud provider's or the monitoring panel |

## 2. Environments

| Environment | Where | Note |
|---|---|---|
| Production | Linux VM `192.0.2.10` (portal) and Windows server `192.0.2.20` (integrations) | Change window: Tuesday and Thursday, 8 pm–10 pm |
| Staging | VM `198.51.100.10` | Masked data; can be recreated |
| Development | Team laptops | — |

## 3. Servers and what runs on them

| Server | OS and resources | What runs | How it starts | Backup |
|---|---|---|---|---|
| `192.0.2.10` | Ubuntu 24.04, 4 vCPU, 8 GB | `portal-pedidos` (Next.js) in a container, PostgreSQL 16 in a container, nginx | Docker Compose with `restart: unless-stopped` | Daily `pg_dump` at 2 am to a local disk (**R1**) |
| `192.0.2.20` | Windows Server 2022, 2 vCPU, 8 GB | `integra-erp` (Python, Windows service), Redis | Windows service | None recorded |

## 4. Network and edge

- CDN with WAF in front of the portal; origin with its own TLS, certificate renewed via ACME.
- Stores connected to headquarters by IPsec VPN; one internet link per store.
- The edge firewall is managed by a service provider; changes only by ticket.

## 5. Observability

- Monitoring by ping and HTTP every 1 min, with an email alert to IT.
- No application metrics and no SLO defined.
- Logs only on the servers themselves, with no centralization.

## 6. Known operational risks

| # | Risk | Status | Decision |
|---|---|---|---|
| R1 | PostgreSQL backup on the same disk as the server and never restored | Open | — |
| R2 | Single server for the portal: a VM outage takes orders down | Open | Accepted by the IT manager until 2026-12 |
| R3 | Windows server clock without NTP configured | To be confirmed | — |

## 7. House rules

1. Production is read-only for audits; every change goes through the window and a ticket.
2. An infrastructure report is internal: it is kept on the IT SharePoint, never in a repository.
3. Commit and push only when the person responsible for the repository asks.
