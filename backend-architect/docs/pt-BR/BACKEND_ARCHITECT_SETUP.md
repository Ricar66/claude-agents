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
// O comando é dividido como o bash faria, sem expandir nada, e o que o bash expandiria é recusado;
// a checagem roda sobre os argumentos reais, com o nome do programa normalizado. Qualquer erro também nega.
// É uma lista do que não pode: barra o acidente e o comando injetado comum, não é fronteira de segurança
// (o agente edita código e roda testes, então pode executar código do projeto).
import { readFileSync } from "node:fs";

const negar = (motivo) => {
  process.stderr.write(`architect: comando bloqueado (${motivo}). Rode um comando por vez, sem cd; se precisar disso, peça ao usuário ou ao Claude principal.\n`);
  process.exit(2);
};

// Divide como o bash, sem expandir: aspas simples são literais; em aspas duplas, \ só escapa " \ $ e crase.
// Recusa operadores, $ e crase, barra invertida fora de aspas e caracteres de glob, chaves e parênteses fora de aspas.
function dividir(s) {
  if (/[\r\n]/.test(s)) negar("quebra de linha");
  const args = [];
  let atual = null;
  let i = 0;
  while (i < s.length) {
    const c = s[i];
    if (c === " " || c === "\t") { if (atual !== null) { args.push(atual); atual = null; } i++; continue; }
    if (atual === null) {
      atual = "";
      if (c === "#") negar("comentário do shell");
    }
    if (c === "'") {
      const fim = s.indexOf("'", i + 1);
      if (fim < 0) negar("aspas sem fechar");
      atual += s.slice(i + 1, fim);
      i = fim + 1;
      continue;
    }
    if (c === '"') {
      let j = i + 1;
      while (j < s.length && s[j] !== '"') {
        if (s[j] === "$" || s[j] === "`") negar("encadeamento, redirecionamento ou substituição");
        if (s[j] === "\\" && j + 1 < s.length && '"\\$`'.includes(s[j + 1])) { atual += s[j + 1]; j += 2; continue; }
        atual += s[j];
        j++;
      }
      if (j >= s.length) negar("aspas sem fechar");
      i = j + 1;
      continue;
    }
    if (";&|<>$`".includes(c)) negar("encadeamento, redirecionamento ou substituição");
    if (c === "\\") negar("barra invertida fora de aspas (ponha o caminho entre aspas)");
    if ("*?[]{}()!".includes(c)) negar("caractere especial do shell fora de aspas (ponha entre aspas)");
    atual += c;
    i++;
  }
  if (atual !== null) args.push(atual);
  return args;
}

// Nome do programa sem pasta, sem extensão do Windows e em minúsculas.
const nomeDe = (p) => p.replace(/^.*[\\/]/, "").toLowerCase().replace(/\.(exe|cmd|bat|ps1|sh)$/, "");
// Argumentos antes do "--" (depois dele, tudo é dado do programa).
const antesDoFim = (a) => (a.includes("--") ? a.slice(0, a.indexOf("--")) : a);
const palavras = (a) => antesDoFim(a).filter((x) => !x.startsWith("-"));

const PASTA = new Set(["cd", "pushd", "popd", "set-location", "sl", "chdir"]);
const SHELLS = new Set(["bash", "sh", "zsh", "dash", "ksh", "fish", "csh", "tcsh", "cmd", "powershell", "pwsh", "wsl", "busybox",
  "env", "sudo", "doas", "su", "runas", "xargs", "nohup", "timeout", "nice", "ionice", "stdbuf", "command", "builtin", "exec",
  "eval", "source", ".", "time", "watch", "start", "start-process", "invoke-expression", "iex", "invoke-command", "icm", "script",
  "parallel", "setsid", "chroot", "unshare", "flock", "strace"]);
const CODIGO_NA_LINHA = {
  node: /^(-e|--eval|-p|--print|-pe|-ep)$|^--(eval|print)=|^data:/,
  bun: /^(-e|--eval|-p|--print)$|^--(eval|print)=/,
  python: /^-[A-Za-z]*c/, python3: /^-[A-Za-z]*c/, py: /^-[A-Za-z]*c/, pythonw: /^-[A-Za-z]*c/,
  ruby: /^-[A-Za-z]*e/, perl: /^-[A-Za-z]*[eE]/, php: /^-r$/, lua: /^-e$/, rscript: /^-e$/, osascript: /^-e$/,
};
const GIT_PROIBIDOS = new Set(["push", "commit", "reset", "checkout", "switch", "restore", "stash", "clean", "rebase", "merge",
  "cherry-pick", "revert", "am", "apply", "fetch", "pull", "clone", "ls-remote", "submodule", "worktree", "update-ref",
  "filter-branch", "filter-repo", "notes", "replace", "gc", "prune", "send-email", "rm"]);
const INSTALA = new Set(["i", "install", "ci", "it", "cit", "install-test", "install-ci-test", "add", "remove", "rm", "uninstall",
  "un", "update", "up", "upgrade", "publish", "link", "ln", "unlink"]);
const BAIXA_E_RODA = new Set(["exec", "x", "dlx", "create", "init"]);
const SERVIDOR = new Set(["dev", "start", "serve", "preview", "deploy", "release"]);
const REDE = new Set(["curl", "wget", "invoke-webrequest", "iwr", "invoke-restmethod", "irm", "ssh", "scp", "sftp", "rsync", "nc",
  "ncat", "netcat", "socat", "nmap", "ping", "telnet", "ftp", "tracert", "traceroute", "test-netconnection", "tnc", "openssl",
  "nslookup", "dig", "host", "gh", "hub", "glab"]);
const NUVEM = new Set(["docker", "docker-compose", "podman", "kubectl", "helm", "terraform", "tofu", "terragrunt", "pulumi",
  "ansible", "ansible-playbook", "systemctl", "service", "sc", "pm2", "nssm", "vercel", "netlify", "fly", "flyctl", "railway",
  "heroku", "az", "aws", "gcloud", "gsutil", "firebase", "wrangler", "supabase"]);
const REMOVE = new Set(["rm", "del", "erase", "rmdir", "rd", "remove-item", "ri", "shred", "unlink"]);
const LEITORES = new Set(["cat", "type", "more", "less", "head", "tail", "get-content", "gc", "bat", "strings", "xxd", "od",
  "hexdump", "base64", "grep", "egrep", "fgrep", "rg", "findstr", "select-string", "sls", "awk", "gawk", "sed", "cut", "sort",
  "uniq", "diff", "cmp", "cp", "copy", "copy-item", "mv", "move", "move-item"]);
const MIGRACAO = /prisma (migrate (deploy|reset|dev|resolve)|db (push|execute|seed))|dotnet ef database|knex migrate|sequelize db:|typeorm migration:run|alembic (upgrade|downgrade)|flask db (upgrade|downgrade)|rails db:/;

function ehSegredo(arg) {
  const b = arg.replace(/^.*[\\/]/, "");
  if (b === ".env") return true;
  if (/^\.env\./i.test(b) && !/^\.env\.(example|sample|template|dist)$/i.test(b)) return true;
  if (/\.(pem|key|pfx|p12)$/i.test(b)) return true;
  if (/^id_(rsa|ed25519|ecdsa|dsa)$/i.test(b)) return true;
  if (/(^|[\\/])\.ssh([\\/]|$)/.test(arg) || /credenciais|credentials/i.test(arg) || /^(senha|password)[^\\/]*\.txt$/i.test(b)) return true;
  return false;
}

function verificarGit(a) {
  let i = 0;
  while (i < a.length && a[i].startsWith("-")) {
    const o = a[i];
    if (o === "-c" || o.startsWith("--config-env") || o.startsWith("--exec-path")) negar("configuração do git na linha (pode executar comando)");
    i += ["-C", "--git-dir", "--work-tree", "--namespace", "--super-prefix"].includes(o) ? 2 : 1;
  }
  const sub = a[i];
  const r = a.slice(i + 1);
  if (GIT_PROIBIDOS.has(sub)) negar("Git que altera histórico, arquivos ou remoto, ou usa rede");
  if (sub === "branch" && r.some((x) => /^(-d|-D|--delete|-m|-M|--move|-c|-C|--copy|-f|--force|-u|--set-upstream-to|--unset-upstream|--edit-description)/.test(x))) negar("Git que altera branch ou remoto");
  if (sub === "remote" && r.some((x) => ["add", "set-url", "remove", "rm", "rename", "set-head", "prune", "update", "set-branches"].includes(x))) negar("Git que altera branch ou remoto");
  if (sub === "tag" && !(r.length === 0 || r.includes("-l") || r.includes("--list"))) negar("Git que altera histórico ou arquivos");
  if (sub === "config" && !r.some((x) => /^(--get|--get-all|--get-regexp|--list|-l|--show-origin|--show-scope)$/.test(x))) negar("git config que grava");
}

// npx, bunx, npm exec, pnpm exec: nega pacote baixado na hora e confere o comando de dentro.
function verificarExecutor(a) {
  if (a.some((x) => /^(-y|--yes|-p|--package|-c|--call)$|^--(package|call)=/.test(x))) negar("execução de pacote baixado na hora ou de comando embutido");
  const i = a.findIndex((x) => !x.startsWith("-"));
  if (i < 0) return;
  if ((a[i] === "playwright" || a[i] === "cypress") && a[i + 1] === "install") negar("download de navegador");
  verificar(a.slice(i));
}

function verificar(argv) {
  if (!argv.length || !argv[0]) negar("vazio");
  if (argv[0].includes("=")) negar("variável de ambiente antes do comando");
  const prog = nomeDe(argv[0]);
  const a = argv.slice(1);
  const pal = palavras(a);
  const texto = [prog, ...a].join(" ");

  if (PASTA.has(prog)) negar("mudar de pasta (use git -C <pasta> ou o argumento de pasta da ferramenta)");
  if (prog === "command" && a.length === 2 && (a[0] === "-v" || a[0] === "-V")) return; // só diz se o programa existe
  if (SHELLS.has(prog)) negar("shell ou comando que embrulha outro (rode o comando direto)");
  if (prog === "printenv" || ((prog === "set" || prog === "export" || prog === "declare" || prog === "typeset") && (a.length === 0 || a.includes("-p")))) negar("variáveis de ambiente (podem ter segredo)");
  if (["get-childitem", "gci", "dir", "ls", "get-item", "gi"].includes(prog) && a.some((x) => /^env:/i.test(x))) negar("variáveis de ambiente (podem ter segredo)");
  if (CODIGO_NA_LINHA[prog] && antesDoFim(a).some((x) => CODIGO_NA_LINHA[prog].test(x))) negar("código na linha de comando (use um arquivo do projeto ou peça ao usuário)");
  if (prog === "deno" && (pal[0] === "eval" || a.some((x) => /^(https?|data|npm|jsr):/i.test(x)))) negar("código na linha de comando ou baixado na hora");

  if (["npx", "pnpx", "bunx"].includes(prog)) return verificarExecutor(a);
  if (["uvx", "pipx"].includes(prog)) negar("execução de pacote baixado na hora");
  if (["npm", "pnpm", "yarn", "bun"].includes(prog)) {
    const p = a.findIndex((x) => !x.startsWith("-"));
    if (p >= 0 && (a[p] === "exec" || a[p] === "x")) return verificarExecutor(a.slice(p + 1));
    if (prog === "yarn" && pal.length === 0 && !a.some((x) => /^(-v|--version|-h|--help)$/.test(x))) negar("instalação ou publicação de pacote");
    if (pal.some((x) => INSTALA.has(x)) || (pal.includes("audit") && pal.includes("fix"))) negar("instalação ou publicação de pacote");
    if (pal[0] === "version" && pal.length > 1) negar("npm version cria commit e tag");
    if (pal.some((x) => BAIXA_E_RODA.has(x))) negar("execução de pacote baixado na hora");
    if (pal.some((x) => SERVIDOR.has(x))) negar("servidor ou deploy");
  }
  if (["pip", "pip3", "uv", "poetry", "pipenv", "conda"].includes(prog) && pal.some((x) => ["install", "add", "remove", "uninstall", "update", "sync", "lock", "upgrade"].includes(x))) negar("instalação de pacote Python");
  if (["python", "python3", "py"].includes(prog)) {
    const m = a.findIndex((x) => /^-[A-Za-z]*m$/.test(x));
    if (m >= 0 && ["pip", "ensurepip"].includes(a[m + 1])) negar("instalação de pacote Python");
  }
  if (prog === "dotnet" && ["add", "remove", "restore", "tool", "workload", "nuget"].includes(pal[0])) negar("pacote .NET");
  if (prog === "go" && pal[0] === "run" && pal.some((x) => x.includes("@"))) negar("execução de pacote baixado na hora");
  if ((prog === "cargo" && ["add", "install", "publish", "update"].includes(pal[0])) || (prog === "go" && ["get", "install"].includes(pal[0]))
    || (prog === "gem" && pal[0] === "install") || (prog === "composer" && ["require", "install", "update"].includes(pal[0]))) negar("instalação de pacote");
  if (MIGRACAO.test(texto)) negar("migração ou alteração de banco");
  if (REDE.has(prog)) negar("rede");
  if (NUVEM.has(prog)) negar("deploy, container, cluster, nuvem ou serviço");
  if (REMOVE.has(prog)) negar("remoção de arquivo (use Edit ou Write; apagar só com pedido)");
  if (prog === "find" && a.some((x) => /^-(delete|exec|execdir|ok|okdir|fprint|fprint0|fprintf|fls)$/.test(x))) negar("find que apaga, executa ou grava");
  if (LEITORES.has(prog) && a.some(ehSegredo)) negar("leitura de arquivo de segredo");
  if (prog === "git") verificarGit(a);
}

try {
  const entrada = JSON.parse(readFileSync(0, "utf8"));
  const bruto = entrada?.tool_input?.command;
  if (typeof bruto !== "string") negar("entrada sem comando em texto");
  const comando = bruto.trim();
  if (!comando) negar("vazio");
  verificar(dividir(comando));
  process.exit(0);
} catch (e) {
  negar(`erro ao analisar o comando: ${e?.message ?? e}`);
}
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

Efeito: commit, push, instalação e migração passam a ser pedidos ao usuário ou ao Claude principal. O script divide o comando como o bash faria e recusa o que o bash expandiria, então aspas, caminho do programa, opção global do git e código na linha (`node -e`, `python -c`) não servem de desvio; qualquer erro dentro dele nega o comando. Testado com 148 comandos fictícios (41 liberados, 107 barrados) e 6 entradas inválidas (todas negadas), sem executar nenhum. Lista do que não pode não é fronteira: um programa chamado por outro nome escapa; as regras da seção 5.1 e a revisão do diff continuam valendo.

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
