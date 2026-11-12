# Backend Architect — installation, permissions and operation

## 1. What is in the package

```text
backend-architect/
├── agent/
│   ├── pt-BR/backend-architect.md     ← agent in Portuguese
│   └── en/backend-architect.md        ← the same agent in English
├── org-context-example/
│   ├── pt-BR/SKILL.md                 ← fictitious example of organizational context
│   └── en/SKILL.md
└── docs/
    ├── pt-BR/                         ← installation guide, playbook, evaluations
    └── en/
```

Both versions have the same `name`: **install one language only**. The usage guide for the collection's four agents is in [`GUIDE.md`](../../../GUIDE.md), at the repository root.

## 2. What is different from the audit agents

The Security Sentinel and the Infra Guardian only read. The Backend Architect **implements**: it has `Edit`, `Write` and `Bash`, reads the project's `CLAUDE.md` as a convention and runs the project's tests. Therefore:

- it writes only when the request is for implementation (`implement`, `refactor` or an explicit fix);
- deploy, a real database, a migration outside a disposable local database, dependency installation, commit and push need specific authorization in the conversation;
- the permission layers in section 5 are the real barrier; the agent's text is not.

## 3. Requirements and installation

- Claude Code with file-based subagents. Checked on Claude Code **2.1.221** against the official subagent documentation (2026-11-12).
- Fields used: `name`, `description`, `tools`, `model`, `permissionMode`, `color`, `skills`. All are on the official list.
- `model: inherit`: the agent uses the model of the session that called it.
- `permissionMode: default`: the agent asks for confirmation according to the permission rules. **This field is ignored when the main session is in `auto`, `acceptEdits` or `bypassPermissions`** (the session's mode applies), and in `plan` the agent stays in `plan`. There is no field to limit writing by folder. Do not use `bypassPermissions`.

Windows (PowerShell):

```powershell
Copy-Item .\backend-architect\agent\en\backend-architect.md "$env:USERPROFILE\.claude\agents\"
```

Linux and macOS:

```bash
cp backend-architect/agent/en/backend-architect.md ~/.claude/agents/
```

For a single project only: copy to `<project>/.claude/agents/`. After installing or updating, open a **new session**: an open session may keep the previous version of the agent.

## 4. Organizational context (optional and local)

The agent preloads the `backend-architect-org-context` skill if it exists: systems, stacks and versions, house standards (authentication, API errors, naming, migrations), sensitive integrations and rules. Create it at `~/.claude/skills/backend-architect-org-context/SKILL.md`, with a `name` equal to the folder's, without `disable-model-invocation`. Fictitious template: `backend-architect/org-context-example/en/SKILL.md`. Never put a password, token or connection string in it; the file is internal and does not go into a public repository.

## 5. Recommended permissions

Nothing here comes turned on. The `permissions` rules apply to the main conversation **and** to subagents; the official order is deny, then ask, then allow.

### 5.1 Deny what must never happen without you

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

Patterns starting with `./` apply from the folder where Claude Code was opened, including subfolders (monorepo). To apply in any folder, in `~/.claude/settings.json`, replace `./` with `//` (e.g. `Read(//**/.env)`); `**/.env` without a prefix is also relative to the current folder. `.env.example` is left out on purpose: the agent needs to read the variable names.

Merge it into your `~/.claude/settings.json` or the project's `.claude/settings.json`. A limit from the documentation itself: Bash patterns that try to constrain arguments are fragile (the same program called by another path or inside `sh -c` escapes). Test first, because the rules also affect your normal use.

### 5.2 Isolated work

For larger changes, run the agent in a separate worktree and review the diff before integrating. The frontmatter accepts `isolation: worktree`, which makes the agent always work on an isolated copy of the repository; it does not come turned on because it changes the flow (the changes stay in the worktree until you integrate them). On native Windows, Claude Code's Bash sandbox does not exist; strong isolation is a VM or a container.

### 5.3 Hook that denies what the agent never does on its own

Hooks can live in the subagent's frontmatter and only apply while it runs. Hooks of agents in `~/.claude/agents/` do not depend on the folder trust dialog; those in the project's `.claude/agents/` do. Unlike the Security Sentinel hook (a list of what is allowed), this one is a list of what is not: build, test, lint and project scripts follow the normal permission flow; chaining, `cd`, push, commit, package install, migration, network, deploy, server, file removal and reading a secret file are denied.

**Proposal** (create and enable only if you want this layer; the same script serves the Frontend Architect):

1. Save the script at `~/.claude/hooks/architect-bash-guard.mjs`:

```js
// PreToolUse (Bash) hook for backend-architect and frontend-architect.
// Denies what these agents must never do on their own; everything else follows the normal permission flow.
import { readFileSync } from "node:fs";

const input = JSON.parse(readFileSync(0, "utf8"));
const command = String(input?.tool_input?.command ?? "").trim();

const deny = (reason) => {
  process.stderr.write(`architect: command blocked (${reason}). Run one command at a time, without cd; if you need this, ask the user or the main Claude.\n`);
  process.exit(2);
};

if (!command) deny("empty");
if (/[;&|`\n\r]|\$\(|>|</.test(command)) deny("chaining, redirection or substitution");
if (/^(cd|pushd|Set-Location)\b/i.test(command)) deny("changing folder (use git -C <folder> or the tool's folder argument)");

const rules = [
  [/^git( -C ("[^"]+"|\S+))? (push|commit|reset|checkout|switch|restore|stash|clean|rebase|merge|cherry-pick|revert|tag|am|apply)\b/, "Git that changes history or files"],
  [/^git( -C ("[^"]+"|\S+))? (branch|remote) .*(-d|-D|--delete|add|set-url|remove|rm)\b/, "Git that changes a branch or remote"],
  [/^(npm|pnpm|yarn|bun) (i|install|ci|add|remove|rm|uninstall|update|up|upgrade|publish|link|dlx)\b/, "package install or publish"],
  [/^(npx|pnpx|bunx) (-y|--yes)\b/, "running a package downloaded on the fly"],
  [/^npx (playwright|cypress) install\b/, "browser download"],
  [/^(pip3?|uv|poetry|pipenv|conda) (install|add|remove|uninstall|update|sync|lock)\b|^python3? -m pip\b/, "Python package install"],
  [/^dotnet (add|remove|restore|tool|workload|nuget)\b/, ".NET package"],
  [/^(cargo (add|install|publish|update)|go (get|install)|gem install|composer (require|install|update))\b/, "package install"],
  [/prisma (migrate (deploy|reset|dev)|db (push|execute|seed))|dotnet ef database|knex migrate|sequelize db:|typeorm migration:run|alembic (upgrade|downgrade)|flask db (upgrade|downgrade)|rails db:/, "migration or database change"],
  [/^(curl|wget|Invoke-WebRequest|iwr|Invoke-RestMethod|irm|ssh|scp|sftp|rsync|nc|ncat|nmap|ping|telnet|ftp|tracert|traceroute|Test-NetConnection)\b/i, "network"],
  [/^(docker|podman|kubectl|helm|terraform|tofu|systemctl|service|sc|pm2|nssm|vercel|netlify|fly|railway|heroku|az|aws|gcloud)\b/i, "deploy, container, cluster, cloud or service"],
  [/^(npm|pnpm|yarn|bun) (run )?(dev|start|serve|preview|deploy|release)\b/, "server or deploy"],
  [/^(rm|del|erase|rmdir|rd|Remove-Item|shred)\b/i, "file removal (use Edit or Write; delete only on request)"],
  [/(^|\s)(cat|type|more|less|head|tail|Get-Content|gc|bat|strings|xxd)\b.*(\.env(\.(?!example)[\w.-]+)?\b(?!\.example)|\.pem\b|\.key\b|\.pfx\b|\.p12\b|id_rsa|id_ed25519)/i, "reading a secret file"],
  [/^(env|printenv|set)$|^Get-ChildItem Env:/i, "environment variables (may hold secrets)"],
];
for (const [re, reason] of rules) if (re.test(command)) deny(reason);
process.exit(0);
```

2. Add to the installed agent's frontmatter:

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "node ~/.claude/hooks/architect-bash-guard.mjs"
```

On Windows, if `~` is not expanded by the hook's shell, use the absolute path (`node C:/Users/<you>/.claude/hooks/architect-bash-guard.mjs`).

Effect: commit, push, install and migration become requests to the user or the main Claude. Tested with 63 fictitious commands (20 allowed, 43 denied), without running any of them. A list of what is not allowed is not a boundary: a program called by another name escapes; the rules in section 5.1 and reviewing the diff still apply.

## 6. How to invoke

- `Use the backend-architect agent to run discover on this repository.`
- `Use the backend-architect agent for api-contract of the orders endpoint, without implementing.`
- `Use the backend-architect agent to implement cursor pagination on GET /pedidos, with tests.`
- `Use the backend-architect agent for database-review of the sales report queries.`
- `Use the backend-architect agent for migration-plan of replacing cliente with cliente_id.`
- `Use the backend-architect agent for technology-assessment: keep Express or move to Fastify.`
- `Use the backend-architect agent to run verify on the current branch's changes.`

State the mode, the scope (folder, route, branch) and whether it may implement. Without "implement", it delivers a design or a review.

## 7. Collaboration

Backend owns APIs, rules and the database; Frontend owns the UI; Sentinel owns the security audit; Guardian owns infrastructure. The full protocol (contract before code, owner of each file, handoff, reviews) is in the [collection guide](../../../GUIDE.md). Subagents do not call one another: you or the main Claude forward the work, from the handoff the agent writes.

## 8. Validation done on this package

Recorded in `BACKEND_ARCHITECT_EVALS.md`, section 4: frontmatter parsing with a YAML parser, a check of the fields against the official list and behavior scenarios. `claude plugin validate <folder>` exists from Claude Code **2.1.233**.

## 9. Limitations

- With `Edit`, `Write` and `Bash`, the limits depend on the agent's text and on the layers in section 5. An instruction is not a barrier.
- `permissionMode` does not protect when the main session is in a permissive mode.
- The agent does not look things up on the internet: versions and support stay "not verified" until someone checks them; the playbook's versions are from 2026-11-12.
- A model can make mistakes: review every diff before integrating.

## 10. Maintenance

- `pt-BR` and `en` change together.
- Review the playbook's versions every 6 months and whenever a new version of Node.js, .NET, Python, TypeScript, OpenAPI or the ORM in use comes out.
- Changed the agent? Run at least BE-01, BE-04, BE-05 and BE-13 from the evaluations again and record the results.
