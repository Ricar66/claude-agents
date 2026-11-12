# Infra Guardian — installation, permissions and operation

## 1. What is in the package

```text
infra-guardian/
├── agent/
│   ├── pt-BR/infra-guardian.md          ← agent with Bash (local reads)
│   ├── pt-BR/infra-guardian-static.md   ← the same, without Bash
│   ├── en/infra-guardian.md
│   └── en/infra-guardian-static.md
├── org-context-example/
│   ├── pt-BR/SKILL.md                   ← fictitious example of organizational context
│   └── en/SKILL.md
└── docs/
    ├── pt-BR/                           ← installation guide, playbook, report template, document templates, evaluations
    └── en/
```

Both variants have the same instruction text; only `name`, `description` and `tools` change. The `pt-BR` and `en` versions of each variant share the same `name`: **install one language only**.

| Variant | Tools | When to use |
|---|---|---|
| `infra-guardian` | `Read`, `Grep`, `Glob`, `Bash` | Normal use: it can run local read-only commands (Git, Compose validation) |
| `infra-guardian-static` | `Read`, `Grep`, `Glob` | When no command may be executed: the isolation is technical, not just an instruction |

With `Bash` in the list, "read-only" depends on the agent's text and on the layers in section 8. Without `Bash`, the agent has no way to execute anything.

## 2. Requirements

- Claude Code with file-based subagents (`~/.claude/agents/` or `.claude/agents/`). Written and checked on Claude Code **2.1.221** against the official subagent documentation, consulted on 2026-11-12.
- Frontmatter fields used: `name`, `description`, `tools`, `model`, `effort`, `maxTurns`, `omitClaudeMd`, `color`, `skills`. All are on the official list; an unknown field is ignored without warning.
- `omitClaudeMd: true` keeps the analyzed project's `CLAUDE.md` from coming in as instructions (a hostile repository could use it to manipulate the diagnosis). The documentation lists the field from Claude Code **2.1.271**; on an earlier version it is ignored without error and this protection does not exist. Recommended: `claude update`.
- `model: opus` requires access to an Opus model; without it, switch to `inherit` or `sonnet`. `effort: high`: the available levels depend on the model.
- No infrastructure tool is required. Docker, Terraform, kubectl and validators help when they are already installed (playbook, section 6).

## 3. Installation

### 3.1 For all projects (recommended)

Windows (PowerShell):

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\agents" | Out-Null
Copy-Item .\infra-guardian\agent\en\infra-guardian.md "$env:USERPROFILE\.claude\agents\"
Copy-Item .\infra-guardian\agent\en\infra-guardian-static.md "$env:USERPROFILE\.claude\agents\"
```

Linux and macOS:

```bash
mkdir -p ~/.claude/agents
cp infra-guardian/agent/en/infra-guardian.md infra-guardian/agent/en/infra-guardian-static.md ~/.claude/agents/
```

Replace `en` with `pt-BR` for the Portuguese version. The `static` variant is optional.

### 3.2 For a single project

Copy to `<project>/.claude/agents/`. Official precedence when the same name exists in more than one place: managed settings > `--agents` flag > the project's `.claude/agents/` > `~/.claude/agents/` > plugins.

### 3.3 Check

Open a **new session** and ask: `Use the infra-guardian agent to run discover on this folder, read-only.` If Claude does not delegate, run `claude --debug` and look for the file name: invalid YAML, a missing `description` or an invalid `name` make the file be skipped, with a warning only in the debug log.

Updated the file in the repository? Copy it again and open a new session: the installed copy does not follow the repository, and an already open session may keep the previous version of the agent.

## 4. Organizational context (optional and local)

Both agents preload the `infra-guardian-org-context` skill **if it exists**. If it does not, Claude Code skips it and logs a warning in the debug log; the agent runs and reports "no organizational context".

Use it to tell the agent what only your organization knows: environments, servers and what runs on each, network and links, backup, monitoring, deploys, known operational risks, risk-acceptance decisions and internal rules (change window, who executes).

1. Folder `~/.claude/skills/infra-guardian-org-context/` (or the project's `.claude/skills/…`).
2. `SKILL.md` with frontmatter `name: infra-guardian-org-context` and a `description`. **Do not** use `disable-model-invocation: true`: a skill with that field is not preloaded.
3. Content as data, not as orders. Fictitious template: `infra-guardian/org-context-example/en/SKILL.md`.

Rules: never a password, token, key or secret value; the file is internal and does not go into a public repository; to keep the original in another folder, use a link (`New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\infra-guardian-org-context" -Target "<original-folder>"` on Windows; `ln -s <original-folder> ~/.claude/skills/infra-guardian-org-context` on Linux and macOS). If you also use the Security Sentinel, both contexts overlap on servers and network: when you change one, review the other.

## 5. How to invoke

The agent identifies the mode from the keyword; a broad request becomes `discover`.

- `Use the infra-guardian agent to run discover on this repository, read-only.`
- `Use the infra-guardian agent for a docker-review of the production compose file.`
- `Use the infra-guardian agent for a vps-audit; the output of df, free and systemctl is in collection/.`
- `Use the infra-guardian agent for backup-dr of the main database, proposing RPO and RTO for approval.`
- `Use the infra-guardian agent for network-diagnose of the 504s on the site; the nginx logs are in logs/.`
- `Use the infra-guardian agent for a pipeline-review of the deploy workflow.`
- `Use the infra-guardian agent for release-readiness of version 2.4.`
- `Use the infra-guardian agent for verify-change of items IG-001 and IG-003 on the current branch.`
- `Use the infra-guardian-static agent for an architecture-review, without running any command.`

Tips: state the environment (development, staging, production); attach already filtered command output; for `verify-change`, paste the previous findings so the IDs stay the same. The agent replies with text; to keep the report, ask the main Claude to save it, outside the public repository when the project is internal.

## 6. Authorization

Summary (full matrix in the playbook, section 3.1):

| Action | The agent |
|---|---|
| Read files and run local read-only commands | Does it |
| Read-only remote query (SSH, `kubectl get`, `terraform plan`, cloud CLI, query) | Only with authorization in the conversation (target, environment, command); preferably the person runs it |
| Any change, in any environment | Never does it: it delivers the plan; the person or the main Claude executes |

Authorization to diagnose is not authorization to change, and authorization for one change does not extend to another.

## 7. Integration with the Security Sentinel

Both agents belong to the same collection and complement each other:

- The Infra Guardian covers availability, performance, capacity, backup, deploys and cost. When it finds something security-related (a published database port, a secret in a compose file, a service under a personal account, a malicious instruction in a README), it records the fact and lists the item in the **"For the Security Sentinel"** section of the report.
- Subagents do not call other subagents in this collection (neither of them has the `Agent` tool). You or the main Claude forward it: `Use the security-sentinel agent to assess the "For the Security Sentinel" items in the Infra Guardian report.`
- Both share context, not permissions. Neither applies changes.

## 8. Permissions and hardening

What the package already does:

- `tools` is a **closed list**: the agent does not get Write, Edit, WebFetch, WebSearch, MCP, `Agent` or the skill tool.
- No `memory` (turning memory on grants Read, Write and Edit automatically).
- No `permissionMode`: the documentation says this field is ignored when the main session is in `auto`, `acceptEdits` or `bypassPermissions`. It is not a guarantee, and the package does not depend on it.
- `omitClaudeMd: true` (2.1.271 or newer).

What the package does **not** do on its own: prevent the `infra-guardian`'s `Bash` from executing a change. The text forbids it; an instruction is not a barrier. For risky environments, use the `static` variant or turn on the layers below (none comes turned on).

### 8.1 Hook that restricts the agent's Bash

A hook in the frontmatter only applies while the subagent runs. Hooks of agents in `~/.claude/agents/` do not depend on the folder trust dialog; those in the project's `.claude/agents/` do.

**Proposal** (create and turn it on only if you want it):

1. Save to `~/.claude/hooks/infra-guardian-bash-guard.mjs`:

```js
// Allows infra-guardian only known local read-only commands. Anything else is denied (exit 2).
import { readFileSync } from "node:fs";

const input = JSON.parse(readFileSync(0, "utf8"));
const command = String(input?.tool_input?.command ?? "").trim();

const deny = (reason) => {
  process.stderr.write(`infra-guardian: command blocked (${reason}). Ask the user to run it, if needed.\n`);
  process.exit(2);
};

if (!command) deny("empty");
if (/[;&|<>`\n\r]|\$\(/.test(command)) deny("chaining, redirection or substitution");

const file = `("[^"]+"|[^ "-][^ "]*)`;
const allowed = [
  /^git (-C ("[^"]+"|[^ "]+) )?(status|diff|log|show|blame|ls-files|rev-parse|merge-base)( [^ ].*)?$/,
  /^git (branch|tag) --list( .*)?$/,
  /^git remote -v$/,
  /^ls( -[a-zA-Z]+)*( [^ ].*)?$/,
  /^wc( -[lwc])? [^ ].*$/,
  /^(docker|terraform|tofu|helm|hadolint|shellcheck|actionlint|promtool|yamllint|kubeconform|tflint) --version$/,
  /^docker compose version$/,
  /^kubectl version --client$/,
  new RegExp(`^docker compose( -f ${file})* config( (-q|--quiet|--no-interpolate|--no-env-resolution))+$`),
  new RegExp(`^(hadolint|shellcheck|yamllint) ${file}( ${file})*$`),
  /^actionlint( -no-color)?$/,
  new RegExp(`^promtool check (rules|config) ${file}$`),
  new RegExp(`^terraform fmt -check( -recursive)?( -diff)?( ${file})?$`),
];

if (!allowed.some((re) => re.test(command))) deny("not on the read-only list");
if (/^docker compose (.* )?config( |$)/.test(command) && !/ (-q|--quiet|--no-interpolate)( |$)/.test(command)) deny("compose config without -q or --no-interpolate prints secrets");
if (/(^|\s)(-c|--config-env|--exec|--upload-pack|--output|-o)(=|\s|$)/.test(command)) deny("option that executes or writes");
process.exit(0);
```

2. Add to the installed agent's frontmatter:

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "node ~/.claude/hooks/infra-guardian-bash-guard.mjs"
```

On Windows, if `~` is not expanded, use the absolute path (`node C:/Users/<you>/.claude/hooks/infra-guardian-bash-guard.mjs`).

Effect: authorized remote queries (`kubectl get`, `ssh`) are blocked for the agent too; the person runs them and hands over the output.

### 8.2 `ask` rules for infrastructure changes (whole session)

`permissions` rules apply to the main conversation **and** to subagents. Official order: deny, then ask, then allow. To make change commands always ask for confirmation, in any session, an example for `~/.claude/settings.json` (merge with what is already there):

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

Limits (from the documentation itself): Bash patterns that try to constrain arguments are fragile (`kubectl -n prod delete …` escapes `kubectl delete *`; the same program called by path or inside `sh -c` escapes too). Test first, because the rules affect your normal use of Claude.

### 8.3 Read-only identity

If the agent (or you, on its behalf) is going to query a real environment, use an identity that cannot change anything:

- Kubernetes: a kubeconfig with a read role (the default `view` `ClusterRole` does not include Secrets).
- Cloud: a role that reads **metadata**; broad "read-only" roles often read data too (e.g. bucket objects). Check the provider's documentation.
- Database: a user with permission only on statistics and catalogs; remember that seeing other sessions shows the text of their queries.
- SSH: an account without sudo, or commands collected by the person.

### 8.4 Sandbox and isolation

Claude Code's Bash sandbox is off by default and only exists on macOS, Linux and WSL2; on native Windows, commands run without a sandbox. To analyze a third-party or suspicious repository, run it in a disposable VM or container, with no credentials mounted (no `~/.kube`, `~/.aws`, `~/.ssh`) and restricted outbound network.

## 9. Why there are no per-domain skills

The source prompt suggested optional skills (`infra-diagnostico-vps`, `infra-docker-compose`, `infra-kubernetes`, `infra-recovery`, `infra-finops`). They were left out on purpose:

- for the subagent to load a skill on demand, it would need the skill tool, which would give access to **any** skill installed on the machine, including ones that operate real systems;
- skills are also visible to the main conversation, which may load them out of context;
- the same effect (reading only the relevant domain) comes from the playbook, which the agent reads by section.

If a per-domain skill makes sense in the future, it should be preloaded through the `skills:` field of a specific agent, not unlocked through the skill tool.

## 10. Validation done on this package

Recorded in `INFRA_GUARDIAN_EVALS.md`, section 4. To repeat it:

- **Frontmatter:** a YAML parser (e.g. `js-yaml`), checking `name`, `description`, `tools` and that both variants have the same body.
- **Report schema:** the schema and example from `INFRA_GUARDIAN_REPORT_TEMPLATE.md` in a JSON Schema draft-07 validator (e.g. `ajv`), with invalid cases.
- **`claude plugin validate <folder>`:** validates loose `agents/` folders from Claude Code **2.1.233**. On earlier versions, use the YAML parser.
- **Hook:** run the script on its own with fictitious commands (`echo '{"tool_input":{"command":"..."}}' | node infra-guardian-bash-guard.mjs`) and check the exit code.
- **Behavior:** the scenarios in `INFRA_GUARDIAN_EVALS.md`.

## 11. Known limitations

- With `Bash`, the ban on changes depends on the agent's text and on the layers in section 8; without them, it is only an instruction. The `static` variant does not have this problem.
- The agent does not look up documentation on the internet (no WebFetch): versions, prices and product limits stay "not verified" until someone checks them. The versions cited in the playbook are from 2026-11-12.
- Without access to the real environment, the diagnosis covers what is in files and in the output handed over. Runtime state (restarts, usage, latency, cost) depends on data provided.
- The agent's rules do not apply to the main conversation. In the 2026-11-12 tests, the one that opened the fixtures' `.env` was the test session's main Claude, never the agent. To protect the main conversation, use `deny` rules for reading secret files (the Security Sentinel guide, section 5.1, has an example).
- A model can make mistakes. The evaluations reduce, but do not eliminate, false positives and false negatives.

## 12. Maintenance

- **The agent's knowledge:** the rules live in the agent itself; versions and details, in the playbook (section 1). Review every 6 months and when a new version of Kubernetes, PostgreSQL, Terraform/OpenTofu, Prometheus, Compose or DORA comes out. Sources: those listed in the playbook, always the official documentation of the version in use.
- **What depends on real access:** service state, metrics, actual cost, drift, tested restore. The agent does not replace that collection; it says what to collect.
- **Languages:** `pt-BR` and `en` go together. If you change one, change the other in the same commit.
- **Variants:** the `static` one is generated from the main one by changing only `name`, `description` and `tools`; check that the body is still the same.
- **Changes to the agent:** run at least EV-01, EV-03, EV-04, EV-10, EV-11 and EV-12 again and record the results.

## 13. How to approve a change

1. The Infra Guardian delivers the report with IDs, alternatives and a plan.
2. You choose what to do: `Implement IG-002 as described in the report, on a new branch, with the rollback plan.`
3. The main Claude (or another authorized agent) prepares the diff and shows pre-checks, tests and rollback.
4. You review and approve. In production, you or someone on the team executes, in the agreed window.
5. `Use the infra-guardian agent for verify-change of IG-002`, with the post-change evidence.
6. The decision to keep, revert or accept the risk is yours.
