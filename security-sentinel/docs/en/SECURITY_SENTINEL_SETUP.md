# Security Sentinel — installation, permissions and operation

## 1. What the package contains

```text
security-sentinel/
├── agent/
│   ├── pt-BR/security-sentinel.md     ← agent in Portuguese
│   └── en/security-sentinel.md        ← same agent in English
├── org-context-example/
│   └── SKILL.md                       ← fictional example of organizational context
└── docs/
    ├── pt-BR/                          ← setup guide, playbook, report template, evaluations
    └── en/
```

Both versions of the agent have the same `name` (`security-sentinel`). **Install only one.**

## 2. Requirements

- Claude Code with support for file-based subagents (`~/.claude/agents/` or `.claude/agents/`). This package was written and checked on Claude Code **2.1.221** against the official subagents documentation (`code.claude.com/docs/en/sub-agents`), consulted on 2026-11-12.
- Frontmatter fields used: `name`, `description`, `tools`, `model`, `effort`, `maxTurns`, `omitClaudeMd`, `color`, `skills`. All of them are in the official list; fields that Claude Code does not recognize are ignored without warning.
- `omitClaudeMd: true` prevents the audited project's `CLAUDE.md` from entering the agent as an instruction (a hostile repository could use it to manipulate the audit). The documentation lists this field from Claude Code **2.1.271** onward; in an earlier version it is ignored without error and this protection does not exist. Recommended: `claude update`.
- `model: opus` requires access to an Opus model. Without it, change it to `inherit` or `sonnet`.
- `effort: high`: the available levels depend on the model.

## 3. Installation

### 3.1 For all projects (recommended)

Windows (PowerShell):

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\agents" | Out-Null
Copy-Item .\security-sentinel\agent\en\security-sentinel.md "$env:USERPROFILE\.claude\agents\security-sentinel.md"
```

Linux and macOS:

```bash
mkdir -p ~/.claude/agents
cp security-sentinel/agent/en/security-sentinel.md ~/.claude/agents/security-sentinel.md
```

For the Portuguese version, just replace `en` with `pt-BR`.

### 3.2 For one project only

Copy the file to `<project>/.claude/agents/security-sentinel.md`. Official precedence when the same name exists in more than one place: managed settings > `--agents` flag > the project's `.claude/agents/` > `~/.claude/agents/` > plugins.

### 3.3 Verify

Open a new session and ask: `Use the security-sentinel agent to run triage on this folder, read-only.` If Claude does not delegate, run `claude --debug` and look for the file name: files with invalid YAML, without `description` or with an invalid `name` are skipped and only show up in the debug log.

Updated the file in the repository? Copy it again. The installed copy does not follow the repository on its own. Then open a **new** Claude Code session: a session that is already open may keep using the agent version it loaded earlier (observed in the 2026-11-12 tests).

## 4. Organizational context (optional and local)

The agent preloads the `security-sentinel-org-context` skill **if it exists**. If it does not exist, Claude Code skips it and logs a warning in the debug log; the agent runs normally and reports "no organizational context".

Use it to tell the agent what only your organization knows: inventory of systems, where they run, network boundaries, service accounts, already known risks, accepted-risk decisions, internal rules and what is authorized by default.

How to create it:

1. Folder `~/.claude/skills/security-sentinel-org-context/` (or the project's `.claude/skills/…`).
2. `SKILL.md` file with frontmatter `name: security-sentinel-org-context` and a `description`. Do **not** use `disable-model-invocation: true`: that field prevents preloading.
3. Content as data, not as orders. Complete, fictional template: `security-sentinel/org-context-example/SKILL.md`.

Rules:

- **Never** put a password, token, key or secret value in it. Write where the credential is and what the risk is.
- The file is internal. Do not commit it to the public repository. If you want history, use a separate private repository or keep it on the machine only.
- To keep the original in another folder without duplicating it, use a link: on Windows, `New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\security-sentinel-org-context" -Target "<original-folder>"`; on Linux and macOS, `ln -s <original-folder> ~/.claude/skills/security-sentinel-org-context`.
- The skill is also visible to the main Claude, which may load it when it sees fit. This is expected: it lives on the machine of someone who already has access to this information.

## 5. Permissions and hardening

What the package already does:

- `tools: Read, Grep, Glob, Bash` is a **closed list**: the agent does not receive Write, Edit, WebFetch, WebSearch, MCP tools or the skills tool.
- No `memory`: enabling memory automatically gives the agent Read, Write and Edit, which would break "read-only".
- `omitClaudeMd: true` (Claude Code 2.1.271 or newer): the audited project's `CLAUDE.md` does not become an instruction for the auditor. Managed policy files still apply.
- No `permissionMode`: the documentation says this field is ignored when the main session is in `auto`, `acceptEdits` or `bypassPermissions`. It is not a guarantee.

What the package does **not** do on its own: prevent Bash from writing, installing or accessing the network. The agent's text forbids it, but an instruction is not a barrier. For a risky environment, enable the layers below (none comes enabled).

### 5.1 Deny rules in `settings.json`

The `permissions.deny` rules apply to the main conversation **and** to subagents, in all modes. Example for `~/.claude/settings.json` (merge it with whatever already exists):

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

Patterns starting with `./` apply from the folder where Claude Code was opened, including subfolders (monorepo). To apply in any folder, in `~/.claude/settings.json`, replace `./` with `//` (e.g. `Read(//**/.env)`); `**/.env` without a prefix is also relative to the current folder. `.env.example` is left out on purpose: the agent needs to read the variable names.

Limits (from the documentation itself): a Bash rule covers the usual form of the command and is **not** a security boundary around the program (`git -C . push` escapes `git push *`). Read rules block the Read tool, not a `cat` through Bash. The rules also affect your normal use of Claude; test before adopting them.

### 5.2 Hook that restricts the agent's Bash

Hooks can live in the subagent's frontmatter and only apply while it runs. Hooks of agents in `~/.claude/agents/` do not depend on the folder trust dialog; those in the project's `.claude/agents/` do. A `PreToolUse` hook does not override a deny rule.

**Proposal** (create and enable only if you want this layer):

1. Save the script at `~/.claude/hooks/sentinel-bash-guard.mjs`:

```js
// PreToolUse (Bash) hook for security-sentinel: allows only known read-only commands; anything else is denied (exit 2).
// The command is split the way bash would split it, without expanding anything, and whatever bash would expand is refused;
// the check runs on the real arguments. Any unexpected error also denies (fail closed).
import { readFileSync } from "node:fs";

const deny = (reason) => {
  process.stderr.write(`security-sentinel: command blocked (${reason}). Ask the user to run it, if necessary.\n`);
  process.exit(2);
};

// Splits like bash, without expanding: single quotes are literal; inside double quotes, \ only escapes " \ $ and backtick.
// Refuses operators, $ and backtick, backslash outside quotes, and glob, brace and parenthesis characters outside quotes.
function split(s) {
  if (/[\r\n]/.test(s)) deny("line break");
  const args = [];
  let current = null;
  let i = 0;
  while (i < s.length) {
    const c = s[i];
    if (c === " " || c === "\t") { if (current !== null) { args.push(current); current = null; } i++; continue; }
    if (current === null) {
      current = "";
      if (c === "#") deny("shell comment");
    }
    if (c === "'") {
      const end = s.indexOf("'", i + 1);
      if (end < 0) deny("unclosed quotes");
      current += s.slice(i + 1, end);
      i = end + 1;
      continue;
    }
    if (c === '"') {
      let j = i + 1;
      while (j < s.length && s[j] !== '"') {
        if (s[j] === "$" || s[j] === "`") deny("shell expansion ($ or backtick)");
        if (s[j] === "\\" && j + 1 < s.length && '"\\$`'.includes(s[j + 1])) { current += s[j + 1]; j += 2; continue; }
        current += s[j];
        j++;
      }
      if (j >= s.length) deny("unclosed quotes");
      i = j + 1;
      continue;
    }
    if (";&|<>".includes(c)) deny("chaining or redirection");
    if (c === "$" || c === "`") deny("shell expansion ($ or backtick)");
    if (c === "\\") deny("backslash outside quotes (put the path in quotes)");
    if ("*?[]{}()!".includes(c)) deny("shell special character outside quotes (put it in quotes)");
    current += c;
    i++;
  }
  if (current !== null) args.push(current);
  return args;
}

// Denies a dangerous long option, abbreviated too (git accepts a unique prefix). The exact name of another option wins
// over the abbreviation (--text is not --textconv). Stops at "--".
const SAFE_EXACT = new Set(["--text"]);
function checkOptions(args, dangerous) {
  for (const a of args) {
    if (a === "--") return;
    if (!a.startsWith("--")) continue;
    const name = a.split("=")[0];
    if (name.length > 2 && !SAFE_EXACT.has(name) && dangerous.some((p) => p.startsWith(name))) deny("option that executes or writes");
  }
}

const GIT_READ = new Set(["status", "diff", "log", "show", "blame", "ls-files", "rev-parse", "merge-base"]);
const GIT_DANGEROUS = ["--output", "--exec", "--upload-pack", "--receive-pack", "--config-env", "--ext-diff", "--textconv", "--open-files-in-pager", "--no-index", "--untracked", "--no-exclude-standard"];
const SCANNERS = new Set(["semgrep", "trivy", "osv-scanner", "gitleaks", "checkov", "codeql", "syft"]);

// Secret file. Normalizes like Windows (case, trailing dot or space, :alternate stream)
// and also looks at the path in <revision>:<path> and in --option=<path>.
function isSecret(arg) {
  const m = String(arg).toLowerCase();
  if (/credenciais|credentials/.test(m) || /(^|[\\/])\.ssh([\\/]|$)/.test(m)) return true;
  const base = (x) => x.replace(/^.*[\\/]/, "");
  const candidates = [base(m).split(":")[0]];
  if (m.includes(":")) candidates.push(base(m.slice(m.indexOf(":") + 1)));
  if (m.includes(":")) candidates.push(base(m.slice(m.lastIndexOf(":") + 1)));
  if (m.includes("=")) candidates.push(base(m.slice(m.lastIndexOf("=") + 1)).split(":")[0]);
  return candidates.some((c0) => {
    const c = c0.replace(/[. ]+$/, "");
    return c === ".env"
      || (c.startsWith(".env.") && !/^\.env\.(example|sample|template|dist)$/.test(c))
      || /\.(pem|key|pfx|p12)$/.test(c)
      || /^id_(rsa|ed25519|ecdsa|dsa)$/.test(c)
      || /^(senha|password)[^\\/]*\.txt$/.test(c);
  });
}

// Does the git subcommand show file content (and not only names or statistics)?
function showsContent(sub, r) {
  const patch = r.some((x) => /^(-p|-u|--patch|--cc|--dd|--remerge-diff|--diff-merges|-c$|-m$|-U\d*$|--unified|--word-diff|--color-words|--binary|--full-diff|-L)/.test(x));
  if (["blame", "cat-file", "grep"].includes(sub)) return true;
  if (sub === "show" || sub === "diff") return patch || !r.some((x) => /^(--stat|--name-only|--name-status|--numstat|--shortstat|--summary|--dirstat|--raw|--no-patch|-s$|--quiet)/.test(x));
  if (sub === "log") return patch || !r.some((x) => /^(--stat|--name-only|--name-status|--numstat|--shortstat|--summary|--dirstat|--raw|--oneline|--format|--pretty|--no-patch|-s$)/.test(x));
  return false;
}

// A git glob or special pathspec (':(glob)', ':!') picks files without naming them; in a command that shows
// content, that would open a secret without the name .env appearing.
// Excluding a secret (':(exclude).env', ':!.env') is the safe use and passes.
const isTrickyPathspec = (x) => {
  if (x.startsWith("-")) return false;
  const excluded = /^:(\(exclude\)|!|\^)(.+)$/.exec(x);
  if (excluded) return !isSecret(excluded[2]);
  return /[*?[]/.test(x) || /^:[(/]/.test(x);
};
function validate(args) {
  const [prog, ...a] = args;
  if (prog === "git") {
    let i = 0;
    if (a[0] === "-C") { if (a.length < 3) deny("git -C without a folder or subcommand"); i = 2; }
    const sub = a[i];
    const rest = a.slice(i + 1);
    const ok = GIT_READ.has(sub)
      || (sub === "branch" && rest[0] === "--list")
      || (sub === "remote" && rest.length === 1 && rest[0] === "-v");
    if (!ok) deny("not on the read-only list");
    checkOptions(rest, GIT_DANGEROUS);
    if (showsContent(sub, rest) && rest.some(isSecret)) deny("reading a secret file");
    if (showsContent(sub, rest) && rest.some(isTrickyPathspec)) deny("glob or special pathspec in a command that shows content");
    return;
  }
  if (prog === "ls") {
    if (a.every((x) => /^-[a-zA-Z1]+$/.test(x) || !x.startsWith("-"))) return;
    deny("ls option not on the list");
  }
  if (prog === "wc") {
    const rest = /^-[lwcm]$/.test(a[0] ?? "") ? a.slice(1) : a;
    if (rest.length && rest.every((x) => !x.startsWith("-"))) return;
    deny("wc usage not on the list");
  }
  if (SCANNERS.has(prog) && a.length === 1 && a[0] === "--version") return;
  deny("not on the read-only list");
}

try {
  const input = JSON.parse(readFileSync(0, "utf8"));
  const raw = input?.tool_input?.command;
  if (typeof raw !== "string") deny("input without a text command");
  const command = raw.trim();
  if (!command) deny("empty");
  const args = split(command);
  if (!args.length) deny("empty");
  validate(args);
  process.exit(0);
} catch (e) {
  deny(`error parsing the command: ${e?.message ?? e}`);
}
```

2. Add to the installed agent's frontmatter:

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "node ~/.claude/hooks/sentinel-bash-guard.mjs"
```

On Windows, if `~` is not expanded by the hook's shell, use the absolute path (`node C:/Users/<you>/.claude/hooks/sentinel-bash-guard.mjs`).

Effect: the agent loses the ability to run scanners (only `--version` passes). The script splits the command the way bash would and refuses whatever bash would expand (`$`, backtick, globs and backslash outside quotes); the check runs on the real arguments, abbreviated options included, and any error inside it denies the command. Limit: a secret that is already committed shows up in any `git show <commit>` or `git log -p` that changes it, because the agent needs to read diffs to do its job; the hook only denies asking for it by name, with a glob or with a special pathspec. The defense is not committing secrets (the agent flags it when it finds one) and the rule of never reproducing a value. When you want a scanner, run it yourself, or ask the main Claude to, with the command the agent suggests, and hand the output to the agent.

### 5.3 Isolation for an untrusted repository

To audit third-party or suspicious code, run Claude Code in a disposable VM or container, with no credentials mounted and with restricted network egress. An instruction in the prompt does not replace isolation.

## 6. How to invoke

Phrases that work (the agent identifies the mode by the keyword):

- `Use the security-sentinel agent to run deep-audit on this repository, read-only; no external tests.`
- `Use the security-sentinel agent to run diff-review on the changes in the current branch.`
- `Use the security-sentinel agent to run network-review on the local proxy, firewall and Docker configurations, without applying changes.`
- `Use the security-sentinel agent to run ai-agent-review on the AI integration, MCP and webhooks.`
- `Use the security-sentinel agent to run verify-fix on findings SS-001 and SS-002.`
- `Use the security-sentinel agent to run threat-model on the payment flow.`
- `Use the security-sentinel agent to run release-gate for version 2.3, considering the open findings.`

Tips:

- State the scope: folder, branch, commit, environment. For `verify-fix`, paste the previous findings table so the IDs are kept.
- For active testing, write down the scope: target, window, environment, who authorized it. Without this, the agent refuses the active part.
- The agent answers with text. To keep the report, ask the main Claude to save it; for internal projects, save it outside the public repository.

## 7. Scanning tools (optional)

The agent works without any of them. When one is installed and you authorize it, it performs the preflight (version, network, execution of project code, secrets in the output, scope) before running. The tool table, with what each one sends over the network and how to run it offline, is in the playbook, section 6.

The package does not install anything.

## 8. Validation done on this package

What was checked and how is recorded in `SECURITY_SENTINEL_EVALS.md`, section 4, and in the repository history. To repeat it:

- **Frontmatter:** parse with a YAML parser (e.g. `js-yaml`), checking `name`, `description` and the `tools` list.
- **Report schema:** validation of the schema and of the example in `SECURITY_SENTINEL_REPORT_TEMPLATE.md` with a JSON Schema draft-07 validator (e.g. `ajv`).
- **`claude plugin validate <folder>`:** validates loose `agents/` folders from Claude Code **2.1.233** onward. In an earlier version, use the YAML parser.
- **Behavior:** scenarios in `SECURITY_SENTINEL_EVALS.md`.

## 9. Known limitations

- The prohibition on writing and on network access through Bash depends on the agent's text, the deny rules and the hook. Without 5.1 and 5.2, it is only an instruction.
- Without WebFetch, the agent does not query NVD, OSV, KEV or EPSS. CVEs and scores stay "not verified" until someone queries them and hands over the data.
- Static analysis proves neither exploitation nor its absence. Network and production are assessed only from what is in files.
- A model can be wrong. The evaluations reduce, but do not eliminate, false positives and false negatives.
- Prompt injection is mitigated, not eliminated: hence the layers in section 5.

## 10. Maintenance

- **Standards:** review the versions cited in the playbook every 6 months and whenever a new edition is released (OWASP, NIST, CWE Top 25, CVSS). Record the date of the check.
- **Languages:** `pt-BR` and `en` go together. If you change one, change the other in the same commit.
- **Agent changes:** re-run at least EV-01, EV-02, EV-08, EV-13, EV-15 and EV-16 and record the results.
- **Claude Code:** new versions bring frontmatter fields and validation commands; check the subagents documentation before adopting them.

## 11. How to approve a remediation

Sentinel does not fix anything. The suggested flow:

1. Sentinel delivers the report with IDs.
2. You choose the IDs and the wave: `Fix SS-001 and SS-004 as described in the report, on a new branch, with regression tests.`
3. The main Claude (or a separate remediation agent, preferably in a worktree) makes the change and shows the diff, the tests and the rollback plan.
4. You review and approve the diff.
5. `Use the security-sentinel agent to run verify-fix on SS-001 and SS-004 on branch <name>.`
6. The decision to merge and to release is yours. An accepted exception needs an owner and a deadline.

## 12. CI integration (proposal, not enabled)

Nothing here is created automatically. If the team wants continuous scanning:

| Item | Target file | Requires | Caution |
|---|---|---|---|
| SAST | `.github/workflows/codeql.yml` or Semgrep | GitHub Advanced Security (CodeQL on a private repository) or Semgrep with local rules | Semgrep with `--config auto` downloads rules and sends metrics; use versioned rules and `--metrics=off` |
| Dependencies | Dependabot (`.github/dependabot.yml`) or OSV-Scanner | Nothing beyond GitHub, in the case of Dependabot | Online OSV-Scanner sends package names and versions to the OSV API |
| Secrets | GitHub push protection or Gitleaks in CI | Repository administrator permission | Output with `--redact` |
| Images and IaC | Trivy, Checkov | Vulnerability database (download) | Pin the version and check what is downloaded |
| Fork PRs | Repository policy | — | Never `pull_request_target` with checkout of the PR code and secrets; minimal `permissions` per job |

Before enabling: preserve the existing workflows, choose who receives the alerts, define what blocks a merge and test on a branch.
