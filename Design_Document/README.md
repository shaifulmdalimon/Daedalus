# Nausica v2 — Design Documents

Rebuild of Nausica as a **plugin-driven engineering workbench**. The current
Nausica (this repository) is the **behavioral and UI reference**: everything it
does — file tree, context menu, profiles, plugin manager, sub-tools, time
tracking, update system — is the baseline the rebuild must match (or
deliberately, documentedly improve on). The core/plugin boundary, however, is
redesigned from first principles.

## Documents

| Document | Content |
|----------|---------|
| [SRS.md](SRS.md) | Software Requirements Specification — vision, user classes, full functional requirements (core, plugin framework, reference plugins), NFRs, acceptance criteria. |
| [SAD.md](SAD.md) | Software Architecture Design — the redesigned core/plugin boundary, component design, manifest schema v2, protocols (context, provider, output), UI design, parity matrix vs. Nausica v1, roadmap, risks. |

## Reading order

1. **SRS.md §1–2** — what the product is and who uses it.
2. **SRS.md §4** — functional requirements (the contract the build must satisfy).
3. **SAD.md §3** — the boundary rules. This is the heart of the redesign: the
   core knows *no* plugin; plugins know the core *only* through the manifest
   schema, the context JSON, the capability registry, and the subprocess/output
   protocol.
4. **SAD.md §10** — parity matrix: every Nausica v1 capability mapped to a v2
   requirement, so "we like whatever current Nausica does" is verifiable.

## Key design stance (short version)

- **Core = host.** Project registry, workspace file tree, context-menu host,
  profile & plugin manager, settings, time tracking, notifications. Zero tool
  knowledge: no CAN, no artifact, no DOCX, no OCR code in core.
- **Plugin = folder.** A directory with a `plugin.yaml` (manifest v2) plus its
  payload (scripts, prompts, templates). Dropped into the plugin directory, it
  appears in the menu. This is how "right-click run mobaXterm" or "right-click
  run Claude Code" become first-party plugins, not core features.
- **Out-of-process by default.** Plugins run as subprocesses under the same
  interpreter (v1 behavior, kept). The v1 `method` handler — plugins calling
  arbitrary `MainWindow` methods by string name — is the biggest boundary leak
  in v1 and is replaced by a typed **capability registry** (SAD §5.4).
- **Dynamic menus without core callbacks.** v1's `dynamic_provider` callback
  (app code handed to the engine per artifact type) becomes a **provider
  protocol**: the host runs the plugin's own script with the context JSON and
  reads a JSON child list back (SAD §5.6).
- **Proven v1 semantics kept.** Three-layer overrides
  (plugin default < profile < project-local `.nausica.yaml`), condition DSL,
  profile persistence per project, plugin-manager save guard, show-all-files
  tree with junk hidden and per-profile extension filter.

## Status

| Doc | Status | Date |
|-----|--------|------|
| SRS.md | Draft v0.9 — for review | 2026-09-26 |
| SAD.md | Draft v0.9 — for review | 2026-09-26 |
