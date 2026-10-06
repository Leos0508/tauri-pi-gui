# 0002 walking skeleton

Status: draft
Slice: 001
Depends on: [0001-architecture.md](0001-architecture.md)
Branch: `slice/001-walking-skeleton`

## Goal

Prove the transport and the process lifecycle end to end, with no pi involved.

When this slice is done, a click in the window travels to Rust, out to a Node process, back
through Rust, and into the UI, and the whole thing dies cleanly when the window closes. Nothing
about the agent works yet.

The point is to have the plumbing settled and tested before 20k lines of forked renderer and a
143 MB dependency tree are sitting on top of it.

## Success criteria

Observable, in order of how much they matter:

1. `pnpm tauri dev` opens a window showing a status line that moves from `starting` to
   `ready`, with the Node version and the sidecar pid.
2. Clicking Ping shows the round-trip latency in milliseconds and increments a counter.
3. Closing the window leaves no `node` process behind. `pgrep -f pi-sidecar` returns nothing.
4. `kill -9` on the Tauri process also leaves no `node` process behind.
5. `cargo test` passes for framing, envelope parsing, the channel allowlist, payload limits,
   request correlation, timeouts, and sidecar exit.
6. A protocol test drives the sidecar over stdio with no GUI and no Rust, and passes.
7. `pnpm typecheck` and `node --test sidecar` pass.

## Non-goals

- No pi import, no session, no provider, no transcript, no model list.
- No `window.piApp` shim and no forked renderer. Those are slice 002.
- No packaging. No `.deb`, no AppImage, no resources wiring.
- No multi-window. Window identity goes in the envelope now but only one window exists.

## Prerequisites

Verified on 2026-10-06:

| Requirement | State |
| --- | --- |
| Rust | `cargo` 1.99.0, `rustc` 1.99.0 via rustup 1.29.1 |
| `cargo check` in `src-tauri` | passes, 62 s cold |
| Node | `/usr/bin/node` 22.23.1 from the Fedora `nodejs22` RPM, plus fnm's 24.18.0 |
| pnpm | 12.6.0 |
| Tauri CLI | 2.12.1, from `node_modules` |
| `WebKitWebDriver` | `/usr/bin/WebKitWebDriver` |
| Tauri Linux deps | webkit2gtk-4.1 2.54.0, gtk+-3.0 3.24.52, libsoup-3.0 3.6.6, librsvg-2.0 2.62.3, openssl 3.5.9 |

Notes:

- `cargo` lives at `~/.cargo/bin` and is added to `PATH` by `~/.cargo/env`, sourced from
  `.bashrc` and `.zshrc`. A non-interactive shell does not have it. Source `~/.cargo/env` or
  use a login shell when running `cargo` from a script.
- `ayatana-appindicator3-0.1` has no `pkg-config` entry and `libayatana-appindicator-gtk3-devel`
  is not in the Fedora 44 repositories. Fedora provides `appindicator3-0.1.pc` from
  `libappindicator-gtk3-devel` instead. `cargo check` passes anyway, because the tray feature is
  off. Do not enable `tauri`'s `tray-icon` feature without resolving this, and note that v1 has
  no tray.
- Node 22.23 runs TypeScript directly, verified with a `.ts` file and no flag. Use that for
  development and tests. Ship a bundle later, in slice 003, when packaging starts.
- The sidecar must not inherit an fnm multishell `PATH`. Resolution and persistence rules are in
  [0001 section 9.2](0001-architecture.md).

## Deliverables

```
shared/
  protocol.ts          envelope types and runtime guards
  channels.ts          channel constants, payload size limits, v1 allowlist
sidecar/
  package.json
  src/
    index.ts           startup, stdin loop, dispatch, shutdown
    transport.ts       JSONL read and write, request ids, error envelopes
    log.ts             stderr logging, the only module allowed to call console
    handlers/
      app.ts           app.ping, app.shutdown
  test/
    protocol.test.ts   envelope and dispatch unit tests
src-tauri/src/
  main.rs, lib.rs
  sidecar.rs           spawn, reader, correlation, timeout, restart, shutdown
  forward.rs           the pi_invoke command
  channels.rs          allowlist from shared/channels.ts, kept in sync by hand
  paths.rs             Node resolution, script path, log path
  error.rs             transport error codes
src/
  main.tsx             minimal status UI, replaced in slice 002
  ipc/transport.ts     invoke and listen, used directly for now
  styles/base.css
```

## Protocol

### Envelopes

```ts
// shared/protocol.ts

export const PROTOCOL_VERSION = 1;

export type RequestEnvelope = {
  id: string;
  windowId: string;
  channel: string;
  payload: unknown;
};

export type ResponseEnvelope =
  | { id: string; ok: true; payload: unknown }
  | { id: string; ok: false; error: TransportFailure };

export type EventEnvelope = {
  event: string;
  windowId: string | null;
  seq: number;
  payload: unknown;
};

export type TransportFailure = {
  code: TransportFailureCode;
  message: string;
  detail?: string;
};

export type TransportFailureCode =
  | "NOT_IMPLEMENTED"
  | "CHANNEL_NOT_ALLOWED"
  | "PAYLOAD_TOO_LARGE"
  | "INVALID_ENVELOPE"
  | "INTERNAL";
```

Rust adds `windowId` from the invoking window. The renderer never supplies it and the sidecar
never trusts one it receives on a request, because slice 004 will have more than one window.

`seq` is per event name, monotonic, assigned by the sidecar. The renderer drops an event whose
`seq` is not greater than the last it saw for that name.

### Framing

Newline-delimited JSON, UTF-8, one object per line, no embedded newlines, no framing header.
Both sides buffer until `\n`. Maximum line length is 8 MiB; exceeding it is
`INVALID_ENVELOPE` and drops the line.

### Channels in this slice

Only two.

| Channel | Payload | Returns |
| --- | --- | --- |
| `app.ping` | `{}` | `{ protocolVersion, nodeVersion, piVersion, pid, startedAt }` |

`piVersion` is `null` in this slice because pi is not loaded. It becomes a string in slice 003.

| Event | Payload |
| --- | --- |
| `sidecar.status` | `{ state: "starting" \| "ready" \| "restarting" \| "down", detail?: string }` |

The sidecar emits `ready` as soon as its stdin reader is installed. `app.ping` exists as an
explicit liveness check and to exchange versions, not as the handshake.

## Rust

### Sidecar state

```rust
enum Status { Starting, Ready, Restarting, Down }

struct PendingRequest {
    channel: String,
    deadline: Instant,
    reply: oneshot::Sender<Result<Value, TransportFailure>>,
}

struct SidecarInner {
    status: Status,
    child: Option<Child>,
    stdin: Option<ChildStdin>,
    pending: HashMap<String, PendingRequest>,
    next_id: u64,
    restarts: Vec<Instant>,
}
```

### Spawn

```rust
let mut cmd = Command::new(node_path);
cmd.arg(script_path)
    .stdin(Stdio::piped())
    .stdout(Stdio::piped())
    .stderr(Stdio::piped());
unsafe {
    cmd.pre_exec(|| {
        libc::setsid();
        libc::prctl(libc::PR_SET_PDEATHSIG, libc::SIGTERM);
        Ok(())
    });
}
```

`setsid()` makes the child a session leader, so its process group id equals its pid and
`killpg` reaches bash and git grandchildren later. `PR_SET_PDEATHSIG` covers the case where the
Tauri process dies without running any shutdown code.

### Environment scrubbing

The sidecar must not inherit pi's own session environment. If the user starts this app from a
terminal that pi spawned, the child would receive `PI_SESSION_ID`, `PI_SESSION_FILE`,
`PI_PROVIDER`, `PI_MODEL` and `PI_REASONING_LEVEL` and pi would adopt them.

Remove every variable whose name starts with `PI_` from the child environment, then set only
what this app intends:

```rust
cmd.env_remove("PI_SESSION_ID").env_remove("PI_SESSION_FILE");
// ... and the rest, plus a sweep for any remaining PI_* keys
cmd.env("PI_TAURI_GUI_SIDECAR", "1");
```

### Reader task

One task per sidecar life reads stdout line by line. For each line:

- Parse. On failure, log, increment a malformed counter, and continue. Ten malformed lines
  marks the sidecar unhealthy and triggers a restart.
- If the object has `id`, resolve the matching pending request. An unknown id is logged and
  dropped, because it is almost always a response to a request that already timed out.
- If the object has `event`, emit `pi:event` to the window named by `windowId`, or broadcast
  when it is `null`.

A second task pipes stderr into the log file and keeps the last 200 lines in memory so the UI
can show why a sidecar died.

### Timeouts

A sweep runs every 250 ms and fails any pending request past its deadline.

| Channel | Deadline |
| --- | --- |
| `app.ping` | 5 s |
| default | 30 s |

`app.shutdown` gets 2 s and its failure is not surfaced, because shutdown proceeds regardless.

### Restart policy

On unexpected exit: fail all pending requests with `SIDECAR_NOT_RUNNING`, emit
`sidecar.status: down`, then restart with backoff of 500 ms, 2 s, and 8 s, at most three
attempts within five minutes. Exceeding that sets status `down` for good and the UI offers a
manual Restart.

An in-flight prompt is never replayed. In this slice there are no prompts, but the rule is set
now so it is not invented later under pressure.

### Shutdown

In order, on window close and on app exit:

1. Send `app.shutdown`, wait up to 500 ms for the response.
2. Close stdin. Wait up to 2 s for the child to exit.
3. `killpg(pid, SIGTERM)`. Wait up to 2 s.
4. `killpg(pid, SIGKILL)`.

Use the pid as the process group id, which is valid because of `setsid()`.

## Sidecar

`src/index.ts` does five things:

1. Install the stdin line reader.
2. Emit `sidecar.status: ready`.
3. Dispatch each request to a handler, timing it and catching throws into `INTERNAL`.
4. Handle `app.shutdown` by flushing and exiting 0.
5. Exit on stdin EOF and on `SIGTERM`.

`src/transport.ts` owns serialization and the guarantee that exactly one line is written per
envelope. It queues writes so two concurrent handlers cannot interleave a partial line.

`src/log.ts` is the only module that calls `console`, and it writes to stderr with levels. A
stray `console.log` on stdout corrupts the protocol, so add a test that imports every handler
module and asserts stdout was untouched.

`app.ping` returns:

```json
{
  "protocolVersion": 1,
  "nodeVersion": "v22.23.1",
  "piVersion": null,
  "pid": 41337,
  "startedAt": "2026-10-06T02:51:51.000Z"
}
```

## Renderer

Deliberately minimal and thrown away in slice 002.

- A status line reading `starting`, then `ready (node v22.23.1, pid 41337)`, or `down`.
- A Ping button that calls `app.ping` and shows `N ms`.
- A counter of pings sent.
- The last 200 lines of sidecar stderr behind a disclosure, so a failure is diagnosable
  without a terminal.

`src/ipc/transport.ts` exposes `invoke(channel, payload)` and `on(event, handler)` and nothing
else. Slice 002 wraps it in the `piApp` shim.

## Failure cases implemented now

These are cheap here and expensive later.

| Case | Behaviour |
| --- | --- |
| Node not found or too old | Dialog naming the requirement and the paths checked, then exit. No stack trace. |
| Channel not in the allowlist | `CHANNEL_NOT_ALLOWED`, logged, never forwarded. |
| Payload over the limit | `PAYLOAD_TOO_LARGE`, never forwarded. |
| Handler throws | `INTERNAL` with the message; detail carries the stack in dev only. |
| Malformed line from the sidecar | Logged, counted, restart after ten. |
| Sidecar exits mid-request | Pending requests fail with `SIDECAR_NOT_RUNNING`. |
| Request past deadline | `SIDECAR_TIMEOUT`. |
| App killed with `SIGKILL` | Sidecar dies from `PR_SET_PDEATHSIG`. |
| Window closed | Sidecar exits and its process group is gone. |

## Tests

**Rust unit, `cargo test`**

- Envelope parsing accepts valid requests, responses and events.
- Envelope parsing rejects malformed JSON, a missing `id`, and a wrong field type.
- The allowlist accepts `app.ping` and `app.shutdown` and rejects anything else.
- A payload over the per-channel limit is rejected before the sidecar sees it.
- A response resolves the pending request with the matching id and no other.
- An unknown response id is dropped without panicking.
- A pending request expires at its deadline.
- Sidecar exit fails every pending request.

**Rust integration**

- Spawn the real sidecar, ping it, assert the version fields.
- `shutdown()` returns only after the child is reaped and `kill(pid, 0)` fails.
- Restart: kill the sidecar out of band, assert it comes back and status returns to `ready`.

**Sidecar, `node --test sidecar/test`**

- Transport writes exactly one line per envelope, with no embedded newline, for a payload
  containing a newline in a string.
- `app.ping` returns the expected shape.
- An unknown channel returns `NOT_IMPLEMENTED`.
- A malformed input line produces an error response and does not crash the process.
- A handler that throws produces `INTERNAL` with a message.
- Importing every handler leaves stdout empty.

**Protocol, no GUI**

Spawn `node sidecar/src/index.ts` and drive it over stdio: read the `ready` event, send a ping,
assert the response, send an unknown channel, assert `NOT_IMPLEMENTED`, close stdin, assert the
process exits within 2 s. This runs in CI without a display and is the fastest way to catch a
protocol regression.

**Not automated**

The `SIGKILL` orphan case and the "no `node` process after closing the window" case are checked
by hand with `pgrep -f pi-sidecar`. Automating them is more trouble than it is worth at this
size. Record both in the pull request body.

## Steps

Each step ends with a check. Do not start the next one until it passes.

| # | Step | Check |
| --- | --- | --- |
| 1 | Remove the scaffold demo: `greet`, `App.tsx`, `App.css`, `src/assets/`, `public/vite.svg`, `public/tauri.svg` | `pnpm build` succeeds |
| 2 | Rust toolchain | **Done.** `cargo check` passes in 62 s |
| 3 | Write `shared/protocol.ts` and `shared/channels.ts` | `pnpm typecheck` passes |
| 4 | Write `sidecar/src/{log,transport,index}.ts` and `handlers/app.ts` | `node sidecar/src/index.ts`, type a ping line, get a response |
| 5 | Write `sidecar/test/protocol.test.ts` | `node --test sidecar/test` passes |
| 6 | Write `src-tauri/src/{channels,error}.rs` | `cargo test` passes for the allowlist |
| 7 | Write `src-tauri/src/sidecar.rs`: spawn, reader, correlation, timeout | Rust integration test pings a real sidecar |
| 8 | Write `src-tauri/src/{forward,paths}.rs` | `cargo test` passes |
| 9 | Wire `lib.rs`: spawn on setup, shutdown on exit, register `pi_invoke` | `pnpm tauri dev` shows status `ready` |
| 10 | Write the minimal renderer and `src/ipc/transport.ts` | Ping works and shows a latency |
| 11 | Write the protocol test and run the manual orphan checks | `pgrep -f pi-sidecar` is empty after close |

Step 2 is done. Steps 1 and 3 are independent of each other and of the toolchain, so they can
happen in either order.

## Verification

```bash
pnpm typecheck
node --test sidecar/test
cargo test --manifest-path src-tauri/Cargo.toml
node sidecar/test/protocol.e2e.ts
pnpm tauri dev                     # Ping, then close; then: pgrep -f pi-sidecar
kill -9 <tauri-pid>                # then: pgrep -f pi-sidecar
```

The slice is done when section Success criteria all hold, not when the tests pass.

## What slice 002 needs from this

- `NOT_IMPLEMENTED` working, so the forked renderer fails visibly on unbuilt channels.
- `windowId` present in every envelope, filled in by Rust.
- `src/ipc/transport.ts` stable, because `pi-app.ts` wraps it.
- The status and stderr surfaces, because the forked renderer will need somewhere to report a
  dead sidecar.

## Open questions

- Whether the malformed-line threshold should be a count or a rate. A count is simpler and
  good enough at this size.
- Whether `app.shutdown` earns its place given stdin EOF and `PR_SET_PDEATHSIG`. Keep it until
  measured otherwise.
- Whether the stderr ring buffer belongs in Rust or in a log file the UI reads. In memory is
  simpler and the sidecar restarts rarely.
