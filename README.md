# SAP Dev Kit

Command-line toolkit for ABAP development on SAP systems.

Sync code from SAP to local files, edit in any editor, branch and review with git, run quality checks in your pipeline, and push changes back - all from the terminal.

102 commands. Single binary. No runtime dependencies. No SAP-side installation.

Works with S/4HANA, ECC, and NetWeaver AS ABAP 7.50+ - any system with ADT endpoints enabled.

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
sapdev workspace init ZFINANCE                # sync package from SAP
cd abap/ZFINANCE
git init && git add -A && git commit -m "Initial sync"

git checkout -b fix/exchange-rate              # branch for the fix
vim zcl_journal_entry.clas.abap               # edit in any editor

sapdev workspace diff S4H/ZFINANCE            # diff against SAP baseline
sapdev check syntax ZCL_JOURNAL_ENTRY --source zcl_journal_entry.clas.abap
sapdev check atc ZCL_JOURNAL_ENTRY            # validate before pushing

git add -A && git commit -m "Fix exchange rate"
git push origin fix/exchange-rate             # push branch for code review

# After approval
sapdev workspace push S4H/ZFINANCE            # push to SAP
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
sapdev workspace init ZLEGACY                 # sync package
cd abap/ZLEGACY
git init && git add -A && git commit -m "Initial sync"

sapdev clean-core assess ZLEGACY              # Clean Core scan (levels A/B/C/D)
sapdev check atc ZCL_LEGACY_REPORT --variant YOUR_VARIANT  # or any ATC variant

sapdev clean-core prep S4H/ZLEGACY            # download fix context for D/C objects

git checkout -b fix/zcl-legacy-report         # branch per fix
vim zcl_legacy_report.clas.abap               # replace deprecated API calls

sapdev check syntax ZCL_LEGACY_REPORT --source zcl_legacy_report.clas.abap
sapdev check atc ZCL_LEGACY_REPORT            # validate the fix

git add -A && git commit -m "Replace CALL FUNCTION with class-based API"
git push origin fix/zcl-legacy-report         # code review via pull request

# After approval
sapdev workspace push S4H/ZLEGACY            # push fix to SAP
sapdev clean-core assess ZLEGACY              # re-assess to confirm
```

See [Clean Core assessment and remediation](#clean-core-assessment-and-remediation) for the full command reference.

---

### 3. CI/CD Quality Gates - syntax, ATC, unit tests in your pipeline

Run syntax checks, ATC analysis, and unit tests from any CI system. Every command returns `--json` output and exit codes built for automation.

```
  git push          CI Pipeline                               SAP System
 ┌──────────┐      ┌──────────────────────────────────┐      ┌──────────┐
 │ Branch   │─────>│ sapdev check syntax ... --json   │<────>│  ADT     │
 │ pushed   │      │ sapdev check atc ...   --json   │      │  APIs    │
 └──────────┘      │ sapdev check unit ...  --json   │      └──────────┘
                   └──────────┬───────────────────────┘
                              │
                    0 = pass, 1 = issues, 2 = error
                              │
                   ┌──────────v───────────────────────┐
                   │ ✓ Merge    or    ✗ Block PR      │
                   └──────────────────────────────────┘
```

```bash
sapdev check syntax ZCL_EXAMPLE --json
sapdev check atc ZCL_EXAMPLE --variant CLEAN_CORE --json
sapdev check unit ZCL_EXAMPLE --json

# Check a local draft before uploading (no write to SAP)
sapdev check syntax ZCL_EXAMPLE --source local-draft.abap --json

# Fail the pipeline if ATC finds issues
sapdev check atc ZCL_EXAMPLE --json | jq -e '.findings | length == 0'
```

---

### 4. Explore and Understand Systems - navigate code from the terminal

Connect to any SAP system and browse, search, read source, navigate definitions, and preview data. No Eclipse required.

```
  You (terminal)                                SAP System
 ┌──────────────────────────────┐              ┌──────────────┐
 │ sapdev package list          │──── ADT ────>│ Package tree  │
 │ sapdev object tree ZFINANCE  │   HTTP/S     │ Object info   │
 │ sapdev source get ZCL_FOO    │<────────────>│ Source code   │
 │ sapdev code references ...   │              │ Definitions   │
 │ sapdev data query T000       │              │ Table data    │
 └──────────────────────────────┘              └──────────────┘
```

```bash
sapdev package list                            # discover Z*/Y* packages
sapdev object tree ZFINANCE                    # browse package contents
sapdev source get ZCL_JOURNAL_ENTRY            # read source code

sapdev code definition ZCL_JOURNAL_ENTRY --line 42 --col 10   # go to definition
sapdev code element-info ZCL_JOURNAL_ENTRY --line 42 --col 10  # type + docs
sapdev code references ZCL_JOURNAL_ENTRY                       # who uses this?

sapdev data query BKPF --rows 5 --where "BUKRS = '1000'"      # preview data
sapdev data sql --query "SELECT * FROM bkpf WHERE gjahr = '2025' UP TO 10 ROWS"
```

---

### 5. AI-Assisted Development - agents discover and use commands safely

Every command has machine-readable safety annotations. AI agents can explore what's available, distinguish reads from writes, preview before committing, and parse structured output.

```
  AI Agent (Claude, Cursor, Copilot, ...)
 ┌───────────────────────────────────────────────┐
 │ 1. sapdev tools --json        → discover ops  │
 │ 2. sapdev source get ZCL_FOO  → read source   │
 │ 3. sapdev check atc ZCL_FOO   → find issues   │
 │ 4. ... edit source locally ...                 │
 │ 5. sapdev source put --dry-run → preview       │
 │ 6. sapdev source put --yes     → apply         │
 └───────────────────────────────────────────────┘
         │                   ^
         │  shell commands   │  --json responses
         v                   │
 ┌───────────────────────────────────────────────┐
 │              SAP System (via ADT)             │
 └───────────────────────────────────────────────┘
```

```bash
sapdev tools --json    # full catalog with safety metadata per command
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

## Download

| Platform | Binary |
|----------|--------|
| **Linux** x64 | [sapdev](linux/sapdev) |
| **macOS** x64 | [sapdev](macos/sapdev) |
| **Windows** x64 | [sapdev.exe](windows/sapdev.exe) |

```bash
# Linux
curl -LO https://github.com/openkash/sap-dev-kit/raw/main/linux/sapdev
chmod +x sapdev && sudo mv sapdev /usr/local/bin/

# macOS
curl -LO https://github.com/openkash/sap-dev-kit/raw/main/macos/sapdev
chmod +x sapdev && sudo mv sapdev /usr/local/bin/

# Windows (PowerShell)
Invoke-WebRequest -Uri https://github.com/openkash/sap-dev-kit/raw/main/windows/sapdev.exe -OutFile sapdev.exe
```

Verify: `sapdev --version`

## Quick Start

```bash
sapdev init                          # create .sapdev.json template
# Edit .sapdev.json with your SAP host, SID, client, username (see Configuration below)
export SAPDEV_PASSWORD='your-password'
sapdev system-check                  # verify connectivity
```

---

## How This Compares to abapGit

abapGit is the established standard for ABAP + git. SAP Dev Kit takes a different approach.

**abapGit** - "SAP is the working directory, git is the archive." Runs as an ABAP program inside SAP. Full git client with branch and merge. Serializes complete object metadata as portable XML.

**SAP Dev Kit** - "Local files are the working directory, SAP is the remote." Runs on your workstation as a CLI binary. Syncs source code to local files; you use standard git directly.

| | abapGit | SAP Dev Kit |
|---|---------|-------------|
| **Runs where** | Inside SAP (ABAP program) | Your workstation (CLI binary) |
| **Installation** | Must install on each SAP system | Nothing on SAP - just needs ADT enabled |
| **Git integration** | Built-in git client | Syncs files to disk; you use git directly |
| **Object serialization** | Full metadata as portable XML | Source code (abapGit-compatible filenames) |
| **Scope** | Version control | Version control + quality checks + refactoring + code navigation + transport management + data preview |
| **CI/CD** | Requires SAP-side program | Every command has `--json` output and exit codes |
| **AI agent support** | Not designed for it | Tool annotations, `--json`, `--dry-run` |

They're complementary. Use abapGit when you need branch/merge operations managed inside SAP or portable cross-system deployment. Use SAP Dev Kit when you want to script from outside, run quality gates in a pipeline, or can't install programs on the SAP system.

---

## Detailed Feature Reference

### Workspace sync (git-like commands)

| sapdev command | git equivalent | What it does |
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
sapdev clean-core assess ZPACKAGE           # discover + classify (A/B/C/D)
sapdev clean-core prep S4H/ZPACKAGE         # download source + fix context for D/C objects
# ... AI agent or human edits .clas.abap / .prog.abap files ...
sapdev clean-core apply S4H/ZPACKAGE        # push fixes back to SAP (lock → write → activate)
```

```bash
sapdev clean-core report S4H/ZPACKAGE       # regenerate summary report
sapdev clean-core executive                 # cross-package executive dashboard
```

### Quality checks

```bash
sapdev check syntax ZCL_EXAMPLE                       # syntax check
sapdev check syntax ZCL_EXAMPLE --source draft.abap   # check draft (no write)
sapdev check atc ZCL_EXAMPLE --variant CLEAN_CORE     # ATC with specific variant
sapdev check unit ZCL_EXAMPLE                         # ABAP Unit tests
sapdev check cds-syntax ZSALES_I                      # CDS DDL syntax
sapdev check atc-exempt <markerId> --reason FPOS --justification "..."
```

### Code navigation

```bash
sapdev code definition ZCL_EXAMPLE --line 42 --col 10   # go to definition
sapdev code element-info ZCL_EXAMPLE --line 42 --col 10  # type info + docs
sapdev code references ZCL_EXAMPLE                       # where-used list
sapdev code snippets ZCL_EXAMPLE                         # code at each usage site
```

### Source code management

```bash
sapdev source get ZCL_EXAMPLE                    # download
sapdev source put ZCL_EXAMPLE --file src.abap    # upload (lock -> write -> unlock -> activate)
sapdev source push ZCL_FOO ZCL_BAR               # bulk upload with batch activation
sapdev source format < dirty.abap > clean.abap   # pretty-print via SAP formatter
```

### Object lifecycle

```bash
sapdev create class ZCL_NEW --package ZDEV --description "New class"
sapdev create ddl-source ZSALES_I --package ZDEV --description "Sales CDS view"
# 21 types: class, interface, program, include, function-group, function-module,
# table, structure, data-element, domain, ddl-source, access-control,
# metadata-extension, annotation-definition, service-definition, service-binding,
# message-class, auth-field, auth-object, package, ...

sapdev object search ZCL_* --type CLAS --package ZDEV
sapdev object tree ZDEV
sapdev object info ZCL_EXAMPLE
sapdev object history ZCL_EXAMPLE
sapdev object activate ZCL_FOO ZCL_BAR
sapdev object delete ZCL_OLD --dry-run
```

### Refactoring (evaluate -> preview -> execute)

```bash
sapdev refactor rename ZCL_EXAMPLE --line 10 --col 5                     # evaluate (read-only)
sapdev refactor rename ZCL_EXAMPLE --line 10 --col 5 --new-name better   # execute

sapdev refactor extract-method ZCL_EXAMPLE --start-line 10 --end-line 20 --include implementations
sapdev refactor extract-method ZCL_EXAMPLE --start-line 10 --end-line 20 --name helper

sapdev refactor move ZCL_EXAMPLE --package ZNEW_PKG --transport DEVK900001
sapdev refactor quickfix ZCL_EXAMPLE --line 10 --col 5 --apply 1
```

### Transport management

```bash
sapdev transport list
sapdev transport create --description "Fix #42" --package ZDEV
sapdev transport get DEVK900001
sapdev transport release DEVK900001
sapdev transport delete DEVK900001 --dry-run
```

### Data preview

```bash
sapdev data query T000 --rows 10 --where "MANDT = '100'"
sapdev data cds I_CURRENCY --rows 50
sapdev data sql --query "SELECT carrid, connid FROM sflight WHERE price > 500"
```

### DDIC and CDS metadata

```bash
sapdev ddic table T000                            # table definition + DDL fields
sapdev ddic data-element MANDT
sapdev ddic domain XFELD
sapdev ddic structure BAPI_USER_DETAIL
sapdev cds element-info I_BUSINESSPARTNER         # recursive field tree
sapdev cds repository-access I_BUSINESSPARTNER    # entity -> DDL source
sapdev cds annotations                            # annotation definitions
```

### Service bindings and behavior definitions

```bash
sapdev service publish ZSRV_SALES --version 0001
sapdev service unpublish ZSRV_SALES --version 0001
sapdev bdef get ZI_SALESORDER
sapdev bdef create ZI_SALESORDER --package ZDEV --description "Sales BDEF"
```

---

## Configuration

`sapdev init` creates a `.sapdev.json` template:

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
      "password_env": "SAPDEV_PASSWORD",
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
export SAPDEV_PASSWORD='your-password'
```

Multiple connections - switch with `-c`:

```bash
sapdev package list -c dev
sapdev package list -c prod
```

**Resolution order:** CLI flags > `.sapdev.json` > built-in defaults.

**Requirements:** ADT endpoints enabled on the SAP system (standard on S/4HANA; available on NetWeaver 7.50+). No programs, transports, or installations required on the SAP system - connects over HTTPS.

<details>
<summary>Optional config sections</summary>

```json
{
  "defaults": {
    "connection": "dev",
    "data_dir": "sapdev",
    "source_dir": "abap",
    "session_file": "sapdev/.session.json"
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

Run `sapdev <group> <command> --help` for detailed options and examples.

</details>

## License

Elastic License 2.0 (ELv2) - see [LICENSE](./LICENSE).

Free to use. Two restrictions: you may not (1) offer this software as a hosted/managed service, or (2) circumvent any license key functionality.
