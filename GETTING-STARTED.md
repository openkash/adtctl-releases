# Getting Started with adtctl

A guided 20-minute walkthrough using real SAP objects. By the end you will have:

- Confirmed your system and credentials work
- Read source code from a live ABAP object
- Run a quality check and seen real findings
- Previewed an upload (without writing anything)
- Learned where to go next for every other workflow

All examples below target `ZSFLIGHT`, SAP's standard flight booking demo package. It's pre-installed on most S/4HANA sandbox systems. Replace the package name if yours is different — `adtctl package list` shows what's available on your system.

---

## Before you start

You need:

1. The `adtctl` binary on your `PATH`. Test: `adtctl --version`.
2. An SAP system with ADT endpoints enabled (standard on S/4HANA, available on NetWeaver 7.50+).
3. SAP credentials — host, SID (3-letter system ID), client (3-digit number), username, password.

---

## Act 1 — Connect (3 minutes)

**Goal:** confirm `adtctl` can reach your system and authenticate.

```bash
adtctl init
```

```
Created config: .adtctl.json
Edit the file to add your SAP connection details.
```

The generated `.adtctl.json` has more fields than you need. Edit it down to the **minimum 4 fields**:

```json
{
  "connections": {
    "dev": {
      "host": "your-sap-host.example.com",
      "sid": "DEV",
      "client": "100",
      "username": "YOUR_USER"
    }
  }
}
```

Export your password to the environment — never commit it:

```bash
export ADTCTL_PASSWORD='your-password'
```

Test connectivity:

```bash
adtctl system-check
```

**Expected output:**

```
Connecting to your-sap-host.example.com...
Connection "dev" → DEV @ your-sap-host.example.com:443 (client 100, user YOUR_USER)

✓ connection     TCP reachable
✓ auth           CSRF token acquired

2/2 checks passed.
```

If you see `401 Unauthorized`, your password env var isn't set or the username/client is wrong. If you see `404` on every call, your system doesn't have ADT services enabled — ask your Basis team.

> **Tip:** run `adtctl discover` to see exactly which ADT services your system exposes. Not all systems have everything (e.g., ECC tends to lack batch activation and data preview).

---

## Act 2 — Browse (3 minutes)

**Goal:** read actual data from your system. This is Hello World.

```bash
adtctl package list
```

**Expected output** — a list of Z/Y packages on your system:

```
Connecting to your-sap-host.example.com...
Found 27 packages matching "Z*, Y*":

NAME                   DESCRIPTION
─────────────────────  ────────────────────────────────────────
Z001                   Customer development class
ZAIF_DEMO              Package for AIF demos
ZCUSTOM_DEVELOPMENT    Cloud-Ready Custom Development
ZSFLIGHT               SFLIGHT demo application objects
...
```

Browse the contents of the SFLIGHT demo package (or pick any package from your list):

```bash
adtctl object tree ZSFLIGHT
```

**Expected output** — a flat listing of every object in the package:

```
Connecting to your-sap-host.example.com...
TYPE       NAME                       DESCRIPTION
────────── ────────────────────────── ────────────────────────────────────────
FUGR/F     ZSFLIGHT_CUST              Customer master data
FUGR/F     ZSFLIGHT_TRVL              Travel agency master data
CLAS/OC    ZCL_SFLIGHT_BOOKING_DAO    Booking data access object
CLAS/OC    ZCL_SFLIGHT_BOOKING_MGR    Booking manager
CLAS/OC    ZCL_SFLIGHT_CALCULATOR     Price and tax calculations
INTF/OI    ZIF_SFLIGHT_BOOKING_SVC    Booking service contract
PROG/P     ZSFLIGHT_ALV_TEST          ALV test program
...
```

Read the source of one of those classes:

```bash
adtctl source get ZCL_SFLIGHT_BOOKING_DAO | head -20
```

**Expected output** — the actual ABAP source, printed to stdout:

```
Connecting to your-sap-host.example.com...
CLASS zcl_sflight_booking_dao DEFINITION PUBLIC FINAL CREATE PUBLIC.
* Feature: Booking data access object — reads + writes booking data
* DB access on DDIC tables — SELECT + UPDATE/INSERT/DELETE
* Expected: Level D (P1 findings for DML, P2 for SELECT)

  PUBLIC SECTION.
    TYPES:
      BEGIN OF ty_booking,
        carrid    TYPE s_carr_id,
        connid    TYPE s_conn_id,
        fldate    TYPE s_date,
        bookid    TYPE s_book_id,
        ...
```

> **Teaching moment:** write operations are always *explicit*. Reading never writes. If a command doesn't include `put`, `push`, `create`, `delete`, `apply`, or `release`, it's safe.

---

## Act 3 — Check (5 minutes)

**Goal:** run a quality check on a real object without modifying anything.

Syntax check against the live (active) version:

```bash
adtctl check syntax ZCL_SFLIGHT_BOOKING_DAO
```

**Expected output** (clean object):

```
Connecting to your-sap-host.example.com...
Resolving ZCL_SFLIGHT_BOOKING_DAO...
Checking active version of ZCL_SFLIGHT_BOOKING_DAO...
Syntax check: 0 message(s), ok
Object: ZCL_SFLIGHT_BOOKING_DAO — OK (no messages)
```

ATC (Code Inspector) is the richer check. Variants are site-specific; list them first:

```bash
adtctl check atc-variants | head -10
```

**Expected output:**

```
Connecting to your-sap-host.example.com...
222 ATC variants found.

VARIANT                         DESCRIPTION
──────────────────────────────  ────────────────────────────────────────
ABAP_CLOUD_DEVELOPMENT_DEFAULT  Default ATC variant for ABAP Cloud Development
CLEAN_CORE                      Clean Core compliance
...
```

Now run a check against the `CLEAN_CORE` variant (change the variant name if your system uses a different one):

```bash
adtctl check atc ZCL_SFLIGHT_BOOKING_DAO --variant CLEAN_CORE
```

**Expected output** (findings — SFLIGHT is intentionally "dirty" for demos):

```
Connecting to your-sap-host.example.com...
Resolving ZCL_SFLIGHT_BOOKING_DAO...
Running ATC check on ZCL_SFLIGHT_BOOKING_DAO (variant: CLEAN_CORE)...
ATC check: 6 finding(s), level D, NOT ok
Object: ZCL_SFLIGHT_BOOKING_DAO — Level D (Not Clean)
Findings: 6

  [P1] Usage of APIs — Updating DDIC database tables or DDIC table views is not allowed (...source/main#start=103,0)
  [P1] Usage of APIs — Updating DDIC database tables or DDIC table views is not allowed (...source/main#start=96,0)
  [P2] Usage of APIs — Reading from DDIC database tables or DDIC table views is not recommended (...source/main#start=65,0)
  ...
```

`[P1]`/`[P2]` are priority levels. `Level D` is the Clean Core classification — D means "not cloud-ready." Expected for SFLIGHT; the package exists to demonstrate findings.

> **Exit codes matter.** These commands return:
> - `0` = no issues
> - `1` = issues found (not a crash, just findings)
> - `2` = actual error (bad config, network, SAP rejected the request)
>
> That's the CI/CD contract. `adtctl check atc X && echo OK || echo FAIL`.

---

## Act 4 — Preview a write (5 minutes)

**Goal:** learn the safe-write pattern. *Preview first, apply second.*

Every write command has `--dry-run`. It validates parameters, resolves transports, and shows what would happen — without actually doing anything.

**Preflight — confirm your user has lock/activate authorization:**

```bash
adtctl system-check ZCL_SFLIGHT_BOOKING_DAO --write
```

> Note: `--write` on `system-check` actually runs *all* checks (connection, auth, read, write) because each one depends on the previous. The write check is just added to the list.

**Expected output:**

```
Connecting to your-sap-host.example.com...
Connection "dev" → DEV @ your-sap-host.example.com:443 (client 100, user YOUR_USER)

✓ connection     TCP reachable
✓ auth           CSRF token acquired
✓ read           ZCL_SFLIGHT_BOOKING_DAO → /sap/bc/adt/oo/classes/zcl_sflight_booking_dao
✓ write          lock/unlock OK

4/4 checks passed.
```

If you see `✗ write` — your user lacks `S_DEVELOP` authorization, or another user holds the lock. You want to catch this *now*, not mid-push.

**Now preview an upload.** Use a sandbox object you own — don't write to SFLIGHT (it's shared demo data):

```bash
# Download a sandbox class to a file, then edit it
adtctl source get ZCL_YOUR_SANDBOX_CLASS > sample.abap
# ... normally you'd edit sample.abap here ...

adtctl source put ZCL_YOUR_SANDBOX_CLASS --file sample.abap --dry-run
```

**Expected output:**

```
Connecting to your-sap-host.example.com...
Read 3933 bytes from sample.abap
Would write: CLAS/OC ZCL_SFLIGHT_BOOKING_DAO
  File: sample.abap (3933 bytes)
  Activate: true
```

If it looks right, drop `--dry-run` to apply. Write commands prompt for confirmation — pass `-y`/`--yes` in scripts to skip.

> **Pattern:** `--dry-run` for preview, `-y` for batch, neither for interactive work. Both flags are on every write command.

---

## Act 5 — Learn more (4 minutes)

**Goal:** know where to go next for whatever you're trying to do.

**`adtctl recipes`** — structured, parameterized workflows for multi-step jobs. Each recipe names the commands it uses and what to expect. Today the recipe library is small (one recipe, `clean-core-assess`); more will ship in future releases. Run:

```bash
adtctl recipes
```

**`adtctl discover`** — shows which ADT services your target system supports. Useful when a command returns 404 and you want to know if it's a system-capability issue or something else:

```bash
adtctl discover
```

**Expected output:**

```
Connecting to your-sap-host.example.com...
DEV — 800 ADT services discovered

  Core
    ✓ Object search                object search
    ✓ Object types catalog         object types
    ✓ Batch activation             object activate (multi)
    ✓ Transport management         transport list, transport get, transport create, ...
  Quality
    ✓ ATC checks                   check atc, clean-core assess
    ✓ Unit tests (async)           check unit
  ...
```

Service counts vary by system version (S/4HANA Cloud ≈800, on-premise S/4 ≈600, ECC much less).

**`adtctl tools`** — every command with read/write/idempotent annotations. Useful for scripting or letting an AI agent drive the tool. `adtctl tools --json` emits machine-readable output (~105 entries).

**`adtctl <group> <command> --help`** — per-command flags and examples.

**[README.md](README.md)** — full feature reference and `.adtctl.json` schema.

---

## Common workshop pitfalls

- **`workspace init` on a huge package** hangs for minutes. First run on a small package (<100 objects) to feel the rhythm. For large packages use `--filter <name-pattern>`, `--type CLAS`, or `--depth 1` to limit scope.
- **Shared SAP system, many attendees.** Pass `--concurrency 1` to `workspace init` to avoid overwhelming ICM. Default is 5 — fine alone, punishing at scale.
- **`--variant CLEAN_CORE` not found.** Variants are site-specific. Run `adtctl check atc-variants` to see what's available — most S/4 systems also have `ABAP_CLOUD_DEVELOPMENT_DEFAULT`.
- **Certificate errors.** If your SAP system uses a self-signed cert, set `NODE_EXTRA_CA_CERTS=/path/to/ca.pem` before running adtctl.
- **Stale session.** If CSRF errors loop, delete `.adtctl/.session.json` and retry — the next call re-authenticates from scratch.
- **Don't commit `.adtctl/`.** It caches auth cookies. Add it to `.gitignore`.

---

## What next

- **Using adtctl in CI/CD?** See README § CI/CD Quality Gates.
- **Syncing a whole package as a git repo?** See README § Workspace sync. Start small: `adtctl workspace init ZSMALL_PACKAGE --dry-run`.
- **Clean Core assessment?** `adtctl recipes clean-core-assess` walks through the `assess → executive` flow.
- **Running the tool from Claude, Cursor, or another AI agent?** See README § AI-Assisted Development. `adtctl tools --json` is the catalog; `adtctl <cmd> --help` returns a discoverable interface per command.

Good luck.
