# Gap note: host-effect floor (net / process / env-mutation / real-OS I/O)

**Context:** port-readiness review 2026-07-22 (`claude/mycelium-readiness-gaps`),
driven by the `gha-runner-ctl` and `tg-agent-relay` port targets. Full plan &
sequencing: `mycelium-lang` `docs/planning/PORT-READINESS-2026-07-22.md`. This note
records the gaps that this repo (`mycelium-std-sys` / `-sys-host`) owns.

`mycelium-std-sys` is the **audited real-OS floor** — the `@std-sys`/`wild` FFI seam's
intended home. Today it provides genuine but minimal host contact; the capabilities the
two network-service ports need are absent.

## Present today (real)

- `sys.rs`: `exit(i32)`, `get_env(&str) -> Option<String>` (read-only), `args()`.
- `io.rs`: real stdio — `read_to_end`, `read_line`, `write_out`, `write_err`, `flush_out`.
- `fs.rs`: thin `std::fs` — `read`, `write`, `exists`, `create_dir_all`, `remove_file`.
- `time.rs`: `wall_nanos`, `mono_nanos`, `sleep_nanos` (+ `OsClock` in `-sys-host`).

## Missing (required by both ports)

| Gap | Needed by | Note |
|---|---|---|
| **Networking** — TCP client, TLS, HTTP/1.1 client, DNS | runner (GitHub API), relay (Telegram) | No `net` module anywhere. Proposed as a new `std-net` phylum riding this floor. |
| **Process / subprocess** — spawn/exec, args, `wait`, exit-status, signals, `kill` | runner (`podman`/`git`/`gh`), relay (adapters/`piper`/`ffmpeg`) | Only `exit` exists. Proposed `std-process`. |
| **Env mutation / cwd** — `set_var`, `current_dir` | both (minor) | env is read-only today. |
| **Real-OS fs floor (M-541)** — wire `RealFs`; metadata, dir listing, rename/copy, permissions (`0o600`) | both | The richer `mycelium-std-fs` API runs on `InMemoryFs` only until `RealFs` lands here. |
| **FIFO / named-pipe** — `mkfifo`/open | relay inbound transport | Unix-specific. |
| **Host-call registry population** — the `wild:` ops the seam dispatches to | everything above | The FFI seam (`mycelium-l1`/`interp`) elaborates `wild { name(..) }` to a `wild:name` op, but the registry is empty by design (RFC-0028 §4.3). The concrete host functions authored here are what make the seam execute. |

## Why now / dependency

These are the **Tier-0/Tier-1** capabilities in the plan. The real-OS floor here is the
first customer of the FFI host-execution seam tracked in `mycelium-l1`
(`docs/GAP-ffi-host-and-surface.md`). Neither port target can perform its core I/O
until this floor exists; both are otherwise portable (they are synchronous, and their
pure logic already type-checks/run — see `gha-runner-ctl/mycelium-port/`).
