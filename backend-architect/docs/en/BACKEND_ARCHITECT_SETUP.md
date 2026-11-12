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
// The command is split the way bash would split it, without expanding anything, and whatever bash would expand is refused;
// the check runs on the real arguments, with the program name normalized. Any error also denies.
// Git only with read-only subcommands; the rest is a list of what is not allowed: it stops accidents and common
// injected commands, it is not a security boundary (the agent edits code and runs tests, so it can run project code).
import { readFileSync, existsSync } from "node:fs";
import { join, dirname } from "node:path";

const deny = (reason) => {
  process.stderr.write(`architect: command blocked (${reason}). Run one command at a time, without cd; if you need this, ask the user or the main Claude.\n`);
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
        if (s[j] === "$" || s[j] === "`") deny("chaining, redirection or substitution");
        if (s[j] === "\\" && j + 1 < s.length && '"\\$`'.includes(s[j + 1])) { current += s[j + 1]; j += 2; continue; }
        current += s[j];
        j++;
      }
      if (j >= s.length) deny("unclosed quotes");
      i = j + 1;
      continue;
    }
    if (";&|<>$`".includes(c)) deny("chaining, redirection or substitution");
    if (c === "\\") deny("backslash outside quotes (put the path in quotes)");
    if ("*?[]{}()!".includes(c)) deny("shell special character outside quotes (put it in quotes)");
    current += c;
    i++;
  }
  if (current !== null) args.push(current);
  return args;
}

// Program name: without folder, without Windows extension, in lowercase and without a trailing version (python3.12 becomes python).
function nameOf(p) {
  const raw = p.replace(/^.*[\\/]/, "").toLowerCase();
  const n = raw.replace(/\.(exe|cmd|bat|ps1|sh)$/, "").replace(/\d+(\.\d+)*$/, "").replace(/[-_.]$/, "");
  if (!n) deny("empty or odd program name");
  return n === "nodejs" ? "node" : n;
}

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

// Words (arguments that are not options). Stops at "--" only when it comes after a subcommand that runs
// a script or program (the rest are that script's arguments); otherwise "--" hides nothing.
function wordsUntil(a, runsThings) {
  const out = [];
  for (const x of a) {
    if (x === "--") { if (out.some((w) => runsThings.has(w))) break; continue; }
    if (!x.startsWith("-")) out.push(x);
  }
  return out;
}

const FOLDER = new Set(["cd", "pushd", "popd", "set-location", "sl", "chdir"]);
const SHELLS = new Set(["bash", "sh", "zsh", "dash", "ksh", "fish", "csh", "tcsh", "cmd", "powershell", "pwsh", "wsl", "busybox",
  "env", "sudo", "doas", "su", "runas", "xargs", "nohup", "timeout", "nice", "ionice", "stdbuf", "command", "builtin", "exec",
  "eval", "source", ".", "time", "watch", "start", "start-process", "invoke-expression", "iex", "invoke-command", "icm", "script",
  "parallel", "setsid", "chroot", "unshare", "flock", "strace", "cross-env", "dotenv", "concurrently"]);
const ALWAYS_DOWNLOADS = new Set(["pnpx", "uvx", "pipx", "corepack"]);
const INLINE_CODE = {
  node: /^(-e|--eval|-p|--print|-pe|-ep)$|^--(eval|print)=|^data:|^--(import|loader|experimental-loader|require)=(data|https?):/,
  bun: /^(-e|--eval|-p|--print)$|^--(eval|print)=/,
  python: /^-[A-Za-z]*c/, py: /^-[A-Za-z]*c/, pythonw: /^-[A-Za-z]*c/,
  ruby: /^-[A-Za-z]*e/, perl: /^-[A-Za-z]*[eE]/, php: /^-r$/, lua: /^-e$/, rscript: /^-e$/, osascript: /^-e$/,
};
const GIT_READ = new Set(["status", "diff", "log", "show", "blame", "ls-files", "ls-tree", "rev-parse", "merge-base", "cat-file",
  "describe", "shortlog", "grep", "for-each-ref", "rev-list", "name-rev", "check-ignore", "check-attr", "count-objects", "version"]);
const GIT_DANGEROUS = ["--output", "--exec", "--upload-pack", "--receive-pack", "--config-env", "--ext-diff", "--textconv",
  "--open-files-in-pager", "--filters"];
const SAFE_EXACT = new Set(["--text"]);
// npm aliases and pnpm, yarn and bun equivalents that install, remove, publish or touch the account and registry.
const PACKAGE_DENIED = new Set(["i", "in", "ins", "inst", "insta", "instal", "install", "isnt", "isnta", "isntal", "isntall", "add",
  "ci", "clean-install", "ic", "install-clean", "isntall-clean", "it", "install-test", "cit", "clean-install-test", "sit",
  "install-ci-test", "uninstall", "unlink", "remove", "rm", "r", "un", "update", "up", "upgrade", "udpate", "publish", "unpublish",
  "link", "ln", "dedupe", "ddp", "prune", "rebuild", "rb", "import", "dlx", "create", "init", "explore", "edit", "login", "logout",
  "adduser", "add-user", "token", "owner", "access", "deprecate", "dist-tag", "team", "org", "star", "unstar", "hook", "profile"]);
const PM_VALUE = new Set(["--prefix", "-C", "--dir", "--cwd", "--filter", "-F", "--workspace", "--userconfig", "--globalconfig",
  "--cache", "--registry", "--loglevel", "--reporter"]);
const RUNS_SCRIPT = new Set(["run", "run-script", "rum", "urn", "test", "t", "tst", "exec", "x"]);
const SERVER = new Set(["dev", "start", "restart", "stop", "serve", "preview", "deploy", "release"]);
const NETWORK = new Set(["curl", "wget", "invoke-webrequest", "iwr", "invoke-restmethod", "irm", "ssh", "scp", "sftp", "rsync", "nc",
  "ncat", "netcat", "socat", "nmap", "ping", "telnet", "ftp", "tracert", "traceroute", "test-netconnection", "tnc", "openssl",
  "nslookup", "dig", "host", "gh", "hub", "glab"]);
const CLOUD = new Set(["docker", "docker-compose", "podman", "kubectl", "helm", "terraform", "tofu", "terragrunt", "pulumi",
  "ansible", "ansible-playbook", "systemctl", "service", "sc", "pm2", "nssm", "vercel", "netlify", "fly", "flyctl", "railway",
  "heroku", "az", "aws", "gcloud", "gsutil", "firebase", "wrangler", "supabase"]);
const REMOVES = new Set(["rm", "del", "erase", "rmdir", "rd", "remove-item", "ri", "shred", "unlink"]);
const LIST_ONLY = new Set(["ls", "dir", "get-childitem", "gci", "stat", "test", "which", "where", "command"]);
const MIGRATION = /prisma (migrate (deploy|reset|dev|resolve)|db (push|execute|seed))|dotnet ef database|knex migrate|sequelize db:|typeorm migration:run|alembic (upgrade|downgrade)|flask db (upgrade|downgrade)|rails db:/;

let CWD = process.cwd();

// Denies a dangerous long option, abbreviated too (git accepts a unique prefix). Stops at "--".
function checkOptions(args, dangerous) {
  for (const a of args) {
    if (a === "--") return;
    if (!a.startsWith("--")) continue;
    const name = a.split("=")[0];
    if (name.length > 2 && !SAFE_EXACT.has(name) && dangerous.some((p) => p.startsWith(name))) deny("option that executes or writes");
  }
}

// Does the git subcommand show file content (and not only names or statistics)?
function showsContent(sub, r) {
  const patch = r.some((x) => /^(-p|-u|--patch|--cc|-U\d*$|--unified|--word-diff|--color-words|--binary|--full-diff|-L)/.test(x));
  if (["blame", "cat-file", "grep"].includes(sub)) return true;
  if (sub === "show" || sub === "diff") return patch || !r.some((x) => /^(--stat|--name-only|--name-status|--numstat|--shortstat|--summary|--dirstat|--raw|--no-patch|-s$|--quiet)/.test(x));
  if (sub === "log") return patch;
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
function checkGit(a) {
  let i = 0;
  while (i < a.length && a[i].startsWith("-")) {
    const o = a[i];
    if (o === "-c" || o.startsWith("--config-env") || o.startsWith("--exec-path")) deny("inline git configuration (can run a command)");
    i += ["-C", "--git-dir", "--work-tree", "--namespace", "--super-prefix"].includes(o) ? 2 : 1;
  }
  const sub = a[i];
  const r = a.slice(i + 1);
  if (sub === undefined) return;
  const pos = r.filter((x) => !x.startsWith("-"));
  let ok = GIT_READ.has(sub);
  if (sub === "branch") ok = !r.some((x) => /^(-d|-D|--delete|-m|-M|--move|-c|-C|--copy|-f|--force|-u|--set-upstream-to|--unset-upstream|--edit-description)/.test(x))
    && (pos.length === 0 || r.some((x) => /^(--list|-l|-a|--all|-r|--remotes|-v|-vv|--contains|--no-contains|--merged|--no-merged|--points-at)/.test(x)));
  if (sub === "tag") ok = !r.some((x) => /^(-a|-s|-u|-f|-d|-m|-F|--annotate|--sign|--local-user|--force|--delete|--message|--file)/.test(x))
    && (pos.length === 0 || r.some((x) => /^(--list|-l|--contains|--no-contains|--merged|--no-merged|--points-at|-n)/.test(x)));
  if (sub === "remote") ok = r.length === 0 || (r.length === 1 && r[0] === "-v") || r[0] === "get-url";
  if (sub === "config") ok = r.some((x) => /^(--get|--get-all|--get-regexp|--list|-l|--show-origin|--show-scope)$/.test(x));
  if (sub === "stash" || sub === "worktree") ok = r[0] === "list";
  if (sub === "reflog") ok = r[0] === undefined || r[0] === "show" || (r[0].startsWith("-") && !r.includes("expire") && !r.includes("delete"));
  if (!ok) deny("Git not on the read-only list (commit, push, checkout and the like stay with the user)");
  checkOptions(r, GIT_DANGEROUS);
  if (sub === "grep" && r.some((x) => /^-[A-Za-z]*O/.test(x))) deny("option that executes or writes");
  if (showsContent(sub, r) && r.some(isSecret)) deny("reading a secret file");
  if (showsContent(sub, r) && r.some(isTrickyPathspec)) deny("glob or special pathspec in a command that shows content");
  if (sub === "grep" && r.some((x) => /^--(no-index|untracked|no-exclude-standard)/.test(x))) deny("git grep outside tracked files (may read a secret)");
}

// Is the package in node_modules/.bin of this folder or of a folder above?
// A scoped package (@scope/name) only counts if that exact package is installed; without a scope, the .bin is enough.
function localBin(pkg) {
  const scoped = pkg.startsWith("@");
  if (scoped ? !/^@[\w.-]+\/[\w.-]+$/.test(pkg) : !/^[\w.-]+$/.test(pkg)) return false;
  let d = CWD;
  for (let k = 0; k < 40; k++) {
    if (scoped && existsSync(join(d, "node_modules", ...pkg.split("/"), "package.json"))) return true;
    if (!scoped) for (const ext of ["", ".cmd", ".exe", ".ps1"]) if (existsSync(join(d, "node_modules", ".bin", pkg + ext))) return true;
    const parent = dirname(d);
    if (parent === d) break;
    d = parent;
  }
  return false;
}

// npx, bunx, npm exec, bun x (may download) and pnpm exec, yarn exec (local only): checks the package and the inner command.
function checkRunner(a, localOnly) {
  if (a.some((x) => /^(-y|--yes|-p|--package|-c|--call)$|^--(package|call|yes)=/.test(x))) deny("running a package downloaded on the fly or an embedded command");
  const noInstall = a.some((x) => x === "--no" || x === "--no-install");
  const i = a.findIndex((x) => !x.startsWith("-"));
  if (i < 0) return;
  const pkg = a[i];
  if (/^.+@/.test(pkg)) deny("package with a version, downloaded on the fly");
  if ((pkg === "playwright" || pkg === "cypress") && a[i + 1] === "install") deny("browser download");
  if (!localOnly && !noInstall && !localBin(pkg)) deny("a package outside the project's node_modules would be downloaded on the fly (use npx --no or ask the user)");
  check(a.slice(i));
}

// pip, uv, poetry and the like: "run" executes what comes after; the rest is checked word by word.
function checkSuffixes(a) {
  for (let k = 0; k < a.length; k++) if (!a[k].startsWith("-")) check(a.slice(k));
}

function checkManager(prog, a) {
  let i = 0;
  while (i < a.length && a[i].startsWith("-")) i += PM_VALUE.has(a[i]) ? 2 : 1;
  const sub = a[i];
  if (sub === undefined) {
    if (prog === "yarn" && !a.some((x) => /^(-v|--version|-h|--help)$/.test(x))) deny("package install or publish");
    return;
  }
  if (sub === "exec" || (sub === "x" && prog !== "yarn")) {
    // a download or embedded-command option can come before the subcommand (npm --yes exec x)
    if (a.slice(0, i).some((x) => /^(-y|--yes|-p|--package|-c|--call)$|^--(package|call|yes)=/.test(x))) deny("running a package downloaded on the fly or an embedded command");
    return checkRunner(a.slice(i + 1), prog === "pnpm" || prog === "yarn");
  }
  const w = wordsUntil(a, RUNS_SCRIPT);
  if (w.some((x) => PACKAGE_DENIED.has(x))) deny("package install or publish, or registry account");
  if (w.includes("audit") && w.includes("fix")) deny("package install or publish");
  if (w.includes("version") && w.length > w.indexOf("version") + 1) deny("npm version creates a commit and a tag");
  if (w.includes("config") && w.some((x) => ["set", "delete", "edit"].includes(x))) deny("package manager configuration");
  if (prog === "yarn" && w.includes("set")) deny("package install or publish");
  const script = ["run", "run-script", "rum", "urn"].includes(sub) ? a.slice(i + 1).find((x) => !x.startsWith("-")) : sub;
  if (SERVER.has(script)) deny("server or deploy");
}

function check(argv) {
  if (!argv.length || !argv[0]) deny("empty");
  if (argv[0].includes("=")) deny("environment variable before the command");
  const prog = nameOf(argv[0]);
  const a = argv.slice(1);
  const text = [prog, ...a].join(" ");

  if (FOLDER.has(prog)) deny("changing folder (use git -C <folder> or the tool's folder argument)");
  if (prog === "command" && a.length === 2 && (a[0] === "-v" || a[0] === "-V")) return; // only says whether the program exists
  if (SHELLS.has(prog)) deny("shell or command that wraps another (run the command directly)");
  if (ALWAYS_DOWNLOADS.has(prog)) deny("running a package downloaded on the fly");
  if (prog === "printenv" || ((prog === "set" || prog === "export" || prog === "declare" || prog === "typeset") && (a.length === 0 || a.includes("-p")))) deny("environment variables (may hold secrets)");
  if (["get-childitem", "gci", "dir", "ls", "get-item", "gi"].includes(prog) && a.some((x) => /^env:/i.test(x))) deny("environment variables (may hold secrets)");
  if (INLINE_CODE[prog] && a.some((x) => INLINE_CODE[prog].test(x))) deny("inline code on the command line (use a project file or ask the user)");
  if (prog === "deno" && (a.includes("eval") || a.some((x) => /^(https?|data|npm|jsr):/i.test(x)))) deny("inline code or code downloaded on the fly");
  if (prog === "git") return checkGit(a);
  if (!LIST_ONLY.has(prog) && a.some(isSecret)) deny("reading a secret file");

  if (prog === "npx" || prog === "bunx") return checkRunner(a, false);
  if (["npm", "pnpm", "yarn", "bun"].includes(prog)) return checkManager(prog, a);
  if (prog === "python" || prog === "py" || prog === "pythonw") {
    for (let k = 0; k < a.length; k++) {
      const x = a[k];
      if (!x.startsWith("-") || x === "-") break;
      if (/^-[A-Za-z]*[WX]$/.test(x)) { k++; continue; }
      const m = /^-[A-Za-z]*m(.*)$/.exec(x);
      if (m) { if (["pip", "ensurepip"].includes(m[1] || a[k + 1])) deny("Python package install"); break; }
    }
  }
  if (["pip", "uv", "poetry", "pipenv", "conda", "pdm", "hatch", "rye"].includes(prog)) {
    const w = wordsUntil(a, new Set(["run"]));
    if (w[0] === "run") {
      if (a.some((x) => /^--with/.test(x))) deny("running a package downloaded on the fly");
      checkSuffixes(a.slice(a.indexOf("run") + 1));
    } else if (w.some((x) => ["install", "add", "remove", "uninstall", "update", "sync", "lock", "upgrade"].includes(x))) deny("Python package install");
  }
  if (prog === "dotnet") {
    const w = wordsUntil(a, new Set(["run", "test", "exec"]));
    if (w[0] !== "ef" && w.some((x) => ["add", "remove", "restore", "tool", "workload", "nuget"].includes(x))) deny(".NET package");
  }
  if (prog === "go" && wordsUntil(a, new Set()).some((x, k, p) => (["get", "install"].includes(x)) || (p[0] === "run" && x.includes("@")))) deny("package install or package downloaded on the fly");
  if (prog === "cargo" && wordsUntil(a, new Set(["run", "test"])).some((x) => ["add", "install", "publish", "update"].includes(x))) deny("package install");
  if (prog === "gem" && wordsUntil(a, new Set(["exec"])).includes("install")) deny("package install");
  if (prog === "composer" && wordsUntil(a, new Set(["run", "run-script", "exec"])).some((x) => ["require", "install", "update"].includes(x))) deny("package install");
  if (MIGRATION.test(text)) deny("migration or database change");
  if (NETWORK.has(prog)) deny("network");
  if (CLOUD.has(prog)) deny("deploy, container, cluster, cloud or service");
  if (REMOVES.has(prog)) deny("file removal (use Edit or Write; delete only on request)");
  if (prog === "find" && a.some((x) => /^-(delete|exec|execdir|ok|okdir|fprint|fprint0|fprintf|fls)$/.test(x))) deny("find that deletes, executes or writes");
}

try {
  const input = JSON.parse(readFileSync(0, "utf8"));
  if (typeof input?.cwd === "string" && input.cwd) CWD = input.cwd;
  const raw = input?.tool_input?.command;
  if (typeof raw !== "string") deny("input without a text command");
  const command = raw.trim();
  if (!command) deny("empty");
  check(split(command));
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
          command: "node ~/.claude/hooks/architect-bash-guard.mjs"
```

On Windows, if `~` is not expanded by the hook's shell, use the absolute path (`node C:/Users/<you>/.claude/hooks/architect-bash-guard.mjs`).

Effect: commit, push, install and migration become requests to the user or the main Claude. The script splits the command the way bash would and refuses whatever bash would expand, so quotes, the program's path, a version in the name (`python3.12`), a git global option and inline code (`node -e`, `python -c`) do not work as a bypass. In git only read-only subcommands pass, and none that shows the content of a secret file (`git show HEAD:.env`). `npx` (and `npm exec`, `bunx`) only runs a package that is already in the project's `node_modules/.bin`, or with `--no`: without a terminal, npx would download the package without asking. Any error inside the script denies the command. A scoped package (`@scope/name`) must be installed under that exact name, and `--yes` before the subcommand (`npm --yes exec`) is denied too. Secret limit: a secret that is already committed shows up in any `git show <commit>` or `git log -p` that changes it, because the agent needs to read diffs to do its job; the hook only denies asking for it by name, with a glob or with a special pathspec. The defense is not committing secrets (the agent flags it when it finds one) and the rule of never reproducing a value. Tested with 238 fictitious commands (61 allowed, 177 denied) and 6 invalid inputs (all denied), without running any of them. A list of what is not allowed is not a boundary: a program called by another name escapes; the rules in section 5.1 and reviewing the diff still apply.

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
