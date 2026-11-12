# Backend Architect — instalação, permissões e operação

## 1. O que vem no pacote

```text
backend-architect/
├── agent/
│   ├── pt-BR/backend-architect.md     ← agente em português
│   └── en/backend-architect.md        ← mesmo agente em inglês
├── org-context-example/
│   ├── pt-BR/SKILL.md                 ← exemplo fictício de contexto organizacional
│   └── en/SKILL.md
└── docs/
    ├── pt-BR/                         ← guia de instalação, playbook, avaliações
    └── en/
```

As duas versões têm o mesmo `name`: **instale um idioma só**. O guia de uso dos quatro agentes da coleção está em [`GUIA.pt-BR.md`](../../../GUIA.pt-BR.md), na raiz do repositório.

## 2. O que muda em relação aos agentes de auditoria

O Security Sentinel e o Infra Guardian só leem. O Backend Architect **implementa**: tem `Edit`, `Write` e `Bash`, lê o `CLAUDE.md` do projeto como convenção e roda os testes do projeto. Por isso:

- ele só escreve quando o pedido é de implementação (`implement`, `refactor` ou correção explícita);
- deploy, banco real, migração fora de banco local descartável, instalação de dependência, commit e push precisam de autorização específica na conversa;
- as camadas de permissão da seção 5 são a barreira de verdade; o texto do agente não é.

## 3. Requisitos e instalação

- Claude Code com subagentes em arquivo. Conferido no Claude Code **2.1.221** contra a documentação oficial de subagentes (12/11/2026).
- Campos usados: `name`, `description`, `tools`, `model`, `permissionMode`, `color`, `skills`. Todos estão na lista oficial.
- `model: inherit`: o agente usa o modelo da sessão que o chamou.
- `permissionMode: default`: o agente pede confirmação conforme as regras de permissão. **Esse campo é ignorado quando a sessão principal está em `auto`, `acceptEdits` ou `bypassPermissions`** (vale o modo da sessão), e em `plan` o agente fica em `plan`. Não existe campo para limitar escrita por pasta. Não use `bypassPermissions`.

Windows (PowerShell):

```powershell
Copy-Item .\backend-architect\agent\pt-BR\backend-architect.md "$env:USERPROFILE\.claude\agents\"
```

Linux e macOS:

```bash
cp backend-architect/agent/pt-BR/backend-architect.md ~/.claude/agents/
```

Só para um projeto: copie para `<projeto>/.claude/agents/`. Depois de instalar ou atualizar, abra uma **sessão nova**: uma sessão aberta pode manter a versão anterior do agente.

## 4. Contexto organizacional (opcional e local)

O agente pré-carrega a skill `backend-architect-org-context` se ela existir: sistemas, stacks e versões, padrões da casa (autenticação, erros de API, nomes, migrações), integrações sensíveis e regras. Crie em `~/.claude/skills/backend-architect-org-context/SKILL.md`, com `name` igual ao da pasta, sem `disable-model-invocation`. Modelo fictício: `backend-architect/org-context-example/pt-BR/SKILL.md`. Nunca coloque senha, token ou connection string; o arquivo é interno e não vai para repositório público.

## 5. Permissões recomendadas

Nada aqui vem ativado. As regras de `permissions` valem para a conversa principal **e** para os subagentes; a ordem oficial é deny, depois ask, depois allow.

### 5.1 Negar o que nunca deve acontecer sem você

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./**/*.pem)",
      "Read(./**/*.key)",
      "Read(./**/*.pfx)",
      "Bash(git push *)"
    ],
    "ask": [
      "Bash(git commit *)",
      "Bash(npm install *)",
      "Bash(npm ci *)",
      "Bash(pnpm add *)",
      "Bash(yarn add *)",
      "Bash(pip install *)",
      "Bash(dotnet add *)",
      "Bash(npx prisma migrate *)",
      "Bash(dotnet ef database *)",
      "Bash(docker compose up *)"
    ]
  }
}
```

Junte ao seu `~/.claude/settings.json` ou ao `.claude/settings.json` do projeto. Limite da própria documentação: padrões de Bash que tentam restringir argumentos são frágeis (o mesmo programa chamado por outro caminho ou dentro de `sh -c` escapa). Teste antes, porque as regras afetam também o seu uso normal.

### 5.2 Trabalho isolado

Para mudanças maiores, rode o agente num worktree separado e revise o diff antes de integrar. O frontmatter aceita `isolation: worktree`, que faz o agente sempre trabalhar numa cópia isolada do repositório; ele não vem ligado porque muda o fluxo (as alterações ficam no worktree até você integrar). No Windows nativo o sandbox do Bash do Claude Code não existe; isolamento forte é VM ou container.

## 6. Como invocar

- `Use o agente backend-architect para discover deste repositório.`
- `Use o agente backend-architect para api-contract do endpoint de pedidos, sem implementar.`
- `Use o agente backend-architect para implement da paginação por cursor em GET /pedidos, com testes.`
- `Use o agente backend-architect para database-review das consultas do relatório de vendas.`
- `Use o agente backend-architect para migration-plan da troca de cliente por cliente_id.`
- `Use o agente backend-architect para technology-assessment: manter Express ou ir para Fastify.`
- `Use o agente backend-architect para verify das mudanças da branch atual.`

Diga o modo, o escopo (pasta, rota, branch) e se pode implementar. Sem "implemente", ele entrega desenho ou revisão.

## 7. Colaboração

Backend é dono de APIs, regras e banco; Frontend, de UI; Sentinel, da auditoria de segurança; Guardian, de infraestrutura. O protocolo completo (contrato antes do código, dono de cada arquivo, handoff, revisões) está no [guia da coleção](../../../GUIA.pt-BR.md). Os subagentes não chamam uns aos outros: quem encaminha é você ou o Claude principal, a partir do handoff que o agente escreve.

## 8. Validação feita neste pacote

Registrada em `BACKEND_ARCHITECT_EVALS.md`, seção 4: parse do frontmatter com um parser YAML, conferência dos campos contra a lista oficial e cenários de comportamento. `claude plugin validate <pasta>` existe a partir do Claude Code **2.1.233**.

## 9. Limitações

- Com `Edit`, `Write` e `Bash`, os limites dependem do texto do agente e das camadas da seção 5. Instrução não é barreira.
- `permissionMode` não protege quando a sessão principal está em modo permissivo.
- O agente não consulta a internet: versões e suporte ficam como "não verificado" até alguém conferir; as versões do playbook são de 12/11/2026.
- Um modelo pode errar: revise todo diff antes de integrar.

## 10. Manutenção

- `pt-BR` e `en` mudam juntos.
- Revise as versões do playbook a cada 6 meses e quando sair versão nova de Node.js, .NET, Python, TypeScript, OpenAPI ou do ORM em uso.
- Mudou o agente? Rode de novo pelo menos BE-01, BE-04, BE-05 e BE-13 das avaliações e registre.
