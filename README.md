<div align="center">

<img src="resources/icons/icon_256x256.png" alt="last-tftp" width="112" />

# last-tftp

**A cross-platform, modern TFTP tool — CLI client, embedded server and desktop GUI in one binary.**

[![CI](https://github.com/lph12168x/last-tftp/actions/workflows/ci.yml/badge.svg)](https://github.com/lph12168x/last-tftp/actions/workflows/ci.yml)
![Rust](https://img.shields.io/badge/rust-1.75%2B-orange)
![License](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue)

[English](./README.md) · [简体中文](./README.zh-CN.md)

</div>

---

Implements RFC 1350 plus the option extensions (RFC 2347 / 2348 / 2349) and RFC 7440
sliding-window transfers, over both IPv4 and IPv6. Protocol details live in
[docs/protocol-notes.md](./docs/protocol-notes.md), architecture in
[DESIGN.md](./DESIGN.md).

## Highlights

- **One binary, four modes** — `get`, `put`, `server`, `gui` (plus a `probe` stub).
- **RFC 7440 sliding window** — `--window` batches acknowledgements; roughly 2× faster on the
  same link (see [Measurements](#measurements)).
- **Real option negotiation** — `blksize`, `timeout`, `tsize`, `windowsize` via OACK.
- **Embedded server** — multi-socket model, serves any directory, optional write support,
  IPv4/IPv6 dual stack.
- **Progress you can trust** — live percentage bar and throughput stats in both CLI and GUI,
  with `--json` for scripting.
- **egui desktop GUI** — Server / Client / Transfers / Settings panels with native file dialogs.
- **Zero-argument start** — running the binary with no subcommand (or double-clicking it)
  opens the GUI.

## Install

Prebuilt binaries for Linux, Windows and macOS are attached to each
[GitHub release](https://github.com/lph12168x/last-tftp/releases) by CI.

Or build from source:

```bash
cargo build --release
./target/release/last-tftp --help
```

## Usage

A transfer target is always `HOST:FILE`, where `HOST` may be a hostname, an IPv4 address or
an IPv6 literal.

### Download

```bash
last-tftp get 192.168.1.1:firmware.bin -o ./fw.bin \
    --blksize 1468 --window 8
```

### Upload

```bash
last-tftp put -l ./local.bin 192.168.1.1:remote.bin \
    --blksize 1468 --window 8
```

### Built-in server

```bash
# Serve ./share on 6969, accept writes
last-tftp server --root ./share --port 6969 --allow-write

# Dual-stack bind on the standard port (needs root / admin)
sudo last-tftp server --root ./share --port 69 --ipv6
```

### GUI

```bash
last-tftp gui
```

### Options

| Flag | Default | Commands | Description |
|------|---------|----------|-------------|
| `--port` | `69` | `get` `put` `probe` | Peer UDP port |
| `--port` | `6969` | `server` | Bind port |
| `--root` | `.` | `server` | Directory to serve |
| `--allow-write` | off | `server` | Accept WRQ |
| `--blksize` | `1468` | `get` `put` | Block size, 18…65464 |
| `--window` | `1` | `get` `put` | RFC 7440 windowsize |
| `--timeout` | `5` | `get` `put` | Per-block timeout, seconds |
| `--retries` | `6` | `get` `put` | Retry budget |
| `--resume` | off | `get` | Stub — restarts from block 1, see [Known limitations](#known-limitations) |
| `--no-progress` | off | `get` `put` | Disable the progress bar |
| `--json` | off | `get` `put` | Print stats as JSON to stdout |
| `--ipv6` | off | all (global) | Prefer IPv6 |

## GUI

Four panels: **Server** (root, port, write toggle), **Client** (host / port / remote path /
local path, download and upload), **Transfers** (live progress per transfer) and **Settings**.

Platform behaviour worth knowing:

- On Windows the binary is built with `windows_subsystem`, so launching the GUI does not flash
  a console window. CLI subcommands re-attach to the parent console, so `stdout` still works
  from `cmd` and PowerShell.
- **Browse** uses the native file dialog. For a download select a *directory* (the remote file
  name is appended); for an upload select a *file*.
- The GUI needs a display server; it cannot start in a headless environment.

## Protocol support

| RFC | Feature | Status |
|-----|---------|--------|
| RFC 1350 | TFTP, revision 2 | ✅ Complete |
| RFC 2347 | Option extension | ✅ Complete |
| RFC 2348 | `blksize` | ✅ Complete |
| RFC 2349 | `timeout` / `tsize` | ✅ Complete |
| RFC 7440 | `windowsize` | ✅ Client + server |
| RFC 2090 | Multicast | ❌ Not implemented |

## Platforms

| Platform | Status | Notes |
|----------|--------|-------|
| **Linux x86_64** | ✅ Verified | Wayland + X11; built in CI |
| **Windows x86_64** | ✅ CI build | winit win32 backend, native file dialog, icon injected by `rcedit` |
| **macOS x86_64 / aarch64** | ✅ CI build | winit cocoa backend, native file dialog |

Cross-compiling locally:

```bash
cargo build --release                                  # native
rustup target add x86_64-pc-windows-gnu
sudo apt install gcc-mingw-w64-x86-64                  # mingw toolchain
./scripts/build.sh all                                 # tries every platform
```

CI builds all three platforms and uploads each binary as an artifact; tag pushes (`v*`) attach
them to the release.

## Tests

```bash
cargo test --release                    # whole workspace
cargo test --release -p last-tftp-core  # protocol engine only
```

Current state of the tree: **35 passing, 1 ignored, 0 failing**.

| Suite | Count | Covers |
|-------|-------|--------|
| `last-tftp-core` unit | 25 | Packet encode/decode, option negotiation, window math, target parsing |
| `last-tftp-core` integration | 3 | End-to-end GET/PUT with OACK, remote error propagation |
| `last-tftp` GUI | 7 | Headless egui render + in-process mock server |

The ignored test is a long-running GET under packet loss:
`cargo test --release -- --ignored`.

## Measurements

Early local-loopback numbers for a 512 KiB download — historical, not re-run on every commit:

| Scenario | Result |
|----------|--------|
| 524288 B GET, `--window 1` | ~199 ms |
| 524288 B GET, `--window 8` | ~92 ms (≈2.2×) |

## Roadmap status

| Phase | Scope | Status |
|-------|-------|--------|
| P0 | Workspace + two crates | ✅ |
| P1 | Packet codec | ✅ |
| P2 | Option negotiation | ✅ |
| P3 | Client GET (follows the server ephemeral port) | ✅ |
| P4 | Client PUT (RFC 7440 sliding window) | ✅ |
| P5 | Embedded server (multi-socket model) | ✅ |
| P6 | `windowsize` on both GET and PUT | ✅ |
| P7 | IPv6 dual stack | ✅ |
| P8 | Progress bar + throughput stats (CLI/GUI) | ✅ |
| P9 | GUI (eframe 0.32) | ✅ |
| P10 | Resume (`--resume`) | ⚠️ Stub |
| P11 | Protocol documentation | ✅ |

## Known limitations

- **`--resume` is a stub.** It restarts from block 1 and overwrites the existing file. A real
  implementation needs server-side `tsize` negotiation plus a client-side offset.
- **Headless GUI.** No display server, no window.
- **Large files.** The server reads the whole file into memory; beyond that, streaming I/O is
  required.
- **Trailing-ACK noise.** After the final ACK the client may report a timeout even though the
  file was written correctly. `cmp` the file to confirm.
- **`probe` is not implemented** and exits with an error; use `get` to check peer capabilities.

## License

MIT or Apache-2.0, at your option.
