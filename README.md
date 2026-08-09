# PierrotKnowledge

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
crates/pk-core/       the engine; nothing here knows about a UI or a transport
  src/okf/            the Open Knowledge Format model, parser and validator
  src/index/          SQLite FTS5: tokenisation, sync, search, the link graph
  src/service/        the one API both frontends drive
    mod.rs            Knowledge, and the path containment check everything passes
    tree.rs           the navigation tree
    graph.rs          the link graph
  src/render.rs       Markdown to sanitised events, HTML, plus the outline
  src/plugin.rs       running an external program as a plugin
  src/bin/            pk-plugin-tidy, the reference plugin
crates/pk-gui/        the desktop shell (egui) — widgets only, no logic
crates/pk-mcp/        MCP server for external agents
docs/DECISIONS.md     why things are the way they are
docs/MCP.md           driving the knowledge base from an agent
docs/TARGETS.md       measured results against the target metrics
```

## The application

Three panes: a navigation tree and search on the left, the note in the middle
(rendered, or a CodeMirror editor over the raw Markdown including frontmatter),
and on the right the OKF v0.2 signals that matter — `type`, `status`, trust tier,
staleness, tags — plus the note's outline, outgoing links, backlinks, and
conformance findings.

| | |
|---|---|
| `Ctrl+O` | Quick switcher. Subsequence matching, so `fts` finds `full-text-search.md`. |
| `Ctrl+E` | Toggle reading and editing |
| `Ctrl+S` | Save |
| `Alt+←` / `Alt+→` | Back and forward through the notes you have visited |
| `Esc` | Close whatever is open |

Clicking an outline entry jumps to that heading; clicking a tag searches for it.

**Graph view.** Notes as circles sized by how many others they connect to, links
as lines. It opens on the neighbourhood of the note you are reading rather than
the whole vault — the whole-vault picture is the screenshot people expect and the
view they rarely use, while the local one answers "what is this connected to".
`index.md` and `log.md` are excluded: they link to everything beside them, and
including them turns the picture into noise. The layout is a small force
simulation written out longhand rather than pulled from a library. A third
scope, 分析, lists the maintenance numbers instead of a picture: the hubs
everything points at and the isolated notes nothing reaches.

A note can also be saved as a standalone HTML file (ツール → HTML で保存):
the same sanitised render, with app-internal links flattened to text and
external links kept.

**Pinned notes** sit above the tree. Pins are a preference about how you work,
not knowledge, so they live in the application's settings keyed by workspace —
nothing is written into the vault, and `.rag/` can still be deleted freely.

Notes are rendered in Rust from a sanitised event stream, never as markup: the
parser in `pk-core` flattens raw HTML to text and strips `javascript:` and
`data:` URLs before any widget is drawn, because notes are written by agents
and arrive from imported vaults. The desktop shell and the old web view share
that one sanitiser (`pk_core::render::rewrite_events`) — there is no second,
drift-prone implementation in the GUI. Wikilinks and Markdown links both
resolve; a link to a document that does not exist yet is shown in red and
offers to create it, because OKF §6.1 makes a dangling link legal — it may be
knowledge not yet written.

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
| Windows installer | **3.70 MB** | 311.7 MB |
| Linux, single architecture | — | 129.8 MB (AppImage) |
| The application's own code | 7.84 MB (`pk-app.exe`) | 8.4 MB (`obsidian.asar.gz`) |

Ours are measured builds; the installer also carries `pk-mcp.exe` and the
reference plugin. Obsidian's are the published sizes of its
[official release artefacts](https://github.com/obsidianmd/obsidian-releases/releases).
Its Windows and macOS installers are universal, which is why the
single-architecture AppImage is the fairer per-platform number.

The last row is the interesting one: **the two applications' own code is
comparable in size.** The difference is the runtime. Obsidian is Electron and
ships its own copy of Chromium and Node; this renders native widgets (egui,
OpenGL) and ships no web runtime at all — the earlier Tauri build rendered into
the OS's WebView2, which saved install size but cost 175 MB of RAM. See
[D16](docs/DECISIONS.md#d16).

### Memory

**Ours, measured** (release build, demo vault, process tree — trivially, since
there is only one process): **132.4 MB private bytes**, 130.7 MB working set,
of which the knowledge engine is ~6 MB and the rest is the GUI toolkit, the
GPU driver, and the system CJK fonts loaded at runtime. The 50–150 MB target
is met. Obsidian is still not installed on the development machine, so its
number stays unpublished rather than invented.

### What is different by design

| | PierrotKnowledge | Obsidian |
|---|---|---|
| Note format | Markdown + YAML frontmatter, conforming to a published spec (OKF v0.2) | Markdown + YAML frontmatter, conventions are the app's own |
| Conformance checking | built in, validated against the spec's own reference bundles | — |
| Japanese search | character bigrams; a two-character query works | depends on the platform's search |
| Agent access | an MCP server in the box, sharing one containment check with the UI | via community plugins |
| Plugins | external processes; cannot write outside the bundle | JavaScript in the app, with full app privileges |
| Runtime | native widgets (egui + OpenGL) | bundled Chromium |
| Licence | MIT | proprietary, free for personal use |

The plugin row is the honest trade: Obsidian's model is far more powerful and is
why its ecosystem exists. This one is far more contained.

## Building

Requires Rust 1.95+ (tested on 1.97.1). The floor comes from `libsqlite3-sys`,
whose build script uses `cfg_select!`, not from anything this project writes.
Node 20+ and pnpm are used for the convenience scripts (the installer wrapper
and the target sweeper), not for the application itself.

```sh
pnpm build             # the three binaries, then the NSIS installer
pnpm test              # cargo test --workspace
pnpm sweep             # drop build artefacts nothing can reach any more
```

The installer script needs `makensis`: it looks for `NSIS_DIR`, then the copy a
previous Tauri build cached under `%LOCALAPPDATA%\tauri\NSIS`, then the PATH.
See [D16](docs/DECISIONS.md#d16) for why there is no `tauri build` any more.

A working tree with every configuration built is about 3.6 GB — and that
number is build cache, not the application (the installer is 3.7 MB). Two
scripts keep it in check: `pnpm sweep` deletes only the artefacts nothing can
reach any more, which costs no rebuild (see [D14](docs/DECISIONS.md#d14)); and
`pnpm clean:dev` deletes all of `target/debug` — the tests-and-development
cache, 2.6 GB when fully built — which cargo rebuilds the next time you run
the tests. If you only *use* the app, `clean:dev` is the one you want.

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

**Measured:** 3.70 MB installer (target ≤ 15 MB ✅) · 132.4 MB resident
(target 50–150 MB ✅ — the engine is ~6 MB of it; see
[D16](docs/DECISIONS.md#d16)) · 209 tests · all four of Google's own OKF
reference bundles validate conformant with zero warnings.

## Getting an icon on the desktop

Double-click **`Desktopにアイコンを作る.cmd`** in this folder. It builds the
application if it has not been built yet, then puts a `PierrotKnowledge`
shortcut on your desktop. Nothing is installed and no elevation is asked for.

Run it with `-Install` instead to install the application properly. The NSIS
installer creates its own desktop shortcut; see `scripts/installer.nsh`.

The icon is a closed three-node graph — the shape of a note and the notes it
links to. `crates/pk-gui/icon-preview.png` shows it from 16 px to 256 px, which
is the only test that matters for an icon; regenerate both with
`crates/pk-gui/make-icon.ps1`.

## Trying it

```sh
cargo run -p pk-core --example scaffold -- ./demo-knowledge
cargo run -p pk-gui
```

Then open `demo-knowledge` from the app, or point an agent at it (ツール →
エージェント接続 does this from inside the app for the six supported tools):

```sh
claude mcp add knowledge -- pk-mcp ./demo-knowledge
```

## Licence

MIT.
