# Agents guide — how to use and how to call each one

**English** · [Português](GUIA.pt-BR.md)

This guide gathers in one place what each agent in the collection does, when to use each one, how to call it and how they work together. Installation and permission details for each agent are in its own `docs/`.

## 1. The four agents

| Agent | Role | Can it change files? | Use when |
|---|---|---|---|
| `security-sentinel` | Security auditor | **No** (read-only) | You want to know whether there is a vulnerability, exposure, secret, or an AI or dependency risk |
| `infra-guardian` | SRE, infrastructure, database, costs | **No** (only reads and proposes a plan) | Server, Docker, Kubernetes, network, deploy, backup, monitoring, capacity, incident, cost |
| `infra-guardian-static` | The same, without any command | **No** | When nothing can be executed, not even a read command |
| `backend-architect` | Backend architect and engineer | **Yes**, when you ask for implementation | APIs, business rules, database, integrations, migrations, server tests |
| `frontend-architect` | Frontend architect and engineer | **Yes**, when you ask for implementation | Screens, UX, design system, accessibility, responsiveness, performance, charts |

Rule of thumb: **the two auditors say what is wrong and how to fix it; the two architects design and, if you ask, implement.** Neither deploys, pushes, migrates a real database or installs a dependency without your authorization in the conversation.

## 2. Quick install

Copy to `~/.claude/agents/` (all projects) or `<project>/.claude/agents/` (one project). Choose **one language** per agent.

```powershell
# Windows (PowerShell), from the root of this repository
foreach ($a in "security-sentinel","infra-guardian","backend-architect","frontend-architect") {
  Copy-Item ".\$a\agent\en\$a.md" "$env:USERPROFILE\.claude\agents\"
}
Copy-Item ".\infra-guardian\agent\en\infra-guardian-static.md" "$env:USERPROFILE\.claude\agents\"
```

```bash
# Linux and macOS
for a in security-sentinel infra-guardian backend-architect frontend-architect; do
  cp "$a/agent/en/$a.md" ~/.claude/agents/
done
cp infra-guardian/agent/en/infra-guardian-static.md ~/.claude/agents/
```

After installing or updating, **open a new session** of Claude Code: a session that is already open may keep the previous version.

## 3. How to call

The form that always works: **"Use the `<name>` agent to run `<mode>` on `<scope>`"**. State the scope (folder, route, screen, branch) and, for the architects, whether it may implement.

### 3.1 Security Sentinel

| Mode | What for |
|---|---|
| `triage` | Quick sweep, top risks |
| `diff-review` | Review a PR, commit or branch |
| `deep-audit` | Full audit by layers |
| `threat-model` | Threat model |
| `network-review` | Network, proxy, firewall, DNS, TLS, Docker, by files |
| `ai-agent-review` | LLM, agents, MCP, RAG, webhooks |
| `supply-chain` | Dependencies and CI |
| `incident-triage` | Support for a security incident |
| `remediation-plan` | Fix plan in waves |
| `verify-fix` | Check fixes (`SS-xxx` IDs) |
| `release-gate` | Release recommendation |

- `Use the security-sentinel agent to run deep-audit on this repository, read-only; do not run external tests.`
- `Use the security-sentinel agent for diff-review of the current branch's changes.`
- `Use the security-sentinel agent for verify-fix of findings SS-001 and SS-002.`

### 3.2 Infra Guardian

| Mode | What for |
|---|---|
| `discover` | Inventory and current architecture |
| `architecture-review` | Decisions and proposal, with an ADR |
| `vps-audit` | Server (from the collection you paste) |
| `docker-review` · `k8s-review` | Compose, Dockerfile, manifests |
| `network-diagnose` | Path DNS → TLS → CDN → proxy → app → database |
| `cloud-iac` | Terraform, OpenTofu, state, drift |
| `pipeline-review` | CI/CD and deploy |
| `observability` | SLI, SLO, alerts, runbooks |
| `database-ops` | Database operations |
| `backup-dr` | Backup, restore, proposed RPO and RTO |
| `capacity-plan` · `finops` | Capacity and costs |
| `incident-triage` | Availability incident |
| `migration-plan` · `release-readiness` · `verify-change` | Migration, readiness, verification after the change (`IG-xxx` IDs) |

- `Use the infra-guardian agent for a vps-audit; the output of df, free and systemctl is in collection/.`
- `Use the infra-guardian agent for backup-dr of the main database, proposing RPO and RTO for approval.`
- `Use the infra-guardian-static agent for an architecture-review, without running any command.`

### 3.3 Backend Architect

| Mode | What for |
|---|---|
| `discover` | Backend map with evidence |
| `design` · `api-contract` | Solution and API contract |
| `technology-assessment` | Keep, evolve or migrate, with a matrix and scores |
| `implement` · `refactor` | Implement a small slice, with tests |
| `code-review` | Review with findings by severity |
| `database-review` · `migration-plan` | Database, queries, safe migrations |
| `performance` · `integration` | Measure before and after; third-party integrations |
| `test-strategy` · `incident-analysis` | Test strategy; failure analysis |
| `release-readiness` · `verify` | Readiness and verification |

- `Use the backend-architect agent to run discover on this repository.`
- `Use the backend-architect agent for api-contract of the orders endpoint, without implementing.`
- `Use the backend-architect agent to implement cursor pagination on GET /pedidos, with tests.`

### 3.4 Frontend Architect

| Mode | What for |
|---|---|
| `discover` | Map of routes, screens, components, tokens, data |
| `ux-review` · `design-system` | Experience and components/tokens |
| `architecture-review` · `technology-assessment` | Rendering, data, state; technology decisions |
| `implement` · `refactor` | Implement a local change, with tests |
| `responsive-audit` · `accessibility-audit` | Phone/tablet/desktop; WCAG 2.2 AA |
| `performance` | Core Web Vitals with measurement |
| `api-integration` · `realtime-ui` | API consumption; real time |
| `dashboard-data-viz` · `three-d` | Charts; 3D with a fallback |
| `test-strategy` · `release-readiness` · `verify` | Tests, readiness, verification |

- `Use the frontend-architect agent for a ux-review of the orders screen.`
- `Use the frontend-architect agent for an accessibility-audit of the registration form, with a keyboard test.`
- `Use the frontend-architect agent to implement the empty, error and loading states in the orders list, with tests.`

## 4. How they work together

| Owner of | Agent |
|---|---|
| APIs, business rules, database | `backend-architect` |
| UI, UX, design system, API consumption | `frontend-architect` |
| Security audit | `security-sentinel` |
| Infrastructure, SRE, cloud, FinOps | `infra-guardian` |

Subagents **do not call one another**. Each one ends with a **handoff** (to whom, what to do, files, risks, how to verify), and you or the main Claude forward it.

**A feature that crosses layers, step by step:**

1. `backend-architect` in `design` or `api-contract`: contract (schema, errors, authentication, pagination, cache, compatibility).
2. `frontend-architect` in `ux-review` or `design-system`: journey and screen states on top of that contract.
3. You approve the contract and decide who owns each file.
4. Each architect in `implement` on its own side, with tests.
5. `security-sentinel` in `diff-review` and, if it touches a server, deploy or database, `infra-guardian` in `release-readiness`.
6. Fixes; then `verify` (architects), `verify-fix` (Sentinel), `verify-change` (Guardian).
7. You decide the release.

Example of a chained request to the main Claude: `Use the backend-architect for an api-contract of the orders export; then the frontend-architect for a ux-review of the export screen using that contract; do not implement yet.`

## 5. Done means (applies to all)

Context understood · project preserved · coherent contract · cohesive, typed code · failure states tested · tests executed or open items declared · accessibility and performance assessed in proportion to the risk · documentation aligned · handoff done · risks explicit.

## 6. Technology: keep, evolve or migrate

The architects always compare **(A) keep and improve**, **(B) incremental evolution** and **(C) migration**, with a score from 0 to 5 per criterion and evidence. Initial weights: business fit 25%, compatibility and migration 20%, security or accessibility and maturity 15%, operations or performance 15%, scalability and integration 10%, team and ecosystem 10%, total cost and vendor lock-in 5%. Without a benchmark, the score is not positive. Nobody swaps a stack because of fashion.

## 7. Your organization's context

Each agent can preload a local `<agent>-org-context` skill with what only your organization knows (systems, servers, standards, risks). For the `frontend-architect`, that is where the **mandatory visual standard** lives (colors, fonts, logos, images). The skill lives at `~/.claude/skills/<agent>-org-context/SKILL.md`, **outside this repository**, with no password or token. Fictitious templates in `<agent>/org-context-example/`.

## 8. Permissions: what is an instruction and what is a barrier

- Each agent's text forbids dangerous actions, but **an instruction is not a barrier**. The real barriers are: each agent's `tools` list, the `permissions` rules (deny, ask, allow) in `settings.json` and, for the auditors, the hooks proposed in the installation guides.
- `permissionMode` in the agent is ignored when the main session is in `auto`, `acceptEdits` or `bypassPermissions`. For the architects, who edit code, work in a session in the default mode or in a separate worktree and review the diff.
- On native Windows there is no Claude Code Bash sandbox.

## 9. Where to read more

| Agent | Installation and permissions | Playbook | Evaluations |
|---|---|---|---|
| Security Sentinel | [`SECURITY_SENTINEL_SETUP.md`](security-sentinel/docs/en/SECURITY_SENTINEL_SETUP.md) | [`SECURITY_SENTINEL_PLAYBOOK.md`](security-sentinel/docs/en/SECURITY_SENTINEL_PLAYBOOK.md) | [`SECURITY_SENTINEL_EVALS.md`](security-sentinel/docs/en/SECURITY_SENTINEL_EVALS.md) |
| Infra Guardian | [`INFRA_GUARDIAN_SETUP.md`](infra-guardian/docs/en/INFRA_GUARDIAN_SETUP.md) | [`INFRA_GUARDIAN_PLAYBOOK.md`](infra-guardian/docs/en/INFRA_GUARDIAN_PLAYBOOK.md) | [`INFRA_GUARDIAN_EVALS.md`](infra-guardian/docs/en/INFRA_GUARDIAN_EVALS.md) |
| Backend Architect | [`BACKEND_ARCHITECT_SETUP.md`](backend-architect/docs/en/BACKEND_ARCHITECT_SETUP.md) | [`BACKEND_ARCHITECT_PLAYBOOK.md`](backend-architect/docs/en/BACKEND_ARCHITECT_PLAYBOOK.md) | [`BACKEND_ARCHITECT_EVALS.md`](backend-architect/docs/en/BACKEND_ARCHITECT_EVALS.md) |
| Frontend Architect | [`FRONTEND_ARCHITECT_SETUP.md`](frontend-architect/docs/en/FRONTEND_ARCHITECT_SETUP.md) | [`FRONTEND_ARCHITECT_PLAYBOOK.md`](frontend-architect/docs/en/FRONTEND_ARCHITECT_PLAYBOOK.md) | [`FRONTEND_ARCHITECT_EVALS.md`](frontend-architect/docs/en/FRONTEND_ARCHITECT_EVALS.md) |

A language model can make mistakes: check the evidence and review every diff before acting.
