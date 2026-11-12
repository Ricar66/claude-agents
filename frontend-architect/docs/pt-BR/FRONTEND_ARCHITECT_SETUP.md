# Frontend Architect — instalação, permissões e operação

## 1. O que vem no pacote

```text
frontend-architect/
├── agent/
│   ├── pt-BR/frontend-architect.md    ← agente em português
│   └── en/frontend-architect.md       ← mesmo agente em inglês
├── org-context-example/
│   ├── pt-BR/SKILL.md                 ← exemplo fictício de padrão visual e contexto
│   └── en/SKILL.md
└── docs/
    ├── pt-BR/                         ← guia de instalação, playbook, avaliações
    └── en/
```

As duas versões têm o mesmo `name`: **instale um idioma só**. O guia de uso dos quatro agentes da coleção está em [`GUIA.pt-BR.md`](../../../GUIA.pt-BR.md).

## 2. O que muda em relação aos agentes de auditoria

O Frontend Architect **implementa**: tem `Edit`, `Write` e `Bash`, segue o `CLAUDE.md` do projeto e roda os testes do projeto. Ele só escreve quando o pedido é de implementação; deploy, API real, instalação de dependência, servidor de desenvolvimento não pedido, commit e push precisam de autorização específica. As camadas da seção 5 são a barreira de verdade.

## 3. Requisitos e instalação

- Claude Code com subagentes em arquivo. Conferido no Claude Code **2.1.221** contra a documentação oficial de subagentes (12/11/2026).
- Campos usados: `name`, `description`, `tools`, `model`, `permissionMode`, `color`, `skills`, todos da lista oficial.
- `model: inherit` usa o modelo da sessão. `permissionMode: default` é ignorado quando a sessão principal está em `auto`, `acceptEdits` ou `bypassPermissions` (vale o modo da sessão), e em `plan` o agente fica em `plan`. Não existe campo para limitar escrita por pasta. Não use `bypassPermissions`.

Windows (PowerShell):

```powershell
Copy-Item .\frontend-architect\agent\pt-BR\frontend-architect.md "$env:USERPROFILE\.claude\agents\"
```

Linux e macOS:

```bash
cp frontend-architect/agent/pt-BR/frontend-architect.md ~/.claude/agents/
```

Só para um projeto: `<projeto>/.claude/agents/`. Depois de instalar ou atualizar, abra uma **sessão nova**.

## 4. Padrão visual da organização (opcional e local)

O agente pré-carrega a skill `frontend-architect-org-context` se ela existir. É nela que fica o **padrão visual obrigatório** da sua organização: cores com hex e uso, tipografia, logos (caminho dos arquivos e regras de uso), imagens permitidas, padrões de tela, sistemas e regras. Crie em `~/.claude/skills/frontend-architect-org-context/SKILL.md`, com `name` igual ao da pasta e sem `disable-model-invocation`. Modelo fictício: `frontend-architect/org-context-example/pt-BR/SKILL.md`.

Com a skill presente, o agente aplica esse padrão nos projetos da organização, salvo quando o projeto já tiver tema próprio aprovado. Sem a skill, ele segue os tokens que encontrar no repositório.

## 5. Permissões recomendadas

Nada vem ativado. As regras de `permissions` valem para a conversa principal e para os subagentes; a ordem oficial é deny, depois ask, depois allow.

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Bash(git push *)"
    ],
    "ask": [
      "Bash(git commit *)",
      "Bash(npm install *)",
      "Bash(npm ci *)",
      "Bash(pnpm add *)",
      "Bash(yarn add *)",
      "Bash(npx playwright install *)",
      "Bash(npm run dev *)",
      "Bash(npm run deploy *)"
    ]
  }
}
```

Padrões de Bash são frágeis (outro caminho ou `sh -c` escapam); teste antes. Para mudanças grandes, use um worktree separado e revise o diff; o frontmatter aceita `isolation: worktree` (não vem ligado porque muda o fluxo de integração). No Windows nativo não há sandbox do Bash.

## 6. Como invocar

- `Use o agente frontend-architect para discover deste projeto.`
- `Use o agente frontend-architect para ux-review da tela de pedidos.`
- `Use o agente frontend-architect para accessibility-audit do formulário de cadastro, com teste de teclado.`
- `Use o agente frontend-architect para responsive-audit da tabela de clientes no celular.`
- `Use o agente frontend-architect para implement dos estados de vazio, erro e carregando na lista de pedidos, com testes.`
- `Use o agente frontend-architect para design-system: unificar os botões e tokens de cor.`
- `Use o agente frontend-architect para dashboard-data-viz do painel de vendas.`

Diga o modo, a tela ou jornada e se pode implementar.

## 7. Colaboração

Frontend é dono de UI, UX e consumo de APIs; Backend, de APIs e banco; Sentinel, da auditoria de segurança; Guardian, de infraestrutura. Mudança de contrato é combinada com o Backend Architect antes. Protocolo completo no [guia da coleção](../../../GUIA.pt-BR.md).

## 8. Validação feita neste pacote

Registrada em `FRONTEND_ARCHITECT_EVALS.md`, seção 4.

## 9. Limitações

- Com `Edit`, `Write` e `Bash`, os limites dependem do texto do agente e das camadas da seção 5.
- O agente não abre navegador nem mede Core Web Vitals sozinho: medida de campo vem de RUM; medida de laboratório precisa de ferramenta do projeto e autorização para subir o servidor.
- Scanner de acessibilidade não substitui teste manual; o agente declara o que foi testado à mão.
- Sem internet: versões e suporte ficam "não verificados" até alguém conferir; as do playbook são de 12/11/2026.
- Um modelo pode errar: revise todo diff e a tela antes de integrar.

## 10. Manutenção

- `pt-BR` e `en` mudam juntos.
- Revise o playbook a cada 6 meses e quando sair versão nova de React, Next.js, TypeScript, WCAG ou das métricas de Core Web Vitals.
- Mudou o agente? Rode de novo pelo menos FE-01, FE-03, FE-07 e FE-14 e registre.
