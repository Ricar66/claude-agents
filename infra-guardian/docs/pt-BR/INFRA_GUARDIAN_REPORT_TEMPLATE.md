# Infra Guardian — modelo de relatório

Dois formatos para o mesmo conteúdo: Markdown, para pessoas, e JSON, para ferramentas. O agente responde em Markdown; o JSON sai quando o usuário pedir.

Regras que valem nos dois:

- Nenhum segredo, token ou dado pessoal; evidência sanitizada.
- Número (latência, custo, RPO, RTO) só medido ou com premissas escritas.
- Comandos listados como executados são só os que de fato rodaram, com quem rodou.

## 1. Modelo em Markdown

````markdown
# Relatório de infraestrutura — <projeto> — <modo> — <AAAA-MM-DD>

## 1. Cabeçalho

- Projeto / repositório / commit / ambiente: <só o que foi comprovado; o resto "não comprovado">
- Objetivo: <pergunta que este relatório responde>
- Modo: <modo> · Data e fuso: <AAAA-MM-DD HH:MM, fuso>
- Fontes: <arquivos, saídas coladas, contexto organizacional>
- Acessos usados: <nenhum acesso remoto | quais, autorizados por quem>
- Comandos executados: <comando — quem rodou — quando> (sem credencial nem saída sensível)
- Ferramentas não executadas: <e por quê>
- Limitações: <o que não foi visto>

## 2. Visão executiva

- Estado geral: <ESTÁVEL OBSERVADO | EM RISCO | CRÍTICO | INCONCLUSIVO> — <motivo e evidência>
- Problemas prioritários (até 5): <IG-xxx — uma linha cada>
- O que pode falhar, impacto e urgência: <...>
- Custo e risco de manter o atual: <...>

## 3. Arquitetura AS-IS

```text
<diagrama em texto: usuário → borda → proxy → app → banco; jobs; dependências externas; pontos únicos de falha marcados com [SPOF]>
```

## 4. Achados

### IG-001 — <título>

| Campo | Valor |
|---|---|
| Domínio / serviço / ambiente | |
| Severidade | P0 / P1 / P2 / P3 |
| Confiança | Confirmado / Provável / Hipótese / Não verificado |
| Evidência | `arquivo:linha`, configuração, métrica com horário, comando autorizado |
| Sintoma observado × causa provável | |
| Modo de falha e gatilho | |
| Impacto | usuários, disponibilidade, integridade, latência, custo, raio de impacto |
| Mecanismos compensatórios / evidência contrária | |
| Alternativas | A: <...> · B: <...> · Manter o atual: <...> |
| Recomendação e justificativa | |
| Esforço e custo | P / M / G — medido ou estimado (premissas) |
| Pré-checks e dependências | |
| Validação e métricas de aceite | |
| Risco da mudança e rollback | |
| Dono e autorização pendente | |
| Status | Aberto / Planejado / Implementado (não verificado) / Verificado / Descartado com evidência / Risco aceito |

## 5. Plano de evolução

| Horizonte | Ação | Achados | Esforço | Impacto | Dependências | Dono |
|---|---|---|---|---|---|---|
| 0–24 h | | | | | | |
| Antes do próximo deploy | | | | | | |
| Próximos 30 dias | | | | | | |
| 90 dias | | | | | | |

## 6. Gate de release (quando pedido)

<RECOMENDA LIBERAR | RECOMENDA LIBERAR COM EXCEÇÕES | NÃO RECOMENDA | INCONCLUSIVO> — motivos. Exceções: <achado — dono — prazo>. Não é autorização de deploy.

## 7. Cobertura

| Domínio | Situação | Evidência ou limitação |
|---|---|---|
| Servidor e SO | avaliado / parcial / não avaliado / não aplicável | |
| ... | | |

### Para o Security Sentinel

| Fato | Local | Por que é de segurança |
|---|---|---|

## 8. Runbook resumido

| Risco | Detectar | Diagnosticar | Estabilizar | Reverter | Recuperar | Validar |
|---|---|---|---|---|---|---|
````

## 2. Correspondência entre rótulos e JSON

O JSON usa valores em inglês, sem acento, para ser estável entre idiomas.

| Rótulo no relatório | Valor no JSON |
|---|---|
| P0 / P1 / P2 / P3 | `P0` / `P1` / `P2` / `P3` |
| Confirmado / Provável / Hipótese / Não verificado | `confirmed` / `probable` / `hypothesis` / `not_verified` |
| Aberto / Planejado / Implementado (não verificado) / Verificado / Descartado com evidência / Risco aceito | `open` / `planned` / `implemented_not_verified` / `verified` / `dismissed_with_evidence` / `risk_accepted` |
| ESTÁVEL OBSERVADO / EM RISCO / CRÍTICO / INCONCLUSIVO | `STABLE_OBSERVED` / `AT_RISK` / `CRITICAL` / `INCONCLUSIVE` |
| RECOMENDA LIBERAR / COM EXCEÇÕES / NÃO RECOMENDA / INCONCLUSIVO | `RECOMMEND_RELEASE` / `RECOMMEND_RELEASE_WITH_EXCEPTIONS` / `NOT_RECOMMENDED` / `INCONCLUSIVE` |
| avaliado / parcial / não avaliado / não aplicável | `assessed` / `partial` / `not_assessed` / `not_applicable` |
| 0–24 h / antes do próximo deploy / 30 dias / 90 dias | `0_24h` / `before_next_deploy` / `30_days` / `90_days` |
| Esforço P / M / G | `S` / `M` / `L` |
| desenvolvimento / homologação / produção / desconhecido | `development` / `staging` / `production` / `unknown` |

Domínios (`domain`): `server_os`, `containers`, `kubernetes`, `cloud_iac`, `network`, `database`, `backup_dr`, `observability`, `cicd`, `performance`, `finops`, `incident`, `iot`, `ai_agents`, `saas`.

## 3. Saída em JSON

Schema JSON draft-07. Regras condicionais: `dismissed_with_evidence` exige `dismissal_evidence`; `risk_accepted` exige `risk_acceptance`; `verified` exige `verification_evidence`; confiança `confirmed` ou `probable` exige `failure_mode`; esforço `estimated` exige `assumptions`; estado `STABLE_OBSERVED` exige `state_evidence`; gate com exceções exige pelo menos uma exceção.

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
        "excerpt": { "type": "string", "description": "Trecho mínimo e sanitizado; nunca segredo ou dado pessoal" },
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

### Exemplo mínimo válido

Fictício: uma aplicação em Compose numa VPS única.

```json
{
  "schema_version": "1.0",
  "header": {
    "project": "loja-exemplo",
    "repository": "exemplo/loja",
    "commit": "a1b2c3d",
    "environment": "production",
    "objective": "Avaliar o risco operacional do deploy atual",
    "mode": "docker-review",
    "assessed_at": "2026-03-01T14:00:00-03:00",
    "timezone": "America/Sao_Paulo",
    "sources": ["compose.yaml", "scripts/backup.sh", "saída de df -h colada pelo usuário em 2026-03-01"],
    "limitations": ["Sem acesso ao servidor; sem métricas de execução"],
    "access_used": [],
    "commands_executed": [
      { "command": "docker compose config -q", "executed_by": "agent", "scope": "compose.yaml" }
    ],
    "tools_not_executed": ["hadolint: não instalado"],
    "org_context": "absent"
  },
  "executive_summary": {
    "overall_state": "AT_RISK",
    "top_issues": ["IG-001 backup sem restore testado", "IG-002 log sem rotação"],
    "what_can_fail": "Perda do banco sem cópia restaurável comprovada; disco cheio por log",
    "cost_of_status_quo": "Não medido"
  },
  "architecture": {
    "as_is_diagram": "usuário -> nginx (container) -> app (container) -> postgres (container, volume local) [SPOF: VPS única]",
    "single_points_of_failure": ["VPS única", "volume local do banco"]
  },
  "coverage": [
    { "domain": "containers", "status": "assessed", "evidence_or_limitation": "compose.yaml lido inteiro" },
    { "domain": "backup_dr", "status": "partial", "evidence_or_limitation": "script lido; execução e restore não vistos" },
    { "domain": "kubernetes", "status": "not_applicable", "evidence_or_limitation": "Sem manifests nem charts no repositório" }
  ],
  "findings": [
    {
      "id": "IG-001",
      "domain": "backup_dr",
      "service": "postgres",
      "environment": "production",
      "title": "Backup diário no mesmo disco e nunca restaurado",
      "severity": "P1",
      "confidence": "confirmed",
      "evidence": [
        { "kind": "file", "location": "scripts/backup.sh:4", "excerpt": "pg_dump ... > /var/backups/loja.sql" }
      ],
      "failure_mode": "Perda do disco ou do servidor leva junto o banco e o backup",
      "impact": { "integrity": "Perda dos pedidos desde a criação", "blast_radius": "Toda a loja" },
      "alternatives": [
        { "name": "A", "description": "Copiar o dump para armazenamento externo com versionamento" },
        { "name": "Manter o atual", "description": "Só se o negócio aceitar a perda total, por escrito" }
      ],
      "recommendation": "Cópia externa diária e simulação de restore mensal em ambiente isolado",
      "effort": { "size": "S", "basis": "estimated", "assumptions": "Um bucket existente; 2 h de engenharia" },
      "validation": ["Restore em VM isolada com contagem de pedidos igual à origem", "Tempo de restore registrado"],
      "rollback": "Remover o passo de cópia externa; o backup local continua",
      "status": "open"
    },
    {
      "id": "IG-002",
      "domain": "containers",
      "service": "app",
      "environment": "production",
      "title": "Log do container sem rotação",
      "severity": "P2",
      "confidence": "confirmed",
      "evidence": [
        { "kind": "file", "location": "compose.yaml:12", "excerpt": "sem bloco logging; driver padrão json-file" }
      ],
      "failure_mode": "Log cresce até encher o disco e para o banco no mesmo host",
      "impact": { "availability": "Parada da aplicação e do banco" },
      "recommendation": "Definir logging com max-size e max-file, ou driver local",
      "effort": { "size": "S", "basis": "estimated", "assumptions": "Uma linha por serviço e um redeploy" },
      "validation": ["docker inspect --format \"{{.HostConfig.LogConfig}}\" mostra a rotação", "Tamanho do log estável após 7 dias"],
      "rollback": "Remover o bloco logging e recriar o container",
      "status": "planned"
    }
  ],
  "evolution_plan": {
    "before_next_deploy": [ { "action": "Rotação de log", "finding_ids": ["IG-002"], "effort": "S" } ],
    "30_days": [ { "action": "Backup externo e simulação de restore", "finding_ids": ["IG-001"], "effort": "S" } ]
  },
  "for_security_sentinel": [],
  "runbook": [
    {
      "risk": "Disco cheio",
      "detect": "Alerta de uso de disco acima do limite combinado",
      "stabilize": "Identificar o diretório que cresceu e o dono antes de liberar espaço",
      "validate": "Aplicação responde e o banco aceita escrita"
    }
  ]
}
```

## 4. Validação

O schema e o exemplo acima foram validados com `ajv` (draft-07) em 12/11/2026, junto com casos inválidos que o schema precisa recusar (achado sem evidência, descartado sem evidência, confirmado sem modo de falha, esforço estimado sem premissas, ID fora do padrão, campo desconhecido, estado estável sem evidência, gate com exceções vazias, mais de cinco problemas prioritários). Registro em `INFRA_GUARDIAN_EVALS.md`, seção 4.
