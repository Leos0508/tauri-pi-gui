# 0001 architecture

Status: accepted
Date: 2026-10-06
Target platform: Linux x64 only

This document describes the target state of `tauri-pi-gui`, a Linux desktop shell for the
`pi` coding agent built on Tauri 2. It replaces the Electron-based `pi-gui` for this
developer's own use. Implementation work is tracked as slices; section 15 defines the first
one, and later slices get their own specs.

## 1. Problem

`pi-gui` is an Electron app. It works, and on this machine it installs to 478 MB and
uses about 628 MB of resident memory while running, of which roughly 270 MB is Chromium:
a 192 MB Electron binary, 42 MB of Chromium locales, and a 15 MB Chromium license file.

`pi` itself is a Node library. `@earendil-works/pi-coding-agent` is imported in-process
by `pi-gui`'s main process and drives sessions, providers, tools, and transcript files.
There is no Rust implementation of pi, and writing one would mean reimplementing
providers, sessions, compaction, and the tool loop.

So the goal is to remove Chromium, not to remove Node. A JavaScript runtime ships in the
finished app no matter which desktop framework is used.

## 2. Goals

- Run one agent thread to completion: prompt in, streamed response out.
- Reuse pi's runtime rather than reimplement any part of it.
- Keep the Rust layer thin enough for one developer to read and maintain.
- Reach a usable core loop before building any secondary feature.
- Stay compatible with the `pi` CLI on the same machine, sharing `~/.pi`.

## 3. Non-goals

- Windows and macOS. Builds and code paths assume Linux.
- Feature parity with `pi-gui`.
- Removing Node from the shipped product. It cannot be removed without reimplementing pi.
- Extension views, terminal, file explorer, review and diff, worktrees, turn checkpoints,
  scheduled tasks, notifications, and MCP settings. All deferred.
- Reading or writing `pi-gui`'s own data files.

## 4. Constraints

| Constraint | Value | Consequence |
| --- | --- | --- |
| Node | System Node, 22.19.0 or newer | This machine has `/usr/bin/node` from the Fedora `nodejs22` RPM at 22.23.1, which satisfies pi. |
| Rust toolchain | Not installed yet | `rustup` and `cargo` are prerequisites before any Rust builds. |
| Webview | WebKitGTK 4.1, system provided | Same engine as `pi-gui` on Linux. Not Chromium. |
| Build output | `.deb` and AppImage | AppImage cannot declare dependencies, so the app must check Node at startup. |
| Developers | One, with AI assistance | The architecture optimizes for a small number of readable files over layering. |

## 5. Decisions

These were settled during planning. Each has a consequence worth remembering.

| Decision | Choice | Reason |
| --- | --- | --- |
| Shell | Tauri 2 | Removes Chromium. Stated motivation is bundle size, memory, and learning Tauri. |
| Rust layer | Thin. Pass-through transport. | Rust holds no domain types, so one forwarder command covers every channel instead of 132 Tauri commands. |
| pi runtime | Node sidecar process | pi runs unmodified on the Node version its maintainers support. No compatibility layer to debug. |
| pi integration | Vendored `pi-sdk-driver` | `pi-gui` already solved session supervision, leases, and runtime loading. Reuse it. |
| Node delivery | System Node | No 100 MB runtime in the bundle. Acceptable because this is a personal tool on Fedora. |
| Renderer | Written fresh | Only a core subset is needed. `pi-gui`'s 28k-line renderer carries features that are deferred. |
| Layout | Flat | `src/`, `shared/`, `sidecar/`, `src-tauri/`. No pnpm workspace, no `packages/`. |
| Data | Separate from `pi-gui` | The app owns its UI state. pi's sessions, auth, and settings stay in `~/.pi` and are shared with the CLI. |
| Windows | Multiple, designed for. One in v1. | Window identity is part of the protocol from the start, but the first slice ships single-window. |
| E2E | `tauri-driver` | WebKitWebDriver is present at `/usr/bin/WebKitWebDriver`. |

## 6. System overview

Three processes.

```
+---------------------------------------------------------------+
| Tauri process (Rust binary)                                   |
|                                                               |
|   +---------------------------+                               |
|   | Webview: WebKitGTK 4.1    |                               |
|   | React 19 renderer         |                               |
|   |   window.pi.transport     |                               |
|   +-------------+-------------+                               |
|                 | invoke("pi_invoke", { channel, payload })    |
|                 | listen("pi:event", cb)                      |
|   +-------------v-------------+                               |
|   | rust: forwarder           |  channel allowlist            |
|   |   sidecar.rs / forward.rs |  request correlation          |
|   +-------------+-------------+                               |
+-----------------|---------------------------------------------+
                  | JSONL on stdin / stdout
                  |
+-----------------v---------------------------------------------+
| Node sidecar process                                          |
|   node <resources>/sidecar/index.mjs                          |
|   pi runtime, sessions, providers, models, tools              |
|   shares ~/.pi with the pi CLI                                |
|   spawns its own children: bash, git, extension processes     |
+---------------------------------------------------------------+
```

A request travels renderer to Rust to sidecar and back. A streaming event travels sidecar to
Rust to renderer with no renderer request involved.

The sidecar is a normal Node process with full user privileges, the same trust level as the
`pi` CLI. It runs agent-generated bash. Nothing in this design tries to sandbox it.

## 7. Renderer

Fresh React 19 app under `src/`, built by Vite.

Structure:

```
src/
  main.tsx
  app/            screen composition, routing between sessions
  features/
    threads/      session list, create, rename, delete
    conversation/ timeline, composer, streaming state
    runtime/      model and provider selection
  ipc/            transport client, window.pi shim, fake transport for tests
  ui/             shared primitives
  styles/         tokens and base styles
```

The renderer never imports Node or pi. It talks to the transport client only.

The v1 timeline renders every message without virtualization. `pi-gui` spent significant
effort on a virtualized, measured viewport with scroll anchoring, and none of that is reused
here. A long transcript will be slow. This is accepted for v1 and tracked in section 17.

## 8. Rust shell

Thin by design. Rust owns windows, the sidecar process, and the transport. It owns no agent
domain types.

```
src-tauri/src/
  main.rs        entry
  lib.rs         builder, plugins, command registration
  sidecar.rs     spawn, JSONL framing, request correlation, lifecycle, shutdown
  forward.rs     the pi_invoke command
  channels.rs    channel allowlist and per-channel payload size limits
  paths.rs       Node resolution, app data and resource directories
  error.rs       error codes shared with the renderer
```

Rust responsibilities:

- Resolve the Node executable and spawn the sidecar.
- Frame and parse newline-delimited JSON in both directions.
- Correlate requests and responses by id, with a timeout per request.
- Forward events to the correct window.
- Enforce a channel allowlist before forwarding.
- Supervise the sidecar: detect exit, fail pending requests, restart with backoff.
- Kill the whole sidecar process tree on shutdown.

Rust does not know what a session, a model, or a transcript is.

### 8.1 The single forwarder

Because Rust is a pipe, the renderer's entire API surface maps to one command:

```rust
#[tauri::command]
async fn pi_invoke(
    state: State<'_, SidecarHandle>,
    window: Window,
    channel: String,
    payload: serde_json::Value,
) -> Result<serde_json::Value, TransportError>;
```

and one event:

```
pi:event   { channel: String, payload: Value }
```

This is the reason the channel count does not multiply the Rust work. Adding a channel later
means adding a TypeScript handler in the sidecar and a channel constant in `shared/`, with no
Rust change.

The cost is that Tauri's capability system cannot scope permissions per channel, since there
is only one command. `channels.rs` therefore holds the allowlist in Rust, plus a maximum
payload size per channel. Any channel not in the list is rejected before the sidecar sees it.

## 9. Node sidecar

A Node process under `sidecar/`, started by Rust.

```
sidecar/
  package.json
  src/
    index.ts        startup, handshake, stdin loop, dispatch
    transport.ts    JSONL read and write, request ids, error envelopes
    log.ts          stderr logging with levels
    handlers/       one module per channel group
      app.ts
      workspace.ts
      session.ts
      conversation.ts
      runtime.ts
    pi/             vendored from pi-gui, trimmed
      driver.ts
      session-supervisor.ts
      runtime-supervisor.ts
      session-schema.ts
      session-lease.ts
      session-usage.ts
      debug/
      compat/       private upstream pi API seams
```

### 9.1 Vendored code

`sidecar/src/pi/` is vendored from `pi-gui`:

- `packages/pi-sdk-driver/src` (about 8.8k lines)
- `packages/session-driver/src` (about 1.1k lines)

Both are MIT licensed. Keep the license header and a `sidecar/src/pi/VENDOR.md` recording the
source commit and what was removed.

Trim to what v1 needs. Remove extension bridges, MCP config, scheduled tasks, turn capture,
checkpoint hooks, package manager fallback, and desktop extension code. Expect to keep roughly
a third of the driver. The `compat/` directory stays, because those seams are what break when
pi changes.

Track `@earendil-works/pi-coding-agent` versions deliberately. The compat seams were written
against pi 0.87.1 and the tree now has 1.0.0. Treat a pi upgrade as its own change with its own
verification run.

### 9.2 Node version handling

Resolve the executable in this order and stop at the first that reports 22.19.0 or newer:

1. An explicit path from the app's settings file.
2. `node` on `PATH`.
3. `/usr/bin/node`, `/usr/bin/node-22`, `/usr/local/bin/node`.
4. nvm, fnm, volta, and asdf install directories under the user's home.

If none qualifies, show a dialog naming the requirement and how to install it, then exit. Do not
fail with a stack trace.

### 9.3 When the sidecar writes

stdout carries the protocol and nothing else. All logging goes to stderr, which Rust captures
and writes to the app log directory. A stray `console.log` corrupts the stream, so `log.ts` is
the only module allowed to call `console`.

## 10. The boundary contract

### 10.1 Framing

Newline-delimited JSON in both directions. One object per line, no embedded newlines. This
mirrors pi's own `--mode rpc` protocol, which is a good precedent to follow.

Request, Rust to sidecar:

```json
{ "id": "1f2c", "channel": "session.list", "payload": { "workspaceId": "w1" } }
```

Success response:

```json
{ "id": "1f2c", "ok": true, "payload": { "sessions": [] } }
```

Failure response:

```json
{ "id": "1f2c", "ok": false, "error": { "code": "SESSION_NOT_FOUND", "message": "...", "detail": "..." } }
```

Event, sidecar to Rust, with no request:

```json
{ "event": "session.event", "payload": { "sessionId": "s1", "event": { "type": "..." } } }
```

`id` is opaque to the sidecar. `detail` is optional and carries a stack trace in development
only.

### 10.2 Error model

Codes are stable strings, defined once in `shared/protocol.ts` and mirrored in
`src-tauri/src/error.rs`. The renderer branches on the code, never on the message.

Transport-level failures originate in Rust and are not sidecar responses:

| Code | Meaning |
| --- | --- |
| `SIDECAR_NOT_RUNNING` | Not started, or exited. |
| `SIDECAR_TIMEOUT` | No response within the per-channel deadline. |
| `CHANNEL_NOT_ALLOWED` | Rejected by the Rust allowlist. |
| `PAYLOAD_TOO_LARGE` | Exceeded the per-channel byte limit. |
| `INVALID_ENVELOPE` | Malformed line from either side. |

### 10.3 Ordering and backpressure

Every event carries a monotonic `seq` per channel stream. The renderer drops an event whose
`seq` is not greater than the last one seen for that stream, which makes a restart or a
duplicate harmless.

Streaming token events are batched in the sidecar at 50 ms per session, borrowed from
`pi-gui`. Discrete events such as a turn start or a tool call flush immediately.

The sidecar counts unacknowledged events per window and stops reading stdin above a threshold
rather than growing memory without bound. The renderer acknowledges on a timer.

### 10.4 Binary payloads

v1 carries no binary data. Images for composer attachments arrive with a later slice.

When that slice lands, images go base64 inside the JSON envelope because they are small and
infrequent. Terminal output does not, because it is high volume and continuous. The terminal
slice will use a second path, most likely a Rust-owned PTY streaming bytes directly to the
webview, which is a deliberate exception to thin Rust. That decision is deferred and recorded
here so it is not a surprise later.

### 10.5 v1 channels

Seventeen channels cover the core loop.

| Channel | Payload | Returns |
| --- | --- | --- |
| `app.ping` | none | protocol version, pi version, sidecar pid |
| `app.shutdown` | none | acknowledgement, then exit |
| `workspace.list` | none | workspace list |
| `workspace.add` | path | workspace |
| `workspace.remove` | workspace id | none |
| `session.list` | workspace id | session summaries |
| `session.create` | workspace id, optional model | session summary |
| `session.read` | session id | transcript entries |
| `session.rename` | session id, title | none |
| `session.delete` | session id | none |
| `conversation.prompt` | session id, message | disposition |
| `conversation.steer` | session id, message | disposition |
| `conversation.abort` | session id | none |
| `runtime.models` | none | model records |
| `runtime.authStatus` | none | provider auth state |
| `runtime.setModel` | session id, provider, model id | none |
| `runtime.login` | provider | deferred, see section 16 |

Events:

| Event | Payload |
| --- | --- |
| `session.event` | session id, pi session event |
| `runtime.changed` | model registry or auth changed |
| `sidecar.status` | starting, ready, restarting, down |

## 11. Lifecycle

### 11.1 Startup

Rust starts the sidecar in parallel with window creation, so the window paints while Node boots.
pi's import graph is roughly 143 MB of JavaScript across AWS, OpenAI, Anthropic, Google,
Mistral, `esbuild`, `jiti`, and `shiki`, so expect one to three seconds of cold start.

The renderer shows a starting state until `sidecar.status: ready` arrives. The handshake carries
the protocol version, and a mismatch is a hard error rather than a silent failure.

### 11.2 Process group and orphan prevention

The sidecar spawns bash, git, and extension processes. Killing the sidecar alone leaves those
running. Two measures, both applied:

- Spawn the sidecar with `setsid()` through `pre_exec`, so it leads its own process group.
  Shutdown sends `SIGTERM` to the group, waits, then `SIGKILL` to the group.
- Set `PR_SET_PDEATHSIG` to `SIGTERM` in `pre_exec`, so the sidecar dies if the Tauri process
  dies for any reason, including `SIGKILL`.

The sidecar also exits when stdin reaches EOF, as a third guard.

### 11.3 Failure and restart

When the sidecar exits:

1. Fail every pending request with `SIDECAR_NOT_RUNNING`.
2. Emit `sidecar.status: down` so the renderer can show a banner.
3. Restart with exponential backoff, at most three attempts in five minutes.

An in-flight prompt is never replayed automatically. The session file is the source of truth,
so the renderer re-reads the transcript after a restart and shows whatever pi committed.

### 11.4 Shutdown

On window close or app exit: send `app.shutdown`, wait 2 seconds, `SIGTERM` the group, wait 2
seconds, `SIGKILL` the group. Never leave the sidecar running.

## 12. State and persistence

Two stores, deliberately separate.

**pi's data, in `~/.pi`.** Sessions as JSONL under pi's session directory, `auth.json`,
`models.json`, `settings.json`, skills, and extensions. This app reads and writes it through
the vendored driver, exactly as the `pi` CLI does. It is the source of truth for transcripts.

**This app's data, in the Tauri app data directory.** UI state only: known workspaces, last
open session, window geometry, and the resolved Node path. Tauri's identifier is
`com.phamhoangbao.tauri-pi-gui`, so this lands under `~/.local/share/`.

The app does not read or write `pi-gui`'s `ui-state.json`, `reviewed-files.json`, or
`turn-checkpoints/`. The two apps can be installed side by side without interfering, which the
developer will do during development.

### 12.1 Concurrent access

The `pi` CLI and this app both write session files. `pi-gui` solved this with a session lease,
which is why `session-lease.ts` is in the vendored set. Keep it. Without it, a prompt in the app
and a prompt in the CLI on the same session can interleave and corrupt a transcript.

### 12.2 Write durability

Adopt `pi-gui`'s approach: serialize writes per path, write to a temporary file, `fsync`, then
rename. Keep the previous valid file as a `.bak`. On a decode failure, recover from the backup
and never overwrite unreadable bytes with defaults.

App UI state is low stakes. Session files are not, but pi owns those, so this applies to the
app's own store and to anything the sidecar writes on the app's behalf.

## 13. Security

The renderer has no Node and no filesystem access. Its only route to privileged work is the
`pi_invoke` command, which the sidecar receives as JSON.

Layers:

- Tauri capabilities limit the webview to `core:default` plus the specific plugins in use.
- `channels.rs` allows only known channels and rejects oversized payloads.
- The sidecar validates every payload again before acting on it, because the renderer is the
  less trusted side. Port the shape of `pi-gui`'s `request-validation.ts`, not the file.
- File paths arriving from the renderer are resolved and checked against the known workspace
  roots before use. Never trust a path from the webview.

The sidecar runs agent bash with the user's privileges. That is inherent to running pi and is
the same as the CLI. Nothing here reduces it.

## 14. Packaging

Tauri bundles `.deb` and AppImage for Linux x64.

Bundled as Tauri resources:

- `sidecar/dist/` (our compiled sidecar code)
- `sidecar/node_modules/` (pi and its dependencies)

Two packaging constraints worth recording:

- **Hoisted node_modules.** pnpm's default isolated linker produces a symlink farm that does not
  survive Tauri's resource copying. Set `node-linker=hoisted` in `.npmrc` so `node_modules` is a
  real directory tree, or the packaged sidecar will fail to resolve imports.
- **Payload size.** pi plus provider SDKs is about 143 MB before any code is written: pi 37 MB,
  OpenAI 28 MB, Mistral 24 MB, Anthropic 14 MB, esbuild 12 MB, Google 11 MB, AWS 9.5 MB, plus
  the rest. This ships in every option, including `pi-gui` today. Expect a bundle around 180 to
  200 MB. The Chromium removal is the entire saving, so do not expect a small binary.

A later optimization is to drop unused provider SDKs. That saves roughly 70 MB but depends on
whether pi can load with a reduced provider set, so it is not part of v1.

Build steps:

```
rustup + cargo                  one-time prerequisite, not yet installed
pnpm install
esbuild sidecar/src -> sidecar/dist
pnpm tauri build --bundles deb,appimage
```

Node itself is not bundled. The app checks it at startup per section 9.2.

## 15. v1 scope: the core loop

One slice, end to end. Nothing else.

1. App launches, resolves Node, starts the sidecar, completes the handshake.
2. Add a workspace by picking a folder.
3. List sessions for that workspace from pi's session files.
4. Create a new session.
5. Send a prompt and stream the response into the timeline.
6. Steer and abort a running turn.
7. List models and provider auth state, and set the model for a session.
8. Reopen an existing session and read its transcript.
9. Rename and delete a session.

Single window. No terminal, workbench, review, extensions, scheduled tasks, notifications,
themes, or command palette.

First implementation steps, in order:

1. Remove the scaffold demo (`greet` command, `App.tsx`, default assets).
2. Install `rustup` and confirm `cargo tauri dev` opens a window.
3. Define `shared/protocol.ts` and `shared/channels.ts`.
4. Write `sidecar/src/transport.ts` and a `app.ping` handler.
5. Write `src-tauri/src/sidecar.rs` and `forward.rs`, and get `app.ping` round-tripping.
6. Only then start on pi integration.

Step 5 is the walking skeleton. Get it working before touching pi.

## 16. Deferred

Each of these gets its own spec when it starts.

| Feature | Note |
| --- | --- |
| Provider login (OAuth and API key) | `runtime.login` is stubbed. Users configure providers with the `pi` CLI in the meantime. |
| Terminal | Needs a PTY. Likely Rust-owned with `portable-pty`, which bends the thin-Rust rule on purpose. |
| File explorer and editor | |
| Review and diff | `pi-gui`'s `git-review.ts` and `checkpoint-store.ts` are the reference. |
| Git worktrees | |
| Turn checkpoints and rewind | |
| Extension views | Highest risk on WebKitGTK. `pi-gui` uses opaque-origin sandboxed iframes served from a custom scheme, and Tauri has a filed Linux bug loading custom-scheme HTML into an iframe. |
| Scheduled tasks | |
| Notifications | Tauri plugin. |
| MCP settings | |
| Themes and command palette | |
| Multi-window | Protocol carries window identity from v1. UI ships single-window. |
| Windows and macOS | |

## 17. Risks

**Scope drift on a two-week target.** The core loop is nine features, not three. If it slips,
cut items 6, 8, and 9 and keep prompt and stream working.

**Timeline performance.** No virtualization in v1. Long sessions will stutter. Accept it, and
do not copy `pi-gui`'s viewport code until the feature list is stable.

**pi compat seams.** The vendored driver touches private pi APIs in `compat/`. A pi upgrade can
break session-file rewriting and project settings. Pin the pi version, and treat upgrades as
their own task with a real-provider verification run.

**Node discovery.** Fedora 44 with `nodejs22` is fine. A user on another distro with an older
Node gets a clear error, not a working app. That is accepted, since this is a personal tool.

**Packaging node_modules.** pnpm's isolated linker breaks resource copying. Verify early, in the
first packaged build, not at the end.

**tauri-driver limits.** `tauri-driver` over WebKitWebDriver is less capable than Playwright
over Electron. Native dialogs, file pickers, and some input events are hard to drive. Plan for
the workspace-add step to need a test hook that bypasses the folder picker.

## 18. Open questions

- Does the sidecar need a supervisor inside Node, or is Rust supervision enough? Start with
  Rust only and add an in-process supervisor if crashes prove hard to attribute.
- Should the sidecar serve more than one window from one process, or one sidecar per window?
  Start with one process total, since sessions already serialize internally.
- Is `app.shutdown` needed at all, given stdin EOF and `PR_SET_PDEATHSIG`? Keep it for a clean
  exit path until measured otherwise.

## 19. Directory layout

```
.
  docs/specs/0001-architecture.md
  index.html
  package.json
  pnpm-lock.yaml
  vite.config.ts
  tsconfig.json
  .npmrc                        node-linker=hoisted
  src/                          React renderer
    main.tsx
    app/
    features/{threads,conversation,runtime}/
    ipc/
    ui/
    styles/
  shared/                       types used by renderer and sidecar
    protocol.ts
    channels.ts
    dto/
  sidecar/
    package.json
    src/
      index.ts
      transport.ts
      log.ts
      handlers/
      pi/                       vendored pi-sdk-driver, trimmed
  src-tauri/
    Cargo.toml
    tauri.conf.json
    capabilities/default.json
    src/{main,lib,sidecar,forward,channels,paths,error}.rs
```

`shared/` is imported by both sides through a `tsconfig` path alias and a Vite alias. It holds
types and channel constants only, never runtime logic that depends on Node or on the DOM.

## 20. Verification

Four levels, cheapest first.

**Rust unit tests.** `cargo test` covers JSONL framing, envelope parsing, the channel allowlist,
payload limits, and request correlation including timeout and sidecar exit.

**Sidecar handler tests.** `node --test` covers each handler against a fake pi, plus transport
envelopes and error codes. Node 22.23 runs TypeScript directly, so no build step is needed for
tests.

**Protocol tests.** Spawn the real sidecar as a child process and speak JSONL to it. Assert the
handshake, a full prompt and stream cycle against a stubbed provider, and clean shutdown. This
runs without a window and is the fastest way to catch integration regressions.

**End-to-end.** `tauri-driver` launches the real app through WebKitWebDriver and drives the core
loop: add workspace, create session, prompt, see a streamed response, reopen, rename, delete.
Run against a fixture workspace and a stubbed provider so it is deterministic and offline.

A change to the core loop is not done until the end-to-end path passes against the real app. A
passing handler test does not prove the window, the transport, and the sidecar agree.
