# adtctl

Scriptable access to SAP systems from the terminal.

Sync code, run quality checks, manage transports, assess S/4HANA readiness, and query data. `--json` output on every command, composable with standard Unix tools. No SAP-side installation required.

102 commands. Single binary. No runtime dependencies. Works with S/4HANA, ECC, and NetWeaver AS ABAP 7.50+. Requires ADT endpoints enabled.

**Contents**
- [What You Can Do](#what-you-can-do) — 6 use cases with examples
- [Download](#download) — platform binaries + install
- [Quick Start](#quick-start) — first 3 minutes + troubleshooting
- [How This Compares to abapGit](#how-this-compares-to-abapgit)
- [Detailed Feature Reference](#detailed-feature-reference) — per-command guide
- [Configuration](#configuration) — `.adtctl.json` fields
- [Global Options](#global-options) — flags on every command
- [All Commands](#all-commands) — full catalog
- New users: see **[GETTING-STARTED.md](GETTING-STARTED.md)** for a guided walkthrough.

## What You Can Do

> **Placeholder convention:** `ZCL_YOUR_*`, `ZFINANCE`, `ZLEGACY`, `S4H`, and similar names in examples are placeholders — replace with real values from your system. Run `adtctl package list` to find a package you own.

### 1. ABAP Source in Git - branch, diff, review, push

Sync packages from SAP to local files. Edit in any editor. Use git for version control, branching, and code review. Push changes back to SAP.

```
  SAP System                        Your Workstation                     GitHub
 ┌──────────┐    workspace init    ┌──────────────┐     git push       ┌──────────┐
 │  ABAP    │ ─────────────────>   │ Local Files  │ ────────────────>  │  Remote  │
 │  Objects │                      │  + git repo  │                    │   Repo   │
 └──────────┘    workspace push    └──────────────┘     git pull       └──────────┘
       ^     <─────────────────           │         <────────────────        │
       │                                  │                                  │
       │      workspace pull              v                                  v
       └──────────────────────     Edit in any editor           Pull Request / Review
                                   Branch per fix
                                   Diff + review
```

```bash
adtctl workspace init ZFINANCE                # sync package from SAP
cd abap/S4H/ZFINANCE
git init && git add -A && git commit -m "Initial sync"

git checkout -b fix/exchange-rate              # branch for the fix
vim zcl_your_class.clas.abap               # edit in any editor

adtctl workspace diff S4H/ZFINANCE            # diff against SAP baseline
adtctl check syntax ZCL_YOUR_CLASS --source zcl_your_class.clas.abap
adtctl check atc ZCL_YOUR_CLASS            # validate before pushing

git add -A && git commit -m "Fix exchange rate"
git push origin fix/exchange-rate             # push branch for code review

# After approval
adtctl workspace push S4H/ZFINANCE            # push to SAP
```

---

### 2. S/4HANA Readiness and Clean Core Compliance - assess, fix on branch, review, verify

Run ATC with any variant - Clean Core, S/4HANA readiness, or custom. Classify objects by compliance level, fix violations on a git branch, review via pull request, and re-assess to confirm.

```
  Assess            Branch & Fix         Review             Push & Verify
 ┌─────────┐       ┌─────────────┐      ┌──────────┐      ┌─────────────┐
 │ ATC     │       │ git branch  │      │ Pull     │      │ workspace   │
 │ Scan    │──────>│ Edit source │─────>│ Request  │─────>│ push        │
 │         │       │ Check local │      │ Approve  │      │ Re-assess   │
 │         │       │ Commit      │      │ Merge    │      │ Confirm fix │
 └─────────┘       └─────────────┘      └──────────┘      └─────────────┘
  Level A/B/C/D     Fix one or more      Reviewer sees      Fewer D findings
                    objects per branch   the exact diff
```

```bash
adtctl workspace init ZLEGACY                 # sync package
cd abap/S4H/ZLEGACY
git init && git add -A && git commit -m "Initial sync"

adtctl clean-core assess ZLEGACY              # Clean Core scan (levels A/B/C/D)
adtctl check atc ZCL_YOUR_REPORT --variant YOUR_VARIANT  # or any ATC variant

adtctl clean-core prep S4H/ZLEGACY            # download fix context for D/C objects

git checkout -b fix/zcl-legacy-report         # branch per fix
vim zcl_your_report.clas.abap               # replace deprecated API calls

adtctl check syntax ZCL_YOUR_REPORT --source zcl_your_report.clas.abap
adtctl check atc ZCL_YOUR_REPORT            # validate the fix

git add -A && git commit -m "Replace CALL FUNCTION with class-based API"
git push origin fix/zcl-legacy-report         # code review via pull request

# After approval
adtctl workspace push S4H/ZLEGACY            # push fix to SAP
adtctl clean-core assess ZLEGACY              # re-assess to confirm
```

See [Clean Core assessment and remediation](#clean-core-assessment-and-remediation) for the full command reference.

---

### 3. CI/CD Quality Gates - syntax, ATC, unit tests in your pipeline

Run syntax checks, ATC analysis, and unit tests from any CI system. Every command returns `--json` output and exit codes built for automation.

```
  git push          CI Pipeline                               SAP System
 ┌──────────┐      ┌──────────────────────────────────┐      ┌──────────┐
 │ Branch   │─────>│ adtctl check syntax ... --json   │─────>│  ADT     │
 │ pushed   │      │ adtctl check atc ...   --json   │      │  APIs    │
 └──────────┘      │ adtctl check unit ...  --json   │      └──────────┘
                   └──────────┬───────────────────────┘
                              │
                    0 = pass, 1 = issues, 2 = error
                              │
                   ┌──────────v───────────────────────┐
                   │ ✓ Merge    or    ✗ Block PR      │
                   └──────────────────────────────────┘
```

```bash
adtctl check syntax ZCL_EXAMPLE --json
adtctl check atc ZCL_EXAMPLE --variant CLEAN_CORE --json
adtctl check unit ZCL_EXAMPLE --json

# Check a local draft before uploading (no write to SAP)
adtctl check syntax ZCL_EXAMPLE --source local-draft.abap --json

# Fail the pipeline if ATC finds issues
adtctl check atc ZCL_EXAMPLE --json | jq -e '.findings | length == 0'
```

---

### 4. Explore and Understand Systems - navigate code from the terminal

Connect to any SAP system and browse, search, read source, navigate definitions, and preview data. No Eclipse required.

```
  You (terminal)                                SAP System
 ┌──────────────────────────────┐              ┌──────────────┐
 │ adtctl package list          │──── ADT ────>│ Package tree  │
 │ adtctl object tree ZFINANCE  │   HTTP/S     │ Object info   │
 │ adtctl source get ZCL_FOO    │<────────────>│ Source code   │
 │ adtctl code references ...   │              │ Definitions   │
 │ adtctl data query T000       │              │ Table data    │
 └──────────────────────────────┘              └──────────────┘
```

```bash
adtctl package list                            # discover Z*/Y* packages
adtctl object tree ZFINANCE                    # browse package contents
adtctl source get ZCL_YOUR_CLASS            # read source code

adtctl code definition ZCL_YOUR_CLASS --line 42 --col 10   # go to definition
adtctl code element-info ZCL_YOUR_CLASS --line 42 --col 10  # type + docs
adtctl code references ZCL_YOUR_CLASS                       # who uses this?

adtctl data query BKPF --rows 5 --where "BUKRS = '1000'"      # preview data
adtctl data sql --query "SELECT * FROM bkpf WHERE gjahr = '2025' UP TO 10 ROWS"
```

---

### 5. AI-Assisted Development - agents discover and use commands safely

Every command has machine-readable safety annotations. AI agents can explore what's available, distinguish reads from writes, preview before committing, and parse structured output.

```
  AI Agent (Claude, Cursor, Copilot, ...)
 ┌───────────────────────────────────────────────┐
 │ 1. adtctl tools --json        → discover ops  │
 │ 2. adtctl source get ZCL_FOO  → read source   │
 │ 3. adtctl check atc ZCL_FOO   → find issues   │
 │ 4. ... edit source locally ...                 │
 │ 5. adtctl source put --dry-run → preview       │
 │ 6. adtctl source put --yes     → apply         │
 └───────────────────────────────────────────────┘
         │                   ^
         │  shell commands   │  --json responses
         v                   │
 ┌───────────────────────────────────────────────┐
 │              SAP System (via ADT)             │
 └───────────────────────────────────────────────┘
```

```bash
adtctl tools --json    # full catalog with safety metadata per command
```

```json
{
  "command": "source put",
  "readOnly": false, "destructive": true, "idempotent": false,
  "note": "prompts for confirmation; read-only with --dry-run"
}
```

Works with any agent that has shell access. No MCP server required.

---

### 6. Automate at Scale - governance, migration, and system intelligence

Every command outputs JSON and returns exit codes. Pipe, filter, loop, schedule.

**Technical debt dashboard** - track compliance scores over time:

```bash
# Weekly: assess all custom packages, feed into Grafana
mkdir -p "scores/$(date +%Y-%m-%d)"
for pkg in $(adtctl package list --json | jq -r '.[].name'); do
  adtctl clean-core assess $pkg --json > "scores/$(date +%Y-%m-%d)/${pkg}.json"
done
adtctl clean-core executive    # aggregate cross-package report
```

**Multi-system divergence detection** - catch drift between DEV, QAS, PRD:

```bash
for obj in ZCL_JOURNAL ZCL_POSTING ZCL_APPROVAL; do
  adtctl source get $obj -c dev > "dev/${obj}.abap"
  adtctl source get $obj -c prd > "prd/${obj}.abap"
  diff "dev/${obj}.abap" "prd/${obj}.abap" || echo "DIVERGED: $obj"
done
```

**Dependency mapping** - build a graph of custom code:

```bash
for obj in $(adtctl object search "ZCL_*" --type CLAS --json | jq -r '.[].name'); do
  adtctl code references $obj --json >> dependency-graph.json
done
# Feed into Neo4j, answer: "what breaks if we delete ZCL_LEGACY_HELPER?"
```

---

## Download

**Requires:** ADT endpoints enabled on the SAP system (standard on S/4HANA; available on NetWeaver 7.50+). Nothing installed on SAP — adtctl connects over HTTPS.

| Platform | Binary |
|----------|--------|
| **Linux** x64 | [adtctl](linux/adtctl) |
| **macOS** ARM64 (Apple Silicon) | [adtctl](macos/adtctl) |
| **macOS** x64 (Intel) | [adtctl](macos-x64/adtctl) |
| **Windows** x64 | [adtctl.exe](windows/adtctl.exe) |

```bash
# Linux
curl -LO https://github.com/openkash/adtctl/raw/main/linux/adtctl
chmod +x adtctl && sudo mv adtctl /usr/local/bin/

# macOS (Apple Silicon — M1/M2/M3/M4)
curl -LO https://github.com/openkash/adtctl/raw/main/macos/adtctl
chmod +x adtctl && sudo mv adtctl /usr/local/bin/

# macOS (Intel)
curl -LO https://github.com/openkash/adtctl/raw/main/macos-x64/adtctl
chmod +x adtctl && sudo mv adtctl /usr/local/bin/

# Windows (PowerShell)
Invoke-WebRequest -Uri https://github.com/openkash/adtctl/raw/main/windows/adtctl.exe -OutFile adtctl.exe
```

**macOS only — clear quarantine:** the binary isn't yet code-signed, so Gatekeeper blocks it. One-time fix after install:

```bash
sudo xattr -d com.apple.quarantine /usr/local/bin/adtctl
```

Alternatively: right-click the binary in Finder → Open → confirm "Open anyway".

Verify: `adtctl --version`

## Quick Start

**First 3 minutes** — prove it works:

```bash
adtctl init                          # create .adtctl.json template
# Edit .adtctl.json with your SAP host, SID, client, username
export ADTCTL_PASSWORD='your-password'

adtctl system-check                  # verify connectivity + auth
adtctl package list                  # lists Z*/Y* packages — your first real output
```

**Next 5 minutes** — prove you can write:

```bash
# Browse a package (ZSFLIGHT is SAP's standard demo, available on most S/4 systems)
adtctl object tree ZSFLIGHT

# Pre-flight: confirm your user has lock/activate authorization on a sandbox object
adtctl system-check ZCL_YOUR_SANDBOX_CLASS --write

# Try a read-only sync
adtctl workspace init ZYOUR_PACKAGE --dry-run
```

**Learn the tool:**

```bash
adtctl recipes                       # guided multi-step workflows
adtctl discover                      # which ADT services does your system expose?
adtctl tools                         # every command with safety annotations
```

### Exit codes (for scripts and CI)

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Partial failure — check found issues, some objects failed |
| `2` | Error — bad config, auth failure, SAP error |

### Batch writes

Every write command prompts for confirmation. In scripts, pass `-y`/`--yes` to skip prompts, or `--dry-run` to preview without writing.

### Troubleshooting first-run failures

| Symptom | Likely cause |
|---------|--------------|
| `401 Unauthorized` | `ADTCTL_PASSWORD` not exported, or wrong client/username |
| `404` on every command | ADT services not enabled on the system (ask Basis) |
| `Certificate verification failed` | Self-signed cert — set `NODE_EXTRA_CA_CERTS=/path/to/ca.pem` |
| `CSRF token required` loops | Stale session file — delete `.adtctl/.session.json` and retry |
| `workspace init` hangs on huge package | Narrow it with `--filter`, `--type CLAS`, or `--depth 1` |
| Workshop / shared system unstable | Lower concurrency: `workspace init --concurrency 1` |

**New to adtctl? See [GETTING-STARTED.md](GETTING-STARTED.md) for a guided 20-minute walkthrough.**

---

## How This Compares to abapGit

abapGit is the established standard for ABAP + git. adtctl takes a different approach.

**abapGit** - "SAP is the working directory, git is the archive." Runs as an ABAP program inside SAP. Full git client with branch and merge. Serializes complete object metadata as portable XML.

**adtctl** - "Local files are the working directory, SAP is the remote." Runs on your workstation as a CLI binary. Syncs source code to local files; you use standard git directly.

| | abapGit | adtctl |
|---|---------|-------------|
| **Runs where** | Inside SAP (ABAP program) | Your workstation (CLI binary) |
| **Installation** | Must install on each SAP system | Nothing on SAP - just needs ADT enabled |
| **Git integration** | Built-in git client | Syncs files to disk; you use git directly |
| **Object serialization** | Full metadata as portable XML | Source code (abapGit-compatible filenames) |
| **Scope** | Version control | Version control + quality checks + refactoring + code navigation + transport management + data preview |
| **CI/CD** | Requires SAP-side program | Every command has `--json` output and exit codes |
| **AI agent support** | Not designed for it | Tool annotations, `--json`, `--dry-run` |

They're complementary. Use abapGit when you need branch/merge operations managed inside SAP or portable cross-system deployment. Use adtctl when you want to script from outside, run quality gates in a pipeline, or can't install programs on the SAP system.

---

## Detailed Feature Reference

### Workspace sync (git-like commands)

Workspace paths use `{SID}/{PACKAGE}` where SID is your SAP system's 3-letter ID (e.g. `S4H`, `DEV`) — this lets you work with multiple systems side by side without collision.

| adtctl command | git equivalent | What it does |
|----------------|---------------|--------------|
| `workspace init` | `git clone` | Download all objects from a package |
| `workspace status` | `git status` | Show modified / clean / missing objects |
| `workspace diff` | `git diff` | Unified diff against SAP baseline |
| `workspace pull` | `git pull` | Refresh from SAP (with conflict detection) |
| `workspace push` | `git push` | Upload local changes to SAP |
| `workspace reset` | `git checkout .` | Discard local changes, restore SAP baseline |
| `workspace refresh` | — | Discover and pull new objects added to the package |
| `workspace add` | — | Add specific objects to the workspace |
| `workspace remove` | — | Stop tracking objects |

All workspace write commands (init, pull, push) save progress after each object — Ctrl+C stops gracefully without losing completed work. Re-run the same command to continue.

### Clean Core assessment and remediation

```bash
adtctl clean-core assess ZPACKAGE           # discover + classify (A/B/C/D)
adtctl clean-core prep S4H/ZPACKAGE         # download source + fix context for D/C objects
# ... edit .clas.abap / .prog.abap files ...
adtctl clean-core apply S4H/ZPACKAGE        # push fixes back to SAP (lock → write → activate)
```

```bash
adtctl clean-core report S4H/ZPACKAGE       # regenerate summary report
adtctl clean-core executive                 # cross-package executive dashboard
```

### Quality checks

```bash
adtctl check syntax ZCL_EXAMPLE                       # syntax check
adtctl check syntax ZCL_EXAMPLE --source draft.abap   # check draft (no write)
adtctl check atc ZCL_EXAMPLE --variant CLEAN_CORE     # ATC with specific variant
adtctl check unit ZCL_EXAMPLE                         # ABAP Unit tests
adtctl check cds-syntax ZSALES_I                      # CDS DDL syntax
adtctl check atc-exempt <markerId> --reason FPOS --justification "..."
```

### Code navigation

```bash
adtctl code definition ZCL_EXAMPLE --line 42 --col 10   # go to definition
adtctl code element-info ZCL_EXAMPLE --line 42 --col 10  # type info + docs
adtctl code references ZCL_EXAMPLE                       # where-used list
adtctl code snippets ZCL_EXAMPLE                         # code at each usage site
```

### Source code management

```bash
adtctl source get ZCL_EXAMPLE                    # download
adtctl source put ZCL_EXAMPLE --file src.abap    # upload (lock -> write -> unlock -> activate)
adtctl source push ZCL_FOO ZCL_BAR               # bulk upload with batch activation
adtctl source format < dirty.abap > clean.abap   # pretty-print via SAP formatter
```

### Object lifecycle

```bash
adtctl create class ZCL_NEW --package ZDEV --description "New class"
adtctl create ddl-source ZSALES_I --package ZDEV --description "Sales CDS view"
# 21 types: class, interface, program, include, function-group, function-module,
# function-group-include, table, structure, data-element, domain, ddl-source,
# access-control, metadata-extension, annotation-definition, service-definition,
# service-binding, message-class, auth-field, auth-object, package

adtctl object search ZCL_* --type CLAS --package ZDEV
adtctl object tree ZDEV
adtctl object info ZCL_EXAMPLE
adtctl object history ZCL_EXAMPLE
adtctl object activate ZCL_FOO ZCL_BAR
adtctl object delete ZCL_OLD --dry-run
```

### Refactoring (evaluate -> preview -> execute)

```bash
adtctl refactor rename ZCL_EXAMPLE --line 10 --col 5                     # evaluate (read-only)
adtctl refactor rename ZCL_EXAMPLE --line 10 --col 5 --new-name better   # execute

adtctl refactor extract-method ZCL_EXAMPLE --start-line 10 --end-line 20 --include implementations
adtctl refactor extract-method ZCL_EXAMPLE --start-line 10 --end-line 20 --name helper

adtctl refactor move ZCL_EXAMPLE --package ZNEW_PKG --transport DEVK900001
adtctl refactor quickfix ZCL_EXAMPLE --line 10 --col 5 --apply 1
```

### Transport management

```bash
adtctl transport list
adtctl transport create --description "Fix #42" --package ZDEV
adtctl transport get DEVK900001
adtctl transport release DEVK900001
adtctl transport delete DEVK900001 --dry-run
```

### Data preview

```bash
adtctl data query T000 --rows 10 --where "MANDT = '100'"
adtctl data cds I_CURRENCY --rows 50
adtctl data sql --query "SELECT carrid, connid FROM sflight WHERE price > 500"
```

### DDIC and CDS metadata

```bash
adtctl ddic table T000                            # table definition + DDL fields
adtctl ddic data-element MANDT
adtctl ddic domain XFELD
adtctl ddic structure BAPI_USER_DETAIL
adtctl cds element-info I_BUSINESSPARTNER         # recursive field tree
adtctl cds repository-access I_BUSINESSPARTNER    # entity -> DDL source
adtctl cds annotations                            # annotation definitions
```

### Service bindings and behavior definitions

```bash
adtctl service publish ZSRV_SALES --version 0001
adtctl service unpublish ZSRV_SALES --version 0001
adtctl bdef get ZI_SALESORDER
adtctl bdef create ZI_SALESORDER --package ZDEV --description "Sales BDEF"
```

---

## Configuration

Minimum working config — 4 fields:

```json
{
  "connections": {
    "dev": {
      "host": "sap-dev.example.com",
      "sid": "DEV",
      "client": "100",
      "username": "YOUR_USER"
    }
  }
}
```

Password is always read from an environment variable — never stored in config:

```bash
export ADTCTL_PASSWORD='your-password'
```

**Only `auth: basic` is implemented today.** The schema reserves `oauth` and `x509` for future use; both currently throw.

Multiple connections — switch with `-c`:

```bash
adtctl package list -c dev
adtctl package list -c prod
```

**Resolution order:** CLI flags > `.adtctl.json` > built-in defaults.

<details>
<summary>Optional connection fields</summary>

```json
{
  "connections": {
    "dev": {
      "host": "sap-dev.example.com",
      "sid": "DEV",
      "client": "100",
      "username": "YOUR_USER",
      "port": 44300,                     // default: 443 (secure) or 80
      "secure": true,                    // default: true
      "auth": "basic",                   // only 'basic' implemented today
      "password_env": "ADTCTL_PASSWORD", // env var holding the password
      "language": "EN"                   // default: EN
    }
  }
}
```

</details>

<details>
<summary>Optional top-level sections</summary>

```json
{
  "defaults": {
    "connection": "dev",
    "workspace_dir": "abap",
    "session_file": ".adtctl/.session.json"
  },
  "http": {
    "max_retries": 3,
    "retry_delay_ms": 1000,
    "retry_backoff_factor": 2,
    "circuit_breaker_threshold": 5
  }
}
```

</details>

## Global Options

| Flag | Description |
|------|-------------|
| `--json` | Structured JSON output (all commands) |
| `--verbose` | Debug-level output |
| `--log-file [path]` | Write log to file |
| `--log-level <level>` | File log level: `debug`, `info`, `warning`, `error` |
| `--session-file <path>` | Persist session across invocations |

Common per-command flags: `-c, --connection <name>` (select SAP connection), `--dry-run` (preview without side effects, on write commands), `-y, --yes` (skip confirmation prompts, on destructive commands).

**Don't commit `.adtctl/` to git.** It caches auth cookies and CSRF tokens. Add `.adtctl/` to your `.gitignore`.

## All Commands

<details>
<summary>102 commands across 17 groups</summary>

| Group | Commands |
|-------|----------|
| **object** | `info`, `search`, `tree`, `path`, `types`, `inactive`, `history`, `activate`, `delete` |
| **source** | `get`, `put`, `push`, `format`, `format-settings` |
| **code** | `definition`, `element-info`, `references`, `snippets` |
| **refactor** | `quickfix`, `rename`, `extract-method`, `move` |
| **create** | `class`, `interface`, `program`, `include`, `function-group`, `function-module`, `function-group-include`, `table`, `structure`, `data-element`, `domain`, `ddl-source`, `access-control`, `metadata-extension`, `annotation-definition`, `service-definition`, `service-binding`, `message-class`, `auth-field`, `auth-object`, `package` |
| **check** | `syntax`, `atc`, `unit`, `atc-variants`, `cds-syntax`, `atc-exempt-proposal`, `atc-exempt` |
| **transport** | `info`, `create`, `list`, `get`, `release`, `delete` |
| **data** | `query`, `cds`, `sql` |
| **workspace** | `init`, `status`, `list`, `pull`, `push`, `diff`, `add`, `remove`, `reset` |
| **clean-core** | `assess`, `report`, `executive`, `prep`, `apply` |
| **ddic** | `table`, `table-settings`, `data-element`, `domain`, `table-type`, `lock-object`, `type-group`, `structure`, `view` |
| **cds** | `annotations`, `element-info`, `repository-access`, `test-dependencies` |
| **service** | `publish`, `unpublish` |
| **bdef** | `get`, `create` |
| **package** | `list`, `lookup`, `exists` |
| **config** | `show`, `set-connection` |
| **reference** | `update` |
| *(standalone)* | `init`, `system-check`, `run`, `tools`, `discover`, `recipes` |

Run `adtctl <group> <command> --help` for detailed options and examples.

</details>

## License

Elastic License 2.0 (ELv2) - see [LICENSE](./LICENSE).

Free to use. Two restrictions: you may not (1) offer this software as a hosted/managed service, or (2) circumvent any license key functionality.
