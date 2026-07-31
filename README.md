PierrotKnowledge

A very light desktop knowledge base. Notes are plain Markdown with YAML
frontmatter conforming to [Open Knowledge Format (OKF) v0.2][okf], arranged in
the LLM Wiki three-layer pattern, so that humans, text editors, and AI agents
all read and write the same files.

The application is a viewer, an editor, and an index. It is never the owner of
your notes: delete it and everything you wrote is still there, still readable,
still greppable, still in git.

[okf]: https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md

## Layout of a knowledge base

```
my-knowledge/
├── AGENTS.md          # schema and rules, read by agents first
├── README.md
├── raw/               # LAYER 1  immutable source; agents read, never write
│   ├── sources/  assets/  inbox/
├── wiki/              # LAYER 2  the OKF bundle — the canonical copy
│   ├── index.md       # reserved: directory listing (§8)
│   ├── log.md         # reserved: change history (§9)
│   └── concepts/ entities/ topics/ notes/ decisions/ procedures/ references/ syntheses/
└── .rag/              # LAYER 3  derived; delete it and it rebuilds
    └── fts.sqlite     # BM25 full-text index
```

The layer boundary is enforced in code, not by convention: `raw/` and `.rag/`
are not writable through the app or through MCP.

## Repository layout

```
crates/pk-core/   OKF model, workspace, search index — all logic lives here
crates/pk-app/    Tauri 2 desktop shell
crates/pk-mcp/    MCP server for external agents
ui/               React + Vite + CodeMirror 6 frontend
docs/DECISIONS.md why things are the way they are
docs/MCP.md       driving the knowledge base from an agent
docs/TARGETS.md   measured results against the target metrics
```

## The application

Three panes: a navigation tree and search on the left, the note in the middle
(rendered, or a CodeMirror editor over the raw Markdown including frontmatter),
and on the right the OKF v0.2 signals that matter — `type`, `status`, trust tier,
staleness, tags — plus outgoing links, backlinks, and conformance findings.

Notes are rendered to HTML in Rust, not in the browser: the preview is treated
as a sanitiser first, since notes are written by agents and arrive from imported
vaults. Raw HTML in a note is escaped rather than executed, and `javascript:` and
`data:` URLs are stripped. Wikilinks and Markdown links both resolve; a link to a
document that does not exist yet is shown dashed and offers to create it, because
OKF §6.1 makes a dangling link legal — it may be knowledge not yet written.

The window re-syncs whenever it regains focus, so edits made in another editor
show up without a restart. An unsaved draft always wins over what landed on disk.

## Plugins

A plugin is an external program that reads one JSON object from stdin and writes
one to stdout. That is the entire contract, so a plugin can be written in
whatever language is already on the machine, and no scripting runtime is
embedded in the application.

Drop a directory containing a `plugin.yaml` into the application's plugin
folder (the **プラグイン** menu shows you where):

```yaml
name: Tidy frontmatter
description: Puts frontmatter keys in a predictable order.
scope: document          # document = the open note, workspace = the whole base
command: pk-plugin-tidy
args: []
timeout_seconds: 30
```

`crates/pk-core/src/bin/pk-plugin-tidy.rs` is a working example, and is useful
in its own right on a vault written by several people and several agents.

Two properties keep this safe enough to ship:

- **A plugin never writes files.** It returns *data* describing what it wants
  written, and the application performs the write through the same path an
  agent's write takes — so the layer guard, path containment, and conformance
  reporting all apply. A plugin cannot reach `raw/`, `.rag/`, or anywhere
  outside the bundle, however it spells the path.
- **A plugin only runs when a person clicks.** There are no save hooks or
  startup hooks, the command is shown in the menu before it is run, and the MCP
  server does not expose plugins at all — an agent must not be able to talk the
  application into running a local program.

What this design does *not* give you is Obsidian-style plugins that add panes,
commands, and themes to the interface. That would need a JavaScript runtime
inside the web view and a relaxed content-security policy, which is a
substantially different security posture. Say so if that is what you want.

## Compared with Obsidian

Obsidian is the obvious point of reference, and it is a more capable product:
mobile apps, sync, canvas, a graph view, and a large plugin ecosystem. What
follows is the narrow comparison this project was actually built against.

### Install size

| | PierrotKnowledge | Obsidian 1.13.4 |
|---|---|---|
| Windows installer | **3.01 MB** | 311.7 MB |
| Linux, single architecture | — | 129.8 MB (AppImage) |
| The application's own code | 5.50 MB (`pk-app.exe`) | 8.4 MB (`obsidian.asar.gz`) |

Ours are measured builds; the installer also carries `pk-mcp.exe` and the
reference plugin. Obsidian's are the published sizes of its
[official release artefacts](https://github.com/obsidianmd/obsidian-releases/releases).
Its Windows and macOS installers are universal, which is why the
single-architecture AppImage is the fairer per-platform number.

The last row is the interesting one: **the two applications' own code is
comparable in size.** The difference is almost entirely the runtime. Obsidian is
Electron and ships its own copy of Chromium and Node; this is Tauri and renders
into the WebView2 that is already part of Windows.

### Memory

**Not measured.** Obsidian is not installed on the development machine, and
publishing numbers for software that was never run would be worthless. Our own
figures are in [docs/TARGETS.md](docs/TARGETS.md): 180.7 MB private bytes for
the whole process tree, of which **5.9 MB is the engine** and the rest is
WebView2.

A like-for-like comparison is also less obvious than it looks. Electron's
Chromium is private to the application, so all of it counts against it; WebView2
is a shared OS component whose working set is largely shared pages, and its
marginal cost is lower on a machine already running Edge. Either measurement
needs that caveat attached to be honest.

### What is different by design

| | PierrotKnowledge | Obsidian |
|---|---|---|
| Note format | Markdown + YAML frontmatter, conforming to a published spec (OKF v0.2) | Markdown + YAML frontmatter, conventions are the app's own |
| Conformance checking | built in, validated against the spec's own reference bundles | — |
| Japanese search | character bigrams; a two-character query works | depends on the platform's search |
| Agent access | an MCP server in the box, sharing one containment check with the UI | via community plugins |
| Plugins | external processes; cannot write outside the bundle | JavaScript in the app, with full app privileges |
| Runtime | the OS web view | bundled Chromium |
| Licence | MIT | proprietary, free for personal use |

The plugin row is the honest trade: Obsidian's model is far more powerful and is
why its ecosystem exists. This one is far more contained.

## Building

Requires Rust 1.93+ (tested on 1.97.1), Node 20+, and pnpm. The floor comes from
`libsqlite3-sys`, not from anything this project writes.

```sh
pnpm install:ui        # once
pnpm build             # frontend, then the Tauri bundle
pnpm test              # cargo test --workspace
```

The Tauri config declares no `beforeBuildCommand`; the frontend build is an
explicit preceding step. See [D5](docs/DECISIONS.md#d5).

## Status

| Phase | Scope | State |
|-------|-------|-------|
| 0 | Workspace, Tauri shell, size baseline | done |
| 1 | OKF v0.2 model, validator, index/log generation, workspace | done |
| 2 | SQLite FTS5 search, CJK-aware tokenisation, incremental sync | done |
| 3 | Service layer: query, read, layer-guarded write | done |
| 4 | Built-in MCP server ([docs](docs/MCP.md)) | done |
| 5 | Editor, preview, navigation, search | done |
| 6 | Measured against the target metrics ([results](docs/TARGETS.md)) | done |

**Measured:** 3.01 MB installer (target ≤ 15 MB ✅) · 180.7 MB resident
(target 50–150 MB ❌ — 5.9 MB of that is the engine, the rest is WebView2; see
[D10](docs/DECISIONS.md#d10)) · 164 tests · all four of Google's own OKF
reference bundles validate conformant with zero warnings.

## Getting an icon on the desktop

Double-click **`Desktopにアイコンを作る.cmd`** in this folder. It builds the
application if it has not been built yet, then puts a `PierrotKnowledge`
shortcut on your desktop. Nothing is installed and no elevation is asked for.

Run it with `-Install` instead to install the application properly. The NSIS
installer creates its own desktop shortcut, via
`crates/pk-app/installer-hooks.nsh` — Tauri's template makes Start Menu entries
only and exposes no option for a desktop one, so hooks are the supported way in.

The icon is a closed three-node graph — the shape of a note and the notes it
links to. `crates/pk-app/icon-preview.png` shows it from 16 px to 256 px, which
is the only test that matters for an icon; regenerate both with
`crates/pk-app/make-icon.ps1`.

## Trying it

```sh
cargo run -p pk-core --example scaffold -- ./demo-knowledge
pnpm install:ui && pnpm build
```

Then open `demo-knowledge` from the app, or point an agent at it:

```sh
claude mcp add knowledge -- pk-mcp ./demo-knowledge
```

## Licence

Apacheライセンス
