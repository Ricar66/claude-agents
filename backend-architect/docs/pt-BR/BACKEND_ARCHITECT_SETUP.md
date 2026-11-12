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
      "Read(./**/.env)",
      "Read(./**/.env.local)",
      "Read(./**/.env.production)",
      "Read(./**/.env.development)",
      "Read(./**/.env.test)",
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

Os padrões com `./` valem a partir da pasta onde o Claude Code foi aberto, inclusive subpastas (monorepo). Para valer em qualquer pasta, no `~/.claude/settings.json`, troque `./` por `//` (ex.: `Read(//**/.env)`); `**/.env` sem prefixo também é relativo à pasta atual. O `.env.example` fica de fora de propósito: o agente precisa ler os nomes das variáveis.

Junte ao seu `~/.claude/settings.json` ou ao `.claude/settings.json` do projeto. Limite da própria documentação: padrões de Bash que tentam restringir argumentos são frágeis (o mesmo programa chamado por outro caminho ou dentro de `sh -c` escapa). Teste antes, porque as regras afetam também o seu uso normal.

### 5.2 Trabalho isolado

Para mudanças maiores, rode o agente num worktree separado e revise o diff antes de integrar. O frontmatter aceita `isolation: worktree`, que faz o agente sempre trabalhar numa cópia isolada do repositório; ele não vem ligado porque muda o fluxo (as alterações ficam no worktree até você integrar). No Windows nativo o sandbox do Bash do Claude Code não existe; isolamento forte é VM ou container.

### 5.3 Hook que barra o que o agente nunca faz sozinho

Hooks podem ficar no frontmatter do subagente e só valem enquanto ele roda. Hooks de agentes em `~/.claude/agents/` não dependem do diálogo de confiança da pasta; os de `.claude/agents/` do projeto, sim. Diferente do hook do Security Sentinel (lista do que pode), este é uma lista do que não pode: build, teste, lint e scripts do projeto seguem o fluxo normal de permissão; encadeamento, `cd`, push, commit, instalação de pacote, migração, rede, deploy, servidor, remoção de arquivo e leitura de arquivo de segredo são barrados.

**Proposta** (crie e ative só se quiser essa camada; o mesmo script serve ao Frontend Architect):

1. Salve o script em `~/.claude/hooks/architect-bash-guard.mjs`:

```js
// Hook PreToolUse (Bash) do backend-architect e do frontend-architect.
// Barra o que esses agentes nunca podem fazer sozinhos; o resto segue o fluxo normal de permissão.
import { readFileSync } from "node:fs";

const entrada = JSON.parse(readFileSync(0, "utf8"));
const comando = String(entrada?.tool_input?.command ?? "").trim();

const negar = (motivo) => {
  process.stderr.write(`architect: comando bloqueado (${motivo}). Rode um comando por vez, sem cd; se precisar disso, peça ao usuário ou ao Claude principal.\n`);
  process.exit(2);
};

if (!comando) negar("vazio");
if (/[;&|`\n\r]|\$\(|>|</.test(comando)) negar("encadeamento, redirecionamento ou substituição");
if (/^(cd|pushd|Set-Location)\b/i.test(comando)) negar("mudar de pasta (use git -C <pasta> ou o argumento de pasta da ferramenta)");

const regras = [
  [/^git( -C ("[^"]+"|\S+))? (push|commit|reset|checkout|switch|restore|stash|clean|rebase|merge|cherry-pick|revert|tag|am|apply)\b/, "Git que altera histórico ou arquivos"],
  [/^git( -C ("[^"]+"|\S+))? (branch|remote) .*(-d|-D|--delete|add|set-url|remove|rm)\b/, "Git que altera branch ou remoto"],
  [/^(npm|pnpm|yarn|bun) (i|install|ci|add|remove|rm|uninstall|update|up|upgrade|publish|link|dlx)\b/, "instalação ou publicação de pacote"],
  [/^(npx|pnpx|bunx) (-y|--yes)\b/, "execução de pacote baixado na hora"],
  [/^npx (playwright|cypress) install\b/, "download de navegador"],
  [/^(pip3?|uv|poetry|pipenv|conda) (install|add|remove|uninstall|update|sync|lock)\b|^python3? -m pip\b/, "instalação de pacote Python"],
  [/^dotnet (add|remove|restore|tool|workload|nuget)\b/, "pacote .NET"],
  [/^(cargo (add|install|publish|update)|go (get|install)|gem install|composer (require|install|update))\b/, "instalação de pacote"],
  [/prisma (migrate (deploy|reset|dev)|db (push|execute|seed))|dotnet ef database|knex migrate|sequelize db:|typeorm migration:run|alembic (upgrade|downgrade)|flask db (upgrade|downgrade)|rails db:/, "migração ou alteração de banco"],
  [/^(curl|wget|Invoke-WebRequest|iwr|Invoke-RestMethod|irm|ssh|scp|sftp|rsync|nc|ncat|nmap|ping|telnet|ftp|tracert|traceroute|Test-NetConnection)\b/i, "rede"],
  [/^(docker|podman|kubectl|helm|terraform|tofu|systemctl|service|sc|pm2|nssm|vercel|netlify|fly|railway|heroku|az|aws|gcloud)\b/i, "deploy, container, cluster, nuvem ou serviço"],
  [/^(npm|pnpm|yarn|bun) (run )?(dev|start|serve|preview|deploy|release)\b/, "servidor ou deploy"],
  [/^(rm|del|erase|rmdir|rd|Remove-Item|shred)\b/i, "remoção de arquivo (use Edit ou Write; apagar só com pedido)"],
  [/(^|\s)(cat|type|more|less|head|tail|Get-Content|gc|bat|strings|xxd)\b.*(\.env(\.(?!example)[\w.-]+)?\b(?!\.example)|\.pem\b|\.key\b|\.pfx\b|\.p12\b|id_rsa|id_ed25519)/i, "leitura de arquivo de segredo"],
  [/^(env|printenv|set)$|^Get-ChildItem Env:/i, "variáveis de ambiente (podem ter segredo)"],
];
for (const [re, motivo] of regras) if (re.test(comando)) negar(motivo);
process.exit(0);
```

2. Acrescente ao frontmatter do agente instalado:

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "node ~/.claude/hooks/architect-bash-guard.mjs"
```

No Windows, se `~` não for expandido pelo shell do hook, use o caminho absoluto (`node C:/Users/<você>/.claude/hooks/architect-bash-guard.mjs`).

Efeito: commit, push, instalação e migração passam a ser pedidos ao usuário ou ao Claude principal. Testado com 63 comandos fictícios (20 liberados, 43 barrados), sem executar nenhum. Lista do que não pode não é fronteira: um programa chamado por outro nome escapa; as regras da seção 5.1 e a revisão do diff continuam valendo.

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
