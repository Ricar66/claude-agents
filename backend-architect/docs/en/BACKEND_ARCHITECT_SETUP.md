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

Merge it into your `~/.claude/settings.json` or the project's `.claude/settings.json`. A limit from the documentation itself: Bash patterns that try to constrain arguments are fragile (the same program called by another path or inside `sh -c` escapes). Test first, because the rules also affect your normal use.

### 5.2 Isolated work

For larger changes, run the agent in a separate worktree and review the diff before integrating. The frontmatter accepts `isolation: worktree`, which makes the agent always work on an isolated copy of the repository; it does not come turned on because it changes the flow (the changes stay in the worktree until you integrate them). On native Windows, Claude Code's Bash sandbox does not exist; strong isolation is a VM or a container.

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
