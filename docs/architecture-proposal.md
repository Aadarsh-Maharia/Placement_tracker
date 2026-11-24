# Placement Tracker: Architecture Proposal

| | |
|---|---|
| Status | **Draft, awaiting approval.** No application code is written yet. |
| Date | 2026-11-24 |
| Scope | Engineering plan for a local-first Windows desktop app, built and packaged entirely in the cloud (Claude Code Web plus GitHub Actions) |

---

## 0. Summary

| Decision | Recommendation |
|---|---|
| App framework | **Tauri 2** (Rust core, system WebView2 UI). Tauri 2.12.6 is current as of Nov 2026. |
| UI | React + TypeScript + Vite. All assets bundled; no CDN or remote fonts. |
| Database | **SQLite with full-database encryption (SQLCipher v4 format)** through `rusqlite`. A Phase 0 spike chooses between SQLCipher and SQLite3MultipleCiphers in SQLCipher-compatible mode. |
| Custom fields | **Typed EAV**: field definitions plus typed value tables. Queries compile from a filter AST into parameterized SQL in Rust. |
| Key management | Random 256-bit data key, wrapped by **Windows DPAPI** by default. A master password (Argon2id) is optional. A one-time **recovery key** is shown at setup. |
| Installer | **NSIS `-setup.exe`, per-user install (no admin needed).** The release build embeds the **WebView2 Evergreen offline installer** (`offlineInstaller`). |
| WebView2 mode | **Evergreen offline installer: yes. Fixed Version: no.** See §8.3. |
| CI/CD | Linux jobs for lint and unit tests (cheap). **`windows-2025` runners** build, run Windows tests, do install smoke tests, and package. Tag-triggered draft **GitHub Releases**. |
| Your machine | You only download the `.exe` from GitHub's web UI. No Node, Rust, Git or Visual Studio. |
| Privacy | Synthetic data only. Secret and PII scanners plus a fixture linter run in CI. The app makes no network calls, and this is enforced at build time and checked at runtime. |

The repository is currently **empty** (private, no commits, no default branch yet). This document will be its first commit.

---

## 1. Final technology stack

### 1.1 Stack

| Layer | Choice | Notes |
|---|---|---|
| Shell / windowing | Tauri 2.12.x | Uses the OS WebView2 on Windows. Capability-based IPC permissions. |
| Backend language | Rust stable, pinned in `rust-toolchain.toml` | The `x86_64-pc-windows-msvc` target supports Windows 10+ (Rust ≥ 1.78). |
| DB driver | `rusqlite` with a bundled SQLCipher build | Synchronous, simple, no build-time database needed (unlike `sqlx` macros). |
| Migrations | `rusqlite_migration` with embedded `.sql` files | Forward-only, and they run inside a transaction. |
| Crypto | RustCrypto `argon2`, `chacha20poly1305`, `zeroize`, `secrecy`; the `windows` crate for DPAPI | No hand-rolled cryptography. |
| Type bridge | `tauri-specta` (fallback `ts-rs`) | Generates TypeScript types and command bindings from the Rust signatures. |
| Frontend | React 19, TypeScript (strict), Vite | |
| UI kit | shadcn/ui (Radix primitives) + Tailwind | Components live in the repo, so there is no runtime dependency on a vendor. |
| Data/grid/forms | TanStack Query, TanStack Table, react-hook-form, zod | zod schemas are generated from the custom-field definitions. |
| Tests | `cargo test`, `proptest`, Vitest, Testing Library, Playwright (web build with mocked IPC), WebdriverIO + `tauri-driver` (real app on Windows) | |
| Packaging | Tauri bundler: NSIS (primary), MSI (optional, enterprise) | |
| CI | GitHub Actions: `ubuntu-latest` + `windows-2025` (+ `windows-11-arm` later) | |

### 1.2 Trade-offs of the framework choice

Criteria, in order: (1) the app can be developed and tested mostly in a Linux cloud container, (2) Windows security posture and update burden for a one-person project, (3) installer size and offline install, (4) memory and disk footprint, (5) ecosystem maturity.

| Option | For | Against | Verdict |
|---|---|---|---|
| **Tauri 2** | Core logic and UI build and test on Linux; only packaging needs Windows. Microsoft patches the WebView2 engine (Evergreen). Small binaries (~10 MB app). Rust gives memory safety and good SQLite/crypto crates. Strict IPC capability model. | Two languages (Rust + TS). The WebView2 runtime is an external dependency (mitigated in §8). Windows-only bugs show up only in CI. | **Recommended** |
| Electron | Self-contained Chromium, so no WebView2 dependency. Huge ecosystem. One language (TS). | **We would have to re-ship Chromium for every security fix** (an ongoing burden). ~90–120 MB installer and higher RAM. Node in the main process increases attack surface. Native SQLCipher modules need per-Electron-version rebuilds. | Viable. Rejected because of the patching burden. |
| .NET 9/10 WPF or WinUI 3 | Native Windows UX, DPAPI built in, mature MSI tooling. | **Cannot run or UI-test in the Linux container**, so every UI change needs a Windows CI round trip. WinUI 3 also needs the Windows App SDK runtime. Self-contained publish is ~70+ MB. | Rejected for this workflow |
| Wails (Go + WebView2) | Same WebView2 model as Tauri, Go backend. | Smaller ecosystem, weaker plugin/permission model, less mature bundler. | Rejected |
| Flutter desktop | Single codebase, good UI. | Windows build needs Windows. Weaker SQLCipher/desktop plugin story. Larger runtime. | Rejected |

**An important nuance.** Once the WebView2 offline installer is embedded, Tauri's installer is *larger* than an Electron one: ~135 MB versus ~100 MB. Tauri still wins on **security maintenance**, because Microsoft patches the Evergreen WebView2 through its own updater while Electron's Chromium is patched only when *we* ship a release. It also wins on runtime footprint and cloud testability.

---

## 2. Desktop architecture

```
┌──────────────────────────── Placement Tracker.exe (one process + WebView2 children) ─────────────────────────────┐
│                                                                                                                  │
│  WebView2 (UI, untrusted zone)                     Rust host (trusted zone)                                      │
│  ┌──────────────────────────────┐   typed IPC     ┌───────────────────────────────────────────────────────────┐  │
│  │ React app                    │ ─────────────▶  │ src-tauri: thin command layer (validation, error mapping)  │  │
│  │ • views, forms, tables       │  (JSON, allow-  │        │                                                   │  │
│  │ • no storage of records      │   listed cmds)  │        ▼                                                   │  │
│  │   (no localStorage/IDB)      │ ◀─────────────  │ pt-core (pure Rust library, no Tauri dependency)           │  │
│  │ • CSP: self only             │   DTOs only     │  ├─ domain   (entities, validation, custom-field types)   │  │
│  └──────────────────────────────┘                 │  ├─ store    (rusqlite, migrations, query compiler, FTS)   │  │
│                                                   │  ├─ crypto   (key hierarchy, DPAPI, Argon2id, AEAD)        │  │
│                                                   │  ├─ backup   (export/import, verify, restore)              │  │
│                                                   │  └─ io       (CSV/JSON import/export, attachments)         │  │
│                                                   └───────────────────────────────────────────────────────────┘  │
│                                                                    │                                             │
│                                              %LOCALAPPDATA%\PlacementTracker\ (encrypted DB, wrapped keys, ...)  │
└──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.1 Repository layout (proposed)

```
/
├─ crates/
│  └─ pt-core/            # all business logic; builds and tests on Linux and Windows
├─ src-tauri/             # Tauri app crate: commands, capabilities, tauri.conf.json
│  ├─ capabilities/       # least-privilege permission sets
│  └─ tauri.conf.json
├─ ui/                    # React + Vite frontend
├─ tools/
│  ├─ synth/              # deterministic synthetic-data generator (Rust bin)
│  └─ privacy-lint/       # fixture/PII checks used in CI
├─ tests/e2e/             # WebdriverIO + tauri-driver specs (Windows CI)
├─ docs/                  # this proposal, ADRs, threat model, release runbook
└─ .github/workflows/     # ci.yml, windows.yml, release.yml, audit.yml
```

### 2.2 Design rules

1. **`pt-core` has no Tauri dependency.** About 80% of the logic is then testable with plain `cargo test` in the Linux container.
2. **The UI holds no persistent data.** It never writes records to `localStorage`, IndexedDB or cookies. All state is fetched through IPC, and TanStack Query's cache is memory-only.
3. **Keys never cross IPC.** The webview never sees key material or file paths of the database.
4. **Commands are coarse and validated.** Each command takes a typed DTO, is validated again in Rust (the UI validation is only for UX), and returns typed errors. There is no generic "run SQL" command.
5. **Least-privilege Tauri capabilities.** Only `core:default`, `dialog` (open/save, used for backup paths and attachments) and `single-instance` are enabled. **No `shell`, `http`, `fs` (scoped JS file access), `updater` or `opener` for URLs in v1.**
6. **Single instance.** A second launch focuses the existing window, so two processes never open the database.
7. **Isolation pattern.** Tauri's isolation pattern is enabled to sandbox IPC messages from third-party npm code.
8. **Hardened webview.** Strict CSP (`default-src 'self'; connect-src ipc: http://ipc.localhost; object-src 'none'; frame-src 'none'`), devtools off in release, no remote origins, and Markdown notes rendered through a sanitizer (no raw HTML).

---

## 3. Database architecture

### 3.1 Engine configuration

- One encrypted SQLite file `placement.db`, plus `attachments.db` (same key, attached), so the main database stays small and fast while the backup stays a single encryption mechanism.
- `PRAGMA journal_mode=WAL; foreign_keys=ON; temp_store=MEMORY; secure_delete=ON; synchronous=NORMAL`.
  - `temp_store=MEMORY` keeps temporary tables and sort spill **off disk in plaintext**. With SQLCipher, temp files are otherwise a known leak vector.
- `STRICT` tables, so SQLite enforces column types.
- One writer connection behind a mutex, plus one reader connection. Calls run on `spawn_blocking`.
- IDs are **UUIDv7** stored as 16-byte BLOBs: time-sortable and merge-safe if import or sync is ever added.
- Timestamps are UTC Unix milliseconds (`INTEGER`). Date-only fields (deadlines, drive dates) are ISO `YYYY-MM-DD` text.
- **Money is integer minor units plus an ISO-4217 code** (for example paise + `INR`). Floats are never used for salary.

### 3.2 Core schema (v1 sketch)

| Table | Purpose / key columns |
|---|---|
| `company` | name, website, industry, HQ/location, archived_at |
| `opportunity` | company_id, title, kind (full-time / internship / PPO / other), source, drive_date, apply_deadline, location(s), JD URL, eligibility notes |
| `compensation_component` | opportunity_id or offer_id, kind (fixed, variable, joining bonus, retention, ESOP/RSU, relocation, stipend), amount_minor, currency, period, vesting notes. Indian CTC breakdowns are not a single number. |
| `pipeline_stage` | user-configurable ordered stages (Applied → OA → Interview → Offer → Accepted/Rejected), is_terminal, color |
| `application` | opportunity_id, stage_id, applied_at, priority, outcome, withdrawn_at |
| `round` | application_id, kind (OA, GD, technical, HR, …), scheduled_at, duration, mode, result, feedback |
| `offer` | application_id, received_at, decision_deadline, decision, decided_at |
| `contact` | company_id, name, role, email, phone, notes |
| `note` | owner_type, owner_id, body (Markdown), created_at |
| `tag`, `entity_tag` | free tagging |
| `attachment_meta` / `attachments.db:blob` | file name, MIME type, size, SHA-256; content stored as a BLOB in the attached encrypted DB (size cap, for example 25 MB) |
| `custom_field_*` | see §4 |
| `saved_view` | entity_type, name, filter AST (JSON), sort, visible columns |
| `audit_log` | ts, entity, action, changed field keys. Supports history/undo; retention is configurable. |
| `search_fts` | FTS5 index over names, titles, notes and text custom fields. It lives inside the encrypted DB. |
| `app_meta` | schema version, created_by_app_version, install id (random, never transmitted) |

Polymorphic owners (`note`, `attachment_meta`, `entity_tag`, `custom_field_value`) get **delete-cascade triggers**, because SQLite cannot enforce foreign keys on polymorphic references.

### 3.3 Migrations

- Forward-only, numbered SQL files compiled into the binary. `PRAGMA user_version` tracks the version.
- Before any migration the app takes an **automatic pre-migration backup** (§13). Migrations run in a single transaction followed by `PRAGMA integrity_check` and `foreign_key_check`.
- If the database's schema is **newer** than the app supports (a downgrade), the app refuses to open it read-write and explains why. No data is touched.
- CI test: for every historical schema version there is a synthetic fixture DB, and each one migrates to head with a content check.

---

## 4. Custom field architecture

### 4.1 Options compared

| Approach | For | Against |
|---|---|---|
| Runtime `ALTER TABLE ADD COLUMN` | Fast native queries | User DDL collides with app migrations. SQLite type changes need table rebuilds. Schema drift per install makes migrations untestable. |
| JSON column per entity | Simple writes, flexible | Indexing user keys needs runtime-generated expression indexes. Weak typing. Renaming a key rewrites every row. |
| **Typed EAV** | Fixed, testable schema. Real types and constraints. Indexable. Rename and archive are trivial. | More verbose queries (one join per filtered field). Hidden behind a query compiler. Adequate up to ~10⁵ records. |

**Recommendation: typed EAV.**

### 4.2 Schema

```sql
custom_field_def(
  id BLOB PK, entity_type TEXT CHECK (entity_type IN ('company','opportunity','application','round','offer','contact')),
  key TEXT,            -- stable slug, unique per entity_type
  label TEXT, help TEXT,
  field_type TEXT,     -- see 4.3
  config TEXT,         -- JSON: min/max, precision/scale, currency, max_length, regex, date bounds
  required INTEGER, default_json TEXT, position INTEGER,
  archived_at INTEGER, created_at INTEGER, updated_at INTEGER,
  UNIQUE(entity_type, key)
) STRICT;

custom_field_option(id BLOB PK, field_id BLOB FK, label TEXT, color TEXT, position INTEGER, archived_at INTEGER) STRICT;

custom_field_value(
  entity_id BLOB, field_id BLOB FK,
  v_text TEXT, v_int INTEGER, v_real REAL, v_option BLOB FK,
  updated_at INTEGER,
  PRIMARY KEY(entity_id, field_id)
) STRICT;
CREATE INDEX cfv_text   ON custom_field_value(field_id, v_text);
CREATE INDEX cfv_int    ON custom_field_value(field_id, v_int);
CREATE INDEX cfv_option ON custom_field_value(field_id, v_option);

custom_field_value_multi(entity_id BLOB, field_id BLOB, option_id BLOB, PRIMARY KEY(entity_id, field_id, option_id)) STRICT;
```

### 4.3 Field types and storage mapping

| Type | Stored in | Notes |
|---|---|---|
| short text, long text, URL, email, phone | `v_text` | Validated by type. Long text is FTS-indexed. |
| integer | `v_int` | |
| decimal | `v_int` (scaled by `config.scale`) | Exact; no float rounding |
| currency | `v_int` minor units; currency in `config` or per value | |
| percent / rating | `v_int` | |
| boolean | `v_int` 0/1 | |
| date | `v_text` ISO date | Sorts lexicographically |
| datetime | `v_int` UTC ms | Displayed in the local time zone |
| single select | `v_option` | |
| multi select | `custom_field_value_multi` | |

Out of scope for v1: formula/computed fields, relation-to-entity fields, per-field encryption (the whole database is already encrypted).

### 4.4 Behaviour

- **Single source of truth in Rust.** `pt-core` validates every write against the definition. The UI receives the definitions and generates zod schemas and form widgets from them.
- **Type changes** are allowed only along lossless paths (int → decimal, short → long text, single → multi select). Other conversions use "create a new field + guided copy" with a dry-run preview of rows that would fail.
- **Delete = archive** by default (values kept and hidden). Permanent purge is explicit and covered by the pre-change backup.
- **Querying.** A typed filter AST (`{field, op, value}` trees with AND/OR) compiles to parameterized SQL with one `LEFT JOIN custom_field_value AS cfN` per referenced field. User input is never concatenated into SQL. Limits: 16 custom-field joins per query and 200 fields per entity type.
- **Import/export.** CSV columns map to built-in or custom fields, with a per-row validation report.

---

## 5. Security and encryption

### 5.1 Threat model

| Threat | In scope? | Mitigation |
|---|---|---|
| Laptop lost or stolen, disk read offline | Yes | Full-DB encryption. Key wrapped by DPAPI, which is bound to the Windows logon secret. Recommend BitLocker as well. |
| Another local Windows account reads the files | Yes | Per-user install dir and data under `%LOCALAPPDATA%` (user ACLs) plus encryption |
| Someone uses your unlocked, logged-in session | Optional | **Optional master password** and auto-lock after idle |
| Backup file copied (USB, cloud drive) | Yes | Backups are always passphrase-encrypted (§13) |
| Malicious content in pasted notes (XSS → IPC abuse) | Yes | Sanitized rendering, strict CSP, isolation pattern, narrow commands, no network capability |
| Data exfiltration by the app or its dependencies | Yes | No network code paths. `cargo-deny` bans HTTP crates. No `http` capability. CSP blocks `connect-src`. Runtime network test (§12). |
| Malware running as your user | **No** (out of scope) | No user-mode app can defend against this. Documented as a known limit. |
| Supply-chain attack on build | Partially | Lockfiles, pinned action SHAs, `cargo-deny`/`npm audit`, minimal dependencies, verify the WebView2 installer's Microsoft signature |

### 5.2 Key hierarchy

```
DEK (256-bit random, generated at first run, held only in Rust memory and zeroized on lock/exit)
 ├─ used as the SQLCipher raw key:  PRAGMA key = "x'<64 hex>'"   (skips PBKDF2, so opening is fast)
 └─ stored only in wrapped forms, in keys\keyring.json:
     ├─ dpapi:     CryptProtectData(DEK, entropy=app-specific, CurrentUser scope)          ← default unlock
     ├─ password:  XChaCha20-Poly1305(KEK = Argon2id(password, salt, m=64–256 MiB, t≈3))  ← optional
     └─ recovery:  XChaCha20-Poly1305(KEK = Argon2id(recovery key, salt))                  ← shown once at setup
```

- **Default (DPAPI only):** no password prompt, and data is still encrypted at rest. This protects against offline disk access and other accounts, not against someone using your session.
- **Master password enabled:** the DPAPI wrap is removed, and the app asks for the password at launch and after idle lock. Argon2id parameters are calibrated to about 0.5–1 s on the device.
- **Recovery key** (24-word or base32 string, shown once and confirmed by re-entry) protects against losing the DPAPI secret after a Windows reinstall or profile reset, and against a forgotten master password. **Without it, a forgotten password or wiped profile means unrecoverable data.** This is a deliberate trade-off that the UI states clearly.
- Rekeying (change password, rotate DEK) uses `PRAGMA rekey`, preceded by an automatic backup.

### 5.3 Engine choice (Phase 0 spike)

| Option | For | Risk |
|---|---|---|
| A. `rusqlite` `bundled-sqlcipher-vendored-openssl` | 15+ years of production use; standard SQLCipher v4 format | Windows MSVC builds with vendored OpenSSL have a history of friction. There is a public report of an **ARM64 write crash**. Needs Perl/NASM on the runner. |
| B. SQLite3MultipleCiphers in SQLCipher-v4-compatible mode | No OpenSSL. Developed primarily on Windows. Actively maintained (v2.7.1, Nov 2026). | No official `rusqlite` feature yet, so it needs a small in-repo `-sys` shim crate. |

Both produce the **same on-disk format** (SQLCipher v4), so switching later does not break user data. Decision rule: use A if it builds and passes the crypto test suite on `windows-2025` x64 with acceptable build time. Otherwise use B. ARM64 (§7.3) must pass the same suite before shipping.

---

## 6. Local storage design

```
%LOCALAPPDATA%\Programs\Placement Tracker\        ← app binaries (per-user NSIS install; replaced on upgrade)

%LOCALAPPDATA%\com.placementtracker.app\          ← all user data (never roams, never touched by upgrades)
├─ data\
│  ├─ placement.db  (+ -wal, -shm)                ← encrypted
│  └─ attachments.db                              ← encrypted
├─ keys\keyring.json                              ← wrapped DEKs only (never plaintext)
├─ backups\
│  ├─ auto\YYYYMMDD-HHMMSS.ptbak                  ← rotating: 7 daily + 4 weekly + pre-migration
│  └─ pre-restore\                                ← previous DB kept after a restore
├─ logs\app.log                                   ← rotated, 5 × 1 MB, redacted (§12.3)
├─ config.json                                    ← UI prefs only (theme, window size); no record data
└─ EBWebView\                                     ← WebView2 profile (cache only; app stores no records here)
```

- **`%LOCALAPPDATA%`, not `%APPDATA%` (Roaming).** Roaming profiles can sync to domain servers. OneDrive "Known Folder Move" redirects Desktop/Documents/Pictures but **not AppData**, so data stays on the machine.
- The data directory is **preserved across upgrades and uninstalls by default**. The NSIS uninstaller has an opt-in "Delete application data" checkbox. Tauri's template already supports this, and `/DELETEAPPDATA` does it silently.
- Atomic writes for small files (write temp → `FlushFileBuffers` → rename).
- When the user picks a backup destination inside a OneDrive/Dropbox/Google Drive folder, the app warns that the (encrypted) file will leave the machine.
- Optional later: a portable mode (data next to the exe). Not in v1.

---

## 7. Windows compatibility

### 7.1 Support matrix

| Platform | Status | How it is verified |
|---|---|---|
| Windows 11 x64 (23H2/24H2/25H2) | **Primary** | CI on `windows-2025` (Server 2025 shares the 24H2 code base). Manual smoke test by you on each release. |
| Windows 10 x64 22H2 | Supported | Same binaries. Not available as a GitHub-hosted runner, so it needs a manual smoke test per release (you, or a Windows 10 VM). |
| Windows 10 x64 1809 (LTSC 2019) | **Best effort** | Rust binaries need Windows 10+ and WebView2 supports this OS, but consumer 1809 has been out of servicing since 2020. Only LTSC 2019 is still patched. No hosted runner exists, so this can be tested only manually or on a self-hosted VM. |
| Windows 11 ARM64 | **Phase 5** | x64 build runs under Prism emulation today. Native build on `windows-11-arm` runner (§7.3). |
| Windows 7/8/8.1, 32-bit | Not supported | Rust and WebView2 support have ended |

### 7.2 Compatibility measures

- **Static VC++ runtime** (Tauri's `staticVCRuntime`, default `true`), so no VC++ Redistributable is needed. CI asserts this with `dumpbin /dependents` (no `VCRUNTIME140.dll`).
- `minimumWebview2Version` is set in `tauri.conf.json` to the oldest version CI tested. The installer upgrades older runtimes.
- Per-monitor-v2 DPI awareness (handled by tao/WebView2). Test at 100% / 150% / 200% scaling.
- Long-path-safe file handling (`\\?\` prefixes where needed). Unicode user names and paths are tested in CI (a user profile path with non-ASCII characters).
- **Windows 10 has passed its general end of support (Oct 2025).** WebView2 continues to receive updates on Windows 10 22H2 under Microsoft's extended policy (reported "at least until October 2028"; to be re-verified at each major release).

### 7.3 ARM64 investigation (separate future target)

| Item | Finding / plan |
|---|---|
| Rust target | `aarch64-pc-windows-msvc` is Tier 1 |
| Runner | `windows-11-arm` is GitHub-hosted (free for public repos; $0.010/min for private) |
| WebView2 | ARM64 offline installer exists; select per-arch at bundle time |
| Risk | SQLCipher + OpenSSL ARM64 write crash reported upstream. Engine B (§5.3) avoids OpenSSL. |
| Signing | Azure Artifact Signing action doesn't run on ARM runners, so we build on ARM and sign in a follow-up x64 job |
| Deliverable | Separate `..._arm64-setup.exe`. No universal installer. |
| Exit criteria | Full `pt-core` + E2E suite green on `windows-11-arm`. Manual test on a real ARM device. |

---

## 8. Installer architecture

### 8.1 Format

| | NSIS `-setup.exe` | MSI (WiX v3) |
|---|---|---|
| Admin rights | **Not needed** (`installMode: currentUser`) | Needed (per-machine) |
| UX | Simple wizard, start-menu shortcut, uninstall entry, optional "delete app data" | Standard, silent `msiexec /qn` |
| Enterprise (GPO/Intune) | Possible via `/S` | Native |
| **Use** | **Primary deliverable** | Optional secondary artifact, off by default |

### 8.2 Deliverables per release

| File | WebView2 mode | Approx. size* | Audience |
|---|---|---|---|
| `PlacementTracker_<ver>_x64-setup.exe` | **`offlineInstaller`** | ~135–145 MB | **Default download**; works fully offline |
| `PlacementTracker_<ver>_x64-online-setup.exe` | `embedBootstrapper` | ~8–12 MB | Optional; for machines that already have WebView2 (all Windows 11) or are online |
| `SHA256SUMS.txt` | | | Integrity check |

\*Estimates: Tauri app ≈ 8–12 MB compressed, plus 127 MB (Tauri's published figure for the WebView2 offline installer). Measured sizes from Phase 0 will replace these.

Install flow (offline variant): check for the WebView2 Evergreen runtime. If missing, or older than `minimumWebview2Version`, run the embedded Microsoft installer silently. Then install the app per-user, create the Start-menu entry, and launch. If WebView2 is already present (true on every Windows 11 machine), the 127 MB payload is simply not executed.

### 8.3 WebView2 distribution options compared

| Mode | Internet at install | Added size | Runtime security updates | Disk on client | Notes |
|---|---|---|---|---|---|
| `downloadBootstrapper` (Tauri default) | **Required** | 0 MB | Automatic (Evergreen) | Shared | Fails offline if runtime is missing |
| `embedBootstrapper` | **Required** | ~1.8 MB | Automatic (Evergreen) | Shared | Same as above, slightly more robust |
| **`offlineInstaller` (Evergreen Standalone)** | **No** | ~127 MB | **Automatic (Evergreen)** once online | Shared with Edge and other apps | Microsoft's recommended mode; Microsoft keeps it patched |
| `fixedRuntime` (Fixed Version) | No | ~180 MB compressed (Microsoft says the binaries are >250 MB) | **None: we must re-release the app for every Chromium security fix** | Private copy per app | Microsoft recommends it only for strict compatibility needs |
| `skip` | — | 0 | — | — | App fails to start if the runtime is missing |

**Recommendation: Evergreen offline installer (`offlineInstaller`) for the default download, plus the small `embedBootstrapper` variant as an optional extra.**

Reasons:
1. **Offline install is guaranteed**, which meets requirements 6 and 8.
2. **Security updates stay Microsoft's job.** After install, the shared Evergreen runtime updates itself whenever the machine is online. With Fixed Version, every WebView2/Chromium CVE (Edge ships security updates every 1–2 weeks) becomes *our* release obligation. A one-person project cannot meet that, and missing one leaves a stale browser engine rendering your data.
3. **Size is acceptable and mostly unused.** Windows 11 always ships WebView2, so the payload runs only on the rare machine that lacks it. Fixed Version would add ~180 MB *and* install a private copy on every machine.
4. **Permanently air-gapped machines:** neither Evergreen nor Fixed updates itself there. With Evergreen, every new app installer we ship also carries a newer standalone runtime, so updating the app updates the engine.

Tauri passes `--disable-features=msSmartScreenProtection` by default, so WebView2 does not send navigation reputation checks to Microsoft. The app also loads only bundled local content.

### 8.4 Code signing

Unsigned installers trigger SmartScreen "Windows protected your PC" (click *More info → Run anyway*). **On Windows 11 with Smart App Control enabled, unsigned apps can be blocked outright.**

| Option | Cost | Constraint |
|---|---|---|
| Unsigned + SHA-256 checksums | Free | SmartScreen warnings; possible Smart App Control block |
| Azure Artifact Signing (formerly Trusted Signing) | ~US$10/month | **Individuals: US/Canada only.** Organizations: more countries. Integrates with GitHub Actions via OIDC. |
| OV certificate on a cloud HSM (SSL.com eSigner, Certum, DigiCert KeyLocker…) | ~US$150–400/year | Works from CI; reputation builds over time |
| SignPath Foundation | Free | **Open-source projects only.** Not available while the repo is private. |

**Proposal:** ship v0.x **unsigned** for your own use, with checksums. Before distributing to anyone else, take an explicit signing decision (see open question Q4). The release pipeline will have a signing step that is wired in but disabled.

---

## 9. Cloud development workflow

```
You (browser only)                Claude Code Web (Linux container)           GitHub
──────────────────                ─────────────────────────────────           ──────────────────────────────
describe feature  ───────────▶    edit code on claude/* branch
                                  cargo test (pt-core, SQLCipher)
                                  vitest + Playwright (mock IPC)
                                  privacy-lint, clippy, eslint
                                  git push  ───────────────────────────────▶  ci.yml (Linux) ──▶ windows.yml
                                  read CI logs via GitHub tools ◀────────────  (build, tests, NSIS installer)
                                  fix → push (loop)                                     │
review PR in GitHub UI ◀───────────────────────────────────────────────────────────────┘
download installer artifact from the PR's Actions run (zip) → run on your PC
approve & merge; for releases: Claude pushes tag → release.yml → draft Release → you click "Publish"
```

- **Container setup.** The current Claude Code container already has Rust 1.97, Node 22 and pnpm. We will save a cloud-environment setup script (system `libssl-dev`, Playwright browsers already present, optional `webkit2gtk-4.1` for `cargo check` of `src-tauri` on Linux) so each new session starts ready.
- **What only Windows CI can verify:** DPAPI, installer behaviour, WebView2 rendering quirks, real-app E2E, static-CRT check. These run on every PR targeting `main`, and on demand via `workflow_dispatch`.
- **Iteration cost.** A Windows CI round trip takes about 10–20 minutes. To reduce it: Rust build caching, building Windows only when relevant paths change, and keeping logic in `pt-core` where the loop takes seconds.
- **Your role:** describe requirements, review PRs, download and try installers, approve releases. You never need a local toolchain or Git client.

---

## 10. CI/CD architecture

### 10.1 Workflows

| Workflow | Trigger | Runner | Steps |
|---|---|---|---|
| `ci.yml` | every push / PR | `ubuntu-latest` | `cargo fmt --check`, `clippy -D warnings`, `cargo test -p pt-core`, `cargo deny check` (licenses, advisories, banned crates), `pnpm lint`, `tsc --noEmit`, `vitest`, Playwright (web build + mocked IPC), **privacy gates** (§12), gitleaks |
| `windows.yml` | PR → `main`, `workflow_dispatch`, path-filtered | `windows-2025` | `cargo test` (incl. DPAPI and Windows paths); `tauri build` (online variant for PRs); static-CRT assertion; **silent install `/S`, launch, E2E (WebdriverIO + tauri-driver + matching msedgedriver), upgrade-over-previous, uninstall** checks; upload artifact (**retention 5 days**) |
| `release.yml` | tag `v*.*.*` | `windows-2025` (+ `windows-11-arm` later) | Full build of **both** installer variants; verify the embedded WebView2 installer's **Authenticode signature is Microsoft's**; (optional) code signing; SBOM (`cargo cyclonedx`, `npm sbom`); `SHA256SUMS`; create **draft** GitHub Release with assets |
| `audit.yml` | weekly cron | `ubuntu-latest` | `cargo audit`, `pnpm audit`, re-run `cargo deny` against new advisories |
| Dependabot | weekly | — | Cargo, npm, GitHub Actions updates (grouped) |

### 10.2 Pipeline hardening

- All third-party actions are **pinned by commit SHA**. Each workflow declares minimal `permissions:` (default `contents: read`).
- PR workflows use `pull_request`, never `pull_request_target`, and have no secrets.
- `concurrency: cancel-in-progress` per branch.
- Toolchains are pinned: `rust-toolchain.toml`, `packageManager` in `package.json`, committed `Cargo.lock` / `pnpm-lock.yaml`, `--locked` / `--frozen-lockfile`.
- One version source: `src-tauri/Cargo.toml`, which `tauri.conf.json` and `package.json` are checked against in CI.

### 10.3 Cost and storage (the repo is **private**)

| Item | Allowance (GitHub Free) | Expected use |
|---|---|---|
| Actions compute | $12/month ≈ 1,200 Windows-min or 2,000 Linux-min | Linux CI ~5 min × ~60 runs ≈ $1.80; Windows PR build ~10–15 min (warm cache) × ~30 ≈ $3–4.50; releases ~20 min each ≈ $0.20 → **~$6/month, within the allowance** |
| Artifact storage | **500 MB** | PR builds upload only the ~10 MB online variant (5-day retention). The 140 MB offline variant is built **only for releases**, where it is attached to the Release rather than stored as an artifact. |

Set the Actions **spending limit to $0** so CI pauses instead of billing if the allowance runs out. Making the repo public would make standard runners free and unlock artifact attestations and SignPath, but would publish the source (Q2).

---

## 11. Automated testing

| Level | Tooling | Where | Key cases |
|---|---|---|---|
| Unit (Rust) | `cargo test`, `proptest` | Linux + Windows | Validation; custom-field type rules and conversions; filter-AST → SQL (including injection attempts); money arithmetic; date handling |
| Storage | `cargo test` on temp encrypted DBs | Linux + Windows | Migrations from **every** historical fixture; FK/cascade triggers; FTS sync; WAL recovery after simulated crash |
| Crypto | `cargo test` | Linux + Windows (DPAPI Windows-only) | Wrong key fails cleanly; tampered page detected; rekey; DPAPI round trip; recovery-key unlock; Argon2 params persisted; keys zeroized (best effort) |
| Backup | `cargo test` | Linux + Windows | Export → restore round trip is byte-for-byte equal at the logical level; truncated/tampered backup rejected; restore of older schema migrates; newer schema refused |
| UI unit | Vitest + Testing Library | Linux | Dynamic form renders every field type; validation messages; table column config |
| UI integration | Playwright vs Vite build with `mockIPC` | Linux | Main user flows against an in-memory fake backend |
| Real app E2E | WebdriverIO + `tauri-driver` + msedgedriver | Windows CI | First-run setup → create records → custom field → search → backup → restore → lock/unlock |
| Installer | PowerShell scripts | Windows CI | Silent install without admin; upgrade v(N-1) → vN keeps data; uninstall keeps data by default; `/DELETEAPPDATA` removes it |
| Performance | Criterion + synthetic 10k apps × 50 custom fields | Linux + Windows | List/filter < 100 ms, open DB < 500 ms (budgets tuned in Phase 0) |

**Synthetic data generator (`tools/synth`):** deterministic (seeded) and produces obviously fake values: companies like `Testco Systems 0042`, people like `Candidate 0017`, emails only `@example.com`/`@example.test` (RFC 2606), phones that are invalid by construction (e.g. `+91 00000 xxxxx`; real Indian mobiles start with 6–9), random but plausible salaries. Fixtures are generated in CI or committed with a header that records the generator version and seed.

---

## 12. Privacy verification

### 12.1 Boundary

| Allowed in GitHub / CI / prompts / screenshots | Never allowed |
|---|---|
| Synthetic data from `tools/synth`, schema, code, aggregate counts | Your real placement records, company notes, salaries, contacts, resumes, offer letters, or any real `.db`/`.ptbak` file |

### 12.2 Enforcement in the repo and CI

1. `.gitignore` blocks `*.db`, `*.db-*`, `*.sqlite*`, `*.ptbak`, `exports/`, `attachments/`, `*.csv` outside `fixtures/`.
2. **gitleaks** (secrets) plus custom rules: email addresses outside `example.*`/`*.test`, Indian mobile patterns, PAN/Aadhaar-shaped strings, `₹`/`LPA`/`CTC` figures outside fixtures.
3. **`privacy-lint`**: every file under `fixtures/` must carry the synthetic-provenance header, and binary DB files are rejected unless the generator can reproduce them from the recorded seed.
4. **Network-free build:** `cargo-deny` bans `reqwest`, `hyper`, `ureq`, `isahc`, `curl`, `attohttpc`, and so on in the app dependency graph. ESLint bans `fetch`/`XMLHttpRequest`/`WebSocket` in `ui/` outside the IPC wrapper. The Tauri capability files are snapshot-tested.
5. **Runtime network test (Windows CI):** during E2E, a monitor records TCP/UDP connections opened by the app's process tree (`Get-NetTCPConnection` / ETW). The test fails on any connection that is not loopback IPC. Known WebView2 component-update traffic from `msedgewebview2.exe`, if any, is listed in an explicit allow file with a justification.
6. **CI log hygiene:** tests never print record contents. Rust test output is captured, `RUST_LOG` is unset in CI, and E2E screenshots and traces are captured **only on failure** and contain synthetic data only.

### 12.3 In the application

- No telemetry, analytics, crash upload or update check in v1. Nothing to opt out of.
- **Logs record events and IDs, never values.** A `Redacted<T>` wrapper type makes accidental `Debug` printing of record data a compile error in log macros.
- **"Export diagnostic bundle"** (for bug reports to Claude or GitHub): app/OS/WebView2 versions, schema version, row *counts*, custom-field *types* (not labels), settings, and the last N redacted log lines. Previewed before saving. **This is the only artifact you should share when asking for help.**
- Windows Error Reporting can upload crash dumps containing process memory. The app opts out for its own process (`WerAddExcludedApplication` at install time; to be verified in Phase 0), and the privacy note documents this.

---

## 13. Backup and restoration

### 13.1 Backup format: `.ptbak`

A `.ptbak` file is a **standard SQLCipher v4 database** keyed with the backup passphrase. It contains all tables, attachment blobs and a `backup_manifest` table (app version, schema version, created_at, row counts, SHA-256 per table's canonical dump).

- It is created through `ATTACH … KEY '<passphrase>'` + `sqlcipher_export()`, so **plaintext never touches disk**. (Do not use `VACUUM INTO` to an unencrypted temp file.)
- SQLCipher's per-page HMAC detects tampering. The manifest catches logical corruption.
- **Disaster-recovery property:** because it is the standard SQLCipher format, the backup can be opened with the open-source `sqlcipher` CLI even if this app no longer exists. This will be documented in `docs/recovery.md`.

### 13.2 Backup kinds

| Kind | When | Key | Purpose |
|---|---|---|---|
| Automatic | Daily on first launch, on exit if changed; keep 7 daily + 4 weekly | Backup key derived from DEK (HKDF) | Corruption or mistakes. **Does not protect against machine loss.** |
| Pre-change | Before migration, rekey, restore, field purge, bulk import | Same | Safe rollback |
| **Manual export** | User action; reminder if none in 30 days | **User passphrase** (strength-checked, e.g. zxcvbn) | Machine loss; save to USB or another disk |
| Plain export | Explicit, with warning | none | CSV/JSON for portability. Shown with a warning that the file is unencrypted. |

### 13.3 Restore procedure

1. Open the file with the passphrase. Verify `cipher_integrity_check`, `integrity_check`, the manifest hashes and the schema version (refuse newer).
2. Copy into a staging DB re-keyed with the current DEK. Migrate forward if older.
3. Take a pre-restore backup of the current DB. **Atomically swap** the files (rename), keeping the previous DB in `backups\pre-restore\`.
4. Restart the data layer and show a summary (counts by entity).

Restore on a **new machine** works because the passphrase, not DPAPI, protects the backup. After restore, the app generates a new DEK and DPAPI wrap for that machine.

---

## 14. Release management

- **Versioning:** SemVer. `0.x` until the data format is declared stable. **Database schema version is independent of app version**, and the release notes state the schema bump.
- **Commits:** Conventional Commits. `CHANGELOG.md` generated with `git-cliff` at release time.
- **Branching:** `main` is protected. Work happens on `claude/*` branches → PR → CI green → you merge.
- **Release steps:**
  1. Claude opens a "release vX.Y.Z" PR (version bump plus changelog).
  2. You merge it.
  3. Claude pushes the tag `vX.Y.Z`.
  4. `release.yml` builds, tests and attaches assets to a **draft** Release.
  5. You smoke-test the installer and click **Publish**.
- **Channels:** PR artifacts are dev builds, unsupported and expire after 5 days. `vX.Y.Z-beta.N` tags are pre-releases. `vX.Y.Z` is stable.
- **Updating the app:** download the new `setup.exe` and run it. NSIS upgrades in place, and data in `%LOCALAPPDATA%` is untouched. The in-app updater is **deferred**: release assets in a private repo aren't publicly downloadable, and an update check is a network call that conflicts with the privacy-first default. If the repo later becomes public, we can add Tauri's signed updater as an **opt-in** check.
- **Rollback:** install the previous version's `setup.exe`. If the schema was migrated, the older app will refuse the newer DB, so restore the automatic pre-migration backup. The release notes call out schema bumps.
- **Retention:** keep all published Releases. Delete draft or failed releases.

---

## 15. Phased delivery plan

| Phase | Outcome | Exit criteria |
|---|---|---|
| **0. Spikes and skeleton** (first) | Repo scaffold; CI on Linux + Windows; "hello" Tauri app → **NSIS installer downloadable from Actions**; SQLCipher engine decision (§5.3); DPAPI round trip; tauri-driver E2E smoke; measured installer sizes and build times | You install the skeleton app on your PC from a GitHub artifact, with no local tools |
| 1. Secure core | `pt-core` domain + schema v1 + migrations + key hierarchy + first-run setup (recovery key) | Crypto, migration and storage suites green on both OSes |
| 2. Core UI | Companies, opportunities, applications, rounds, offers; pipeline board; search | E2E main flows green |
| 3. Custom fields + views | Field manager, dynamic forms, filters/sorts, saved views, CSV import/export | Property tests and perf budget green |
| 4. Backup and hardening | Auto/manual backup, restore, diagnostic bundle, runtime network test, installer upgrade tests | Privacy gates all enforced; **v1.0.0** released |
| 5. Extensions | ARM64 build, code signing, optional master-password UX polish, optional MSI | Per §7.3 / §8.4 |

---

## 16. Open questions (answers change the plan)

| # | Question | Default if unanswered |
|---|---|---|
| Q1 | **Who uses the app?** One student tracking their own applications, or a coordinator tracking many students? | Single user, own applications. A coordinator use-case changes the schema (student entity, eligibility rules, bulk import). |
| Q2 | **Repo visibility:** keep private (CI fits in ~$12/month allowance; no attestations/SignPath) or make public (free CI, attestations, free OSS signing)? | Keep private |
| Q3 | **Master password:** optional (default off, DPAPI only) or required at every launch? | Optional, off by default; recovery key always generated |
| Q4 | **Code signing:** is unsigned acceptable for your own use during v0.x? | Yes; decide before sharing with others |
| Q5 | Any must-have fields or workflows (e.g., specific CTC components, placement-cell drive rules, calendar reminders)? | Generic schema in §3.2 + custom fields |

---

## 17. Facts verified for this proposal (Nov 2026)

| Fact | Source |
|---|---|
| Tauri 2.12.6 (16 Nov 2026) is current | v2.tauri.app/release/core |
| WebView2 modes and sizes: bootstrapper 0 / ~1.8 MB, offline ~127 MB, fixed ~180 MB | v2.tauri.app/distribute/windows-installer |
| Fixed Version not auto-updated; binaries >250 MB; Evergreen recommended | learn.microsoft.com/microsoft-edge/webview2/concepts/distribution, …/evergreen-vs-fixed-version |
| Wry default `--disable-features=msWebOOUI,msPdfOOUI,msSmartScreenProtection`; `staticVCRuntime` default true; NSIS `currentUser` default; uninstaller "delete app data" + `/DELETEAPPDATA` | Tauri config schema and NSIS template (tauri-apps/tauri) |
| Rust Windows MSVC targets require Windows 10+ | doc.rust-lang.org/rustc/platform-support/windows-msvc |
| Hosted runners: `windows-2025` (= `windows-latest`), `windows-2022`, `windows-11-arm` | docs.github.com: GitHub-hosted runners reference |
| Private-repo pricing: Windows 2-core $0.010/min; Free plan $12/month included compute; 500 MB artifact storage | docs.github.com: Actions runner pricing; Actions billing |
| Artifact attestations: public repos only on Free/Pro/Team | docs.github.com: artifact attestations |
| Azure Artifact Signing ~$9.99/month; individuals US/Canada only; action not supported on ARM runners | learn.microsoft.com: code signing options; Azure/artifact-signing-action |
| rusqlite `bundled-sqlcipher-vendored-openssl` exists; Windows/ARM64 friction reported; SQLite3MultipleCiphers v2.7.1 supports SQLCipher-compatible scheme | rusqlite README / issue #966; utelle/SQLite3MultipleCiphers |

Items marked "to verify in Phase 0" in the text are assumptions, not yet confirmed facts.
