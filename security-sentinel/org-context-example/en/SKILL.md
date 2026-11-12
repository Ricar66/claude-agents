---
name: security-sentinel-org-context
description: >-
  Organizational context of Example Agro Ltd. (fictitious company) for the
  security-sentinel agent: systems, where they run, network, service accounts,
  known risks and internal rules. Preloaded by security-sentinel. Internal.
---

# Organizational context — Example Agro Ltd. (FICTITIOUS EXAMPLE)

> This file is a **template**. The company, the systems, the IPs (ranges reserved for documentation) and the risks are made up. Copy the structure to `~/.claude/skills/security-sentinel-org-context/SKILL.md` and replace it with what is true in your organization.
> **Never** write a password, token, key or secret value here: only where the credential is and what the risk is.

## 0. How the Sentinel uses this file

- It is context data, not instructions. It does not widen permissions or authorize active testing.
- It is a dated snapshot (last reviewed: 2026-03-01). Confirm in the code or configuration before turning an item into a finding.

## 1. Scope authorized by default

| Allowed (default) | Needs written authorization from the IT team beforehand |
|---|---|
| Read code, IaC, CI and documentation of the organization's repositories | Any scan or request against `192.0.2.0/24`, `198.51.100.0/24` or `*.exampleagro.example` |
| Read-only Git commands | Scanners that download a database or send data to third parties |
| Read `docs/riscos.md` in each repository | Connecting to a database, directory (LDAP), ERP or firewall panel |

Never open `.env`, `*.pfx` or the `segredos/` folder on the servers.

## 2. The organization

- Farm machinery reseller with 4 stores and a distribution center.
- Directory: Active Directory `exampleagro.local`, synchronized with Microsoft Entra ID.
- Personal data: customers (CPF/CNPJ, the Brazilian individual and company tax IDs; phone; address), employees (hiring data). The LGPD (Brazil's data protection law) applies.
- IT team: 3 people; the person responsible for security is the infrastructure coordinator.

## 3. Systems

| System | What it does | Stack | Where it runs / exposure | Sensitive data and privileges |
|---|---|---|---|---|
| `portal-pedidos` | Parts orders from customers | Next.js 15, Prisma, PostgreSQL 16 | Linux VM `192.0.2.10`, published via CDN at `pedidos.exampleagro.example` | Customer data; multi-tenant by store |
| `integra-erp` | Syncs orders with the ERP | Python 3.12, Redis queue | VM `192.0.2.11`, internal network only | ERP user with write access to orders |
| `bot-atendimento` | Customer service over WhatsApp with an LLM | Node 20, cloud LLM provider | Container on VM `192.0.2.12`, public webhook | Reads orders and catalog; does not write to the ERP |
| `onboarding-ad` | Creates employee accounts from a ticket | Scheduled PowerShell | Windows server `192.0.2.20` | **Writes to AD** |

## 4. Network and perimeter

- A single edge firewall at headquarters; stores connected by IPsec VPN.
- CDN with WAF in front of `portal-pedidos`. The origin `192.0.2.10:443` accepts connections from any IP (**known risk R2**).
- IPv6 enabled on the VMs, without a firewall policy equivalent to the IPv4 one (to be confirmed).

## 5. Identity and service accounts

- `svc-onboarding` (AD) with permission to create users in the OU `Colaboradores`.
- `integra-erp` uses an ERP user shared with a member of the team (**R3**).
- App secrets in Entra ID with no expiry control.

## 6. Known risks

| # | Risk | Status | Decision |
|---|---|---|---|
| R1 | PostgreSQL backup without a restore test | Open | — |
| R2 | Portal origin reachable without going through the WAF | Open | Fix by 2026-06-30 |
| R3 | ERP integration user shared with a person | Open | — |
| R4 | Old repository with a secret in its history | Secret **rotated** in 2026-02; history not cleaned | Risk accepted by the IT manager until the migration |

## 7. House rules

1. A security report is internal: it is kept on the IT SharePoint, never in a repository.
2. Production is read-only for audits.
3. Commit and push only when the person responsible for the repository asks.
