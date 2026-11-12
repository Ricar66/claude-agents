# Guia dos agentes — como usar e como chamar cada um

[English](GUIDE.md) · **Português**

Este guia reúne, num lugar só, o que cada agente da coleção faz, quando usar cada um, como chamar e como eles trabalham juntos. Detalhes de instalação e permissões de cada agente ficam no `docs/` dele.

## 1. Os quatro agentes

| Agente | Papel | Pode alterar arquivos? | Use quando |
|---|---|---|---|
| `security-sentinel` | Auditor de segurança | **Não** (só lê) | Quer saber se há vulnerabilidade, exposição, segredo, risco em IA ou dependência |
| `infra-guardian` | SRE, infraestrutura, banco, custos | **Não** (só lê e propõe plano) | Servidor, Docker, Kubernetes, rede, deploy, backup, monitoramento, capacidade, incidente, custo |
| `infra-guardian-static` | O mesmo, sem nenhum comando | **Não** | Quando nada pode ser executado, nem comando de leitura |
| `backend-architect` | Arquiteto e engenheiro de backend | **Sim**, quando você pede implementação | APIs, regras de negócio, banco, integrações, migrações, testes de servidor |
| `frontend-architect` | Arquiteto e engenheiro de frontend | **Sim**, quando você pede implementação | Telas, UX, design system, acessibilidade, responsividade, performance, gráficos |

Regra de bolso: **os dois auditores dizem o que está errado e como corrigir; os dois arquitetos desenham e, se você pedir, implementam.** Nenhum faz deploy, push, migração em banco real ou instala dependência sem você autorizar na conversa.

## 2. Instalação rápida

Copie para `~/.claude/agents/` (todos os projetos) ou `<projeto>/.claude/agents/` (um projeto). Escolha **um idioma** por agente.

```powershell
# Windows (PowerShell), a partir da raiz deste repositório
foreach ($a in "security-sentinel","infra-guardian","backend-architect","frontend-architect") {
  Copy-Item ".\$a\agent\pt-BR\$a.md" "$env:USERPROFILE\.claude\agents\"
}
Copy-Item ".\infra-guardian\agent\pt-BR\infra-guardian-static.md" "$env:USERPROFILE\.claude\agents\"
```

```bash
# Linux e macOS
for a in security-sentinel infra-guardian backend-architect frontend-architect; do
  cp "$a/agent/pt-BR/$a.md" ~/.claude/agents/
done
cp infra-guardian/agent/pt-BR/infra-guardian-static.md ~/.claude/agents/
```

Depois de instalar ou atualizar, **abra uma sessão nova** do Claude Code: uma sessão já aberta pode continuar com a versão anterior.

## 3. Como chamar

A forma que sempre funciona: **"Use o agente `<nome>` para `<modo>` de `<escopo>`"**. Diga o escopo (pasta, rota, tela, branch) e, para os arquitetos, se pode implementar.

### 3.1 Security Sentinel

| Modo | Para quê |
|---|---|
| `triage` | Varredura rápida, top riscos |
| `diff-review` | Revisar PR, commit ou branch |
| `deep-audit` | Auditoria completa por camadas |
| `threat-model` | Modelo de ameaças |
| `network-review` | Rede, proxy, firewall, DNS, TLS, Docker, por arquivos |
| `ai-agent-review` | LLM, agentes, MCP, RAG, webhooks |
| `supply-chain` | Dependências e CI |
| `incident-triage` | Apoio a incidente de segurança |
| `remediation-plan` | Plano de correção em ondas |
| `verify-fix` | Conferir correções (IDs `SS-xxx`) |
| `release-gate` | Recomendação de release |

- `Use o agente security-sentinel para executar deep-audit deste repositório, somente leitura; não faça testes externos.`
- `Use o agente security-sentinel para diff-review das alterações da branch atual.`
- `Use o agente security-sentinel para verify-fix dos achados SS-001 e SS-002.`

### 3.2 Infra Guardian

| Modo | Para quê |
|---|---|
| `discover` | Inventário e arquitetura atual |
| `architecture-review` | Decisões e proposta, com ADR |
| `vps-audit` | Servidor (a partir da coleta que você cola) |
| `docker-review` · `k8s-review` | Compose, Dockerfile, manifests |
| `network-diagnose` | Caminho DNS → TLS → CDN → proxy → app → banco |
| `cloud-iac` | Terraform, OpenTofu, state, drift |
| `pipeline-review` | CI/CD e deploy |
| `observability` | SLI, SLO, alertas, runbooks |
| `database-ops` | Operação de banco |
| `backup-dr` | Backup, restore, RPO e RTO propostos |
| `capacity-plan` · `finops` | Capacidade e custos |
| `incident-triage` | Incidente de disponibilidade |
| `migration-plan` · `release-readiness` · `verify-change` | Migração, prontidão, verificação depois da mudança (IDs `IG-xxx`) |

- `Use o agente infra-guardian para vps-audit; a saída de df, free e systemctl está em coleta/.`
- `Use o agente infra-guardian para backup-dr do banco principal, propondo RPO e RTO para aprovação.`
- `Use o agente infra-guardian-static para architecture-review, sem executar nenhum comando.`

### 3.3 Backend Architect

| Modo | Para quê |
|---|---|
| `discover` | Mapa do backend com evidência |
| `design` · `api-contract` | Solução e contrato de API |
| `technology-assessment` | Manter, evoluir ou migrar, com matriz e nota |
| `implement` · `refactor` | Implementar uma fatia pequena, com testes |
| `code-review` | Revisão com achados por gravidade |
| `database-review` · `migration-plan` | Banco, consultas, migrações seguras |
| `performance` · `integration` | Medida antes e depois; integrações com terceiros |
| `test-strategy` · `incident-analysis` | Estratégia de testes; análise de falha |
| `release-readiness` · `verify` | Prontidão e verificação |

- `Use o agente backend-architect para discover deste repositório.`
- `Use o agente backend-architect para api-contract do endpoint de pedidos, sem implementar.`
- `Use o agente backend-architect para implement da paginação por cursor em GET /pedidos, com testes.`

### 3.4 Frontend Architect

| Modo | Para quê |
|---|---|
| `discover` | Mapa de rotas, telas, componentes, tokens, dados |
| `ux-review` · `design-system` | Experiência e componentes/tokens |
| `architecture-review` · `technology-assessment` | Renderização, dados, estado; decisões de tecnologia |
| `implement` · `refactor` | Implementar alteração local, com testes |
| `responsive-audit` · `accessibility-audit` | Celular/tablet/desktop; WCAG 2.2 AA |
| `performance` | Core Web Vitals com medida |
| `api-integration` · `realtime-ui` | Consumo de API; tempo real |
| `dashboard-data-viz` · `three-d` | Gráficos; 3D com fallback |
| `test-strategy` · `release-readiness` · `verify` | Testes, prontidão, verificação |

- `Use o agente frontend-architect para ux-review da tela de pedidos.`
- `Use o agente frontend-architect para accessibility-audit do formulário de cadastro, com teste de teclado.`
- `Use o agente frontend-architect para implement dos estados de vazio, erro e carregando na lista de pedidos, com testes.`

## 4. Como eles trabalham juntos

| Dono de | Agente |
|---|---|
| APIs, regras de negócio, banco | `backend-architect` |
| UI, UX, design system, consumo de API | `frontend-architect` |
| Auditoria de segurança | `security-sentinel` |
| Infraestrutura, SRE, cloud, FinOps | `infra-guardian` |

Os subagentes **não chamam uns aos outros**. Cada um termina com um **handoff** (para quem, o que fazer, arquivos, riscos, como verificar), e quem encaminha é você ou o Claude principal.

**Feature que cruza camadas, passo a passo:**

1. `backend-architect` em `design` ou `api-contract`: contrato (schema, erros, autenticação, paginação, cache, compatibilidade).
2. `frontend-architect` em `ux-review` ou `design-system`: jornada e estados da tela sobre esse contrato.
3. Você aprova o contrato e decide quem é dono de cada arquivo.
4. Cada arquiteto em `implement` no seu lado, com testes.
5. `security-sentinel` em `diff-review` e, se mexer em servidor, deploy ou banco, `infra-guardian` em `release-readiness`.
6. Correções; depois `verify` (arquitetos), `verify-fix` (Sentinel), `verify-change` (Guardian).
7. Você decide o release.

Exemplo de pedido encadeado ao Claude principal: `Use o backend-architect para api-contract da exportação de pedidos; depois o frontend-architect para ux-review da tela de exportação usando esse contrato; não implemente ainda.`

## 5. Pronto quer dizer (vale para todos)

Contexto entendido · projeto preservado · contrato coerente · código coeso e tipado · estados de falha testados · testes executados ou pendências declaradas · acessibilidade e performance avaliadas na proporção do risco · documentação alinhada · handoff feito · riscos explícitos.

## 6. Tecnologia: manter, evoluir ou migrar

Os arquitetos comparam sempre **(A) manter e melhorar**, **(B) evolução incremental** e **(C) migração**, com nota de 0 a 5 por critério e evidência. Pesos iniciais: adequação ao negócio 25%, compatibilidade e migração 20%, segurança ou acessibilidade e maturidade 15%, operação ou performance 15%, escalabilidade e integração 10%, equipe e ecossistema 10%, custo total e dependência de fornecedor 5%. Sem benchmark, a nota não é positiva. Ninguém troca stack por moda.

## 7. Contexto da sua organização

Cada agente pode pré-carregar uma skill local `<agente>-org-context` com o que só a sua organização sabe (sistemas, servidores, padrões, riscos). Para o `frontend-architect`, é ali que fica o **padrão visual obrigatório** (cores, fontes, logos, imagens). A skill fica em `~/.claude/skills/<agente>-org-context/SKILL.md`, **fora deste repositório**, sem senha nem token. Modelos fictícios em `<agente>/org-context-example/`.

## 8. Permissões: o que é instrução e o que é barreira

- O texto de cada agente proíbe ações perigosas, mas **instrução não é barreira**. As barreiras reais são: a lista `tools` de cada agente, as regras `permissions` (deny, ask, allow) do `settings.json` e, para os auditores, hooks propostos nos guias de instalação.
- `permissionMode` no agente é ignorado quando a sessão principal está em `auto`, `acceptEdits` ou `bypassPermissions`. Para os arquitetos, que editam código, trabalhe em sessão no modo padrão ou em worktree separado e revise o diff.
- No Windows nativo não há sandbox do Bash do Claude Code.

## 9. Onde ler mais

| Agente | Instalação e permissões | Playbook | Avaliações |
|---|---|---|---|
| Security Sentinel | [`SECURITY_SENTINEL_SETUP.md`](security-sentinel/docs/pt-BR/SECURITY_SENTINEL_SETUP.md) | [`SECURITY_SENTINEL_PLAYBOOK.md`](security-sentinel/docs/pt-BR/SECURITY_SENTINEL_PLAYBOOK.md) | [`SECURITY_SENTINEL_EVALS.md`](security-sentinel/docs/pt-BR/SECURITY_SENTINEL_EVALS.md) |
| Infra Guardian | [`INFRA_GUARDIAN_SETUP.md`](infra-guardian/docs/pt-BR/INFRA_GUARDIAN_SETUP.md) | [`INFRA_GUARDIAN_PLAYBOOK.md`](infra-guardian/docs/pt-BR/INFRA_GUARDIAN_PLAYBOOK.md) | [`INFRA_GUARDIAN_EVALS.md`](infra-guardian/docs/pt-BR/INFRA_GUARDIAN_EVALS.md) |
| Backend Architect | [`BACKEND_ARCHITECT_SETUP.md`](backend-architect/docs/pt-BR/BACKEND_ARCHITECT_SETUP.md) | [`BACKEND_ARCHITECT_PLAYBOOK.md`](backend-architect/docs/pt-BR/BACKEND_ARCHITECT_PLAYBOOK.md) | [`BACKEND_ARCHITECT_EVALS.md`](backend-architect/docs/pt-BR/BACKEND_ARCHITECT_EVALS.md) |
| Frontend Architect | [`FRONTEND_ARCHITECT_SETUP.md`](frontend-architect/docs/pt-BR/FRONTEND_ARCHITECT_SETUP.md) | [`FRONTEND_ARCHITECT_PLAYBOOK.md`](frontend-architect/docs/pt-BR/FRONTEND_ARCHITECT_PLAYBOOK.md) | [`FRONTEND_ARCHITECT_EVALS.md`](frontend-architect/docs/pt-BR/FRONTEND_ARCHITECT_EVALS.md) |

Um modelo de linguagem pode errar: confira a evidência e revise todo diff antes de agir.
