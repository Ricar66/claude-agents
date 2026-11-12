# Frontend Architect — installation, permissions and operation

## 1. What is in the package

```text
frontend-architect/
├── agent/
│   ├── pt-BR/frontend-architect.md    ← agent in Portuguese
│   └── en/frontend-architect.md       ← the same agent in English
├── org-context-example/
│   ├── pt-BR/SKILL.md                 ← fictitious example of visual standard and context
│   └── en/SKILL.md
└── docs/
    ├── pt-BR/                         ← installation guide, playbook, evaluations
    └── en/
```

Both versions have the same `name`: **install one language only**. The usage guide for the collection's four agents is in [`GUIDE.md`](../../../GUIDE.md).

## 2. What is different from the audit agents

The Frontend Architect **implements**: it has `Edit`, `Write` and `Bash`, follows the project's `CLAUDE.md` and runs the project's tests. It writes only when the request is for implementation; deploy, a real API, dependency installation, a development server that was not requested, commit and push need specific authorization. The layers in section 5 are the real barrier.

## 3. Requirements and installation

- Claude Code with file-based subagents. Checked on Claude Code **2.1.221** against the official subagent documentation (2026-11-12).
- Fields used: `name`, `description`, `tools`, `model`, `permissionMode`, `color`, `skills`, all on the official list.
- `model: inherit` uses the session's model. `permissionMode: default` is ignored when the main session is in `auto`, `acceptEdits` or `bypassPermissions` (the session's mode applies), and in `plan` the agent stays in `plan`. There is no field to limit writing by folder. Do not use `bypassPermissions`.

Windows (PowerShell):

```powershell
Copy-Item .\frontend-architect\agent\en\frontend-architect.md "$env:USERPROFILE\.claude\agents\"
```

Linux and macOS:

```bash
cp frontend-architect/agent/en/frontend-architect.md ~/.claude/agents/
```

For a single project only: `<project>/.claude/agents/`. After installing or updating, open a **new session**.

## 4. The organization's visual standard (optional and local)

The agent preloads the `frontend-architect-org-context` skill if it exists. That is where your organization's **mandatory visual standard** lives: colors with hex and use, typography, logos (file path and usage rules), permitted images, screen patterns, systems and rules. Create it at `~/.claude/skills/frontend-architect-org-context/SKILL.md`, with a `name` equal to the folder's and without `disable-model-invocation`. Fictitious template: `frontend-architect/org-context-example/en/SKILL.md`.

With the skill present, the agent applies that standard in the organization's projects, except when the project already has an approved theme of its own. Without the skill, it follows the tokens it finds in the repository.

## 5. Recommended permissions

Nothing comes turned on. The `permissions` rules apply to the main conversation and to subagents; the official order is deny, then ask, then allow.

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

Bash patterns are fragile (another path or `sh -c` escapes); test first. For large changes, use a separate worktree and review the diff; the frontmatter accepts `isolation: worktree` (it does not come turned on because it changes the integration flow). On native Windows there is no Bash sandbox.

## 6. How to invoke

- `Use the frontend-architect agent to run discover on this project.`
- `Use the frontend-architect agent for a ux-review of the orders screen.`
- `Use the frontend-architect agent for an accessibility-audit of the registration form, with a keyboard test.`
- `Use the frontend-architect agent for a responsive-audit of the customers table on a phone.`
- `Use the frontend-architect agent to implement the empty, error and loading states in the orders list, with tests.`
- `Use the frontend-architect agent for design-system: unify the buttons and color tokens.`
- `Use the frontend-architect agent for dashboard-data-viz of the sales dashboard.`

State the mode, the screen or journey and whether it may implement.

## 7. Collaboration

Frontend owns UI, UX and API consumption; Backend owns APIs and the database; Sentinel owns the security audit; Guardian owns infrastructure. A contract change is agreed with the Backend Architect first. Full protocol in the [collection guide](../../../GUIDE.md).

## 8. Validation done on this package

Recorded in `FRONTEND_ARCHITECT_EVALS.md`, section 4.

## 9. Limitations

- With `Edit`, `Write` and `Bash`, the limits depend on the agent's text and on the layers in section 5.
- The agent does not open a browser or measure Core Web Vitals on its own: field measurement comes from RUM; laboratory measurement needs a project tool and authorization to start the server.
- An accessibility scanner does not replace manual testing; the agent declares what was tested by hand.
- No internet: versions and support stay "not verified" until someone checks them; the playbook's are from 2026-11-12.
- A model can make mistakes: review every diff and the screen before integrating.

## 10. Maintenance

- `pt-BR` and `en` change together.
- Review the playbook every 6 months and whenever a new version of React, Next.js, TypeScript, WCAG or the Core Web Vitals metrics comes out.
- Changed the agent? Run at least FE-01, FE-03, FE-07 and FE-14 again and record the results.
