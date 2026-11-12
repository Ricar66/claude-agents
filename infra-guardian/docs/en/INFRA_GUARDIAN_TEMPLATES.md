# Infra Guardian — document templates

Five short templates that the agent uses when delivering decisions and plans. Copy the block, fill it in and keep it where your organization keeps operational documentation (not necessarily in the repository).

Rules for all of them: no secrets; numbers only measured or with assumptions; every action with an owner and a verifiable criterion.

## 1. ADR — architecture decision record

```markdown
# ADR-<nnn>: <decision in one sentence>

- Date: <YYYY-MM-DD> · Status: proposed | accepted | superseded by ADR-<nnn>
- Decision makers: <names or roles>

## Context
<The problem, with evidence: incident, metric, requirement, cost. What happens if nothing changes.>

## Options considered
| Option | Cost | Complexity | Operation | Reliability | Migration risk |
|---|---|---|---|---|---|
| Keep the current one | | | | | |
| A | | | | | |
| B | | | | | |

## Decision
<The chosen option and why, tied to the real requirements (load, SLO, team, budget).>

## Consequences
<What improves, what gets worse, what now needs to be operated, when to revisit this decision.>
```

## 2. Change plan

```markdown
# Change: <title> — <environment> — <date and window, with time zone>

| Field | Value |
|---|---|
| Asset and environment | |
| Owner / who executes / who approves | |
| Objective and success criterion | |
| Risk and blast radius | |
| Dependencies | |
| Baseline (metrics before) | |
| Backup and restore available (tested on) | |
| Out-of-band access (if the change can cut off access) | |

## Pre-checks
- [ ] <e.g.: today's backup is restorable; disk space; replicas healthy>

## Steps
1. <step — command or action — expected time — how to know it worked>

## Abort criteria
- <e.g.: error rate above X% for Y min; P95 latency above Z; loss of access>

## Rollback or forward-fix
1. <step — expected time — how to confirm the way back>

## Post-checks and evidence to keep
- [ ] <e.g.: health OK, metrics equal to or better than the baseline for N min>

## Approval
<name — date and time — what was approved (this change, in this window)>
```

## 3. SLO

```markdown
# SLO: <user journey>

- Service and owner: <...>
- Journey: <e.g.: customer sends a message and receives a reply>
- SLI (specification): <what the user feels, e.g.: proportion of messages answered within N s>
- SLI (implementation): <where the measurement comes from and what it cannot see>
- Target and window: <e.g.: X% over a 4-week rolling window> — basis: <measured history or "to be defined after N weeks of measurement">
- Error budget: <how much failure the target allows in the window>
- Error budget policy: <what changes when the budget runs out; who approved; when to review>
- Alerts: <fast and slow budget burn; owner; runbook>
- Not an SLA: <contractual commitments, if any, live in their own document>
```

## 4. Incident and postmortem

```markdown
# Incident <YYYY-MM-DD> — <short title>

- Severity: <P0 | P1 | ...> · Duration: <start — end, with time zone>
- Coordination: <who> · Communication: <who> · Operation: <who>
- Impact: <who was affected, what stopped working, data lost or not>

## Timeline (time zone: <...>)
| Time | Fact | Source |
|---|---|---|

## Hypotheses
| Hypothesis | For | Against | Result |
|---|---|---|---|

## Stabilization
<What was done, by whom, with what risk; what was preserved beforehand.>

## Postmortem (blameless)
- Cause: <...>
- Contributing factors: <...>
- What worked: <...>
- What got in the way: <...>

## Actions
| Action | Owner | Deadline | Verifiable criterion |
|---|---|---|---|
```

## 5. Disaster recovery (DR) plan

```markdown
# DR: <system>

- Owner: <...> · Last review: <date> · Last test: <date and result>
- Accepted RPO: <value> — approved by <who>, on <date>
- Accepted RTO: <value> — approved by <who>, on <date>

## What must come back
| Item | Where it lives | How it is copied | Copy destination | Frequency | Retention | Last restore tested |
|---|---|---|---|---|---|---|
| Database | | | | | | |
| Files and uploads | | | | | | |
| Configuration (no secrets) | | | | | | |
| Secrets and certificates (custody) | | | | | | |
| DNS and IaC | | | | | | |

## Scenarios
| Scenario | Strategy | Summary of steps | Time measured in the last test |
|---|---|---|---|
| Loss of the server | | | |
| Data deletion or corruption | | | |
| Loss of the provider or region | | | |
| Ransomware | | | |

## Restore test
<Isolated environment, steps, integrity and application checks, measured RTO and RPO, problems, next date.>
```
