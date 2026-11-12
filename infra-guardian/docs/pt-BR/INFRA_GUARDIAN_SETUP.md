# Infra Guardian — instalação, permissões e operação

## 1. O que vem no pacote

```text
infra-guardian/
├── agent/
│   ├── pt-BR/infra-guardian.md          ← agente com Bash (leitura local)
│   ├── pt-BR/infra-guardian-static.md   ← mesma coisa, sem Bash
│   ├── en/infra-guardian.md
│   └── en/infra-guardian-static.md
├── org-context-example/
│   ├── pt-BR/SKILL.md                   ← exemplo fictício de contexto organizacional
│   └── en/SKILL.md
└── docs/
    ├── pt-BR/                           ← guia de instalação, playbook, modelo de relatório, modelos de documentos, avaliações
    └── en/
```

As duas variantes têm o mesmo texto de instruções; só mudam `name`, `description` e `tools`. As versões `pt-BR` e `en` de cada variante têm o mesmo `name`: **instale um idioma só**.

| Variante | Ferramentas | Quando usar |
|---|---|---|
| `infra-guardian` | `Read`, `Grep`, `Glob`, `Bash` | Uso normal: pode rodar comandos locais de leitura (Git, validação de compose) |
| `infra-guardian-static` | `Read`, `Grep`, `Glob` | Quando nenhum comando pode ser executado: o isolamento é técnico, não só instrução |

Com `Bash` na lista, "somente leitura" depende do texto do agente e das camadas da seção 8. Sem `Bash`, o agente não tem como executar nada.

## 2. Requisitos

- Claude Code com subagentes em arquivo (`~/.claude/agents/` ou `.claude/agents/`). Escrito e conferido no Claude Code **2.1.221** contra a documentação oficial de subagentes, consultada em 12/11/2026.
- Campos de frontmatter usados: `name`, `description`, `tools`, `model`, `effort`, `maxTurns`, `omitClaudeMd`, `color`, `skills`. Todos estão na lista oficial; campo desconhecido é ignorado sem aviso.
- `omitClaudeMd: true` impede que o `CLAUDE.md` do projeto analisado entre como instrução (um repositório hostil poderia usá-lo para manipular o diagnóstico). A documentação indica o campo a partir do Claude Code **2.1.271**; em versão anterior ele é ignorado sem erro e essa proteção não existe. Recomendado: `claude update`.
- `model: opus` exige acesso a um modelo Opus; sem ele, troque por `inherit` ou `sonnet`. `effort: high`: os níveis disponíveis dependem do modelo.
- Nenhuma ferramenta de infraestrutura é obrigatória. Docker, Terraform, kubectl e validadores ajudam quando já estão instalados (playbook, seção 6).

## 3. Instalação

### 3.1 Para todos os projetos (recomendado)

Windows (PowerShell):

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\agents" | Out-Null
Copy-Item .\infra-guardian\agent\pt-BR\infra-guardian.md "$env:USERPROFILE\.claude\agents\"
Copy-Item .\infra-guardian\agent\pt-BR\infra-guardian-static.md "$env:USERPROFILE\.claude\agents\"
```

Linux e macOS:

```bash
mkdir -p ~/.claude/agents
cp infra-guardian/agent/pt-BR/infra-guardian.md infra-guardian/agent/pt-BR/infra-guardian-static.md ~/.claude/agents/
```

Troque `pt-BR` por `en` para a versão em inglês. A variante `static` é opcional.

### 3.2 Só para um projeto

Copie para `<projeto>/.claude/agents/`. Precedência oficial quando o mesmo nome existe em mais de um lugar: configurações gerenciadas > flag `--agents` > `.claude/agents/` do projeto > `~/.claude/agents/` > plugins.

### 3.3 Conferir

Abra uma **sessão nova** e peça: `Use o agente infra-guardian para discover desta pasta, somente leitura.` Se o Claude não delegar, rode `claude --debug` e procure o nome do arquivo: YAML inválido, `description` ausente ou `name` inválido fazem o arquivo ser pulado, com aviso só no log de debug.

Atualizou o arquivo no repositório? Copie de novo e abra uma sessão nova: a cópia instalada não acompanha o repositório, e uma sessão já aberta pode continuar com a versão anterior do agente.

## 4. Contexto organizacional (opcional e local)

Os dois agentes pré-carregam a skill `infra-guardian-org-context` **se ela existir**. Se não existir, o Claude Code pula e registra um aviso no log de debug; o agente roda e informa "sem contexto organizacional".

Use para dizer o que só a sua organização sabe: ambientes, servidores e o que roda em cada um, rede e links, backup, monitoramento, deploy, riscos operacionais conhecidos, decisões de risco aceito e regras internas (janela de mudança, quem executa).

1. Pasta `~/.claude/skills/infra-guardian-org-context/` (ou `.claude/skills/…` do projeto).
2. `SKILL.md` com frontmatter `name: infra-guardian-org-context` e uma `description`. **Não** use `disable-model-invocation: true`: skill com esse campo não é pré-carregada.
3. Conteúdo como dado, não como ordem. Modelo fictício: `infra-guardian/org-context-example/pt-BR/SKILL.md`.

Regras: nunca senha, token, chave ou valor de segredo; o arquivo é interno e não entra em repositório público; para manter o original em outra pasta, use um link (`New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\infra-guardian-org-context" -Target "<pasta-original>"` no Windows; `ln -s <pasta-original> ~/.claude/skills/infra-guardian-org-context` no Linux e macOS). Se você também usa o Security Sentinel, os dois contextos se sobrepõem em servidores e rede: ao mudar um, revise o outro.

## 5. Como invocar

O agente identifica o modo pela palavra-chave; pedido amplo vira `discover`.

- `Use o agente infra-guardian para discover deste repositório, somente leitura.`
- `Use o agente infra-guardian para docker-review do compose de produção.`
- `Use o agente infra-guardian para vps-audit; a saída de df, free e systemctl está em coleta/.`
- `Use o agente infra-guardian para backup-dr do banco principal, propondo RPO e RTO para aprovação.`
- `Use o agente infra-guardian para network-diagnose dos 504 no site; os logs do nginx estão em logs/.`
- `Use o agente infra-guardian para pipeline-review do workflow de deploy.`
- `Use o agente infra-guardian para release-readiness da versão 2.4.`
- `Use o agente infra-guardian para verify-change dos itens IG-001 e IG-003 na branch atual.`
- `Use o agente infra-guardian-static para architecture-review, sem executar nenhum comando.`

Dicas: diga o ambiente (desenvolvimento, homologação, produção); anexe saídas de comando já filtradas; para `verify-change`, cole os achados anteriores para os IDs se manterem. O agente responde com texto; para guardar o relatório, peça ao Claude principal que salve, fora do repositório público quando o projeto for interno.

## 6. Autorização

Resumo (matriz completa no playbook, seção 3.1):

| Ação | O agente |
|---|---|
| Ler arquivos e rodar comandos locais de leitura | Faz |
| Consulta remota de leitura (SSH, `kubectl get`, `terraform plan`, CLI de cloud, query) | Só com autorização na conversa (alvo, ambiente, comando); de preferência a pessoa roda |
| Qualquer mudança, em qualquer ambiente | Nunca faz: entrega o plano; quem executa é a pessoa ou o Claude principal |

Autorização para diagnosticar não é autorização para mudar, e autorização para uma mudança não se estende a outra.

## 7. Integração com o Security Sentinel

Os dois agentes são da mesma coleção e se complementam:

- O Infra Guardian cuida de disponibilidade, desempenho, capacidade, backup, deploy e custo. Quando encontra algo de segurança (porta de banco publicada, segredo em compose, serviço em conta pessoal, instrução maliciosa num README), registra o fato e lista o item na seção **"Para o Security Sentinel"** do relatório.
- Subagentes não chamam outros subagentes nesta coleção (nenhum dos dois tem a ferramenta `Agent`). Quem encaminha é você ou o Claude principal: `Use o agente security-sentinel para avaliar os itens "Para o Security Sentinel" do relatório do Infra Guardian.`
- Os dois compartilham contexto, não permissões. Nenhum aplica mudança.

## 8. Permissões e endurecimento

O que o pacote já faz:

- `tools` é **lista fechada**: o agente não recebe Write, Edit, WebFetch, WebSearch, MCP, `Agent` nem a ferramenta de skills.
- Sem `memory` (ligar memória dá Read, Write e Edit automaticamente).
- Sem `permissionMode`: a documentação diz que esse campo é ignorado quando a sessão principal está em `auto`, `acceptEdits` ou `bypassPermissions`. Não é garantia, e o pacote não depende dele.
- `omitClaudeMd: true` (2.1.271 ou mais novo).

O que o pacote **não** faz sozinho: impedir que o `Bash` do `infra-guardian` execute mudança. O texto proíbe; instrução não é barreira. Para ambiente de risco, use a variante `static` ou ative as camadas abaixo (nenhuma vem ativada).

### 8.1 Hook que restringe o Bash do agente

Hook no frontmatter vale só enquanto o subagente roda. Hooks de agentes em `~/.claude/agents/` não dependem do diálogo de confiança da pasta; os de `.claude/agents/` do projeto, sim.

**Proposta** (crie e ative só se quiser):

1. Salve em `~/.claude/hooks/infra-guardian-bash-guard.mjs`:

```js
// Permite ao infra-guardian só comandos locais de leitura conhecidos. Qualquer outro é negado (exit 2).
import { readFileSync } from "node:fs";

const entrada = JSON.parse(readFileSync(0, "utf8"));
const comando = String(entrada?.tool_input?.command ?? "").trim();

const negar = (motivo) => {
  process.stderr.write(`infra-guardian: comando bloqueado (${motivo}). Peça ao usuário para rodar, se for necessário.\n`);
  process.exit(2);
};

if (!comando) negar("vazio");
if (/[;&|<>`\n\r]|\$\(/.test(comando)) negar("encadeamento, redirecionamento ou substituição");

const arq = `("[^"]+"|[^ "-][^ "]*)`;
const permitidos = [
  /^git (-C ("[^"]+"|[^ "]+) )?(status|diff|log|show|blame|ls-files|rev-parse|merge-base)( [^ ].*)?$/,
  /^git (branch|tag) --list( .*)?$/,
  /^git remote -v$/,
  /^ls( -[a-zA-Z]+)*( [^ ].*)?$/,
  /^wc( -[lwc])? [^ ].*$/,
  /^(docker|terraform|tofu|helm|hadolint|shellcheck|actionlint|promtool|yamllint|kubeconform|tflint) --version$/,
  /^docker compose version$/,
  /^kubectl version --client$/,
  new RegExp(`^docker compose( -f ${arq})* config( (-q|--quiet|--no-interpolate|--no-env-resolution))+$`),
  new RegExp(`^(hadolint|shellcheck|yamllint) ${arq}( ${arq})*$`),
  /^actionlint( -no-color)?$/,
  new RegExp(`^promtool check (rules|config) ${arq}$`),
  new RegExp(`^terraform fmt -check( -recursive)?( -diff)?( ${arq})?$`),
];

if (!permitidos.some((re) => re.test(comando))) negar("fora da lista de leitura");
if (/^docker compose (.* )?config( |$)/.test(comando) && !/ (-q|--quiet|--no-interpolate)( |$)/.test(comando)) negar("compose config sem -q nem --no-interpolate imprime segredos");
if (/(^|\s)(-c|--config-env|--exec|--upload-pack|--output|-o)(=|\s|$)/.test(comando)) negar("opção que executa ou grava");
process.exit(0);
```

2. Acrescente ao frontmatter do agente instalado:

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "node ~/.claude/hooks/infra-guardian-bash-guard.mjs"
```

No Windows, se `~` não for expandido, use o caminho absoluto (`node C:/Users/<você>/.claude/hooks/infra-guardian-bash-guard.mjs`).

Efeito: consulta remota autorizada (`kubectl get`, `ssh`) também fica bloqueada para o agente; a pessoa roda e entrega a saída.

### 8.2 Regras `ask` para mudanças de infraestrutura (sessão inteira)

As regras de `permissions` valem para a conversa principal **e** para os subagentes. Ordem oficial: deny, depois ask, depois allow. Para que comandos de mudança sempre peçam confirmação, em qualquer sessão, um exemplo para `~/.claude/settings.json` (junte ao que já existir):

```json
{
  "permissions": {
    "ask": [
      "Bash(terraform apply *)",
      "Bash(terraform destroy *)",
      "Bash(tofu apply *)",
      "Bash(tofu destroy *)",
      "Bash(kubectl apply *)",
      "Bash(kubectl delete *)",
      "Bash(kubectl scale *)",
      "Bash(kubectl rollout *)",
      "Bash(kubectl drain *)",
      "Bash(helm upgrade *)",
      "Bash(helm install *)",
      "Bash(helm uninstall *)",
      "Bash(docker compose down *)",
      "Bash(docker system prune *)",
      "Bash(docker volume rm *)",
      "Bash(systemctl restart *)",
      "Bash(systemctl stop *)",
      "Bash(ufw *)",
      "Bash(iptables *)",
      "Bash(nft *)"
    ]
  }
}
```

Limites (da própria documentação): padrões de Bash que tentam restringir argumentos são frágeis (`kubectl -n prod delete …` escapa de `kubectl delete *`; o mesmo programa chamado por caminho ou dentro de `sh -c` também escapa). Teste antes, porque as regras afetam o seu uso normal do Claude.

### 8.3 Identidade só de leitura

Se o agente (ou você, para ele) for consultar ambiente real, use uma identidade que não consiga mudar nada:

- Kubernetes: kubeconfig com um papel de leitura (o `ClusterRole` `view` padrão não inclui Secrets).
- Cloud: papel de leitura de **metadados**; papéis "somente leitura" amplos costumam ler dados também (ex.: objetos de bucket). Confira na documentação do provedor.
- Banco: usuário com permissão só de estatísticas e catálogos; lembre que ver as sessões dos outros mostra o texto das consultas.
- SSH: conta sem sudo, ou comandos coletados pela pessoa.

### 8.4 Sandbox e isolamento

O sandbox do Bash do Claude Code vem desligado e só existe em macOS, Linux e WSL2; no Windows nativo os comandos rodam sem sandbox. Para analisar repositório de terceiros ou suspeito, rode numa VM ou container descartável, sem credenciais montadas (sem `~/.kube`, `~/.aws`, `~/.ssh`) e com saída de rede restrita.

## 9. Por que não há skills por domínio

O prompt de origem sugeria skills opcionais (`infra-diagnostico-vps`, `infra-docker-compose`, `infra-kubernetes`, `infra-recovery`, `infra-finops`). Ficaram de fora de propósito:

- para o subagente carregar skill sob demanda, ele precisaria da ferramenta de skills, que daria acesso a **qualquer** skill instalada na máquina, inclusive as que operam sistemas reais;
- skills ficam visíveis também para a conversa principal, que pode carregá-las fora de contexto;
- o mesmo efeito (ler só o domínio pertinente) vem do playbook, que o agente lê por seção.

Se no futuro uma skill por domínio fizer sentido, ela deve ser pré-carregada pelo campo `skills:` de um agente específico, não liberada pela ferramenta de skills.

## 10. Validação feita neste pacote

Registrada em `INFRA_GUARDIAN_EVALS.md`, seção 4. Para repetir:

- **Frontmatter:** parser YAML (ex.: `js-yaml`), conferindo `name`, `description`, `tools` e se as duas variantes têm o mesmo corpo.
- **Schema do relatório:** schema e exemplo de `INFRA_GUARDIAN_REPORT_TEMPLATE.md` num validador JSON Schema draft-07 (ex.: `ajv`), com casos inválidos.
- **`claude plugin validate <pasta>`:** valida pastas `agents/` soltas a partir do Claude Code **2.1.233**. Em versão anterior, use o parser YAML.
- **Hook:** rode o script isolado com comandos fictícios (`echo '{"tool_input":{"command":"..."}}' | node infra-guardian-bash-guard.mjs`) e confira o código de saída.
- **Comportamento:** cenários de `INFRA_GUARDIAN_EVALS.md`.

## 11. Limitações conhecidas

- Com `Bash`, a proibição de mudança depende do texto do agente e das camadas da seção 8; sem elas, é só instrução. A variante `static` não tem esse problema.
- O agente não consulta documentação na internet (sem WebFetch): versões, preços e limites de produto ficam "não verificados" até alguém conferir. As versões citadas no playbook são de 12/11/2026.
- Sem acesso ao ambiente real, o diagnóstico é do que está em arquivo e nas saídas entregues. Estado de execução (reinícios, uso, latência, custo) depende de dado fornecido.
- As regras do agente não valem para a conversa principal. Nos testes de 12/11/2026, quem abriu o `.env` das fixtures foi o Claude principal da sessão de teste, nunca o agente. Para proteger a conversa principal, use regras `deny` de leitura de arquivos de segredo (o guia do Security Sentinel, seção 5.1, traz um exemplo).
- Um modelo pode errar. As avaliações reduzem, não eliminam, falso positivo e falso negativo.

## 12. Manutenção

- **Conhecimento do agente:** as regras ficam no próprio agente; versões e detalhes, no playbook (seção 1). Revise a cada 6 meses e quando sair versão nova de Kubernetes, PostgreSQL, Terraform/OpenTofu, Prometheus, Compose ou do DORA. Fontes: as listadas no playbook, sempre a documentação oficial da versão em uso.
- **O que depende de acesso real:** estado dos serviços, métricas, custo efetivo, drift, restore testado. O agente não substitui essa coleta; ele diz o que coletar.
- **Idiomas:** `pt-BR` e `en` andam juntos. Mudou um, mude o outro no mesmo commit.
- **Variantes:** a `static` é gerada da principal trocando só `name`, `description` e `tools`; confira que o corpo continua igual.
- **Mudanças no agente:** rode de novo pelo menos EV-01, EV-03, EV-04, EV-10, EV-11 e EV-12 e registre.

## 13. Como aprovar uma mudança

1. O Infra Guardian entrega o relatório com IDs, alternativas e plano.
2. Você escolhe o que fazer: `Implemente IG-002 conforme o relatório, numa branch nova, com o plano de rollback.`
3. O Claude principal (ou outro agente autorizado) prepara o diff e mostra pré-checks, testes e rollback.
4. Você revisa e aprova. Em produção, quem executa é você ou alguém da equipe, na janela combinada.
5. `Use o agente infra-guardian para verify-change de IG-002`, com as evidências do pós-mudança.
6. A decisão de manter, reverter ou aceitar risco é sua.
