# Nausica v2 — Software Architecture Design

**Product:** Nausica (rebuild) — plugin-driven engineering workbench
**Document status:** Draft v0.9 — for review
**Date:** 2026-09-26
**Companion:** [SRS.md](SRS.md) (requirements this design satisfies; IDs referenced throughout)

---

## 1. Introduction & Design Goals

This document specifies the architecture of Nausica v2. The single most
important design decision is the **core/plugin boundary**, redesigned from
first principles. Everything else follows from it.

### 1.1 The v1 problem, precisely

Nausica v1's plugin system (ADR-019) solved *menu declaration* — who gets a
menu item, when, and what roughly happens. But the execution side leaked the
boundary in three places:

| # | v1 leak | Why it's a problem |
|---|---------|--------------------|
| L-1 | `handler: method` — manifest names an arbitrary `MainWindow` method by string; dispatcher does `getattr(main_window, name)`. | A "plugin" is really a core feature wearing a YAML hat. The CANForge menu in v1 is 100% core code (`run_can_forge_cli` etc. live in `Nausica_APP.py`, 7,615 lines). Nothing about CANForge is portable, testable, or addable by a tool developer. |
| L-2 | `dynamic_provider` — the core hands a Python *callback* (defined in `Nausica_APP.py`, artifact-type aware) to the menu engine so the structured_artifact plugin's "Export As…" can enumerate variants. | The engine (generic) is told about one plugin's domain concepts. Adding a second dynamic menu means more app code. |
| L-3 | Subprocess progress dialog captures only `stderr`; "open output" is re-implemented ad hoc per pipeline in app code. | No contract between a tool and the UI for progress, output, or artifact handback; every tool reinvents it. |

Result: the 13 tools in `Dependency_Tools/` are *invoked* declaratively but
*implemented* in core. The extensibility story ("AI agents add plugins by
editing text files") is only half true.

### 1.2 First-principles boundary rules

From "the core is a host for tools it does not know about", we derive:

- **BR-1 Core purity.** Core source contains no reference to any plugin, its
  files, or its domain concepts. (SRS C1; enforced by test, SAD §9.)
- **BR-2 One-way, file-and-CLI-only communication.** Plugin ↔ core
  communicate exclusively via: (a) YAML/JSON files on disk (manifests,
  profiles, context), (b) subprocess argv/env, (c) the stdout protocol,
  (d) named capabilities. No shared Python imports, no in-process callbacks,
  no string-named method dispatch.
- **BR-3 Closed, typed core API.** The set of capabilities is small, named,
  versioned, and closed to text-file authors; extending it is a deliberate
  core change (SRS NFR-05). This is the deliberate trade: v1's `method`
  handler was "open" (any method) and therefore unsafe and unreviewable.
- **BR-4 Plugins own their logic end-to-end.** A plugin is a folder: manifest
  + scripts + prompts + templates + (optionally) its own dialogs. If a
  behavior is tool-specific, it is plugin code.
- **BR-5 Core owns everything shared.** File tree, projects, profiles,
  settings, time, version, notifications, terminal/agent/external-app launch,
  AI connectivity — because every plugin needs them identically.
- **BR-6 Failure containment.** The plugin lifecycle has explicit error
  states; no plugin fault path reaches an unhandled core exception (SRS C2).
- **BR-7 Proven v1 semantics are load-bearing.** Three-layer overrides,
  condition DSL, profile persistence, plugin-manager save guard, show-all
  tree: these worked and users depend on them. The rebuild changes the
  *mechanism* (L-1/L-2/L-3 fixes), not these *semantics*.

### 1.3 Consequences of the rules (what this buys)

- "Right-click run mobaXterm" and "right-click run Claude Code" are
  **plugins with zero payload scripts** — pure manifest + capabilities
  (FR-REF-06/07). In v1, each would be another 100 lines in the main file.
- A tool developer (or an AI agent) ships a tool by copying a folder in.
- Every plugin is runnable headless from a terminal (`python script.py`),
  independently of the GUI — v1 tools mostly already are; the contract now
  *requires* it.
- Core shrinks to ~3,000 lines (NFR-04) because ~4,000+ lines of tool
  orchestration move into plugins.

---

## 2. Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────────┐
│                            NAUSICA CORE (Qt app)                           │
│                                                                            │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────────────────────────┐  │
│  │ Workbench UI │  │  Workbench    │  │            PLUGIN HOST          │  │
│  │              │  │  Services     │  │                                 │  │
│  │ MainWindow   │◄─┤ ProjectRegistry│◄┤ PluginRegistry  (lifecycle)     │  │
│  │ ProjectList  │  │ WorkspaceTree │  │ ManifestValidator (JSON Schema) │  │
│  │ FileTree     │  │ ProfileMgr    │  │ ConditionEvaluator              │  │
│  │ ContextMenu  │  │ SettingsStore │  │ MenuBuilder   (model → QMenu)   │  │
│  │ EventLogDock │  │ TimeTracker   │  │ OverrideResolver (3 layers)     │  │
│  │ Menus/Dialogs│  │ UpdateChecker │  │ ProcessRunner (modes+timeout)   │  │
│  └──────┬───────┘  │ NotificationCenter│  │ ProtocolParser (stdout)     │  │
│         │          │ AI Service      │  │ CapabilityRegistry (typed)    │  │
│         │          └────────┬────────┘  └──────────────┬────────────────┘  │
│         │                   │                          │                   │
│         │        ┌──────────┴─────────┐      ┌─────────┴──────────┐        │
│         └───────►│   ExecutionContext │─────►│   Capabilities     │        │
│                  │ (context JSON)     │      │ (the only plugin→  │        │
│                  └────────────────────┘      │  core API, §5.4)   │        │
└──────────────────────────────────────────────┼────────────────────┼────────┘
                                               │ subprocess (same  │ capability
                                               │ interpreter)      │ calls
                     ┌─────────────────────────┼────────────────────┼────────┐
                     │                          ▼                    ▼        │
                     │   ┌──────────────────────────────────────────────────┐ │
                     │   │              PLUGIN  (a folder)                  │ │
                     │   │  plugin.yaml            (manifest v2, §5.3)      │ │
                     │   │  <scripts>              (its own logic, BR-4)    │ │
                     │   │  providers/<key>.py     (dynamic menus, §5.6)    │ │
                     │   │  prompts/ templates/    (payload)                │ │
                     │   └──────────────────────────────────────────────────┘ │
                     │        canforge · structured_artifact · docconvert    │
                     │        translation · testkit · visual_validation      │
                     │        terminal_launcher · agent_launcher (NEW)       │
                     └────────────────────────────────────────────────────────┘
```

Process model: **one core process + N short-lived plugin subprocesses**
(same interpreter, per SRS A1). No plugin threads, no plugin imports.

Component responsibilities (core) are detailed in §4; the plugin-side
contracts in §5.

---

## 3. The Core/Plugin Boundary Contract

The contract has exactly four artifacts. If it doesn't fit one of these, it
doesn't cross the boundary.

| # | Artifact | Format | Who writes | Who reads |
|---|----------|--------|------------|-----------|
| K-1 | **Manifest** `plugins/<id>/plugin.yaml` | YAML, schema v2 | Tool developer | Core (load time) |
| K-2 | **Context object** (per invocation) | JSON file in temp dir | Core | Plugin script / provider |
| K-3 | **Invocation** | argv tokens + env (`NAUSICA_CTX`, `NAUSICA_PROJECT_ROOT`) + stdout protocol + exit codes | Both | Both |
| K-4 | **Capability call** | Named, schema-checked, synchronous; results returned to the *calling context* (see §5.4 for the mechanics) | Plugin (via manifest `handler: capability` or via sidecar CLI, §5.4.3) | Core |

**Explicitly not part of the contract** (and therefore forbidden):
importing core modules, calling core objects, shared in-memory registries,
string-named method dispatch, GUI callbacks from plugin to core, and any
plugin-to-plugin communication (a plugin that needs another tool shells out
to it directly — same as v1 tools did with each other).

**Trust model.** Manifests and plugin scripts are team-authored and trusted
to be *correct*, not *safe* — so the core validates structure (schema,
capability names, condition keys) at load, but does not sandbox. A malicious
plugin has the user's full privileges; that is accepted (SRS NFR-07) because
v1 had the identical trust level.

---

## 4. Core Component Design

All core components are plain Python modules under `core/`; the Qt shell is
the only component that touches widgets. Target layout:

```
nausica2/
├── Nausica_APP.py          # shell only: MainWindow, dialogs, wiring (~1200 lines)
├── Nausica.ui / Nausica_UI.py   # ported from v1 (UI parity, B-05)
├── core/
│   ├── registry.py         # ProjectRegistry          (FR-CORE-01/02)
│   ├── tree.py             # WorkspaceTree model+proxy (FR-CORE-03)
│   ├── settings.py         # SettingsStore            (FR-CORE-09)
│   ├── time.py             # TimeTracker              (FR-CORE-10)
│   ├── update.py           # UpdateChecker (SVN)      (FR-CORE-11)
│   ├── notify.py           # NotificationCenter + EventLog (FR-CORE-12)
│   ├── ai.py               # AIService (Ollama)       (FR-CORE-13)
│   └── version.py          # version.py + full_version()
├── pluginhost/
│   ├── discovery.py        # scan, rescan             (FR-PLUG-01)
│   ├── validate.py         # JSON-Schema validation   (FR-PLUG-01/04)
│   ├── conditions.py       # ConditionEvaluator       (FR-PLUG-05)
│   ├── overrides.py        # OverrideResolver         (FR-PLUG-13)
│   ├── profiles.py         # ProfileManager (load/save/migrate) (FR-PLUG-14, FR-CORE-06)
│   ├── menubuilder.py      # menu model → QMenu       (FR-CORE-04/06, FR-PLUG-06)
│   ├── runner.py           # ProcessRunner            (FR-PLUG-07/11/12)
│   ├── protocol.py         # ProtocolParser           (FR-PLUG-11)
│   ├── providers.py        # ProviderRunner           (FR-PLUG-10)
│   └── capabilities.py     # CapabilityRegistry       (FR-PLUG-08/09, §5.4)
├── profiles/  config.json  settings.json  time_log.json
├── plugins/                  # ← v1 Dependency_Tools/, manifests bumped to v2
└── Test/
```

### 4.1 MenuBuilder — the context-menu host (FR-CORE-04)

The heart of the host. It works on a **menu model** (pure data), then renders
Qt — this split makes assembly testable headless (NFR-08).

1. `build_model(ctx) -> MenuModel` — pure Python, no Qt:
   - start with built-in actions, gated by the active profile's
     `built_in_actions` toggles and by `ctx.is_file`/`ctx.is_dir`;
   - for each enabled plugin in profile priority order: evaluate each
     `menu_items[]` entry's condition against `ctx` (ConditionEvaluator);
     collect surviving entries into a per-plugin node;
   - a plugin node with zero visible entries is dropped (v1 behavior).
2. `render(model) -> QMenu` — Qt only; `dynamic` nodes resolve their children
   via ProviderRunner **lazily at render time** (providers may be slower than
   condition evaluation; the 250 ms budget applies to model building + static
   rendering — provider calls run within the 3 s provider budget and the menu
   shows the item with children replaced when they land; see §6.3 for the
   timing treatment).

The model is also what the Plugin Status dialog and the parity tests inspect,
so "what menu would the user see" is answerable without a GUI.

### 4.2 ProcessRunner (FR-PLUG-07/11/12)

Owns the full subprocess lifecycle:

- resolves `script_path` against the plugin dir; expands `args_template`
  tokens (v1 set + `{ctx_json}`, `{project_name}` — §5.5.3);
- writes the context file; sets `NAUSICA_CTX`, `NAUSICA_PROJECT_ROOT`,
  `NAUSICA_PLUGIN_ID`;
- runs in one of three modes:
  - `progress`: modal cancellable `QProgressDialog` (v1 look), worker thread
    runs the process, `ProtocolParser` streams `progress`/`log` events to the
    dialog + EventLog. Cancellation → terminate the process tree
    (`taskkill /T` on Windows), report as exit code 2.
  - `terminal`: spawn a PowerShell window with the command (v1
    `run_in_terminal` behavior, keeps output visible); no protocol parsing.
  - `detached`: `Popen`, no wait (dashboards, Flowline); a status-bar
    notification is logged.
- enforces `timeout` (default 300 s, manifest-overridable) → kill + failure;
- maps exit codes (0/1/2/3) to the SRS behaviors, incl. `install_hint`
  dialog for exit 3.

### 4.3 OverrideResolver (FR-PLUG-13)

Pure function: `(item, plugin_id, profile_overrides, local_overrides) → item'`.
Preserves v1 semantics exactly: shallow item copy; `label_override`;
`args_override.handler_args` merged with `replace` (update) or `append`
(update + extend `args_template` via `extra_cli_flags`); local layer wins.
Unit-tested against the v1 `_apply_override`/`_merge_overrides` truth table
(ported from `Test/`).

### 4.4 Profiles (FR-PLUG-14, FR-CORE-06)

- `profiles/*.yaml` load/save with the v2 schema (§5.2) — a strict superset of
  v1 (v1 files load unmodified; the handler shim §5.4.4 covers v1 handler
  names at dispatch time).
- Switching: `ProfileManager.activate(profile_id)` → persist to the project's
  `config.json` entry (ADR-035 semantics) → invalidate menu model cache →
  tree filter re-applied from `file_filter`.
- Save guard (ADR-034) lives in the dialog, not the manager: the manager
  exposes `dirty` state; the dialog owns Save/Discard/Cancel.

### 4.5 Workbench services

Straight ports of v1 behavior with the noted changes:

- **ProjectRegistry** — `config.json` read/write (schema unchanged), CRUD
  dialogs. Exposed to plugins only via the `open_project` capability
  (FR-CORE-09/14).
- **WorkspaceTree** — `QFileSystemModel` + `JsonFileFilterProxy` port;
  `HIDDEN_JUNK_DIRS` moved to `settings.json` (was a constant);
  extension-filter submenu unchanged (B-02).
- **TimeTracker / UpdateChecker / AI Service / NotificationCenter** — ports;
  `AI Service` is the single Ollama client (v1 scattered it between
  `detail_dialog.py` and tools); tools now call it via `ai_assist`/
  `prompt_ai` capabilities.
- **NotificationCenter + EventLogDock** — append-only, tag-aware
  (`[plugin:canforge]`), feeds the dockable panel (FR-CORE-12, NFR-09).

---

## 5. Plugin-Side Contracts

### 5.1 Plugin folder layout (canonical)

```
plugins/<id>/
├── plugin.yaml          # manifest v2 (required)
├── <scripts>.py         # tool logic (any layout; referenced by manifest)
├── providers/           # optional: <key>.py per dynamic submenu
├── prompts/ templates/  # optional payload
├── requirements.txt     # optional: declared python deps (Plugin Status)
└── README.md            # optional
```

### 5.2 Profile YAML (v2) — unchanged from v1 plus:

```yaml
id: default
name: Default
description: ...
tags: [default, full]
built_in_actions:            # v1, preserved
  add: true
  open_submenu: true
  start: true                # new explicit key (v1 implicit)
  delete: true
  rename: true
  copy_path: true
  group_copy_move: true
  preview_directory: true
file_filter:                 # v1, preserved
  show_all: false
  enabled_extensions: [asc, md, ...]
plugins:                     # v1, preserved
  - {id: canforge, enabled: true, priority: 40}
overrides:                   # v1 shape, preserved (§4.3)
  canforge: {label_override: ..., args_override: {...}}
```

### 5.3 Manifest schema v2

```yaml
# ── identity ─────────────────────────────────────────────
id: canforge                 # required, kebab-case, unique across plugins
name: CAN Forge              # required, displayed in menu submenu
version: 2.0.0               # required, SemVer
min_core: "2.0"              # required — plugin unsupported by older cores
description: >-              # optional
  ...
tags: [can, uds]             # optional (profiles may match tags later)

# ── health (drives Plugin Status) ────────────────────────
requires:                    # optional python packages, checked via importlib
  - cantools
external_deps:               # optional PATH binaries, checked by name
  - ffmpeg
install_hint: "pip install cantools"   # optional, shown on exit code 3

# ── static capability usage (validated at load, BR-3) ───
capabilities_used: [terminal, agent, open_in]   # optional, advisory + validated

# ── menu ─────────────────────────────────────────────────
menu_items:
  # action
  - label: "CAN: Convert DBC to Markdown"
    type: action                         # action | submenu | separator
    condition:                           # §5.3.1
      file_extension: [".dbc"]
      is_file: true
      max_selection: 1
    handler: subprocess                  # subprocess | capability
    handler_args:
      script_path: canforge.py
      args_template: ["convert", "{input}", "--format", "md"]
      mode: progress                     # progress | terminal | detached
      timeout: 600
      progress_label: "Converting DBC…"
      show_output_on_success: false      # open resulting file (via protocol)
    # ── submenu with static children ──
  - label: "Import From…"
    type: submenu
    nested: true                         # real child QMenu (v1 ADR-038 key)
    condition: {name_suffix: ".structArt.md", max_selection: 1}
    children:
      - {label: "DOCX …", type: action, handler: subprocess,
         handler_args: {script_path: import_artifact.py,
                        args_template: ["docx", "--into", "{input}", "{pick}"]}}
    # ── submenu with dynamic children ──
  - label: "Export As…"
    type: submenu
    nested: true
    dynamic: export_variants             # → providers/export_variants.py (§5.6)
    provider_timeout: 5
    condition: {name_suffix: ".structArt.md", max_selection: 1}
  # ── capability-only item (zero payload) ──
  - label: "Run mobaXterm here"
    type: action
    condition: {is_file: false}          # folders
    handler: capability
    handler_args: {capability: terminal, app: mobaxterm}
```

**Validation** (load time, JSON Schema + core checks): required fields;
unique `id`; known `type`/`handler`/`mode`; known condition keys (FR-PLUG-05:
unknown key → invalid manifest, fail at load); known capability names;
`min_core` parseable. One bad manifest never blocks others (FR-PLUG-01).

#### 5.3.1 Condition DSL (v1 preserved + 2 additions)

| Key | v1? | Semantics |
|-----|-----|-----------|
| (string shorthand) `"`.dbc`" | ✔ | `file_extension: [.dbc]` + `is_file: true` |
| `file_extension` | ✔ | ext ∈ list (case-insensitive) |
| `name_suffix` | ✔ | basename endswith (case-insensitive) |
| `name_suffix_not` | ✔ | basename must NOT endswith any (ADR-042) |
| `dir_keyword` | ✔ | keyword in relative path (case-insensitive) |
| `is_file` | ✔ | true/false |
| `min_selection` / `max_selection` | ✔ | selection count bounds |
| `project_profile` | **new** | profile id or tag — plugin adapts per profile without a profile-level override |
| `env` | **new** | `{os: windows}`, `{capability: <name>}` (hidden when capability unavailable), `{external_dep: <name>}` (hidden when binary missing) |

All keys AND together. Evaluated purely against the context object — no
filesystem probing at menu time (providers may probe; they are the sanctioned
slow path).

### 5.4 Capabilities (the closed plugin→core API, BR-3)

#### 5.4.1 Invocation mechanics

- **From a menu item:** `handler: capability` + `handler_args` =
  `{capability: <name>, ...args}`. The CapabilityRegistry validates `args`
  against the capability's JSON schema, then executes synchronously on the UI
  thread (all capabilities are UI-ish or fast; `prompt_ai` and `ai_assist`
  run their long work in QThreads like v1's `AIWorker`).
- **From inside a running plugin script** (the important new power): the core
  ships `nausica_cli.py` next to the app; plugins call
  `sys.executable <app_dir>/nausica_cli.py <capability> --args-json '<json>'`,
  which connects to the core over a **named pipe / local loopback socket**
  (one per core process, path passed in `NAUSICA_CORE_ENDPOINT` env). The
  core answers with JSON (`{ok, result}`) — e.g. `pick_file` returns the
  chosen paths, which the script then uses. This is how v1's
  `method`-handler dialogs (source picker in Import From…, conflict dialog)
  are reproduced without any plugin importing core code. Timeouts: 300 s
  default for interactive capabilities; no endpoint (headless run from a
  terminal) → capabilities that need the UI report "not available", pure-data
  capabilities (none today) would still work.

#### 5.4.2 Capability catalog (SRS §4.2.4) — schemas

```
open_in       {app: str, path?: str}          # app ∈ settings.external_apps ∪ {explorer, vscode}
terminal      {app?: str, cwd?: str, command?: str}
agent         {agent: str, cwd?: str, file?: str}   # agent ∈ settings.agents
pick_file     {title: str, filters?: [str], multi?: bool, start?: str}
prompt_ai     {title: str, template_dir?: str, prompt_file?: str, output_ext?: str,
               script_path: str, args_template?: [str]}   # runs the plugin script after dialog
ai_assist     {mode: "grammar"|"translate"|"custom", prompt?: str, file: str,
               script_path?: str}              # body-aware transform; frontmatter never touched
dialog        {kind: "info"|"warn"|"critical"|"confirm", title: str, text: str}
notify        {level: str, text: str}
open_hardware_dialog {config_key: str, widget: "camera"|"serial"}
open_project  {action: "add"|"remove"|"edit"}
```

Unknown capability or bad args → error dialog + log + plugin marked `error`;
never an unhandled exception (BR-6).

#### 5.4.3 Why a pipe, not argv-only

Menu-triggered actions need no pipe (argv + protocol suffice). The pipe
exists for the *conversational* capabilities (`pick_file` return values,
`dialog` confirm results) and for plugins that want AI mid-script. It is the
only long-lived core endpoint; it carries JSON requests only, is bound to the
local process (no network), and is closed at app exit.

#### 5.4.4 v1 handler shim (FR-PLUG-09)

During migration, the registry maps v1 handler names to v2:

| v1 | v2 mapping |
|----|------------|
| `method: run_can_forge_cli` etc. | **not mappable** — these *become* the CANForge plugin's subprocess scripts (the port moves ~1,200 lines of app code into `plugins/canforge/`). The shim covers only the generic ones below. |
| `open_with: {app}` | `capability: open_in` / `terminal` / `agent` |
| `prompt_tool` | `capability: prompt_ai` |
| `method: show_prompt_tool_dialog` | `capability: prompt_ai` |
| `dynamic: "<key>"` (in-app provider) | provider script required; if absent at migration, the item is disabled + flagged in Plugin Status (forces the port, loudly) |

This makes the v1 `method` handlers honest: ~60% were real core services
(cleanly mapped), the rest were tool logic that v1 mislabeled (now plugin
logic).

### 5.5 Context object (K-2)

Written to a temp file per invocation; path passed as `{ctx_json}` token and
`NAUSICA_CTX` env.

```json
{
  "ctx_version": 2,
  "primary": "C:/proj/req/SYS_REQ_001.structArt.md",
  "is_file": true,
  "selected": ["C:/proj/req/SYS_REQ_001.structArt.md"],
  "selection_count": 1,
  "ext": ".md",
  "base_name": "SYS_REQ_001.structArt.md",
  "rel_path": "req/SYS_REQ_001.structArt.md",
  "project": {
    "name": "ATC_Project",
    "test_case_root": "C:/proj",
    "source_code_root": "",
    "profile": "default",
    "local_overrides": "C:/proj/.nausica.yaml"
  }
}
```

Token table for `args_template` (v1 tokens preserved):
`{input}` `{input_dir}` `{input_stem}` `{output}` `{root}` (= project
test_case_root) `{app_dir}` **`{ctx_json}`** **`{project_name}`**.

### 5.6 Provider protocol (K-3, FR-PLUG-10)

For a `dynamic: "<key>"` item, the host runs:

```
sys.executable <plugin_dir>/providers/<key>.py --ctx <ctx_file>
```

- stdout must be a JSON array of child items in the standard item shape
  (`label`, optional `handler`/`handler_args`/`condition` — provider children
  are **not** re-validated against conditions; the provider decides, as in v1).
- Children may themselves carry `handler_args.script_path` relative to the
  plugin dir. Provider-returned capability items must name valid capabilities
  (validated; invalid child dropped + logged).
- Default timeout 3 s (`provider_timeout` per item). No children → disabled
  hint row (v1 behavior). Crash/timeout → item skipped, status `error`, log.
- Canonical use: `plugins/structured_artifact/providers/export_variants.py`
  reads the artifact frontmatter (type resolution, `structart_manifest.yaml`
  curation) and emits the 8-preset Export As… catalog — the domain logic that
  lived in `Nausica_APP.py` moves here (BR-1 fix, L-2).

### 5.7 Output protocol (K-3, FR-PLUG-11)

Stdout lines:

```
NAUSICA-PROTO: {"t":"progress","n":37,"max":100,"msg":"parsing frame 37"}
NAUSICA-PROTO: {"t":"log","level":"warn","msg":"unknown signal FOO"}
NAUSICA-PROTO: {"t":"artifact","path":"C:/out/x.md","kind":"created"}
NAUSICA-PROTO: {"t":"open","path":"C:/out/x.md"}
NAUSICA-PROTO: {"t":"fail","code":3,"msg":"cantools missing"}
```

Any other stdout line = plain log (level info). stderr = plain log (level
error). Exit codes: **0** success (info dialog per `show_output_on_success` /
`artifact`+`open` messages), **1** failure (stderr tail dialog + log), **2**
user-cancel (silent), **3** missing dependency (`install_hint` dialog).

---

## 6. Key Flows

### 6.1 Right-click → menu shown

```
customContextMenuRequested
  → ExecutionContext.build(selection, project)          [pure, fast]
  → MenuBuilder.build_model(ctx)                        [pure; conditions;
       built-ins + per-plugin nodes]                     ~<100 ms]
  → MenuBuilder.render(model)                           [Qt; dynamic items
       get placeholder submenus]
  shown. (Provider runs are deferred to submenu open, §6.3)
```

### 6.2 Action click → subprocess (progress mode)

```
QAction.triggered
  → HandlerDispatch: validate (file exists? — v1 check)
  → ProcessRunner.run(mode=progress)
       → context file written, env set
       → QProgressDialog (modal, cancellable)
       → worker thread: Popen, stream stdout
            ProtocolParser → progress/log events → dialog + EventLog
       → exit 0: close dialog, "Done" info, open `artifact`/`open` paths
         exit 1: stderr-tail dialog + EventLog
         exit 2: silent close
         exit 3: install_hint dialog
```

### 6.3 Dynamic submenu open (lazy provider)

The `Export As…` node is rendered as a real `QMenu` with a placeholder
(enabled hint). On first `aboutToShow`, ProviderRunner fires
`providers/export_variants.py` (≤ 3–5 s). While running the placeholder shows
"loading…"; results replace it; a second open is cached per context signature
(primary path + mtime) for the session. If the provider fails, the hint row
stays and the error goes to status/log. This keeps the 250 ms menu budget
(§4.1) honest: no provider work happens at menu-build time.

### 6.4 Profile switch

```
Switch Profile… → dialog → ProfileManager.activate(id)
  → validate profile, load
  → persist config.json project entry ("profile": id)      [ADR-035]
  → invalidate MenuBuilder cache + tree filter (file_filter)
  → status bar: "Profile: <name>"
```

### 6.5 Plugin Manager save

```
edit enabled/priority/labels in dialog
  → dirty=true
close → if dirty: Save / Discard / Cancel dialog           [ADR-034]
  → Save: profiles.save_yaml (v2 schema) ; if non-active profile: NO switch
  → invalidate cache
```

### 6.6 First launch with v1 data (FR-MIG-01)

```
detect config.json + profiles/*.yaml + Dependency_Tools/
  → copy *.v1.bak for every file we touch
  → Dependency_Tools/* → plugins/* (manifests rewritten: v1→v2 via §5.4.4 map;
     unmappable `method` items flagged in the migration report dialog)
  → _connectionSetting.json → settings.json (defaults merged)
  → profiles migrated in place (superset schema — v1 files usually untouched)
  → migration report dialog (what moved, what was flagged)
```

---

## 7. Data Model (summary)

All persistence is flat files in the app dir or project root (v1 property,
kept — SVN-friendly, portable). See SRS §7 for the table. Invariants:

- `config.json` projects array order = display order (v1).
- A profile file is never written except by the Plugin Manager or profile
  Add/Export; the core never rewrites profiles implicitly.
- `.nausica.yaml` is never created by the core (user/agent-authored).
- Migration writes `*.v1.bak` next to each modified file, once.

## 8. UI Design

### 8.1 Main window (ported from `Nausica.ui`, parity B-05)

```
┌ menubar: File | Profile | AI | About ────────────────────────────────────┐
│ ┌ Project list ┐ ┌ Workspace file tree ──────────────────────────────┐ │
│ │ (config.json)│ │ Name        Size   Type   Date modified           │ │
│ │ + Add button │ │  (all files; junk hidden; extension-filter menu)  │ │
│ └──────────────┘ └───────────────────────────────────────────────────┘ │
│ ┌ Event Log dock (optional, hidden by default) ───────────────────────┐ │
│ │ [10:31:02][canforge] DBC→MD done (37 frames, 1.9 s)                 │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
│ status bar: Nausica 2.0.0-r1234 · ATC_Project · profile: default · 1 sel │
└──────────────────────────────────────────────────────────────────────────┘
```

New vs v1: the Event Log dock (FR-CORE-12) and the "Start" action's explicit
naming. Everything else — labels, views (Full/Basic), web links — is v1.

### 8.2 Context menu layout (rules)

```
<selection-specific built-ins>      ← profile toggles (Open/Open With, Start,
                                       Add, Delete, Rename, Copy Path, …)
───────────────────────────────────
<plugin A submenu>                  ← priority order; submenu title = plugin
<plugin B submenu>                   name (v1 grouping, preserved)
…
```

### 8.3 Dialogs

All v1 dialogs ported 1:1 (Project editor, Switch/Add profile, Plugin
Manager incl. inspection pane + save guard, Plugin Status, Prompt/AI template
editor, About with plugin-health table + version + changelog, Update
notification, Timesheet reminder, message/progress). New: **Migration report**
(§6.6) and **Migration/dependency hint** (exit code 3). No new dialog is
introduced for any v1 flow.

## 9. Error Handling, Logging, Testing Strategy

### 9.1 Error handling (BR-6)

| Fault | Detection | User-visible behavior |
|-------|-----------|-----------------------|
| Bad manifest | JSON Schema at load | Plugin Status `invalid-manifest` + reason; no menu; app fine |
| Unknown capability / bad condition key | load-time validation | same as above (fail at load, not click) |
| Script missing at click | ProcessRunner precheck | v1-style "Missing Script" dialog |
| Script crash (exit ≠ 0) | exit code + stderr tail | failure dialog + EventLog; status `error` |
| Hang | timeout (process 300 s / provider 3 s) | kill, failure dialog |
| Provider malformed JSON | parse check | item skipped, status `error`, log |
| Core uncaught exception | global Qt handler | logged to file + EventLog; app continues |

### 9.2 Logging

Rotating `logs/nausica.log` (5×1 MB) + EventLog dock (session). Every plugin
run line: `ts plugin mode cmd exit duration` — a run is fully reconstructable
(NFR-09).

### 9.3 Test strategy (NFR-08)

- **Unit (headless, no Qt):** manifest validation truth table, condition DSL
  truth table (port v1 `Test/test_structured_artifact.py` condition cases),
  override resolver (port v1 cases), protocol parser, token expansion,
  profile migrate (v1 fixtures → expected v2), menu **model** building
  (model ≠ widgets — assert structure, priorities, pruning).
- **Integration (offscreen Qt):** menu rendering incl. dynamic nodes;
  ProcessRunner against fixture scripts (all exit codes, timeout,
  cancellation); capability registry (schema rejection, pipe round-trip
  `pick_file`/`dialog`).
- **Parity suite** (`Test/parity/`): one script per SRS §3 baseline row
  (B-01…B-23) — headless where possible, `QT_QPA_PLATFORM=offscreen` where
  not; the v1 structured-artifact suites re-targeted (SRS §8.4).
- **Chaos (C2):** segfaulting script, infinite loop, provider printing
  garbage, manifest with duplicate id, plugin dir deleted mid-session —
  assert core stays responsive (automated: UI-thread watchdog timer).
- **Core purity test:** grep core + pluginhost sources for plugin ids,
  tool names (`canforge`, `structart`, `docx`, …) and for
  `getattr(self.main_window` patterns; fail CI on hits (C1, SRS §8.2).

## 10. Parity Matrix — Nausica v1 → Nausica v2

Mechanism: **P** = ported unchanged, **R** = redesigned (behavior preserved,
mechanism per boundary rules), **N** = new, **M** = moved into plugin payload.

| v1 baseline (SRS §3) | v2 home | Mech | Notes |
|---|---|---|---|
| B-01 Projects | `core/registry.py` + dialog | P | |
| B-02 File tree + junk hide + ext filter | `core/tree.py` | P | junk list → settings.json |
| B-03 Built-in menu actions | `pluginhost/menubuilder.py` (built-ins) | P | `start` explicit in profile toggles |
| B-04 Extension/path-keyword branching | condition DSL | P | unchanged keys |
| B-05 Menubar + views + links | shell | P | |
| B-06 Profile switch + persist | `pluginhost/profiles.py` | P | ADR-035 semantics |
| B-07 Plugin Manager + save guard | shell dialog | P | ADR-034 |
| B-08 Plugin health in About | PluginStatus source = PluginRegistry states | R | richer states (SRS FR-CORE-08) |
| B-09 Doc conversion pipelines | `plugins/docconvert/` | M+R | `progress` mode + `artifact`/`open` protocol replaces ad-hoc "open output" |
| B-10 CANForge (all ADR-022…028/036 flows) | `plugins/canforge/` | M | app-side `run_can_forge_cli` etc. become plugin scripts; dashboards `detached` |
| B-11 Structured Artifact menu | `plugins/structured_artifact/` | M+R | Export As… = canonical provider; validation via `validate_artifact.py` subprocess |
| B-12 AI (grammar, prompt tools) | `core/ai.py` + `prompt_ai`/`ai_assist` caps | R | single AI service; detail dialog keeps its own AIWorker via the service |
| B-13 Time tracking + reminder | `core/time.py` | P | |
| B-14 SemVer + SVN update + packaging | `core/update.py`, `core/version.py` | P | |
| B-15 Visual validation | `plugins/visual_validation/` + `open_hardware_dialog` cap | M+R | core keeps settings only |
| B-16 Tree drag-drop move | shell | P | |
| B-17 Feedback buttons | shell | P | |
| B-18 Discovery | `pluginhost/discovery.py` | R | + rescan action, status states |
| B-19 Condition DSL | `pluginhost/conditions.py` | P+N | + `project_profile`, `env` |
| B-20 Handlers | capabilities + subprocess modes | R | `method` removed (§5.4.4); `prompt_tool`→`prompt_ai`; `open_with`→`open_in/terminal/agent` |
| B-21 Menu item shape | manifest schema v2 | P+N | `nested`/`dynamic` kept; `dynamic` now provider protocol |
| B-22 3-layer overrides | `pluginhost/overrides.py` | P | semantics frozen by tests |
| B-23 Profile shape | `profiles/*.yaml` v2 | P+N | superset; `start` toggle explicit |
| — (none in v1) | `plugins/terminal_launcher/` | N | FR-REF-06 (mobaXterm et al.) |
| — (none in v1) | `plugins/agent_launcher/` | N | FR-REF-07 (Claude Code et al.) |
| — (none in v1) | Event Log dock, `nausica_cli` pipe, migration wizard | N | FR-CORE-12, §5.4.3, §6.6 |

## 11. Key Decisions, Risks, Roadmap

### 11.1 Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| D-1 | **Keep PySide2 + `Nausica.ui` workflow** for the rebuild (re-evaluate PySide6 in a later ADR) | UI parity (user requirement) with zero widget-migration risk; the redesign is about the boundary, not the toolkit. PySide6 migration is mechanical and deferred. |
| D-2 | **Subprocess isolation by default; capabilities closed set** | BR-2/BR-3. v1's openness (`method`) was the failure mode. |
| D-3 | **`nausica_cli` + loopback pipe for conversational capabilities** | Minimal surface that restores v1's `method`-dialog power (pickers, confirms) without imports. Only long-lived endpoint in core. |
| D-4 | **Providers as subprocess scripts, lazy on submenu open** | Fixes L-2 while keeping the 250 ms menu budget; domain logic (export catalog) lives with the artifact plugin. |
| D-5 | **`plugins/` replaces `Dependency_Tools/`** (rename at migration) | Names the relationship honestly; tools *are* plugins. v1 dir name survives only in the migration report. |
| D-6 | **Keep all v1 config formats readable** | C3; teams switch without re-entry. |
| D-7 | **Fresh ADR series for v2** under `ArchitectureDecisionRecord/v2/` | v1 ADRs stay the record of v1; v2 decisions reference them by number. |

### 11.2 Risks

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| v1 `method` port underestimates hidden app-state dependencies (e.g. CANForge flowline reads tree state) | Medium | Parity suite per plugin before cutover; flowline-style tools get `ctx` + settings access; anything genuinely needing live UI state is reclassified as a capability (closed set — deliberate, small) |
| Provider latency makes dynamic menus feel slow on first open | Medium | Lazy-on-open + per-context caching (§6.3); provider budgets in manifest; hint row while loading |
| Pipe endpoint adds a security surface | Low | Loopback-only, JSON requests, per-process path, closed at exit; documented in NFR-07 |
| Scope creep: "while we're rebuilding…" | High (process) | This SRS is the contract; anything new is a new FR + ADR, never inline. Parity matrix is the done-definition. |
| Two apps running in parallel during migration | Low | Feature flag + separate `plugins/` per app dir; v1 untouched until parity suite is green (SRS §8) |

### 11.3 Roadmap (each phase ends with the parity subset green)

| Phase | Deliverable | Exit criteria |
|-------|-------------|---------------|
| **M0 Scaffold** | `nausica2/` skeleton, core components empty, test harness offscreen, core-purity test | repo builds, CI green |
| **M1 Core shell** | FR-CORE-01…05, 10 (projects, tree, built-in menu, time) | B-01…03, 13, 16 parity |
| **M2 Plugin framework** | FR-PLUG-01…16: discovery, schema, conditions, overrides, menubuilder, runner, protocol, capabilities, providers, shim | chaos + framework unit suites green; dummy-plugin demo works (SRS §8.3) |
| **M3 Management UI** | FR-CORE-06…09, 11, 12: profiles, plugin manager/status, settings, update, event log | B-05…08 parity; ADR-034/035 guard tests |
| **M4 Reference plugins** | FR-REF-01…08 ports, in priority order: structured_artifact, canforge, docconvert, translation, testkit, terminal_launcher, agent_launcher, visual_validation | per-plugin parity checklists |
| **M5 Cutover** | migration wizard (FR-MIG-01), full parity run, packaging (ADR-015 parity), docs (`plugins/PLUGIN_SPEC.md`), v2 ADR series opened | SRS §8 acceptance 1–6 |

Suggested effort shape: M1–M3 ≈ the "core" a reviewer can hold in one
sitting; M4 is largely porting work that parallelizes per plugin; M5 is
verification + packaging.

## 12. Open Questions (for review)

1. **Capability set additions:** does any v1 tool need live tree/UI state
   beyond the context object? (Expected answer: flowline-style dashboards —
   plan is to pass project root + file and run detached; confirm during M4.)
2. **`project_profile` in conditions** — confirm profiles won't need
   per-plugin *args* variation that only `overrides` can express today
   (overrides already cover it; this key is a convenience).
3. **PySide6** — defer (D-1) or plan in v2.0? Recommended: defer.
4. **Plugin auto-update** (SVN per-plugin rev in Plugin Status) — v1 has app
   update only; defer unless requested.
