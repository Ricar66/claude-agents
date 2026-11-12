# Security Sentinel — modelo de relatório

Este é o formato que o Security Sentinel usa para responder. Ele serve para três coisas: o agente segue a mesma ordem em toda auditoria, quem lê sabe onde achar cada informação e duas execuções podem ser comparadas (IDs estáveis, status por achado).

Regras que valem para o relatório inteiro:

- **Nenhum segredo, token, senha, chave ou dado pessoal** aparece no texto. Use `[REDIGIDO]` e informe só arquivo, linha e tipo.
- Endereços e nomes internos (IPs, hosts, contas) só quando forem necessários para corrigir, e o relatório passa a ser **interno**.
- Tudo que não foi verificado diz "não verificado". Nada de número de CVSS, EPSS ou versão sem fonte.

## 1. Modelo em Markdown

Copie a estrutura abaixo. Seções marcadas "(quando aplicável)" podem sair em modos curtos; a 1 e a 9 nunca saem.

```markdown
# Relatório de segurança — <projeto> — <modo> — <AAAA-MM-DD>

Classificação: <Interno | Público>   ·   Gerado por: security-sentinel   ·   Pedido por: <pessoa ou papel>

## 1. Escopo e stack
- Projeto e versão: <repositório>, branch <nome>, commit <sha curto>
- Modo: <triage | diff-review | deep-audit | threat-model | network-review | ai-agent-review | supply-chain | incident-triage | remediation-plan | verify-fix | release-gate>
- Teste ativo autorizado: <não | sim — escopo, janela, quem autorizou>
- Stack detectada (por arquivo, não por suposição): <linguagens, frameworks, banco, infraestrutura, CI>
- Contexto organizacional carregado: <sim | não>
- Arquivos e áreas lidos: <lista ou contagem por pasta>
- Não acessado e por quê: <ambientes, servidores, segredos, pastas fora do escopo>
- Limitações: <o que reduz a confiança das conclusões>

## 2. Arquitetura e fronteiras de confiança
<componentes, entradas e saídas, onde acontece autenticação e autorização, dados sensíveis, dependências externas>
Cadeias de ataque possíveis (condicionais): <ex.: "se a rota X estiver exposta à internet, SS-003 + SS-007 permitem…">

## 3. Resumo executivo
- Risco geral: <frase argumentada, sem "seguro" nem "invulnerável">
- Principais exposições: <3 a 5, com ID>
- O que não foi inspecionado: <lista curta>
- Prioridade imediata: <ação>

## 4. Achados
| ID | Título | Severidade | Confiança | Camada | Local | Status |
|---|---|---|---|---|---|---|
| SS-001 | … | Alta | Confirmado | API | src/rotas/pedidos.ts:42 | aberto |

### SS-001 — <título>
- Severidade / confiança: <Alta / Confirmado>
- Camada / ativo / tenant: <…>
- Localização: <arquivo:linha ou recurso de configuração>
- Fluxo (origem → validação → autorização → destino): <…>
- Pré-condições: <autenticado? papel? rede?>
- Impacto (C/I/D e negócio): <…>
- Evidência sanitizada: <trecho mínimo, comando e saída resumida, sem segredo>
- CWE / ASVS: <CWE-639 / V8.x.x — só se o mapeamento for claro; senão "sem mapeamento">
- CVE / KEV / EPSS / CVSS: <só se verificados, com fonte e data; senão "não se aplica" ou "não verificado">
- Controles existentes e tentativas de refutar: <o que foi procurado e o resultado>
- Causa raiz: <…>
- Correção recomendada: <mudança mínima na causa, e defesa em profundidade se fizer sentido>
- Teste de regressão: <caso positivo e negativo>
- Rollback e efeitos colaterais: <…>
- Risco residual: <…>
- Dono sugerido / status: <… / aberto>

## 5. Plano de remediação (quando aplicável)
| Onda | Achados | Ação | Esforço | Depende de | Dono | Teste | Rollback |
|---|---|---|---|---|---|---|---|
| Imediato | SS-001 | … | P | — | … | … | … |
Ondas: imediato · antes de produção · próximo ciclo · melhoria contínua.

## 6. Checklist operacional
| Eixo | Situação | Observação |
|---|---|---|
| Rede e perímetro | avaliado / parcial / não avaliado / NÃO APLICÁVEL | … |
| Identidade e IAM | … | … |
| Aplicação | … | … |
| CI/CD e cadeia de suprimentos | … | … |
| Banco e dados | … | … |
| IA e agentes | … | … |
| Logs e resposta | … | … |

## 7. Verificação pós-correção (quando aplicável)
| ID | Resultado | Evidência |
|---|---|---|
| SS-001 | corrigido e verificado / parcialmente mitigado / não corrigido / não verificado / falso positivo justificado / risco aceito | … |

## 8. Release gate (quando pedido)
Recomendação: <GO | GO COM EXCEÇÕES | NÃO RECOMENDADO | INCONCLUSIVO>
Motivos: <…>
Exceções (cada uma com dono e prazo definidos pelo usuário): <…>
A decisão de liberar é humana.

## 9. Cobertura
- Analisado: <camadas e pastas>
- Não analisado: <camadas e motivo>
- Tipo de análise: <estática | configuração | dinâmica autorizada>
- Ferramentas executadas: <nome, versão, comando, data, modo offline?>
- Ferramentas recomendadas e não executadas: <nome e comando sugerido>
```

## 2. Campos do achado

| Campo | Obrigatório | Valores e regras |
|---|---|---|
| `id` | sim | `SS-` + 3 ou mais dígitos. Mantém o número entre execuções quando os achados anteriores forem informados |
| `title` | sim | Curto, descreve o defeito, não a correção |
| `severity` | sim | `critical`, `high`, `medium`, `low`, `info` (impacto se explorado) |
| `confidence` | sim | `confirmed`, `probable`, `hypothesis` (força da evidência; independe da severidade) |
| `layer` | sim | `application`, `identity`, `api`, `data`, `network`, `infrastructure`, `supply_chain`, `ai`, `operations`, `iot`, `other` |
| `location` | sim | Arquivo e linhas, ou recurso de configuração. Nunca inventado |
| `flow` | sim para confirmado e provável | origem → validação → autorização → destino |
| `evidence` | sim | Sanitizada. Sem segredo nem dado pessoal |
| `cwe`, `asvs` | não | Só com mapeamento claro. ASVS cita a versão (ex.: `5.0.0`) |
| `cve` | não | Cada item com `id`, `kev` (true, false ou null), `epss` (score, percentil e data) ou null, `source` e `checked_at` |
| `cvss` | não | Só v4.0 com vetor completo e justificativa; sem vetor, null |
| `status` | sim | `open`, `fixed_verified`, `partially_mitigated`, `not_fixed`, `not_verified`, `false_positive`, `risk_accepted` |

## 3. Saída em JSON

Quando o usuário pedir JSON, o relatório segue o schema abaixo (`schema_version` `1.0`, JSON Schema draft-07, escolhido por ser aceito pela maioria dos validadores). **Nomes de campo e valores enumerados ficam em inglês** e são os mesmos nas versões em português e em inglês deste pacote, para que qualquer ferramenta leia os dois; os textos livres (título, evidência, recomendação) ficam no idioma do relatório.

| No relatório em Markdown | No JSON |
|---|---|
| Crítica, Alta, Média, Baixa, Informativa | `critical`, `high`, `medium`, `low`, `info` |
| Confirmado, Provável, Hipótese | `confirmed`, `probable`, `hypothesis` |
| aberto, corrigido e verificado, parcialmente mitigado, não corrigido, não verificado, falso positivo, risco aceito | `open`, `fixed_verified`, `partially_mitigated`, `not_fixed`, `not_verified`, `false_positive`, `risk_accepted` |
| imediato, antes de produção, próximo ciclo, melhoria contínua | `immediate`, `before_production`, `next_cycle`, `continuous_improvement` |
| avaliado, parcial, não avaliado, NÃO APLICÁVEL | `assessed`, `partial`, `not_assessed`, `not_applicable` |
| GO, GO COM EXCEÇÕES, NÃO RECOMENDADO, INCONCLUSIVO | `GO`, `GO_WITH_EXCEPTIONS`, `NOT_RECOMMENDED`, `INCONCLUSIVE` |
| esforço P, M, G | `S`, `M`, `L` |

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "urn:security-sentinel:report:1.0",
  "title": "Security Sentinel report",
  "type": "object",
  "additionalProperties": false,
  "required": ["schema_version", "generated_at", "mode", "scope", "summary", "findings", "coverage", "tools"],
  "definitions": {
    "nonEmpty": { "type": "string", "minLength": 1 },
    "finding": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "title", "severity", "confidence", "layer", "location", "evidence", "recommendation", "status"],
      "properties": {
        "id": { "type": "string", "pattern": "^SS-[0-9]{3,}$" },
        "title": { "$ref": "#/definitions/nonEmpty" },
        "severity": { "enum": ["critical", "high", "medium", "low", "info"] },
        "confidence": { "enum": ["confirmed", "probable", "hypothesis"] },
        "layer": { "enum": ["application", "identity", "api", "data", "network", "infrastructure", "supply_chain", "ai", "operations", "iot", "other"] },
        "asset": { "type": "string" },
        "location": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "file": { "type": "string" },
            "line_start": { "type": "integer", "minimum": 1 },
            "line_end": { "type": "integer", "minimum": 1 },
            "config": { "type": "string" }
          },
          "anyOf": [{ "required": ["file"] }, { "required": ["config"] }]
        },
        "flow": { "type": "string" },
        "preconditions": { "type": "string" },
        "impact": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "confidentiality": { "type": "string" },
            "integrity": { "type": "string" },
            "availability": { "type": "string" },
            "business": { "type": "string" }
          }
        },
        "evidence": { "$ref": "#/definitions/nonEmpty" },
        "cwe": { "type": "array", "items": { "type": "string", "pattern": "^CWE-[0-9]+$" } },
        "asvs": {
          "type": "array",
          "items": {
            "type": "object",
            "additionalProperties": false,
            "required": ["version", "requirement"],
            "properties": { "version": { "type": "string" }, "requirement": { "type": "string" } }
          }
        },
        "cve": {
          "type": "array",
          "items": {
            "type": "object",
            "additionalProperties": false,
            "required": ["id", "source", "checked_at"],
            "properties": {
              "id": { "type": "string", "pattern": "^CVE-[0-9]{4}-[0-9]{4,}$" },
              "kev": { "type": ["boolean", "null"] },
              "epss": {
                "type": ["object", "null"],
                "additionalProperties": false,
                "required": ["score", "percentile", "date"],
                "properties": {
                  "score": { "type": "number", "minimum": 0, "maximum": 1 },
                  "percentile": { "type": "number", "minimum": 0, "maximum": 1 },
                  "date": { "type": "string" }
                }
              },
              "source": { "type": "string" },
              "checked_at": { "type": "string" }
            }
          }
        },
        "cvss": {
          "type": ["object", "null"],
          "additionalProperties": false,
          "required": ["version", "vector", "justification"],
          "properties": {
            "version": { "enum": ["4.0"] },
            "vector": { "type": "string", "pattern": "^CVSS:4\\.0/" },
            "score": { "type": "number", "minimum": 0, "maximum": 10 },
            "justification": { "type": "string" }
          }
        },
        "existing_controls": { "type": "string" },
        "refutation_attempts": { "type": "string" },
        "root_cause": { "type": "string" },
        "recommendation": { "$ref": "#/definitions/nonEmpty" },
        "regression_test": { "type": "string" },
        "rollback": { "type": "string" },
        "residual_risk": { "type": "string" },
        "owner": { "type": "string" },
        "status": { "enum": ["open", "fixed_verified", "partially_mitigated", "not_fixed", "not_verified", "false_positive", "risk_accepted"] },
        "false_positive_reason": { "type": "string" }
      },
      "allOf": [
        {
          "if": { "properties": { "status": { "const": "false_positive" } } },
          "then": { "required": ["false_positive_reason"] }
        },
        {
          "if": { "properties": { "confidence": { "enum": ["confirmed", "probable"] } } },
          "then": { "required": ["flow"] }
        }
      ]
    }
  },
  "properties": {
    "schema_version": { "const": "1.0" },
    "generated_at": { "type": "string" },
    "classification": { "enum": ["internal", "public"] },
    "mode": { "enum": ["triage", "diff-review", "deep-audit", "threat-model", "network-review", "ai-agent-review", "supply-chain", "incident-triage", "remediation-plan", "verify-fix", "release-gate"] },
    "scope": {
      "type": "object",
      "additionalProperties": false,
      "required": ["project", "active_testing_authorized", "files_read", "not_accessed", "org_context_loaded"],
      "properties": {
        "project": { "$ref": "#/definitions/nonEmpty" },
        "branch": { "type": "string" },
        "commit": { "type": "string" },
        "environment": { "type": "string" },
        "active_testing_authorized": { "type": "boolean" },
        "authorization_ref": { "type": "string" },
        "stack": { "type": "array", "items": { "type": "string" } },
        "files_read": { "type": "array", "items": { "type": "string" } },
        "not_accessed": { "type": "array", "items": { "type": "string" } },
        "org_context_loaded": { "type": "boolean" },
        "limitations": { "type": "array", "items": { "type": "string" } }
      }
    },
    "architecture": { "type": "string" },
    "summary": {
      "type": "object",
      "additionalProperties": false,
      "required": ["overall_risk", "top_risks", "not_inspected"],
      "properties": {
        "overall_risk": { "$ref": "#/definitions/nonEmpty" },
        "top_risks": { "type": "array", "items": { "type": "string" } },
        "not_inspected": { "type": "array", "items": { "type": "string" } },
        "immediate_priority": { "type": "string" }
      }
    },
    "findings": { "type": "array", "items": { "$ref": "#/definitions/finding" } },
    "remediation_plan": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["wave", "finding_ids", "action"],
        "properties": {
          "wave": { "enum": ["immediate", "before_production", "next_cycle", "continuous_improvement"] },
          "finding_ids": { "type": "array", "items": { "type": "string", "pattern": "^SS-[0-9]{3,}$" } },
          "action": { "type": "string" },
          "effort": { "enum": ["S", "M", "L"] },
          "dependencies": { "type": "string" },
          "owner": { "type": "string" },
          "test": { "type": "string" },
          "rollback": { "type": "string" }
        }
      }
    },
    "coverage": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["axis", "status"],
        "properties": {
          "axis": { "enum": ["network", "identity_iam", "application", "ci_supply_chain", "database_data", "ai_agents", "logging_response", "infrastructure", "iot"] },
          "status": { "enum": ["assessed", "partial", "not_assessed", "not_applicable"] },
          "notes": { "type": "string" }
        }
      }
    },
    "tools": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["name", "executed"],
        "properties": {
          "name": { "type": "string" },
          "version": { "type": "string" },
          "command": { "type": "string" },
          "executed": { "type": "boolean" },
          "date": { "type": "string" },
          "network": { "type": "string" },
          "notes": { "type": "string" }
        }
      }
    },
    "verification": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["finding_id", "result"],
        "properties": {
          "finding_id": { "type": "string", "pattern": "^SS-[0-9]{3,}$" },
          "result": { "enum": ["fixed_verified", "partially_mitigated", "not_fixed", "not_verified", "false_positive", "risk_accepted"] },
          "evidence": { "type": "string" }
        }
      }
    },
    "release_gate": {
      "type": ["object", "null"],
      "additionalProperties": false,
      "required": ["recommendation", "reasons"],
      "properties": {
        "recommendation": { "enum": ["GO", "GO_WITH_EXCEPTIONS", "NOT_RECOMMENDED", "INCONCLUSIVE"] },
        "reasons": { "type": "array", "items": { "type": "string" }, "minItems": 1 },
        "exceptions": {
          "type": "array",
          "items": {
            "type": "object",
            "additionalProperties": false,
            "required": ["finding_id", "owner", "deadline"],
            "properties": {
              "finding_id": { "type": "string", "pattern": "^SS-[0-9]{3,}$" },
              "owner": { "type": "string" },
              "deadline": { "type": "string" }
            }
          }
        }
      }
    }
  }
}
```

### Exemplo mínimo válido

Projeto e achado fictícios, só para mostrar a forma.

```json
{
  "schema_version": "1.0",
  "generated_at": "2026-11-12T15:00:00-03:00",
  "classification": "internal",
  "mode": "diff-review",
  "scope": {
    "project": "exemplo-pedidos",
    "branch": "feat/exportar-pedidos",
    "commit": "a1b2c3d",
    "active_testing_authorized": false,
    "stack": ["TypeScript", "Next.js 15", "Prisma", "PostgreSQL"],
    "files_read": ["src/app/api/pedidos/[id]/route.ts", "src/lib/auth.ts", "prisma/schema.prisma"],
    "not_accessed": ["ambiente de produção", ".env"],
    "org_context_loaded": false
  },
  "summary": {
    "overall_risk": "Alto no diff: a rota nova devolve pedidos de outro tenant a qualquer usuário autenticado.",
    "top_risks": ["SS-001"],
    "not_inspected": ["configuração de produção", "CI"]
  },
  "findings": [
    {
      "id": "SS-001",
      "title": "Rota de pedido não filtra por tenant (IDOR/BOLA)",
      "severity": "high",
      "confidence": "confirmed",
      "layer": "api",
      "location": { "file": "src/app/api/pedidos/[id]/route.ts", "line_start": 14, "line_end": 18 },
      "flow": "params.id (URL) → prisma.pedido.findUnique({ where: { id } }) → resposta JSON, sem comparar pedido.tenantId com a sessão",
      "evidence": "findUnique usa só o id; a sessão é lida na linha 9, mas tenantId não entra no where nem é comparado depois.",
      "cwe": ["CWE-639"],
      "existing_controls": "Middleware exige sessão válida; não há checagem de objeto.",
      "refutation_attempts": "Procurei política RLS no banco (schema.prisma e migrações): não há. Procurei wrapper de repositório com tenant: não é usado nesta rota.",
      "root_cause": "Consulta por chave primária sem escopo de tenant.",
      "recommendation": "Buscar com where { id, tenantId: sessao.tenantId } e responder 404 quando não houver resultado.",
      "regression_test": "Usuário do tenant A pedindo pedido do tenant B recebe 404; pedido do próprio tenant continua 200.",
      "rollback": "Reverter o commit da correção; sem migração envolvida.",
      "status": "open"
    }
  ],
  "coverage": [
    { "axis": "application", "status": "partial", "notes": "Só os arquivos do diff e chamadores diretos." },
    { "axis": "network", "status": "not_assessed" },
    { "axis": "iot", "status": "not_applicable" }
  ],
  "tools": [
    { "name": "semgrep", "executed": false, "notes": "Não instalado; comando sugerido no plano." }
  ],
  "release_gate": null
}
```

## 4. Validação

- O schema e o exemplo acima foram conferidos com o `ajv` 6 (draft-07) no momento da criação; ver `SECURITY_SENTINEL_SETUP.md`, seção de validação.
- O agente não valida o próprio JSON. Se o relatório for consumido por outra ferramenta, valide antes de importar.
- SARIF não faz parte deste formato. Quando a equipe precisar de SARIF, ele deve vir de uma ferramenta que gere e valide SARIF 2.1.0 (CodeQL, Semgrep e outras), não de texto escrito pelo modelo.
