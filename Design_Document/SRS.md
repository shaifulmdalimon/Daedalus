# Nausica v2 — Software Requirements Specification

**Product:** Nausica (rebuild) — plugin-driven engineering workbench
**Document status:** Draft v0.9 — for review
**Date:** 2026-09-26
**Companion:** [SAD.md](SAD.md) (Software Architecture Design)
**Reference system:** Nausica v1 (this repository) — behavioral and UI reference

---

## 1. Introduction

### 1.1 Purpose

This SRS specifies the requirements for a ground-up rebuild of Nausica.
Nausica is a Windows desktop application that serves as an ASPICE-aware ALM
(Application Lifecycle Management) hub for automotive/embedded software
engineering: it manages engineering projects, a workspace file tree, and an
extensible right-click context menu driven by plugins, orchestrating 13+
specialized sub-tools (CAN bus tooling, document conversion, structured
artifacts, AI assistance, visual validation) over Markdown+YAML artifacts.

The rebuild keeps **all** of Nausica v1's observable behavior and UI as the
baseline (parity is a first-class requirement, §4.4) and redesigns the
**core/plugin boundary from first principles** so that:

- the core contains no tool-specific code,
- any new capability (including "right-click run mobaXterm", "right-click run
  an AI agent") is delivered as a drop-in plugin folder,
- plugins are isolated, versioned, and cannot crash or corrupt the core.

### 1.2 Scope

**In scope:**

- Core workbench: project registry, workspace file tree, context-menu host,
  profile system, plugin host (discovery/validate/enable), plugin manager and
  status UI, settings, session time tracking, versioning & update checking,
  notifications/event log, local AI service, terminal/external-app launching.
- Plugin framework v2: manifest schema, condition DSL, handlers/capabilities,
  dynamic-menu provider protocol, subprocess output protocol, overrides.
- First-party reference plugins ported from v1 `Dependency_Tools/`: CANForge,
  Structured Artifact, document conversion (MD↔DOCX/PDF/EPUB/PPTX, PDF OCR,
  Image→MD), translation, test tooling (TestConfigurator, test preview, code
  injection), visual validation.
- Two **new** first-party plugins demonstrating the extensibility goal:
  **Terminal/External-App launcher** (mobaXterm, Windows Terminal, …) and
  **Agent launcher** (Claude Code, configurable agent CLIs).

**Out of scope:**

- Reimplementation of any sub-tool's *internal* logic (CANForge's DBC
  parser, docling pipelines, UDS test engines, etc.). Sub-tools are carried
  over as plugin payloads unchanged where possible; only their **integration
  surface** (menu, invocation, progress) changes.
- Multi-user/network deployment. Nausica remains a single-user, local-first
  desktop tool (LAN services such as Ollama or the mosquitto broker are
  external dependencies).
- Web/mobile front ends.

### 1.3 Definitions & Abbreviations

| Term | Meaning |
|------|---------|
| **Core / Host** | The Nausica application itself: shell, workbench services, plugin host. Contains no tool logic. |
| **Plugin** | A directory containing a `plugin.yaml` manifest v2 and arbitrary payload (scripts, prompts, templates, UI). |
| **Manifest** | The declarative `plugin.yaml` describing a plugin's identity, menu contributions, conditions, and handlers. |
| **Profile** | A named, ordered set of enabled plugins with optional overrides, assigned to a project. |
| **Context object** | The JSON snapshot of the current selection/project passed to plugins (SAD §5.5). |
| **Capability** | A typed, core-provided service entry point (open-in, terminal, agent launch, prompt dialog, …) that plugins may invoke by name. Replaces v1's stringly-typed `method` handler. |
| **Provider** | A plugin-side script that answers "what submenu children apply here?" with a JSON list, enabling dynamic menus without core callbacks. |
| **Artifact** | An engineering document (requirement, design, test case) authored as Markdown with YAML frontmatter, per v1's `Schema/`. |
| **Parity** | The rebuilt app offering the same observable behavior as Nausica v1 for a given v1 feature. |
| **FR / NFR** | Functional / Non-functional requirement. |
| **V- / I- / D-** | Verification by test / inspection / demonstration. |

### 1.4 References

- Nausica v1 `CLAUDE.md` (architecture overview, most current description).
- `ArchitectureDecisionRecord/ADR_019_Profile_And_Plugin_Architecture.md` —
  the v1 plugin/profile decision this rebuild supersedes *as an architecture*
  while preserving its proven semantics (profiles, overrides, conditions).
- `plugin_engine.py`, `handler_dispatcher.py` — v1 implementation being
  redesigned (see SAD §10 parity matrix).
- `profiles/default.yaml`, `profiles/whud-t28.yaml`, `config.json` — v1 config
  shapes that must remain readable or migratable.
- `What_I_Know_About_Nausica.md` — behavioral inventory of v1.

---

## 2. Overall Description

### 2.1 Product Perspective

Nausica v2 is a self-contained Windows desktop application. It is the **hub**;
all specialized functionality arrives as plugins in a `plugins/` directory
(v1's `Dependency_Tools/`, renamed). The core never imports plugin code, and
plugins never import core code — they communicate through files (YAML/JSON),
subprocess arguments, a documented stdout protocol, and named core
capabilities.

The core exposes three surfaces:

1. **User surface** — the workbench UI (projects, file tree, context menu,
   profile/plugin management, settings, logs).
2. **Plugin surface** — the plugin framework: discovery, manifest schema v2,
   condition evaluation, handler dispatch, capability registry, context object,
   output protocol.
3. **Persistence surface** — `config.json` (projects), `profiles/*.yaml`,
   project-local `.nausica.yaml`, `settings.json`, `time_log.json`.

### 2.2 User Classes

| Class | Description | Primary needs |
|-------|-------------|---------------|
| **Embedded/automotive engineer** (primary) | Works in one of ~20 engineering projects; authors/reviews Markdown+YAML artifacts; runs conversions, CAN tooling, test configuration. | Fast right-click access to the right tool for the right file; per-project tool sets (profiles); no code editing to adopt a new tool. |
| **Tool developer** (secondary) | Builds new sub-tools and packages them as plugins (e.g. a new converter, a new agent launcher). | A stable, documented, versioned plugin contract; a way to test plugins without shipping the app; clear error reporting for bad manifests. |
| **AI agent** (tertiary, deliberate) | An AI coding agent (Claude Code, etc.) asked to "add a tool to Nausica". | The whole extension path is text files (YAML manifest + script) — no core code generation. This is an explicit v1 goal (ADR-019) that v2 makes structurally true. |
| **Team lead / auditor** | Needs traceability and records (ASPICE). | Time tracking, event log, versioned app + changelog, artifact schema validation surfaced in the UI. |

### 2.3 Operating Environment

- **OS:** Windows 10/11 (64-bit). v1 is Windows-only (`os.startfile`,
  PowerShell terminal, PyInstaller portable packaging); v2 keeps this.
- **Runtime:** Python 3.x, single interpreter for core and all plugin
  subprocesses (v1 behavior). Qt Widgets (PySide2 retained for UI parity —
  SAD §11 decision D-1).
- **External dependencies (optional, feature-gated):** Ollama (local AI,
  LAN), mosquitto (MQTT broker, auto-started by visual-validation plugin),
  FFmpeg (video), svn client (update checking), VS Code / mobaXterm / agent
  CLIs (external-app launch). Missing dependencies must degrade gracefully:
  the affected menu items disable or fail with a clear message; the app
  starts and everything else works.

### 2.4 Design Constraints

- C1: The core must not contain any tool-specific logic (no CAN, artifact,
  conversion, or OCR code paths). Enforced by inspection + a "core purity"
  test that greps the core for plugin identifiers (SAD §9).
- C2: Plugin failure isolation: a malformed manifest, a crashing plugin
  script, or a hung plugin must never crash or freeze the core UI.
- C3: v1 config files must be readable or auto-migratable (config.json,
  profiles/*.yaml, .nausica.yaml, time_log.json) — teams switch over without
  re-entering data.
- C4: The context menu must build within 250 ms for the largest expected
  profile (10 plugins × ~30 items) on a mid-range Windows laptop.
- C5: Extension is text-file-only: adding a fully functional plugin requires
  no modification of core source.

### 2.5 Assumptions & Dependencies

- A1: Engineers run one interpreter environment for core + plugins (same as
  v1); per-plugin `requirements.txt` is installed into that environment.
- A2: The `plugins/` directory is writable by the user (portable install
  layout, per ADR-015).
- A3: AI features depend on Ollama being reachable at the configured URL;
  all AI features are optional.
- A4: v1's behavioral catalog (SRS §3) is complete enough to serve as the
  parity baseline; gaps found during migration are logged as new FRs, not
  silently dropped.

---

## 3. Nausica v1 Behavioral Baseline (Parity Catalog)

The following catalog is the "we like whatever the current Nausica does"
reference. Every row is a parity obligation unless SAD marks it
**Redesigned** (behavior preserved, mechanism changed) or **New**.

### 3.1 Core workbench (v1 `Nausica_APP.py`)

| # | v1 behavior |
|---|-------------|
| B-01 | Left panel lists projects from `config.json` (name, test_case_root, source_code_root); add/edit projects via dialog; selection drives everything else. |
| B-02 | File tree = `QFileSystemModel` rooted at the selected project's `test_case_root` (or source root); **all files shown by default**; junk dirs (`.git`, `.svn`, `__pycache__`, …) hidden; per-profile "Filter by Extension…" submenu (ADR-032), state persisted in profile YAML (`file_filter.*`). |
| B-03 | Right-click context menu: built-in actions (Open / "Open With" submenu for directories, Start with OS default program for files, Add, Delete, Rename, Copy Path, group copy/move with directory target, preview directory) **always present**, plugin submenus appended per active profile. |
| B-04 | Menu branching on file extension (`.json`, `.md`, `.yaml`, `.dbc`, `.asc`, `.epub`, `.mp4`, `.docx`, `.pdf`, …) and relative-path keywords (`test` → test actions, `design` → design actions). |
| B-05 | Menubar: File (open project dir, open at filesystem, terminal, full/basic view, quit), Profile (Switch, Add, Disassociate, Export, Delete, Plugin Manager, Plugin Status), AI/Tools links (web UI, server dashboard, PowerPoint), About (About Nausica, Check for Update, View Changelog). |
| B-06 | Profile switch (dialog, "Switch to it now?" on Add) persists the choice to the project's `config.json` entry (ADR-035); Disassociate and deleting the active profile clear it. |
| B-07 | Plugin Manager dialog: enable/disable plugins per profile, inspect handlers/conditions, YAML save-back; closing with unsaved changes prompts Save / Discard / Cancel (ADR-034); saving a non-active profile does not switch profiles. |
| B-08 | About dialog shows plugin health table (active/inactive per loaded plugin). |
| B-09 | Document conversion pipeline: EPUB→MD, DOCX→MD (docling / markitdown), PDF→MD (DeepSeek OCR), MD→DOCX, MD→PDF — pattern: validate input → `QProgressDialog` → subprocess → open output. |
| B-10 | CANForge: DBC→Markdown, plan-from-MD (chat mode, AI), JSON→ASC, ASC playback/recording, dashboard, open/closed-loop UDS/ISO-TP workflows (ADR-022…028, 036). |
| B-11 | Structured Artifact unified menu: Info, Validate Against Schema, Export As… (dynamic variants, 8-preset catalog), Import From… (DOCX/PDF/Image), AI Assist (grammar, translate); `.structArt.yaml` variant with Info + Validate only (ADR-038…042). |
| B-12 | AI: Ollama-backed grammar check / text improve in test detail dialog (`detail_dialog.py`, `AIWorker` QThread); AI prompt-tool dialog pattern (template editor → script with `--input --prompt-file --output`). |
| B-13 | Time tracking: session logging to `time_log.json` (configurable per-project checkbox); daily timesheet reminder popup (ADR-029). |
| B-14 | Versioning: SemVer `version.py` + `full_version()`; window title + About show it; SVN update checker thread on startup (ADR-012); update dialog with changelog preview; PyInstaller portable packaging (ADR-015). |
| B-15 | Hardware visual validation: camera dialog (OpenCV), MQTT via auto-started mosquitto, FFmpeg recording, Arduino serial (`_connectionSetting.json`). |
| B-16 | Tree drag-and-drop file move (ADR-021). |
| B-17 | Feedback button, local-first feedback (ADR-010/013). |

### 3.2 Plugin framework (v1 ADR-019, `plugin_engine.py`, `handler_dispatcher.py`)

| # | v1 behavior |
|---|-------------|
| B-18 | Auto-discovery of `Dependency_Tools/*/plugin.yaml` at startup; invalid manifests skipped with warnings. |
| B-19 | Condition DSL: `file_extension`, `name_suffix`, `name_suffix_not`, `dir_keyword`, `is_file`, `min_selection`, `max_selection`, string shorthand. |
| B-20 | Handlers: `method` (string-named MainWindow call, optional `args` list, ADR-039), `subprocess` (script + args_template tokens `{input} {input_dir} {input_stem} {output} {root} {app_dir}`; progress-dialog or terminal modes), `prompt_tool` (AI prompt dialog), `open_with` (vscode/explorer/terminal/agent). |
| B-21 | Menu items: `action`, `submenu` (flattened by default), `separator`; nested/dynamic submenus (`nested: true`, `dynamic: "<key>"` with in-app Python provider callback, ADR-038). |
| B-22 | Three-layer overrides: plugin default < profile `overrides` (label_override, args_override with replace/append merge) < project-local `.nausica.yaml` (local wins). |
| B-23 | Profiles: `profiles/*.yaml` with ordered `plugins` (id, enabled, priority), `built_in_actions` toggles, `file_filter`, `overrides`, `tags`; `default.yaml` fallback. |

---

## 4. Functional Requirements

Requirement IDs: **FR-CORE-\*** (core workbench), **FR-PLUG-\*** (plugin
framework), **FR-REF-\*** (reference/first-party plugins). Each carries a
verification method (V-test / I-inspection / D-demonstration).

### 4.1 Core — workbench

**FR-CORE-01 Project registry** *(parity B-01)*
The core SHALL provide a project list with name, `test_case_root`,
`source_code_root`; add/edit/delete via dialog; persistence in `config.json`
(schema-compatible with v1). Projects without a profile fall back to the
default profile. *V (test: registry CRUD + persistence; load v1 config.json
unmodified).*

**FR-CORE-02 Project selection context** *(B-01)*
Selecting a project SHALL load its roots, active profile, local overrides
(`.nausica.yaml` if present), and time-tracking setting, and refresh the file
tree and menu context. *D.*

**FR-CORE-03 Workspace file tree** *(B-02, B-16)*
The core SHALL show a file tree of the selected project root showing **all
files by default**, hiding only the configurable junk-directory list, with
columns Name/Size/Type/Date, drag-and-drop move, and a per-profile
"Filter by Extension…" submenu listing extensions present in the folder;
filter state persisted in the profile YAML (`file_filter.show_all`,
`file_filter.enabled_extensions`). *V.*

**FR-CORE-04 Context-menu host** *(B-03, B-04)*
On right-click in the tree, the core SHALL assemble a menu consisting of:
(a) built-in actions gated by the profile's `built_in_actions` toggles;
(b) one submenu per enabled, condition-satisfied plugin, ordered by profile
priority; (c) separators. Assembly SHALL be ≤ 250 ms for the reference
profile (C4). No plugin Python code SHALL be imported by the core during
assembly. *V (test incl. timing).*

**FR-CORE-05 Built-in file actions** *(B-03)*
The core SHALL always provide (per profile toggles): **Open** (directories:
"Open With" submenu), **Start** (files: OS default program), **Add**
(folder/file), **Delete** (with confirmation), **Rename**, **Copy Path**,
**group copy/move** with directory target, **preview directory**. These are
core features, *not* plugins. *V.*

**FR-CORE-06 Profile management** *(B-06)*
The Profile menu SHALL provide: Switch Profile (dialog; switching applies
immediately and persists to the project's `config.json` entry), Add New
Profile (with optional "Switch to it now?"), Disassociate (revert to default),
Export (save YAML anywhere), Delete (with guard: deleting the active profile
clears the project's assignment; ADR-035 semantics). *V.*

**FR-CORE-07 Plugin Manager** *(B-07)*
A dialog SHALL list every discovered plugin with: enabled state (per current
profile), priority, manifest summary, per-item handler/condition inspection,
and enable/priority editing; changes saved back to the profile YAML. Closing
with unsaved changes SHALL prompt Save / Discard / Cancel — never discard
silently (ADR-034). Saving a non-active profile SHALL NOT switch profiles.
*V.*

**FR-CORE-08 Plugin Status / health** *(B-08)*
The core SHALL report per-plugin status: `ok`, `disabled-in-profile`,
`invalid-manifest` (with reason), `missing-dependency`, `error` (last failure
summary). The About dialog SHALL include this table. *V.*

**FR-CORE-09 Settings**
`settings.json` (v1's `_connectionSetting.json`, superseding it) SHALL hold:
AI endpoint (Ollama URL/model), MQTT broker URL, Arduino serial port,
**named external apps** (mobaXterm, Windows Terminal, editors, agent CLIs —
each a command template with a `{path}` placeholder), junk-directory list,
update-check SVN URL. Missing file → defaults + created on first save. *V.*

**FR-CORE-10 Time tracking** *(B-13)*
Per-project session time SHALL log to `time_log.json` (v1-compatible format)
when enabled; a daily reminder popup at the configured hour prompts a
timesheet entry. *V.*

**FR-CORE-11 Versioning & updates** *(B-14)*
`version.py` with `__version__` (SemVer) + auto-stamped build revision;
`full_version()` used in title bar, About, and update dialog. On startup a
background checker SHALL compare local vs. remote SVN revision and offer an
update dialog with changelog preview from `changelog/CHANGELOG.md`.
*V.*

**FR-CORE-12 Notifications & event log** *(partly new)*
The core SHALL maintain a non-blocking event log (dockable panel): every
plugin launch, progress event, failure, and core action is appended with
timestamp and plugin attribution. Long-running actions SHALL show the v1-style
cancellable progress dialog **and** stream into the log. Errors from plugins
SHALL surface as log entries + message boxes, never as unhandled exceptions.
*D.*

**FR-CORE-13 Local AI service** *(B-12)*
The core SHALL provide a single AI service (Ollama, configurable endpoint)
used by plugins via the `prompt_ai` capability and the `ai_assist`
capability; AI unavailable → dependent menu items disabled with a tooltip,
no crash. *D.*

**FR-CORE-14 External launch capabilities** *(B-03 Open With, B-12)*
The core SHALL implement the capabilities in §4.2.4 (open_in, terminal,
agent, pick_file, prompt_ai, dialog, notify) as the *only* way plugins reach
core services. *I (boundary inspection).*

**FR-CORE-15 Hardware/visual validation support** *(B-15)*
The core SHALL expose serial-port and MQTT settings and an
`open_hardware_dialog` capability so the visual-validation plugin can run
without core knowing camera/MQTT details. (Payload logic stays in the
plugin.) *I.*

### 4.2 Plugin framework

#### 4.2.1 Discovery & lifecycle

**FR-PLUG-01 Discovery** *(B-18)*
At startup (and on a manual "Rescan") the core SHALL scan `plugins/*/` for a
`plugin.yaml`, validate it against the manifest schema v2 (JSON Schema), and
register it. Invalid manifests are skipped, reported in Plugin Status with the
schema error, and never abort startup. *V (test: valid, malformed, missing,
non-dict, duplicate-id manifests).*

**FR-PLUG-02 Lifecycle**
Each plugin SHALL have states `discovered → validated → enabled |
disabled-in-profile → (error)`. Enable/disable takes effect on the next menu
build (no restart). A plugin whose `min_core` version exceeds the running core
is reported `incompatible` and its menu items suppressed. *V.*

**FR-PLUG-03 Versioning**
Manifests carry SemVer `version`; the core stores last-seen versions and the
Plugin Manager shows version changes on rescan (informational). *I.*

#### 4.2.2 Manifest & conditions

**FR-PLUG-04 Manifest schema v2**
`plugin.yaml` SHALL declare: `id` (unique, kebab-case), `name`, `version`,
`min_core`, optional `description`, `tags`, `entry` (default script),
`requires` (python packages for status checking), `external_deps` (named
binaries, e.g. ffmpeg — checked for Plugin Status), `menu_items[]`,
`capabilities_used[]` (static list, validated at load). Full schema: SAD §5.3.
*V (schema tests).*

**FR-PLUG-05 Condition DSL** *(B-19, extended)*
Each menu item SHALL support a declarative `condition` evaluated against the
context object: `file_extension` (list/str), `name_suffix`, `name_suffix_not`,
`dir_keyword`, `is_file`, `min_selection`, `max_selection`, string shorthand
(`".dbc"` → extension + is_file), **new:** `project_profile` (id or tag),
`env` (e.g. `{os: windows}`, `{capability: agent}` — item hidden when the
capability is unavailable). All conditions AND together. Unknown keys →
manifest invalid (fail at load, not at click). *V (truth-table tests).*

**FR-PLUG-06 Menu contribution shape** *(B-21)*
`menu_items[]` items SHALL support `type: action | submenu | separator`,
static `children`, `nested: true` (real submenu), `dynamic: "<provider_key>"`
(provider protocol, §4.2.5), and per-item `handler` + `handler_args`.
Separator and empty-submenu pruning behavior preserved from v1. *V.*

#### 4.2.3 Handlers & capabilities

**FR-PLUG-07 Subprocess handler** *(B-20, Redesigned)*
`handler: subprocess` SHALL run `sys.executable <plugin_dir>/<script_path>`
with token-expanded `args_template` (v1 tokens plus **new** `{ctx_json}` —
path of a written context file — and `{project_name}`), in one of the modes
`progress` (cancellable modal progress + streaming log), `terminal` (new
terminal window, v1 behavior), `detached` (fire-and-forget, e.g. dashboards).
Exit code 0 = success (information dialog), 1 = failure (stderr tail in
dialog + log), 2 = user-cancelled (silent), 3 = missing dependency (friendly
"install hint" dialog if the manifest declares `install_hint`). A subprocess
that exceeds `timeout` (manifest, default 300 s) is killed and reported.
*V.*

**FR-PLUG-08 Capability handler** *(B-20 `method`/`open_with`/`prompt_tool`,
Redesigned — the key boundary fix)*
`handler: capability` SHALL invoke a named, typed core capability with JSON
`args` (schema per capability, SAD §5.4). The v1 stringly-typed `method`
handler and its implicit access to all `MainWindow` state are **removed**.
A manifest naming an unknown capability is invalid at load time. *V + I.*

**FR-PLUG-09 Backward compatibility shim**
During the migration window (SAD §11), the core MAY accept v1 manifests
(`handler: method/open_with/prompt_tool`, `dynamic:` with no provider script)
by mapping them onto v2 equivalents, logging a deprecation warning per use.
The shim SHALL be removable without user-visible change once all first-party
plugins are v2. *V.*

#### 4.2.4 Core capabilities (the complete plugin→core API)

Each capability: name, args schema, behavior, error behavior.

| Capability | Args | Behavior |
|------------|------|----------|
| `open_in` | `{app: "explorer"\|"vscode"\|<named external app>, path?}` | Open path/folder in the named app (settings external-app templates). |
| `terminal` | `{app?, cwd?, command?}` | Open a terminal at cwd, optionally pre-running a command (v1 `run_in_terminal`). |
| `agent` | `{agent: "claude_code"\|<named>, cwd?}` | Launch a configured agent CLI rooted at the selection (FR-REF-07's core side; the *menu* is a plugin). |
| `pick_file` | `{title, filters[], multi?}` | File picker; returns chosen paths to the calling plugin via the output protocol. |
| `prompt_ai` | `{title, prompt_file?, template_dir?, output_ext?}` | v1 prompt-tool dialog: template editor → invokes the plugin's script with `--input --prompt-file --output`. |
| `ai_assist` | `{mode: grammar\|translate\|custom, prompt?, file}` | Body-aware AI transform of an artifact's body (frontmatter never modified). |
| `dialog` | `{kind: info\|warn\|critical\|confirm, title, text}` | Standard message dialogs. |
| `notify` | `{level, text, plugin}` | Non-blocking notification + event log entry. |
| `open_hardware_dialog` | `{config_key}` | v1 camera/serial dialog host (FR-CORE-15). |
| `open_project` | `{action: add\|remove\|edit}` | Project registry dialogs (so a plugin can bootstrap projects). |

*V (per-capability tests) + I (no other core entry points exposed to plugins).*

#### 4.2.5 Dynamic menus (provider protocol)

**FR-PLUG-10 Providers** *(B-21 `dynamic`, Redesigned)*
A `dynamic: "<key>"` item SHALL make the host run
`sys.executable <plugin_dir>/providers/<key>.py --ctx <ctx_file>` with a
timeout (default 3 s, manifest-overridable); the script prints a JSON array of
child items (standard item shape, no conditions re-evaluated for provider
children — provider is trusted code, as in v1). No children → submenu shows a
disabled hint row (v1 behavior). Provider missing/crashing → item skipped +
Plugin Status `error` + log entry; core never freezes (C2). *V.*

#### 4.2.6 Output protocol

**FR-PLUG-11 Output protocol** *(Redesigned — v1 only captured stderr)*
Plugin stdout lines beginning with `NAUSICA-PROTO:` followed by JSON are
protocol messages: `progress {n, max?, msg?}`, `log {level, msg}`, `open
{path}`, `artifact {path, kind}`, `fail {code, msg}`. All other stdout lines
are plain log. Protocol is consumed in `progress` mode to drive the dialog and
event log; in `terminal` mode it is ignored (raw output shown). *V.*

**FR-PLUG-12 Artifact handback**
`artifact`/`open` protocol messages SHALL let a plugin request that the core
open its output file (v1 "→ open output" behavior) without the plugin shelling
out to `os.startfile` on the UI thread. *V.*

#### 4.2.7 Overrides, profiles, isolation

**FR-PLUG-13 Overrides** *(B-22, preserved)*
Profile-level `overrides` and project-local `.nausica.yaml` SHALL apply with
v1 semantics: local wins; `label_override`; `args_override` with
`replace`/`append` merge (append merges `handler_args` keys and extends
`args_template`). *V.*

**FR-PLUG-14 Profiles** *(B-23, preserved)*
`profiles/*.yaml`: ordered `plugins` (id, enabled, priority),
`built_in_actions` toggles, `file_filter`, `overrides`, `tags`, `description`.
Default profile fallback. Profile files are editable by the Plugin Manager and
hand-editable (YAML). *V.*

**FR-PLUG-15 Isolation** *(C2)*
A plugin crash, hang, or invalid output SHALL result in: failure dialog or log
entry, plugin marked `error` in status, core fully responsive. The core SHALL
run no plugin code in its own process, except provider scripts (bounded by
timeout) — and provider scripts are subprocesses, not imports. *V (chaos
tests: segfaulting script, infinite loop, malformed JSON provider output).*

**FR-PLUG-16 Text-only extension** *(C5)*
Adding a fully functional plugin SHALL require only files under
`plugins/<id>/` (manifest + payload). No core edit, no restart (rescan), no
code signing. *D (demonstration with a brand-new dummy plugin).*

### 4.3 Reference plugins (parity with v1 Dependency_Tools)

Each entry = port the v1 tool to a v2 plugin with menu parity.

**FR-REF-01 CANForge plugin** *(B-10, B-04)*
DBC→MD, E2E profile inspect/settings, plan-from-MD (chat/AI), JSON→ASC, ASC
playback/recording/dashboard, open/closed-loop UDS/ISO-TP workflow entries.
Menu labels, conditions (`.dbc`, `.can.md`, `.can.plan.json`, …) and
behaviors match v1. UDS dashboard and Flowline open as `detached`/terminal
windows. *V (parity checklist per menu item).*

**FR-REF-02 Structured Artifact plugin** *(B-11, ADR-038…042)*
Info, Validate Against Schema (with `{root}` manifest layer), Export As…
(**dynamic provider** over the 8-preset catalog — the canonical provider use
case), Import From… (DOCX/PDF/Image), AI Assist (grammar/translate);
`.structArt.yaml` branch (Info + Validate only); `name_suffix_not` exclusion
of legacy `.md` plugins preserved; "Add Feature Dir" scaffold. *V (parity
checklist + the v1 `Test/test_structured_artifact.py` suites re-targeted).*

**FR-REF-03 Document conversion plugin** *(B-09)*
EPUB→MD, DOCX→MD (docling, markitdown), PDF→MD (DeepSeek OCR, prompt_ai),
MD→DOCX, MD→PDF, Image→MD (prompt_ai). All as subprocess handlers with
`progress` mode; outputs opened via `artifact` protocol. *V.*

**FR-REF-04 Translation plugin** *(B-09/B-12)*
Markdown translation via AI (prompt_ai). *V.*

**FR-REF-05 Test tooling plugin** *(B-04)*
TestConfigurator, test preview HTML generation, test code injection, extract
test code, test expected-output recording — conditioned on `dir_keyword:
test` / design as in v1. *V.*

**FR-REF-06 Terminal & external-app launcher plugin** *(NEW — user request)*
Menu items: "Run mobaXterm here", "Run Windows Terminal here", "Run <named
app> with <template>" — all via the `terminal`/`open_in` capabilities, driven
by settings external-app templates. Demonstrates a plugin that is *pure
capability wiring*: zero payload scripts. *D.*

**FR-REF-07 Agent launcher plugin** *(NEW — user request)*
Menu items: "Run Claude Code here", "Run <named agent> on this file/folder"
via the `agent` capability with per-agent config in settings (CLI command,
extra args template). Conditions: any folder, or any file. *D.*

**FR-REF-08 Visual validation plugin** *(B-15)*
Camera dialog, MQTT (auto-started mosquitto), FFmpeg recording, Arduino
serial — using `open_hardware_dialog` capability + settings. *D.*

### 4.4 Parity & migration

**FR-MIG-01 Config migration** *(C3)*
First launch with v1 data SHALL auto-migrate (non-destructively, with a
`*.v1.bak` copy): `config.json` (unchanged schema), `profiles/*.yaml`
(v1→v2: v1 `handler: method/open_with/prompt_tool` items mapped via shim map
in SAD §5.4), `.nausica.yaml`, `time_log.json`, `_connectionSetting.json` →
`settings.json`. Migration is idempotent. *V.*

**FR-MIG-02 Parity verification**
A parity checklist (one row per B-xx in §3) SHALL be maintained in
`Test/parity/` and executed as part of acceptance before cutover. *I.*

---

## 5. External Interface Requirements

### 5.1 User interfaces

- **Workbench UI:** single main window — menubar (File, Profile, AI, About),
  left project list + add button, center file tree (4 columns), status bar
  (version, project, selection), optional dockable event-log panel. Layout,
  labels, and dialog styles follow v1 `Nausica.ui` (parity B-05).
- **Dialogs:** Switch/Add profile, Plugin Manager, Plugin Status, Project
  editor, Prompt/AI template editor, About (with plugin health + version +
  changelog), Update notification, Timesheet reminder, message/progress
  dialogs. All v1 dialog behaviors preserved (incl. ADR-034 guard).
- **Context menu:** see FR-CORE-04. Right-click target: file, folder, or
  empty tree space (folder-level actions).
- **Accessibility/usability:** v1 keyboard behavior retained (Ctrl+F search
  in tree if present in v1 build; menu shortcuts for Plugin Manager).
  No new mandatory interactions.

### 5.2 Software interfaces

| Interface | Direction | Contract |
|-----------|-----------|----------|
| Plugin manifest | plugin → core | YAML, manifest schema v2 (SAD §5.3), JSON-Schema-validated. |
| Context object | core → plugin | JSON file at a temp path, `{ctx_json}` token + `NAUSICA_CTX` env var (SAD §5.5). |
| Subprocess output | plugin → core | Stdout: `NAUSICA-PROTO:` JSON lines + plain log (FR-PLUG-11); exit codes 0/1/2/3. |
| Provider output | plugin → core | Stdout: JSON array of child items, 3 s default timeout (FR-PLUG-10). |
| Capabilities | plugin → core | Named, schema-checked invocations only (SAD §5.4); no other core API exposed. |
| Ollama | core → LAN | OpenAI-compatible REST at settings URL; optional. |
| mosquitto / FFmpeg / svn / external apps | core or plugin → OS | PATH binaries; feature-gated. |

### 5.3 Hardware interfaces

Camera (OpenCV), Arduino serial, MQTT broker — all owned by the visual
validation plugin (FR-REF-08); core only stores settings and hosts the dialog.

### 5.4 Communication interfaces

Local-first. No network calls except: Ollama (LAN, user-configured), SVN
update check (user-initiated or on-startup, LAN), and any tool that itself
needs the network (user's responsibility, surfaced in its output).

---

## 6. Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-01 **Reliability** | Core never crashes due to plugin failure (C2). App starts in < 5 s cold (10 plugins) on reference hardware. Uncaught exceptions in core are logged with stack to the event log and app continues. |
| NFR-02 **Performance** | Menu build ≤ 250 ms (C4); provider call ≤ 3 s default; tree handles 100k-file projects with the v1 model+proxy approach; plugin subprocess launch ≤ 1 s to first progress event for local scripts. |
| NFR-03 **Isolation/Safety** | Plugins run with the user's normal privileges (no sandboxing beyond subprocess); destructive actions (Delete, overwrite) always confirm. File writes by plugins are the user's explicit intent via a menu action. |
| NFR-04 **Maintainability** | Core < 3,000 lines (v1 main file is 7,615); every core component unit-testable headless (`QT_QPA_PLATFORM=offscreen`); plugin contract documented in one file (`plugins/PLUGIN_SPEC.md`) stable across patch versions. |
| NFR-05 **Extensibility** | Text-only extension (C5); adding a capability requires a core change + spec bump — capabilities are an intentionally *closed*, versioned set. |
| NFR-06 **Portability within Windows** | PyInstaller portable build (ADR-015 parity); same-interpreter plugin execution; no installer. |
| NFR-07 **Security** | Local tool; no credentials stored except user's own LAN endpoints. Plugin manifests are trusted (team-authored); unknown capability names rejected at load (no arbitrary code execution via YAML). |
| NFR-08 **Testability** | All framework behavior testable without GUI (engine, resolver, protocol parsers); parity suite (§4.4) runnable headless. |
| NFR-09 **Observability** | Rotating log file (`logs/nausica.log`, 5×1 MB) with plugin-tagged entries; every plugin run reconstructable (cmd, cwd, exit, duration, protocol events) from event log. |
| NFR-10 **Compatibility** | Reads v1 config artifacts (C3); `full_version()` SemVer scheme preserved; changelog Keep-a-Changelog format preserved. |

---

## 7. Data Requirements

| Data | Format | Owner | Migration |
|------|--------|-------|-----------|
| `config.json` | JSON (v1 schema + optional `profile` per project) | Core | read as-is |
| `profiles/*.yaml` | YAML v2 (superset of v1) | Core (Plugin Manager) | auto-migrate v1 handlers |
| `<project>/.nausica.yaml` | YAML (v1 override shape) | Project | read as-is |
| `settings.json` | JSON | Core | from `_connectionSetting.json` + new keys |
| `time_log.json` | JSON (v1 shape) | Core | read as-is |
| `plugins/<id>/` | manifest + payload | Tool developers | v1 `Dependency_Tools/` copied, manifests bumped to v2 |
| `structart_manifest.yaml` (per project) | YAML | Structured Artifact plugin | unchanged (consumed via `{root}` token) |
| `logs/nausica.log` | rotating text | Core | new |

---

## 8. Acceptance Criteria (summary)

1. **Parity:** 100% of §3 baseline rows pass the parity checklist (FR-MIG-02).
2. **Boundary:** core-purity test passes — no plugin identifier appears in
   core source; no `getattr(main_window, <string>)` style dispatch exists
   (C1, FR-CORE-14).
3. **Extension demo:** a reviewer adds a new dummy plugin (manifest + one
   script) by creating files only; it appears in the correct context after
   rescan; a deliberately broken plugin is reported in Plugin Status without
   affecting the UI (FR-PLUG-01/08/15/16).
4. **Reference plugins:** FR-REF-01…08 parity checklists pass; v1
   `Test/test_structured_artifact.py` suites (re-targeted) pass.
5. **NFRs:** startup < 5 s, menu build ≤ 250 ms, isolation chaos tests pass
   (NFR-01/02, C2).
6. **Migration:** a v1 working directory (config, profiles, one project)
   migrates on first launch with zero data loss and unchanged menu behavior
   (FR-MIG-01).

---

## 9. Traceability

Full requirement → v1 baseline → v2 mechanism traceability is maintained in
the SAD parity matrix (SAD §10) and the `Test/parity/` checklist (FR-MIG-02).
ADR numbering for v2 starts fresh in a new series; v1 ADRs remain the record
of v1 decisions and are referenced, not renumbered.
