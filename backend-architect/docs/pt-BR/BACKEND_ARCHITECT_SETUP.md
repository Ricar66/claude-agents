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
// Git só com subcomandos de leitura; o resto é lista do que não pode: barra o acidente e o comando injetado
// comum, não é fronteira de segurança (o agente edita código e roda testes, então pode executar código do projeto).
import { readFileSync, existsSync } from "node:fs";
import { join, dirname } from "node:path";

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

// Nome do programa: sem pasta, sem extensão do Windows, em minúsculas e sem versão no fim (python3.12 vira python).
function nomeDe(p) {
  const n = p.replace(/^.*[\\/]/, "").toLowerCase().replace(/\.(exe|cmd|bat|ps1|sh)$/, "").replace(/\d+(\.\d+)*$/, "").replace(/[-_.]$/, "");
  return n === "nodejs" ? "node" : n;
}

// Arquivo de segredo. Normaliza como o Windows (maiúsculas, ponto ou espaço no fim, :fluxo alternativo)
// e olha também o caminho de <revisão>:<caminho> e de --opcao=<caminho>.
function ehSegredo(arg) {
  const m = String(arg).toLowerCase();
  if (/credenciais|credentials/.test(m) || /(^|[\\/])\.ssh([\\/]|$)/.test(m)) return true;
  const base = (x) => x.replace(/^.*[\\/]/, "");
  const candidatos = [base(m).split(":")[0]];
  if (m.includes(":")) candidatos.push(base(m.slice(m.indexOf(":") + 1)));
  if (m.includes("=")) candidatos.push(base(m.slice(m.lastIndexOf("=") + 1)).split(":")[0]);
  return candidatos.some((c0) => {
    const c = c0.replace(/[. ]+$/, "");
    return c === ".env"
      || (c.startsWith(".env.") && !/^\.env\.(example|sample|template|dist)$/.test(c))
      || /\.(pem|key|pfx|p12)$/.test(c)
      || /^id_(rsa|ed25519|ecdsa|dsa)$/.test(c)
      || /^(senha|password)[^\\/]*\.txt$/.test(c);
  });
}

// Palavras (argumentos que não são opção). Para no "--" só quando ele vem depois de um subcomando que roda
// script ou programa (o resto é argumento desse script); fora disso o "--" não esconde nada.
function palavrasAte(a, rodaCoisas) {
  const out = [];
  for (const x of a) {
    if (x === "--") { if (out.some((w) => rodaCoisas.has(w))) break; continue; }
    if (!x.startsWith("-")) out.push(x);
  }
  return out;
}

const PASTA = new Set(["cd", "pushd", "popd", "set-location", "sl", "chdir"]);
const SHELLS = new Set(["bash", "sh", "zsh", "dash", "ksh", "fish", "csh", "tcsh", "cmd", "powershell", "pwsh", "wsl", "busybox",
  "env", "sudo", "doas", "su", "runas", "xargs", "nohup", "timeout", "nice", "ionice", "stdbuf", "command", "builtin", "exec",
  "eval", "source", ".", "time", "watch", "start", "start-process", "invoke-expression", "iex", "invoke-command", "icm", "script",
  "parallel", "setsid", "chroot", "unshare", "flock", "strace", "cross-env", "dotenv", "concurrently"]);
const BAIXA_SEMPRE = new Set(["pnpx", "uvx", "pipx", "corepack"]);
const CODIGO_NA_LINHA = {
  node: /^(-e|--eval|-p|--print|-pe|-ep)$|^--(eval|print)=|^data:|^--(import|loader|experimental-loader|require)=(data|https?):/,
  bun: /^(-e|--eval|-p|--print)$|^--(eval|print)=/,
  python: /^-[A-Za-z]*c/, py: /^-[A-Za-z]*c/, pythonw: /^-[A-Za-z]*c/,
  ruby: /^-[A-Za-z]*e/, perl: /^-[A-Za-z]*[eE]/, php: /^-r$/, lua: /^-e$/, rscript: /^-e$/, osascript: /^-e$/,
};
const GIT_LEITURA = new Set(["status", "diff", "log", "show", "blame", "ls-files", "ls-tree", "rev-parse", "merge-base", "cat-file",
  "describe", "shortlog", "grep", "for-each-ref", "rev-list", "name-rev", "check-ignore", "check-attr", "count-objects", "version"]);
const GIT_PERIGOSAS = ["--output", "--exec", "--upload-pack", "--receive-pack", "--config-env", "--ext-diff", "--textconv",
  "--open-files-in-pager", "--filters"];
const EXATAS_SEGURAS = new Set(["--text"]);
// Apelidos do npm e equivalentes do pnpm, yarn e bun que instalam, removem, publicam ou mexem em conta e registro.
const PACOTE_PROIBIDO = new Set(["i", "in", "ins", "inst", "insta", "instal", "install", "isnt", "isnta", "isntal", "isntall", "add",
  "ci", "clean-install", "ic", "install-clean", "isntall-clean", "it", "install-test", "cit", "clean-install-test", "sit",
  "install-ci-test", "uninstall", "unlink", "remove", "rm", "r", "un", "update", "up", "upgrade", "udpate", "publish", "unpublish",
  "link", "ln", "dedupe", "ddp", "prune", "rebuild", "rb", "import", "dlx", "create", "init", "explore", "edit", "login", "logout",
  "adduser", "add-user", "token", "owner", "access", "deprecate", "dist-tag", "team", "org", "star", "unstar", "hook", "profile"]);
const VALOR_PM = new Set(["--prefix", "-C", "--dir", "--cwd", "--filter", "-F", "--workspace", "--userconfig", "--globalconfig",
  "--cache", "--registry", "--loglevel", "--reporter"]);
const RODA_SCRIPT = new Set(["run", "run-script", "rum", "urn", "test", "t", "tst", "exec", "x"]);
const SERVIDOR = new Set(["dev", "start", "restart", "stop", "serve", "preview", "deploy", "release"]);
const REDE = new Set(["curl", "wget", "invoke-webrequest", "iwr", "invoke-restmethod", "irm", "ssh", "scp", "sftp", "rsync", "nc",
  "ncat", "netcat", "socat", "nmap", "ping", "telnet", "ftp", "tracert", "traceroute", "test-netconnection", "tnc", "openssl",
  "nslookup", "dig", "host", "gh", "hub", "glab"]);
const NUVEM = new Set(["docker", "docker-compose", "podman", "kubectl", "helm", "terraform", "tofu", "terragrunt", "pulumi",
  "ansible", "ansible-playbook", "systemctl", "service", "sc", "pm2", "nssm", "vercel", "netlify", "fly", "flyctl", "railway",
  "heroku", "az", "aws", "gcloud", "gsutil", "firebase", "wrangler", "supabase"]);
const REMOVE = new Set(["rm", "del", "erase", "rmdir", "rd", "remove-item", "ri", "shred", "unlink"]);
const SO_LISTA = new Set(["ls", "dir", "get-childitem", "gci", "stat", "test", "which", "where", "command"]);
const MIGRACAO = /prisma (migrate (deploy|reset|dev|resolve)|db (push|execute|seed))|dotnet ef database|knex migrate|sequelize db:|typeorm migration:run|alembic (upgrade|downgrade)|flask db (upgrade|downgrade)|rails db:/;

let CWD = process.cwd();

// Nega opção longa perigosa, inclusive abreviada (o git aceita prefixo único). Para no "--".
function verificarOpcoes(args, perigosas) {
  for (const a of args) {
    if (a === "--") return;
    if (!a.startsWith("--")) continue;
    const nome = a.split("=")[0];
    if (nome.length > 2 && !EXATAS_SEGURAS.has(nome) && perigosas.some((p) => p.startsWith(nome))) negar("opção que executa ou grava");
  }
}

// O subcomando do git mostra conteúdo de arquivo (e não só nomes ou estatística)?
function mostraConteudo(sub, r) {
  if (["blame", "cat-file", "grep"].includes(sub)) return true;
  if (sub === "show" || sub === "diff") return !r.some((x) => /^(--stat|--name-only|--name-status|--numstat|--shortstat|--summary|--dirstat|--no-patch|-s$|--quiet|--exit-code)/.test(x));
  if (sub === "log") return r.some((x) => /^(-p|-u|--patch|--cc|--word-diff|-U\d*$|-L|--full-diff)/.test(x));
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
  if (sub === undefined) return;
  const pos = r.filter((x) => !x.startsWith("-"));
  let ok = GIT_LEITURA.has(sub);
  if (sub === "branch") ok = !r.some((x) => /^(-d|-D|--delete|-m|-M|--move|-c|-C|--copy|-f|--force|-u|--set-upstream-to|--unset-upstream|--edit-description)/.test(x))
    && (pos.length === 0 || r.some((x) => /^(--list|-l|-a|--all|-r|--remotes|-v|-vv|--contains|--no-contains|--merged|--no-merged|--points-at)/.test(x)));
  if (sub === "tag") ok = !r.some((x) => /^(-a|-s|-u|-f|-d|-m|-F|--annotate|--sign|--local-user|--force|--delete|--message|--file)/.test(x))
    && (pos.length === 0 || r.some((x) => /^(--list|-l|--contains|--no-contains|--merged|--no-merged|--points-at|-n)/.test(x)));
  if (sub === "remote") ok = r.length === 0 || (r.length === 1 && r[0] === "-v") || r[0] === "get-url";
  if (sub === "config") ok = r.some((x) => /^(--get|--get-all|--get-regexp|--list|-l|--show-origin|--show-scope)$/.test(x));
  if (sub === "stash" || sub === "worktree") ok = r[0] === "list";
  if (sub === "reflog") ok = r[0] === undefined || r[0] === "show" || (r[0].startsWith("-") && !r.includes("expire") && !r.includes("delete"));
  if (!ok) negar("Git fora da lista de leitura (commit, push, checkout e afins ficam com o usuário)");
  verificarOpcoes(r, GIT_PERIGOSAS);
  if (sub === "grep" && r.some((x) => /^-[A-Za-z]*O/.test(x))) negar("opção que executa ou grava");
  if (mostraConteudo(sub, r) && r.some(ehSegredo)) negar("leitura de arquivo de segredo");
}

// O pacote está no node_modules/.bin desta pasta ou de uma pasta acima?
function binLocal(pacote) {
  const bin = pacote.replace(/^@[^/]+\//, "");
  if (!/^[\w.-]+$/.test(bin)) return false;
  let d = CWD;
  for (let k = 0; k < 40; k++) {
    for (const ext of ["", ".cmd", ".exe", ".ps1"]) if (existsSync(join(d, "node_modules", ".bin", bin + ext))) return true;
    const pai = dirname(d);
    if (pai === d) break;
    d = pai;
  }
  return false;
}

// npx, bunx, npm exec, bun x (podem baixar) e pnpm exec, yarn exec (só local): confere o pacote e o comando de dentro.
function verificarExecutor(a, soLocal) {
  if (a.some((x) => /^(-y|--yes|-p|--package|-c|--call)$|^--(package|call)=/.test(x))) negar("execução de pacote baixado na hora ou de comando embutido");
  const semInstalar = a.some((x) => x === "--no" || x === "--no-install");
  const i = a.findIndex((x) => !x.startsWith("-"));
  if (i < 0) return;
  const pacote = a[i];
  if (/^.+@/.test(pacote)) negar("pacote com versão, baixado na hora");
  if ((pacote === "playwright" || pacote === "cypress") && a[i + 1] === "install") negar("download de navegador");
  if (!soLocal && !semInstalar && !binLocal(pacote)) negar("pacote fora do node_modules do projeto seria baixado na hora (use npx --no ou peça ao usuário)");
  verificar(a.slice(i));
}

// pip, uv, poetry e afins: "run" executa o que vem depois; o resto é conferido palavra por palavra.
function verificarSufixos(a) {
  for (let k = 0; k < a.length; k++) if (!a[k].startsWith("-")) verificar(a.slice(k));
}

function verificarGerenciador(prog, a) {
  let i = 0;
  while (i < a.length && a[i].startsWith("-")) i += VALOR_PM.has(a[i]) ? 2 : 1;
  const sub = a[i];
  if (sub === undefined) {
    if (prog === "yarn" && !a.some((x) => /^(-v|--version|-h|--help)$/.test(x))) negar("instalação ou publicação de pacote");
    return;
  }
  if (sub === "exec" || (sub === "x" && prog !== "yarn")) return verificarExecutor(a.slice(i + 1), prog === "pnpm" || prog === "yarn");
  const pal = palavrasAte(a, RODA_SCRIPT);
  if (pal.some((x) => PACOTE_PROIBIDO.has(x))) negar("instalação, publicação de pacote ou conta no registro");
  if (pal.includes("audit") && pal.includes("fix")) negar("instalação ou publicação de pacote");
  if (pal.includes("version") && pal.length > pal.indexOf("version") + 1) negar("npm version cria commit e tag");
  if (pal.includes("config") && pal.some((x) => ["set", "delete", "edit"].includes(x))) negar("configuração do gerenciador de pacotes");
  if (prog === "yarn" && pal.includes("set")) negar("instalação ou publicação de pacote");
  const script = ["run", "run-script", "rum", "urn"].includes(sub) ? a.slice(i + 1).find((x) => !x.startsWith("-")) : sub;
  if (SERVIDOR.has(script)) negar("servidor ou deploy");
}

function verificar(argv) {
  if (!argv.length || !argv[0]) negar("vazio");
  if (argv[0].includes("=")) negar("variável de ambiente antes do comando");
  const prog = nomeDe(argv[0]);
  const a = argv.slice(1);
  const texto = [prog, ...a].join(" ");

  if (PASTA.has(prog)) negar("mudar de pasta (use git -C <pasta> ou o argumento de pasta da ferramenta)");
  if (prog === "command" && a.length === 2 && (a[0] === "-v" || a[0] === "-V")) return; // só diz se o programa existe
  if (SHELLS.has(prog)) negar("shell ou comando que embrulha outro (rode o comando direto)");
  if (BAIXA_SEMPRE.has(prog)) negar("execução de pacote baixado na hora");
  if (prog === "printenv" || ((prog === "set" || prog === "export" || prog === "declare" || prog === "typeset") && (a.length === 0 || a.includes("-p")))) negar("variáveis de ambiente (podem ter segredo)");
  if (["get-childitem", "gci", "dir", "ls", "get-item", "gi"].includes(prog) && a.some((x) => /^env:/i.test(x))) negar("variáveis de ambiente (podem ter segredo)");
  if (CODIGO_NA_LINHA[prog] && a.some((x) => CODIGO_NA_LINHA[prog].test(x))) negar("código na linha de comando (use um arquivo do projeto ou peça ao usuário)");
  if (prog === "deno" && (a.includes("eval") || a.some((x) => /^(https?|data|npm|jsr):/i.test(x)))) negar("código na linha de comando ou baixado na hora");
  if (prog === "git") return verificarGit(a);
  if (!SO_LISTA.has(prog) && a.some(ehSegredo)) negar("leitura de arquivo de segredo");

  if (prog === "npx" || prog === "bunx") return verificarExecutor(a, false);
  if (["npm", "pnpm", "yarn", "bun"].includes(prog)) return verificarGerenciador(prog, a);
  if (prog === "python" || prog === "py" || prog === "pythonw") {
    for (let k = 0; k < a.length; k++) {
      const x = a[k];
      if (!x.startsWith("-") || x === "-") break;
      if (/^-[A-Za-z]*[WX]$/.test(x)) { k++; continue; }
      const m = /^-[A-Za-z]*m(.*)$/.exec(x);
      if (m) { if (["pip", "ensurepip"].includes(m[1] || a[k + 1])) negar("instalação de pacote Python"); break; }
    }
  }
  if (["pip", "uv", "poetry", "pipenv", "conda", "pdm", "hatch", "rye"].includes(prog)) {
    const pal = palavrasAte(a, new Set(["run"]));
    if (pal[0] === "run") {
      if (a.some((x) => /^--with/.test(x))) negar("execução de pacote baixado na hora");
      verificarSufixos(a.slice(a.indexOf("run") + 1));
    } else if (pal.some((x) => ["install", "add", "remove", "uninstall", "update", "sync", "lock", "upgrade"].includes(x))) negar("instalação de pacote Python");
  }
  if (prog === "dotnet") {
    const pal = palavrasAte(a, new Set(["run", "test", "exec"]));
    if (pal[0] !== "ef" && pal.some((x) => ["add", "remove", "restore", "tool", "workload", "nuget"].includes(x))) negar("pacote .NET");
  }
  if (prog === "go" && palavrasAte(a, new Set()).some((x, k, p) => (["get", "install"].includes(x)) || (p[0] === "run" && x.includes("@")))) negar("instalação de pacote ou pacote baixado na hora");
  if (prog === "cargo" && palavrasAte(a, new Set(["run", "test"])).some((x) => ["add", "install", "publish", "update"].includes(x))) negar("instalação de pacote");
  if (prog === "gem" && palavrasAte(a, new Set(["exec"])).includes("install")) negar("instalação de pacote");
  if (prog === "composer" && palavrasAte(a, new Set(["run", "run-script", "exec"])).some((x) => ["require", "install", "update"].includes(x))) negar("instalação de pacote");
  if (MIGRACAO.test(texto)) negar("migração ou alteração de banco");
  if (REDE.has(prog)) negar("rede");
  if (NUVEM.has(prog)) negar("deploy, container, cluster, nuvem ou serviço");
  if (REMOVE.has(prog)) negar("remoção de arquivo (use Edit ou Write; apagar só com pedido)");
  if (prog === "find" && a.some((x) => /^-(delete|exec|execdir|ok|okdir|fprint|fprint0|fprintf|fls)$/.test(x))) negar("find que apaga, executa ou grava");
}

try {
  const entrada = JSON.parse(readFileSync(0, "utf8"));
  if (typeof entrada?.cwd === "string" && entrada.cwd) CWD = entrada.cwd;
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

Efeito: commit, push, instalação e migração passam a ser pedidos ao usuário ou ao Claude principal. O script divide o comando como o bash faria e recusa o que o bash expandiria, então aspas, caminho do programa, versão no nome (`python3.12`), opção global do git e código na linha (`node -e`, `python -c`) não servem de desvio. No git só passam subcomandos de leitura, e nenhum que mostre o conteúdo de um arquivo de segredo (`git show HEAD:.env`). O `npx` (e `npm exec`, `bunx`) só roda pacote que já está no `node_modules/.bin` do projeto, ou com `--no`: sem terminal, o npx baixaria o pacote sem perguntar. Qualquer erro dentro do script nega o comando. Testado com 220 comandos fictícios (58 liberados, 162 barrados) e 6 entradas inválidas (todas negadas), sem executar nenhum. Lista do que não pode não é fronteira: um programa chamado por outro nome escapa; as regras da seção 5.1 e a revisão do diff continuam valendo.

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
