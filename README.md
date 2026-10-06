# tauri-pi-gui

A Linux desktop shell for the [pi](https://github.com/earendil-works/pi) coding agent,
built on [Tauri](https://tauri.app) instead of Electron.

This is a personal experiment, not a replacement for [pi-gui](https://github.com/minghinmatthewlam/pi-gui).
It exists to see what a pi desktop app costs when Chromium is removed and the agent runtime
stays on Node.

## What it is

Three parts:

- `src/` a React renderer, written fresh for a small core feature set
- `src-tauri/` a thin Rust shell that owns windows and the transport, and no agent logic
- `sidecar/` a Node process that runs pi, vendoring `pi-gui`'s driver

pi is a Node library. It runs unmodified in the sidecar on system Node, sharing `~/.pi` with
the `pi` CLI, so sessions, credentials, and skills carry over.

## Requirements

- Linux x64
- Node 22.19.0 or newer on `PATH` or at `/usr/bin/node`
- Fedora 44 with the `nodejs22` package is what this is developed against
- Rust is not installed on the development machine yet and is needed before any Rust build

## Status

Design only. See [docs/specs/0001-architecture.md](docs/specs/0001-architecture.md) for the
target state and the first slice. Nothing runs yet beyond the Tauri scaffold.

## Development

Not yet usable. The planned sequence lives in section 15 of the architecture spec.

## License

MIT. Vendored code under `sidecar/src/pi/` comes from pi-gui, also MIT.
