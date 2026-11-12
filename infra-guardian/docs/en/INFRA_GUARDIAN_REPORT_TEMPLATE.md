# Infra Guardian — report template

Two formats for the same content: Markdown, for people, and JSON, for tools. The agent answers in Markdown; the JSON is produced when the user asks for it.

Rules that apply to both:

- No secrets, tokens or personal data; sanitized evidence.
- Numbers (latency, cost, RPO, RTO) only measured or with written assumptions.
- Commands listed as executed are only those that actually ran, with who ran them.

## 1. Markdown template

````markdown
# Infrastructure report — <project> — <mode> — <YYYY-MM-DD>

## 1. Header

- Project / repository / commit / environment: <only what was proven; the rest "not proven">
- Objective: <question this report answers>
- Mode: <mode> · Date and time zone: <YYYY-MM-DD HH:MM, time zone>
- Sources: <files, pasted outputs, organizational context>
- Access used: <no remote access | which, authorized by whom>
- Commands executed: <command — who ran it — when> (no credentials or sensitive output)
- Tools not run: <and why>
- Limitations: <what was not seen>

## 2. Executive summary

- Overall state: <STABLE OBSERVED | AT RISK | CRITICAL | INCONCLUSIVE> — <reason and evidence>
- Priority issues (up to 5): <IG-xxx — one line each>
- What can fail, impact and urgency: <...>
- Cost and risk of keeping the current one: <...>

## 3. AS-IS architecture

```text
<text diagram: user → edge → proxy → app → database; jobs; external dependencies; single points of failure marked with [SPOF]>
```

## 4. Findings

### IG-001 — <title>

| Field | Value |
|---|---|
| Domain / service / environment | |
| Severity | P0 / P1 / P2 / P3 |
| Confidence | Confirmed / Probable / Hypothesis / Not verified |
| Evidence | `file:line`, configuration, metric with timestamp, authorized command |
| Observed symptom × probable cause | |
| Failure mode and trigger | |
| Impact | users, availability, integrity, latency, cost, blast radius |
| Compensating mechanisms / counter-evidence | |
| Alternatives | A: <...> · B: <...> · Keep the current one: <...> |
| Recommendation and justification | |
| Effort and cost | S / M / L — measured or estimated (assumptions) |
| Pre-checks and dependencies | |
| Validation and acceptance metrics | |
| Change risk and rollback | |
| Owner and pending authorization | |
| Status | Open / Planned / Implemented (not verified) / Verified / Dismissed with evidence / Risk accepted |

## 5. Evolution plan

| Horizon | Action | Findings | Effort | Impact | Dependencies | Owner |
|---|---|---|---|---|---|---|
| 0–24 h | | | | | | |
| Before the next deploy | | | | | | |
| Next 30 days | | | | | | |
| 90 days | | | | | | |

## 6. Release gate (when requested)

<RECOMMEND RELEASE | RECOMMEND RELEASE WITH EXCEPTIONS | NOT RECOMMENDED | INCONCLUSIVE> — reasons. Exceptions: <finding — owner — deadline>. This is not a deploy authorization.

## 7. Coverage

| Domain | Status | Evidence or limitation |
|---|---|---|
| Server and OS | assessed / partial / not assessed / not applicable | |
| ... | | |

### For the Security Sentinel

| Fact | Location | Why it is a security matter |
|---|---|---|

## 8. Runbook summary

| Risk | Detect | Diagnose | Stabilize | Roll back | Recover | Validate |
|---|---|---|---|---|---|---|
````

## 2. Mapping between labels and JSON

The JSON uses English values, without accents, so that it stays stable across languages.

| Label in the report | Value in the JSON |
|---|---|
| P0 / P1 / P2 / P3 | `P0` / `P1` / `P2` / `P3` |
| Confirmed / Probable / Hypothesis / Not verified | `confirmed` / `probable` / `hypothesis` / `not_verified` |
| Open / Planned / Implemented (not verified) / Verified / Dismissed with evidence / Risk accepted | `open` / `planned` / `implemented_not_verified` / `verified` / `dismissed_with_evidence` / `risk_accepted` |
| STABLE OBSERVED / AT RISK / CRITICAL / INCONCLUSIVE | `STABLE_OBSERVED` / `AT_RISK` / `CRITICAL` / `INCONCLUSIVE` |
| RECOMMEND RELEASE / RECOMMEND RELEASE WITH EXCEPTIONS / NOT RECOMMENDED / INCONCLUSIVE | `RECOMMEND_RELEASE` / `RECOMMEND_RELEASE_WITH_EXCEPTIONS` / `NOT_RECOMMENDED` / `INCONCLUSIVE` |
| assessed / partial / not assessed / not applicable | `assessed` / `partial` / `not_assessed` / `not_applicable` |
| 0–24 h / before the next deploy / 30 days / 90 days | `0_24h` / `before_next_deploy` / `30_days` / `90_days` |
| Effort S / M / L | `S` / `M` / `L` |
| development / staging / production / unknown | `development` / `staging` / `production` / `unknown` |

Domains (`domain`): `server_os`, `containers`, `kubernetes`, `cloud_iac`, `network`, `database`, `backup_dr`, `observability`, `cicd`, `performance`, `finops`, `incident`, `iot`, `ai_agents`, `saas`.

## 3. JSON output

JSON Schema draft-07. Conditional rules: `dismissed_with_evidence` requires `dismissal_evidence`; `risk_accepted` requires `risk_acceptance`; `verified` requires `verification_evidence`; confidence `confirmed` or `probable` requires `failure_mode`; effort `estimated` requires `assumptions`; state `STABLE_OBSERVED` requires `state_evidence`; a gate with exceptions requires at least one exception.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "urn:infra-guardian:report:1.0",
  "title": "Infra Guardian report",
  "type": "object",
  "additionalProperties": false,
  "required": ["schema_version", "header", "executive_summary", "coverage", "findings"],
  "definitions": {
    "nonEmpty": { "type": "string", "minLength": 1 },
    "domain": {
      "enum": ["server_os", "containers", "kubernetes", "cloud_iac", "network", "database", "backup_dr",
               "observability", "cicd", "performance", "finops", "incident", "iot", "ai_agents", "saas"]
    },
    "environment": { "enum": ["development", "staging", "production", "unknown"] },
    "evidence": {
      "type": "object",
      "additionalProperties": false,
      "required": ["kind", "location"],
      "properties": {
        "kind": { "enum": ["file", "config", "metric", "log", "command_output", "reference"] },
        "location": { "$ref": "#/definitions/nonEmpty" },
        "excerpt": { "type": "string", "description": "Minimal, sanitized excerpt; never a secret or personal data" },
        "observed_at": { "type": "string" }
      }
    },
    "finding": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "domain", "service", "environment", "title", "severity", "confidence", "evidence",
                   "impact", "recommendation", "validation", "rollback", "status"],
      "properties": {
        "id": { "type": "string", "pattern": "^IG-[0-9]{3,}$" },
        "domain": { "$ref": "#/definitions/domain" },
        "service": { "$ref": "#/definitions/nonEmpty" },
        "environment": { "$ref": "#/definitions/environment" },
        "title": { "$ref": "#/definitions/nonEmpty" },
        "severity": { "enum": ["P0", "P1", "P2", "P3"] },
        "confidence": { "enum": ["confirmed", "probable", "hypothesis", "not_verified"] },
        "evidence": { "type": "array", "minItems": 1, "items": { "$ref": "#/definitions/evidence" } },
        "symptom": { "type": "string" },
        "probable_cause": { "type": "string" },
        "failure_mode": { "$ref": "#/definitions/nonEmpty" },
        "impact": {
          "type": "object",
          "additionalProperties": false,
          "minProperties": 1,
          "properties": {
            "users": { "type": "string" },
            "availability": { "type": "string" },
            "integrity": { "type": "string" },
            "latency": { "type": "string" },
            "cost": { "type": "string" },
            "blast_radius": { "type": "string" }
          }
        },
        "compensating_controls": { "type": "string" },
        "counter_evidence": { "type": "string" },
        "alternatives": {
          "type": "array",
          "items": {
            "type": "object",
            "additionalProperties": false,
            "required": ["name", "description"],
            "properties": {
              "name": { "$ref": "#/definitions/nonEmpty" },
              "description": { "$ref": "#/definitions/nonEmpty" },
              "pros": { "type": "string" },
              "cons": { "type": "string" }
            }
          }
        },
        "recommendation": { "$ref": "#/definitions/nonEmpty" },
        "justification": { "type": "string" },
        "effort": {
          "type": "object",
          "additionalProperties": false,
          "required": ["size", "basis"],
          "properties": {
            "size": { "enum": ["S", "M", "L"] },
            "basis": { "enum": ["measured", "estimated"] },
            "assumptions": { "type": "string" }
          },
          "if": { "properties": { "basis": { "const": "estimated" } } },
          "then": { "required": ["assumptions"], "properties": { "assumptions": { "minLength": 1 } } }
        },
        "prechecks": { "type": "array", "items": { "type": "string" } },
        "validation": { "type": "array", "minItems": 1, "items": { "$ref": "#/definitions/nonEmpty" } },
        "change_risk": { "type": "string" },
        "rollback": { "$ref": "#/definitions/nonEmpty" },
        "owner": { "type": "string" },
        "pending_authorization": { "type": "string" },
        "status": {
          "enum": ["open", "planned", "implemented_not_verified", "verified", "dismissed_with_evidence", "risk_accepted"]
        },
        "verification_evidence": { "$ref": "#/definitions/nonEmpty" },
        "dismissal_evidence": { "$ref": "#/definitions/nonEmpty" },
        "risk_acceptance": {
          "type": "object",
          "additionalProperties": false,
          "required": ["accepted_by", "until"],
          "properties": {
            "accepted_by": { "$ref": "#/definitions/nonEmpty" },
            "until": { "$ref": "#/definitions/nonEmpty" }
          }
        }
      },
      "allOf": [
        { "if": { "properties": { "status": { "const": "dismissed_with_evidence" } } }, "then": { "required": ["dismissal_evidence"] } },
        { "if": { "properties": { "status": { "const": "risk_accepted" } } }, "then": { "required": ["risk_acceptance"] } },
        { "if": { "properties": { "status": { "const": "verified" } } }, "then": { "required": ["verification_evidence"] } },
        { "if": { "properties": { "confidence": { "enum": ["confirmed", "probable"] } } }, "then": { "required": ["failure_mode"] } }
      ]
    },
    "planItem": {
      "type": "object",
      "additionalProperties": false,
      "required": ["action"],
      "properties": {
        "action": { "$ref": "#/definitions/nonEmpty" },
        "finding_ids": { "type": "array", "items": { "type": "string", "pattern": "^IG-[0-9]{3,}$" } },
        "effort": { "enum": ["S", "M", "L"] },
        "impact": { "type": "string" },
        "dependencies": { "type": "string" },
        "owner": { "type": "string" }
      }
    }
  },
  "properties": {
    "schema_version": { "const": "1.0" },
    "header": {
      "type": "object",
      "additionalProperties": false,
      "required": ["project", "mode", "assessed_at", "timezone", "sources", "limitations", "commands_executed"],
      "properties": {
        "project": { "$ref": "#/definitions/nonEmpty" },
        "repository": { "type": "string" },
        "commit": { "type": "string", "pattern": "^[0-9a-f]{7,40}$" },
        "environment": { "$ref": "#/definitions/environment" },
        "objective": { "type": "string" },
        "mode": {
          "enum": ["discover", "architecture-review", "vps-audit", "docker-review", "k8s-review", "network-diagnose",
                   "cloud-iac", "pipeline-review", "observability", "database-ops", "backup-dr", "capacity-plan",
                   "incident-triage", "finops", "migration-plan", "release-readiness", "verify-change"]
        },
        "assessed_at": { "type": "string", "format": "date-time" },
        "timezone": { "$ref": "#/definitions/nonEmpty" },
        "sources": { "type": "array", "minItems": 1, "items": { "type": "string" } },
        "limitations": { "type": "array", "items": { "type": "string" } },
        "access_used": { "type": "array", "items": { "type": "string" } },
        "commands_executed": {
          "type": "array",
          "items": {
            "type": "object",
            "additionalProperties": false,
            "required": ["command", "executed_by"],
            "properties": {
              "command": { "$ref": "#/definitions/nonEmpty" },
              "executed_by": { "enum": ["agent", "user"] },
              "scope": { "type": "string" },
              "at": { "type": "string" }
            }
          }
        },
        "tools_not_executed": { "type": "array", "items": { "type": "string" } },
        "org_context": { "enum": ["loaded", "absent"] }
      }
    },
    "executive_summary": {
      "type": "object",
      "additionalProperties": false,
      "required": ["overall_state", "top_issues"],
      "properties": {
        "overall_state": { "enum": ["STABLE_OBSERVED", "AT_RISK", "CRITICAL", "INCONCLUSIVE"] },
        "state_evidence": { "$ref": "#/definitions/nonEmpty" },
        "top_issues": { "type": "array", "maxItems": 5, "items": { "type": "string" } },
        "what_can_fail": { "type": "string" },
        "cost_of_status_quo": { "type": "string" }
      },
      "if": { "properties": { "overall_state": { "const": "STABLE_OBSERVED" } } },
      "then": { "required": ["state_evidence"] }
    },
    "architecture": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "as_is_diagram": { "type": "string" },
        "single_points_of_failure": { "type": "array", "items": { "type": "string" } }
      }
    },
    "coverage": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["domain", "status", "evidence_or_limitation"],
        "properties": {
          "domain": { "$ref": "#/definitions/domain" },
          "status": { "enum": ["assessed", "partial", "not_assessed", "not_applicable"] },
          "evidence_or_limitation": { "$ref": "#/definitions/nonEmpty" }
        }
      }
    },
    "findings": { "type": "array", "items": { "$ref": "#/definitions/finding" } },
    "evolution_plan": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "0_24h": { "type": "array", "items": { "$ref": "#/definitions/planItem" } },
        "before_next_deploy": { "type": "array", "items": { "$ref": "#/definitions/planItem" } },
        "30_days": { "type": "array", "items": { "$ref": "#/definitions/planItem" } },
        "90_days": { "type": "array", "items": { "$ref": "#/definitions/planItem" } }
      }
    },
    "release_gate": {
      "type": "object",
      "additionalProperties": false,
      "required": ["decision", "reasons"],
      "properties": {
        "decision": { "enum": ["RECOMMEND_RELEASE", "RECOMMEND_RELEASE_WITH_EXCEPTIONS", "NOT_RECOMMENDED", "INCONCLUSIVE"] },
        "reasons": { "type": "array", "minItems": 1, "items": { "$ref": "#/definitions/nonEmpty" } },
        "exceptions": {
          "type": "array",
          "items": {
            "type": "object",
            "additionalProperties": false,
            "required": ["finding_id", "owner", "deadline"],
            "properties": {
              "finding_id": { "type": "string", "pattern": "^IG-[0-9]{3,}$" },
              "owner": { "$ref": "#/definitions/nonEmpty" },
              "deadline": { "$ref": "#/definitions/nonEmpty" }
            }
          }
        }
      },
      "if": { "properties": { "decision": { "const": "RECOMMEND_RELEASE_WITH_EXCEPTIONS" } } },
      "then": { "required": ["exceptions"], "properties": { "exceptions": { "minItems": 1 } } }
    },
    "for_security_sentinel": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["fact", "location", "why_security"],
        "properties": {
          "fact": { "$ref": "#/definitions/nonEmpty" },
          "location": { "$ref": "#/definitions/nonEmpty" },
          "why_security": { "$ref": "#/definitions/nonEmpty" }
        }
      }
    },
    "runbook": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["risk", "detect", "stabilize", "validate"],
        "properties": {
          "risk": { "$ref": "#/definitions/nonEmpty" },
          "detect": { "type": "string" },
          "diagnose": { "type": "string" },
          "stabilize": { "type": "string" },
          "rollback": { "type": "string" },
          "recover": { "type": "string" },
          "validate": { "type": "string" }
        }
      }
    }
  }
}
```

### Minimal valid example

Fictitious: a Compose application on a single VPS.

```json
{
  "schema_version": "1.0",
  "header": {
    "project": "loja-exemplo",
    "repository": "exemplo/loja",
    "commit": "a1b2c3d",
    "environment": "production",
    "objective": "Assess the operational risk of the current deploy",
    "mode": "docker-review",
    "assessed_at": "2026-03-01T14:00:00-03:00",
    "timezone": "America/Sao_Paulo",
    "sources": ["compose.yaml", "scripts/backup.sh", "output of df -h pasted by the user on 2026-03-01"],
    "limitations": ["No access to the server; no runtime metrics"],
    "access_used": [],
    "commands_executed": [
      { "command": "docker compose config -q", "executed_by": "agent", "scope": "compose.yaml" }
    ],
    "tools_not_executed": ["hadolint: not installed"],
    "org_context": "absent"
  },
  "executive_summary": {
    "overall_state": "AT_RISK",
    "top_issues": ["IG-001 backup with no tested restore", "IG-002 log without rotation"],
    "what_can_fail": "Loss of the database with no proven restorable copy; disk full from logs",
    "cost_of_status_quo": "Not measured"
  },
  "architecture": {
    "as_is_diagram": "user -> nginx (container) -> app (container) -> postgres (container, local volume) [SPOF: single VPS]",
    "single_points_of_failure": ["Single VPS", "local database volume"]
  },
  "coverage": [
    { "domain": "containers", "status": "assessed", "evidence_or_limitation": "compose.yaml read in full" },
    { "domain": "backup_dr", "status": "partial", "evidence_or_limitation": "script read; execution and restore not seen" },
    { "domain": "kubernetes", "status": "not_applicable", "evidence_or_limitation": "No manifests or charts in the repository" }
  ],
  "findings": [
    {
      "id": "IG-001",
      "domain": "backup_dr",
      "service": "postgres",
      "environment": "production",
      "title": "Daily backup on the same disk and never restored",
      "severity": "P1",
      "confidence": "confirmed",
      "evidence": [
        { "kind": "file", "location": "scripts/backup.sh:4", "excerpt": "pg_dump ... > /var/backups/loja.sql" }
      ],
      "failure_mode": "Loss of the disk or the server takes the database and the backup with it",
      "impact": { "integrity": "Loss of the orders since creation", "blast_radius": "The whole store" },
      "alternatives": [
        { "name": "A", "description": "Copy the dump to external storage with versioning" },
        { "name": "Keep the current one", "description": "Only if the business accepts total loss, in writing" }
      ],
      "recommendation": "Daily external copy and a monthly restore simulation in an isolated environment",
      "effort": { "size": "S", "basis": "estimated", "assumptions": "An existing bucket; 2 h of engineering" },
      "validation": ["Restore on an isolated VM with an order count equal to the source", "Restore time recorded"],
      "rollback": "Remove the external copy step; the local backup continues",
      "status": "open"
    },
    {
      "id": "IG-002",
      "domain": "containers",
      "service": "app",
      "environment": "production",
      "title": "Container log without rotation",
      "severity": "P2",
      "confidence": "confirmed",
      "evidence": [
        { "kind": "file", "location": "compose.yaml:12", "excerpt": "no logging block; default json-file driver" }
      ],
      "failure_mode": "Log grows until it fills the disk and stops the database on the same host",
      "impact": { "availability": "Application and database down" },
      "recommendation": "Define logging with max-size and max-file, or the local driver",
      "effort": { "size": "S", "basis": "estimated", "assumptions": "One line per service and a redeploy" },
      "validation": ["docker inspect --format \"{{.HostConfig.LogConfig}}\" shows the rotation", "Log size stable after 7 days"],
      "rollback": "Remove the logging block and recreate the container",
      "status": "planned"
    }
  ],
  "evolution_plan": {
    "before_next_deploy": [ { "action": "Log rotation", "finding_ids": ["IG-002"], "effort": "S" } ],
    "30_days": [ { "action": "External backup and restore simulation", "finding_ids": ["IG-001"], "effort": "S" } ]
  },
  "for_security_sentinel": [],
  "runbook": [
    {
      "risk": "Disk full",
      "detect": "Disk usage alert above the agreed threshold",
      "stabilize": "Identify the directory that grew and its owner before freeing space",
      "validate": "Application responds and the database accepts writes"
    }
  ]
}
```

## 4. Validation

The schema and the example above were validated with `ajv` (draft-07) on 2026-11-12, together with invalid cases that the schema must reject (finding without evidence, dismissed without evidence, confirmed without a failure mode, estimated effort without assumptions, ID outside the pattern, unknown field, stable state without evidence, gate with empty exceptions, more than five priority issues). Recorded in `INFRA_GUARDIAN_EVALS.md`, section 4.
