# Security Sentinel — instalação, permissões e operação

## 1. O que vem no pacote

```text
security-sentinel/
├── agent/
│   ├── pt-BR/security-sentinel.md     ← agente em português
│   └── en/security-sentinel.md        ← mesmo agente em inglês
├── org-context-example/
│   └── SKILL.md                       ← exemplo fictício de contexto organizacional
└── docs/
    ├── pt-BR/                          ← guia de instalação, playbook, modelo de relatório, avaliações
    └── en/
```

As duas versões do agente têm o mesmo `name` (`security-sentinel`). **Instale só uma.**

## 2. Requisitos

- Claude Code com suporte a subagentes em arquivo (`~/.claude/agents/` ou `.claude/agents/`). Este pacote foi escrito e conferido no Claude Code **2.1.221** contra a documentação oficial de subagentes (`code.claude.com/docs/en/sub-agents`), consultada em 12/11/2026.
- Campos de frontmatter usados: `name`, `description`, `tools`, `model`, `effort`, `maxTurns`, `omitClaudeMd`, `color`, `skills`. Todos estão na lista oficial; campos que o Claude Code não reconhece são ignorados sem aviso.
- `omitClaudeMd: true` impede que o `CLAUDE.md` do projeto auditado entre no agente como instrução (um repositório hostil poderia usá-lo para manipular a auditoria). A documentação indica esse campo a partir do Claude Code **2.1.271**; em versão anterior ele é ignorado sem erro e essa proteção não existe. Recomendado: `claude update`.
- `model: opus` exige acesso a um modelo Opus. Sem ele, troque por `inherit` ou `sonnet`.
- `effort: high`: os níveis disponíveis dependem do modelo.

## 3. Instalação

### 3.1 Para todos os projetos (recomendado)

Windows (PowerShell):

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\agents" | Out-Null
Copy-Item .\security-sentinel\agent\pt-BR\security-sentinel.md "$env:USERPROFILE\.claude\agents\security-sentinel.md"
```

Linux e macOS:

```bash
mkdir -p ~/.claude/agents
cp security-sentinel/agent/pt-BR/security-sentinel.md ~/.claude/agents/security-sentinel.md
```

Troque `pt-BR` por `en` para a versão em inglês.

### 3.2 Só para um projeto

Copie o arquivo para `<projeto>/.claude/agents/security-sentinel.md`. Precedência oficial quando o mesmo nome existe em mais de um lugar: configurações gerenciadas > flag `--agents` > `.claude/agents/` do projeto > `~/.claude/agents/` > plugins.

### 3.3 Conferir

Abra uma sessão nova e peça: `Use o agente security-sentinel para triage desta pasta, somente leitura.` Se o Claude não delegar, rode `claude --debug` e procure o nome do arquivo: arquivos com YAML inválido, sem `description` ou com `name` inválido são pulados e só aparecem no log de debug.

Atualizou o arquivo no repositório? Copie de novo. A cópia instalada não acompanha o repositório sozinha. E abra uma **sessão nova** do Claude Code: uma sessão já aberta pode continuar usando a versão do agente que carregou antes (visto nos testes de 12/11/2026).

## 4. Contexto organizacional (opcional e local)

O agente pré-carrega a skill `security-sentinel-org-context` **se ela existir**. Se não existir, o Claude Code pula e registra um aviso no log de debug; o agente roda normalmente e informa "sem contexto organizacional".

Use para dizer ao agente o que só a sua organização sabe: inventário de sistemas, onde rodam, fronteiras de rede, contas de serviço, riscos já conhecidos, decisões de risco aceito, regras internas e o que está autorizado por padrão.

Como criar:

1. Pasta `~/.claude/skills/security-sentinel-org-context/` (ou `.claude/skills/…` do projeto).
2. Arquivo `SKILL.md` com frontmatter `name: security-sentinel-org-context` e uma `description`. **Não** use `disable-model-invocation: true`: esse campo impede o pré-carregamento.
3. Conteúdo como dado, não como ordem. Modelo completo e fictício: `security-sentinel/org-context-example/SKILL.md`.

Regras:

- **Nunca** coloque senha, token, chave ou valor de segredo. Escreva onde a credencial está e qual o risco.
- O arquivo é interno. Não versione no repositório público. Se quiser histórico, use um repositório privado à parte ou mantenha só na máquina.
- Para manter o original em outra pasta sem duplicar, use um link: no Windows, `New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\security-sentinel-org-context" -Target "<pasta-original>"`; no Linux e macOS, `ln -s <pasta-original> ~/.claude/skills/security-sentinel-org-context`.
- A skill também fica visível para o Claude principal, que pode carregá-la quando achar pertinente. Isso é esperado: ela mora na máquina de quem já tem acesso a essas informações.

## 5. Permissões e endurecimento

O que o pacote já faz:

- `tools: Read, Grep, Glob, Bash` é uma **lista fechada**: o agente não recebe Write, Edit, WebFetch, WebSearch, ferramentas de MCP nem a ferramenta de skills.
- Sem `memory`: ligar memória dá Read, Write e Edit ao agente automaticamente, o que quebraria o "somente leitura".
- `omitClaudeMd: true` (Claude Code 2.1.271 ou mais novo): o `CLAUDE.md` do projeto auditado não vira instrução do auditor. Arquivos de política gerenciada continuam valendo.
- Sem `permissionMode`: a documentação diz que esse campo é ignorado quando a sessão principal está em `auto`, `acceptEdits` ou `bypassPermissions`. Não é uma garantia.

O que o pacote **não** faz sozinho: impedir que o Bash escreva, instale ou acesse a rede. O texto do agente proíbe, mas instrução não é barreira. Para ambiente de risco, ative as camadas abaixo (nenhuma vem ativada).

### 5.1 Regras de negação no `settings.json`

As regras `permissions.deny` valem para a conversa principal **e** para os subagentes, em todos os modos. Exemplo para `~/.claude/settings.json` (junte ao que já existir):

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
      "Read(~/.ssh/**)",
      "Bash(curl *)",
      "Bash(wget *)",
      "Bash(nmap *)",
      "Bash(ssh *)",
      "Bash(scp *)",
      "Bash(git push *)"
    ]
  }
}
```

Os padrões com `./` valem a partir da pasta onde o Claude Code foi aberto, inclusive subpastas (monorepo). Para valer em qualquer pasta, no `~/.claude/settings.json`, troque `./` por `//` (ex.: `Read(//**/.env)`); `**/.env` sem prefixo também é relativo à pasta atual. O `.env.example` fica de fora de propósito: o agente precisa ler os nomes das variáveis.

Limites (da própria documentação): regra de Bash cobre a forma usual do comando e **não** é fronteira de segurança em volta do programa (`git -C . push` escapa de `git push *`). Regras de Read bloqueiam a ferramenta Read, não um `cat` pelo Bash. As regras afetam também o seu uso normal do Claude; teste antes.

### 5.2 Hook que restringe o Bash do agente

Hooks podem ficar no frontmatter do subagente e só valem enquanto ele roda. Hooks de agentes em `~/.claude/agents/` não dependem do diálogo de confiança da pasta; os de `.claude/agents/` do projeto, sim. Hook `PreToolUse` não passa por cima de regra de negação.

**Proposta** (crie e ative só se quiser essa camada):

1. Salve o script em `~/.claude/hooks/sentinel-bash-guard.mjs`:

```js
// Hook PreToolUse (Bash) do security-sentinel: permite só comandos de leitura conhecidos; o resto é negado (exit 2).
// O comando é dividido como o bash faria, sem expandir nada, e o que o bash expandiria é recusado;
// a checagem roda sobre os argumentos reais. Qualquer erro inesperado também nega (falha fechada).
import { readFileSync } from "node:fs";

const negar = (motivo) => {
  process.stderr.write(`security-sentinel: comando bloqueado (${motivo}). Peça ao usuário para rodar, se for necessário.\n`);
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
        if (s[j] === "$" || s[j] === "`") negar("expansão do shell ($ ou crase)");
        if (s[j] === "\\" && j + 1 < s.length && '"\\$`'.includes(s[j + 1])) { atual += s[j + 1]; j += 2; continue; }
        atual += s[j];
        j++;
      }
      if (j >= s.length) negar("aspas sem fechar");
      i = j + 1;
      continue;
    }
    if (";&|<>".includes(c)) negar("encadeamento ou redirecionamento");
    if (c === "$" || c === "`") negar("expansão do shell ($ ou crase)");
    if (c === "\\") negar("barra invertida fora de aspas (ponha o caminho entre aspas)");
    if ("*?[]{}()!".includes(c)) negar("caractere especial do shell fora de aspas (ponha entre aspas)");
    atual += c;
    i++;
  }
  if (atual !== null) args.push(atual);
  return args;
}

// Nega opção longa perigosa, inclusive abreviada (o git aceita prefixo único). Nome exato de outra opção vence
// a abreviação (--text não é --textconv). Para no "--".
const EXATAS_SEGURAS = new Set(["--text"]);
function verificarOpcoes(args, perigosas) {
  for (const a of args) {
    if (a === "--") return;
    if (!a.startsWith("--")) continue;
    const nome = a.split("=")[0];
    if (nome.length > 2 && !EXATAS_SEGURAS.has(nome) && perigosas.some((p) => p.startsWith(nome))) negar("opção que executa ou grava");
  }
}

const GIT_LEITURA = new Set(["status", "diff", "log", "show", "blame", "ls-files", "rev-parse", "merge-base"]);
const GIT_PERIGOSAS = ["--output", "--exec", "--upload-pack", "--receive-pack", "--config-env", "--ext-diff", "--textconv", "--open-files-in-pager", "--no-index", "--untracked", "--no-exclude-standard"];
const SCANNERS = new Set(["semgrep", "trivy", "osv-scanner", "gitleaks", "checkov", "codeql", "syft"]);

// Arquivo de segredo. Normaliza como o Windows (maiúsculas, ponto ou espaço no fim, :fluxo alternativo)
// e olha também o caminho de <revisão>:<caminho> e de --opcao=<caminho>.
function ehSegredo(arg) {
  const m = String(arg).toLowerCase();
  if (/credenciais|credentials/.test(m) || /(^|[\\/])\.ssh([\\/]|$)/.test(m)) return true;
  const base = (x) => x.replace(/^.*[\\/]/, "");
  const candidatos = [base(m).split(":")[0]];
  if (m.includes(":")) candidatos.push(base(m.slice(m.indexOf(":") + 1)));
  if (m.includes(":")) candidatos.push(base(m.slice(m.lastIndexOf(":") + 1)));
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

// Subcomandos do git que podem mostrar conteúdo de arquivo. Com eles, segredo, curinga e pathspec especial são
// negados qualquer que seja a opção (falha fechada: opções de letra agrupadas, como -3p, escapariam de uma lista).
const MOSTRA_CONTEUDO = new Set(["show", "diff", "log", "blame", "cat-file", "grep"]);
// Curinga ou pathspec especial do git (':(glob)', ':!') escolhe arquivos sem dar o nome; em comando que mostra
// conteúdo, isso abriria um segredo sem que o nome .env apareça.
// Excluir um segredo (':(exclude).env', ':!.env') é o uso seguro e passa.
const pathspecEsperto = (x) => {
  if (x.startsWith("-")) return false;
  const exclui = /^:(\(exclude\)|!|\^)(.+)$/.exec(x);
  if (exclui) return !ehSegredo(exclui[2]);
  return /[*?[]/.test(x) || /^:[(/]/.test(x);
};
function validar(args) {
  const [prog, ...a] = args;
  if (prog === "git") {
    let i = 0;
    if (a[0] === "-C") { if (a.length < 3) negar("git -C sem pasta ou sem subcomando"); i = 2; }
    const sub = a[i];
    const resto = a.slice(i + 1);
    const ok = GIT_LEITURA.has(sub)
      || (sub === "branch" && resto[0] === "--list")
      || (sub === "remote" && resto.length === 1 && resto[0] === "-v");
    if (!ok) negar("fora da lista de leitura");
    verificarOpcoes(resto, GIT_PERIGOSAS);
    if (MOSTRA_CONTEUDO.has(sub) && resto.some(ehSegredo)) negar("leitura de arquivo de segredo");
    if (MOSTRA_CONTEUDO.has(sub) && resto.some(pathspecEsperto)) negar("curinga ou pathspec especial em comando que mostra conteúdo");
    return;
  }
  if (prog === "ls") {
    if (a.every((x) => /^-[a-zA-Z1]+$/.test(x) || !x.startsWith("-"))) return;
    negar("opção de ls fora da lista");
  }
  if (prog === "wc") {
    const resto = /^-[lwcm]$/.test(a[0] ?? "") ? a.slice(1) : a;
    if (resto.length && resto.every((x) => !x.startsWith("-"))) return;
    negar("uso de wc fora da lista");
  }
  if (SCANNERS.has(prog) && a.length === 1 && a[0] === "--version") return;
  negar("fora da lista de leitura");
}

try {
  const entrada = JSON.parse(readFileSync(0, "utf8"));
  const bruto = entrada?.tool_input?.command;
  if (typeof bruto !== "string") negar("entrada sem comando em texto");
  const comando = bruto.trim();
  if (!comando) negar("vazio");
  const args = dividir(comando);
  if (!args.length) negar("vazio");
  validar(args);
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
          command: "node ~/.claude/hooks/sentinel-bash-guard.mjs"
```

No Windows, se `~` não for expandido pelo shell do hook, use o caminho absoluto (`node C:/Users/<você>/.claude/hooks/sentinel-bash-guard.mjs`).

Efeito: o agente perde a capacidade de rodar scanners (só `--version` passa). O script divide o comando como o bash faria e recusa o que o bash expandiria (`$`, crase, glob e barra invertida fora de aspas); a checagem roda sobre os argumentos reais, inclusive opções abreviadas, e qualquer erro dentro dele nega o comando. Limite: um segredo já versionado aparece em qualquer `git show <commit>` ou `git log -p` que o altere, porque o agente precisa ler diffs para trabalhar; o hook barra só o pedido pelo nome, com curinga ou com pathspec especial. A defesa é não versionar segredo (o agente aponta quando acha) e a regra de nunca reproduzir valor. Quando quiser um scanner, rode você mesmo, ou peça ao Claude principal, com o comando que o agente sugerir, e entregue a saída ao agente.

### 5.3 Isolamento para repositório não confiável

Para auditar código de terceiros ou suspeito, rode o Claude Code numa VM ou container descartável, sem credenciais montadas e com saída de rede restrita. Instrução no prompt não substitui isolamento.

## 6. Como invocar

Frases que funcionam (o agente identifica o modo pela palavra-chave):

- `Use o agente security-sentinel para executar deep-audit deste repositório, somente leitura; não faça testes externos.`
- `Use o agente security-sentinel para diff-review das alterações da branch atual.`
- `Use o agente security-sentinel para network-review das configurações locais de proxy, firewall e Docker, sem aplicar mudanças.`
- `Use o agente security-sentinel para ai-agent-review da integração de IA, MCP e webhooks.`
- `Use o agente security-sentinel para verify-fix dos achados SS-001 e SS-002.`
- `Use o agente security-sentinel para threat-model do fluxo de pagamento.`
- `Use o agente security-sentinel para release-gate da versão 2.3, considerando os achados abertos.`

Dicas:

- Diga o escopo: pasta, branch, commit, ambiente. Para `verify-fix`, cole a tabela de achados anterior para os IDs se manterem.
- Para teste ativo, escreva o escopo: alvo, janela, ambiente, quem autorizou. Sem isso, o agente recusa a parte ativa.
- O agente responde com texto. Para guardar o relatório, peça ao Claude principal que salve; para projetos internos, salve fora do repositório público.

## 7. Ferramentas de varredura (opcionais)

O agente funciona sem nenhuma. Quando uma estiver instalada e você autorizar, ele faz o preflight (versão, rede, execução de código do projeto, segredos na saída, escopo) antes de rodar. A tabela de ferramentas, com o que cada uma envia pela rede e como rodar offline, fica no playbook, seção 6.

O pacote não instala nada.

## 8. Validação feita neste pacote

O que foi conferido e como está registrado em `SECURITY_SENTINEL_EVALS.md`, seção 4, e no histórico do repositório. Para repetir:

- **Frontmatter:** parse com um parser YAML (ex.: `js-yaml`), conferindo `name`, `description` e a lista de `tools`.
- **Schema do relatório:** validação do schema e do exemplo de `SECURITY_SENTINEL_REPORT_TEMPLATE.md` com um validador JSON Schema draft-07 (ex.: `ajv`).
- **`claude plugin validate <pasta>`:** valida pastas `agents/` soltas a partir do Claude Code **2.1.233**. Em versão anterior, use o parser YAML.
- **Comportamento:** cenários de `SECURITY_SENTINEL_EVALS.md`.

## 9. Limitações conhecidas

- A proibição de escrita e de rede pelo Bash depende do texto do agente, das regras de negação e do hook. Sem 5.1 e 5.2, é só instrução.
- Sem WebFetch, o agente não consulta NVD, OSV, KEV nem EPSS. CVE e pontuação ficam "não verificados" até alguém consultar e entregar o dado.
- Análise estática não prova exploração nem ausência dela. Rede e produção são avaliadas só pelo que estiver em arquivo.
- Um modelo pode errar. As avaliações reduzem, não eliminam, falso positivo e falso negativo.
- Prompt injection é mitigada, não eliminada: por isso as camadas da seção 5.

## 10. Manutenção

- **Normas:** revise as versões citadas no playbook a cada 6 meses e quando sair edição nova (OWASP, NIST, CWE Top 25, CVSS). Registre a data da consulta.
- **Idiomas:** `pt-BR` e `en` andam juntos. Mudou um, mude o outro no mesmo commit.
- **Mudanças no agente:** rode de novo pelo menos EV-01, EV-02, EV-08, EV-13, EV-15 e EV-16 e registre.
- **Claude Code:** novas versões trazem campos de frontmatter e comandos de validação; confira a documentação de subagentes antes de adotar.

## 11. Como aprovar uma remediação

O Sentinel não corrige. O fluxo sugerido:

1. O Sentinel entrega o relatório com IDs.
2. Você escolhe os IDs e a onda: `Corrija SS-001 e SS-004 conforme o relatório, numa branch nova, com testes de regressão.`
3. O Claude principal (ou um agente de remediação separado, de preferência num worktree) faz a mudança e mostra o diff, os testes e o plano de rollback.
4. Você revisa e aprova o diff.
5. `Use o agente security-sentinel para verify-fix de SS-001 e SS-004 na branch <nome>.`
6. A decisão de merge e de release é sua. Exceção aceita precisa de dono e prazo.

## 12. Integração com CI (proposta, não ativada)

Nada aqui é criado automaticamente. Se a equipe quiser varredura contínua:

| Peça | Arquivo-alvo | Precisa de | Cuidado |
|---|---|---|---|
| SAST | `.github/workflows/codeql.yml` ou Semgrep | GitHub Advanced Security (CodeQL em repositório privado) ou Semgrep com regras locais | Semgrep com `--config auto` baixa regras e envia métricas; use regras versionadas e `--metrics=off` |
| Dependências | Dependabot (`.github/dependabot.yml`) ou OSV-Scanner | Nada além do GitHub, no caso do Dependabot | OSV-Scanner online envia nomes e versões de pacotes à API do OSV |
| Segredos | Push protection do GitHub ou Gitleaks no CI | Permissão de administrador do repositório | Saída com `--redact` |
| Imagens e IaC | Trivy, Checkov | Base de vulnerabilidades (download) | Fixar versão e checar o que é baixado |
| PR de fork | Política do repositório | — | Nunca `pull_request_target` com checkout do código do PR e segredos; `permissions` mínimas por job |

Antes de ativar: preserve os workflows existentes, escolha quem recebe os alertas, defina o que bloqueia merge e teste num branch.
