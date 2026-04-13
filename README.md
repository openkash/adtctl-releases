# adtctl

Scriptable access to SAP systems from the terminal.

Sync code, run quality checks, manage transports, assess S/4HANA readiness, and query data. `--json` output on every command, composable with standard Unix tools. No SAP-side installation required.

102 commands. Single binary. No runtime dependencies. Works with S/4HANA, ECC, and NetWeaver AS ABAP 7.50+. Requires ADT endpoints enabled.

## What You Can Do

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
       └──────────────────────     Edit in VS Code              Pull Request / Review
                                   Branch per fix
                                   Diff + review
```

```bash
adtctl workspace init ZFINANCE                # sync package from SAP
cd abap/ZFINANCE
git init && git add -A && git commit -m "Initial sync"

git checkout -b fix/exchange-rate              # branch for the fix
vim zcl_journal_entry.clas.abap               # edit in any editor

adtctl workspace diff S4H/ZFINANCE            # diff against SAP baseline
adtctl check syntax ZCL_JOURNAL_ENTRY --source zcl_journal_entry.clas.abap
adtctl check atc ZCL_JOURNAL_ENTRY            # validate before pushing

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
cd abap/ZLEGACY
git init && git add -A && git commit -m "Initial sync"

adtctl clean-core assess ZLEGACY              # Clean Core scan (levels A/B/C/D)
adtctl check atc ZCL_LEGACY_REPORT --variant YOUR_VARIANT  # or any ATC variant

adtctl clean-core prep S4H/ZLEGACY            # download fix context for D/C objects

git checkout -b fix/zcl-legacy-report         # branch per fix
vim zcl_legacy_report.clas.abap               # replace deprecated API calls

adtctl check syntax ZCL_LEGACY_REPORT --source zcl_legacy_report.clas.abap
adtctl check atc ZCL_LEGACY_REPORT            # validate the fix

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
 │ Branch   │─────>│ adtctl check syntax ... --json   │<────>│  ADT     │
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
adtctl source get ZCL_JOURNAL_ENTRY            # read source code

adtctl code definition ZCL_JOURNAL_ENTRY --line 42 --col 10   # go to definition
adtctl code element-info ZCL_JOURNAL_ENTRY --line 42 --col 10  # type + docs
adtctl code references ZCL_JOURNAL_ENTRY                       # who uses this?

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
for pkg in $(adtctl package list --json | jq -r '.[].name'); do
  adtctl clean-core assess $pkg --json >> "scores/$(date +%Y-%m-%d).json"
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

**Assembly-line refactoring** - one PR per object:

```bash
adtctl clean-core prep S4H/ZLEGACY
cd clean-core/S4H/ZLEGACY/fix-context
for file in *.clas.abap; do
  obj=$(basename $file .clas.abap | tr 'a-z' 'A-Z')
  git checkout -b "fix/${obj}"
  # fix findings using .context/ metadata
  adtctl check syntax $obj --source $file --json    # validate
  git add -A && git commit -m "Fix: $obj"
  gh pr create --title "Clean core: $obj"
  git checkout main
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

Verify: `adtctl --version`

## Quick Start

```bash
adtctl init                          # create .adtctl.json template
# Edit .adtctl.json with your SAP host, SID, client, username (see Configuration below)
export ADTCTL_PASSWORD='your-password'
adtctl system-check                  # verify connectivity
```

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

| adtctl command | git equivalent | What it does |
|----------------|---------------|--------------|
| `workspace init` | `git clone` | Download all objects from a package |
| `workspace status` | `git status` | Show modified / clean / missing objects |
| `workspace diff` | `git diff` | Unified diff against SAP baseline |
| `workspace pull` | `git pull` | Refresh from SAP (with conflict detection) |
| `workspace push` | `git push` | Upload local changes to SAP |
| `workspace reset` | `git checkout .` | Discard local changes, restore SAP baseline |
| `workspace add` | - | Add new objects to the workspace |
| `workspace remove` | - | Stop tracking objects |

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
# table, structure, data-element, domain, ddl-source, access-control,
# metadata-extension, annotation-definition, service-definition, service-binding,
# message-class, auth-field, auth-object, package, ...

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

`adtctl init` creates a `.adtctl.json` template:

```json
{
  "connections": {
    "dev": {
      "host": "sap-dev.example.com",
      "port": 44300,
      "sid": "DEV",
      "client": "100",
      "secure": true,
      "auth": "basic",
      "username": "YOUR_USER",
      "password_env": "ADTCTL_PASSWORD",
      "language": "EN"
    }
  },
  "defaults": {
    "connection": "dev"
  }
}
```

Passwords are always read from environment variables, never stored in config:

```bash
export ADTCTL_PASSWORD='your-password'
```

Multiple connections - switch with `-c`:

```bash
adtctl package list -c dev
adtctl package list -c prod
```

**Resolution order:** CLI flags > `.adtctl.json` > built-in defaults.

**Requirements:** ADT endpoints enabled on the SAP system (standard on S/4HANA; available on NetWeaver 7.50+). No programs, transports, or installations required on the SAP system - connects over HTTPS.

<details>
<summary>Optional config sections</summary>

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
| `--dry-run` | Preview without side effects (write commands) |
| `--verbose` | Debug-level output |
| `--log-file [path]` | Write log to file |
| `--session-file <path>` | Persist session across invocations |
| `-c, --connection <name>` | Select SAP connection from config |
| `-y, --yes` | Skip confirmation prompts |

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
