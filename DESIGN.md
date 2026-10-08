# penyero — design (draft)

Status: draft proposal. Nothing here is implemented yet. Items marked **(verify)** are assumptions
that must be checked on a real machine before they are relied upon.

This document describes **how** penyero is built to meet [SPEC.md](SPEC.md). The spec is the
contract; anything here can change as long as the spec still holds. Terms (AI workspace, guest,
profile, privileged side, broker, client) are as defined in [SPEC.md §3](SPEC.md#3-threat-model).

## 1. Processes

One program, `penyero`, with subcommands:

| Subcommand | Runs on | Role |
|---|---|---|
| `serve <profile> <name>` | privileged side | The broker for one guest: reads the configuration, resolves the profile, opens the channel into the guest and supervises the relay, runs resources, hosts the approval prompt on its terminal, writes the audit log |
| `relay` | guest | Started by the broker through `guest_exec`, never by hand. Listens on a Unix socket inside the guest and forwards between clients and the broker |
| `list`, `describe`, `request`, `wait`, `retrieve`, `pending`, `cancel` | guest | The client (SPEC.md §6). Each call connects to the relay's socket |

**One broker per guest.** Each broker has one channel, so every message it receives belongs to
that guest, and each guest gets its own approval pane. The guest is identified everywhere
(prompt, audit log) as `<profile>/<name>`.

**Request states.** Each request carries one explicit state, and transitions are the only
place it changes:

| From | Event | To |
|---|---|---|
| (arrival) | `confirm_request` | `pending-approval` |
| (arrival) | no `confirm_request` | `queued` |
| `pending-approval` | `y` or `Y` | `queued` |
| `pending-approval` | `n` | `denied` |
| `queued` | a slot of its resource is free | `executing` |
| `executing` | exits or is stopped (timeout, output cap), with output to review | `pending-release` |
| `executing` | exits or is stopped, nothing to review | `deliverable` (exit 0) or `failed` |
| `executing` | `k` (kill) | `denied` |
| `pending-release` | `y` | `deliverable` or `failed`, by exit status |
| `pending-release` | `n` | `release-denied` |
| `pending-approval`, `queued` | channel lost | `dropped` |
| `pending-approval`, `queued` | `cancel` | deleted |
| any terminal state | complete `retrieve` | deleted |

- The client gets its id on arrival. Checkers, later, run before `pending-approval`.
- An execution has output to review unless `confirm_output = false`, the human answered `Y`, or
  there is nothing to review (below).
- Failures are reviewed like results: stdout may hold partial data, such as rows written before a
  statement timeout. A failed execution skips review only when its stdout is empty. A request the
  broker stops at its `timeout` or the output cap is a failure whose stdout keeps what was written
  (§5).
- A stop (timeout, output cap) or a kill ends the resource's process group (it runs in its own
  session, below). A kill records the request as denied while running, which the agent is told,
  and releases nothing (SPEC.md §7).
- `denied`, `release-denied`, `deliverable`, `failed` and `dropped` are **terminal**: the request
  is settled (SPEC.md §6) and holds whatever `retrieve` will answer: the result, the resource's
  stdout on failure, or the human's message. `retrieve` is a pull, so no state is needed for a
  delivery that breaks off: the request stays terminal and the next `retrieve` starts over. A
  complete `retrieve` deletes the request.
- `cancel` deletes a request that has not started; afterwards its id is unknown.
- `s` (skip) leaves the state as is and moves the request to the back of its kind of prompt.

**One event loop, no threads.** The broker is a single `selectors` loop over the channel pipe, the
running resources' stdout and stderr, the terminal, and timers (heartbeat, resource timeouts).
Queues are views over state, not separate structures:

- **Slots.** A `queued` request starts when fewer than its resource's `concurrency` requests are
  `executing`, in the order requests became `queued`.
- **Prompts.** `pending-approval` requests in arrival order, then `pending-release` ones: first
  those that complete a blocked `wait`, then the rest in arrival order (SPEC.md §7). The broker
  knows each open `wait` and the ids it lists, since it answers them when they settle.
- **Outbox.** Requests in a terminal state.

Only the active prompt reads the terminal. Input that arrives before a prompt is drawn is
flushed (`termios.tcflush`); a prompt whose request is cancelled or dropped is withdrawn with an
entry, and a key in flight either lands on the withdrawn request, where it does nothing, or is
flushed; entries for other requests are printed between prompts, or held while the human types a
message (SPEC.md §7).

The review pager and editor take over the terminal, but the loop keeps running underneath them:
the tool runs as a child, the loop keeps answering heartbeats and clients, and pane output is
buffered until the tool exits. Otherwise a minute in the pager would let the relay's heartbeat
lapse.

Resources run as child processes of the broker, in their own session without a controlling
terminal, with stdin, stdout and stderr connected to the request, and with the fixed environment
of SPEC.md §8 (`PATH`, `HOME`, `USER`, `LANG`/`LC_*`, `PENYERO_P_*`) built from scratch rather
than filtered. They can't prompt on, or write to, the approval pane. The broker collects stdout as
the result and stderr for the pane and the log (SPEC.md §8, the resource contract), both spooled
(§5), so the running view (SPEC.md §7) can show a snapshot of either at any time.

## 2. Configuration resolution

At start, `serve <profile> <name>`:

1. Loads `config.toml` (SPEC.md §8) after checking ownership and permissions.
2. Substitutes `{profile}` and `{name}` in the profile's `guest_exec`. `name` must match
   `[A-Za-z0-9][A-Za-z0-9._-]*`, so it cannot alter the command's shape.
3. Builds the guest's resource table from the profile's `resources` table: each entry's resource
   definition plus that entry's `confirm_request`/`confirm_output` (default `true`). An entry for
   an undefined resource, or `confirm_request = false` on a resource with `write_credential`, is
   a start-up error.
4. Refuses to start if the profile has no resources: such a guest has nothing to request, so
   there is nothing to broker.

`list` and `describe` answer from this table only; a request for anything else is an invalid
request (exit 253).

## 3. Channel and relay

```
privileged side                               guest trusted/shop
┌──────────────┐  guest_exec                  ┌──────────────────────────────┐
│ penyero      │  (docker exec -i             │ penyero relay                │
│ serve        │   cayo-trusted-shop          │   listens on a Unix socket   │
│ trusted shop │ ────── penyero relay) ─────▶ │   in the guest               │
│              │ ◀───── stdin/stdout ───────▶ │                          ▲   │
└──────────────┘                              │ penyero request ... ─────┘   │
                                              └──────────────────────────────┘
```

**Why this shape.** In a normal client-server setup the host listens, the guest dials in, and the
host must then authenticate the caller: tokens, TLS, or kernel peer credentials. Peer credentials
only work when the guest is a separate OS user, and on macOS a host Unix socket can't even be
mounted into a container, because sockets don't cross the hypervisor. The relay reverses who
connects: the broker dials *into* the guest with a command only it controls, so it knows who is on
the other end because it chose the destination. This meets SPEC.md §3's identity requirement with
no secrets and no host listener. Replacing the relay binary gains a guest nothing: whatever speaks
on this guest's channel *is* this guest.

- On start, the broker runs the resolved `guest_exec` followed by `penyero relay`, and keeps
  that process's stdin and stdout as the channel. Nothing listens on the host.
- The channel process is started in a new session with no controlling terminal (`setsid`), and
  its stderr is captured rather than inherited. Otherwise, with a `guest_exec` like `sudo -u`, a
  guest process would share the approval pane's terminal and could inject input into it or write
  escape sequences to it. Captured stderr is shown in the pane escaped, like all guest text.
- The relay runs as whichever user `guest_exec` lands on, and clients must run as that same user
  to reach its socket. With `docker exec -i` that is the image's default user; a command that
  switches user (e.g. `-u 0`) would create a socket the guest's normal user cannot use.
- The relay listens on `<home>/.penyero/relay.sock` (directory mode 0700), where
  `<home>` is the running user's home from the passwd database, not from the environment. The
  relay and the clients start in different environments (`guest_exec` runs no login shell; the
  agent's shell may export `XDG_STATE_HOME` or `XDG_RUNTIME_DIR`), so any path derived from
  environment variables could differ between them and nobody would find the socket. There is no
  override; add one only if a deployment's home cannot hold a Unix socket. Socket paths are
  limited to about 100 bytes (103 on macOS, 107 on Linux); the relay fails with a clear message if
  the path is longer.
- Everything arriving on the channel is untrusted input from that guest. The broker validates
  every message and never trusts a claim of identity in it.

## 4. Handshake and liveness

- **Handshake.** The first line in each direction is a `hello` carrying the protocol version. The
  guest's copy of penyero is independent of the broker's (SPEC.md §3), so versions will drift;
  on a mismatch the broker reports it in its pane and closes the channel, and the relay answers
  clients with an "incompatible broker" error until it exits.
- **Heartbeat.** The broker sends a `ping` periodically (default every 10 s) and the relay answers
  `pong`. Liveness does not rely on end-of-file on the channel: runtimes don't guarantee that an
  exec'd process sees its stdin close when the calling side dies, and they can't always kill it.
  - The relay exits when it hasn't heard from the broker for a few intervals (default 30 s), or
    on end-of-file.
  - The broker treats a missing `pong` the same way as the channel ending: it kills the channel
    process and restarts it with backoff.
- **Takeover.** A relay starting on a socket that is already served by a live relay takes over:
  it replaces the socket. The old relay checks on every heartbeat that the socket path is still
  its own (same inode); once it isn't, it tells its broker it was taken over and exits, and that
  broker reports "another broker took over this guest" in its pane instead of reconnecting. An
  orphaned relay with no broker exits when its heartbeat lapses. On exit a relay removes the
  socket only if it is still its own. Refusing instead would let an orphaned relay lock a
  restarted broker out of its guest. Takeover grants nothing to the guest: any process in it could
  already answer on the socket, and results are only as trustworthy to clients as the guest
  itself (SPEC.md §3).

## 5. Requests, results and channel loss

Requests live in the broker's memory, keyed by id. An id is `<run>-<request>`: the broker run's
id (§8) and ten random base32 characters (50 bits) from the operating system's random source. No
uniqueness check is kept. Within a run, even an unusually busy run of 10,000 requests has a
collision chance around 4 × 10⁻⁸. The run prefix separates runs, but it is random too: two runs
of one guest share a run id with a chance around 5 × 10⁻⁴ over a thousand runs, and then a stale
id is reported as unknown (252) instead of lost. Both sizes get tuned during development (§12).
Random rather than sequential, an id tells an agent nothing about other sessions' requests, and a
mistyped or stale id practically never lands on someone else's. The prompt shows only the request
part. `wait` or `retrieve` for an id from an earlier run answers 248 (lost), with the audit log's
last recorded step for that request on stderr.

A settled request keeps its result until a `retrieve` completes: the client sends `retrieved` after
writing the last byte to its stdout without error, and only then does the broker delete the request
(SPEC.md §6). A `retrieve` that ends early (broken pipe, killed client) leaves the result in place.

**Results are spooled, not held.** A resource's stdout, and its stderr the same way, is captured
with `tempfile.SpooledTemporaryFile`: in memory up to a threshold (1 MiB), then rolled over to a
file in the broker's private temp directory (§8). A rolled-over spool file has no path, so the
prompt's head/tail preview (10 lines each; a request is shown whole up to 20 lines, §12) reads the
spool, and `view` and `edit` first write a named copy, with the extension from `output_syntax`, into
the same directory. An edited copy becomes the released result. Output stops at a per-request cap
(default 64 MiB): the resource is stopped and the request becomes `failed` with the output written
up to the cap. `retrieve` exits 1, and a `penyero:` line on stderr says the cap was reached, the
output is partial, and the request should be narrowed. A resource's `timeout` (SPEC.md §8) ends the
same way, the line saying it timed out. Memory stays flat whatever the result size, and waiting
results cost disk, not RAM.

When the channel ends, the broker marks the guest offline and applies the rules in SPEC.md §6:
requests that have not started become `dropped` (exit 249 on `retrieve`); requests that are
running, awaiting release or settled are kept and flagged "not retrieved". It then restarts the
channel with backoff. When the relay's stdin closes or its heartbeat lapses, it removes its socket
and exits, so clients fail fast with "broker not connected" (exit 254). A broker restart loses the
in-memory requests; the audit log (§8) is the record. Quitting `serve` with requests outstanding
asks for confirmation first; on quit, running resources are stopped and each request's last step
is logged.

## 6. Framing

- **Client ↔ relay:** newline-delimited JSON, one command per connection. The client sends one
  message (`list`, `describe`, `request`, `wait`, `retrieve`, `pending`, `cancel`) and reads
  events until the connection closes: `accepted` (with the id), `settled`, `chunk`, `result`,
  `error`. For `retrieve` the client answers the final `result` with `retrieved` once its stdout
  is written.
- **Relay ↔ broker:** the same messages over the channel, each line wrapped with a connection id
  assigned by the relay: `{"conn": 7, "msg": {...}}`, plus `{"conn": 7, "close": true}`. This
  multiplexes concurrent clients over one channel. The relay does not interpret messages; it only
  wraps and unwraps them.
- **Byte payloads are streamed in chunks.** The request payload travels from the client, and the
  result back to it, as `chunk` messages (`{"chunk": "<base64>"}`, at most 64 KiB of data each),
  followed by a final message without payload (`end` from the client, `result` from the broker).
  Base64 keeps binary content safe. Chunking keeps one large result from blocking the other
  connections sharing the channel, and bounds line length.
- **Line limit.** A line longer than 1 MiB is a protocol error: the relay drops that client's
  connection, and the broker drops that connection id. Neither needs unbounded buffers.
- **Validation and limits (SPEC.md §3).** The broker parses each line with the standard library's
  JSON decoder, catching its recursion limit, and checks the result against a strict schema per
  message type: known keys only, ids of the form `<run>-<request>`, no unknown message types.
  Anything else is a protocol error for that connection. Per guest, it caps the total payload of a
  request (default 16 MiB), the number of outstanding requests (default 64) and the number of open
  connections (default 32); beyond a cap, new requests are refused with exit 253 and a message, not
  queued. The parser and the schema checks are the broker's only exposure to guest bytes, and they
  get fuzzed (§11).

## 7. Other transports

A host-side TCP listener is a possible later transport for hosts that offer networking but no
`guest_exec`. It would need a per-guest secret to identify callers and protection against other
guests on the same network sniffing or spoofing it (network isolation or TLS). Not planned.

## 8. Files

- Everything lives under `~/.penyero` on the privileged side (SPEC.md §8), with no override
  through the environment; tests set `HOME`.
- Audit log: `~/.penyero/log/<profile>/<name>.jsonl`, one JSON line per request step,
  each carrying the request id (SPEC.md §9). A directory per profile keeps names apart without
  restricting them: `a/b-c` and `a-b/c` would collide in a single flat name. `profile/name` is
  unique only while the guest exists
  (the runtime enforces it: container names, VM names, user names), and a guest recreated with the
  same name shares the file, so each `serve` start writes a start record with a **run id** (a
  random six-character base32 token; the record also carries the start time) and every entry
  carries it. The run id is also the first part of every request id (§5).
- Spooled results (§5) and review temp files (SPEC.md §7) live in a privileged-owned temp
  directory with mode 0700, removed with the request or when the broker exits.

## 9. Technology

- **Language: Python**, with a **pinned interpreter managed by uv**, so there is no dependency on
  the stock system Python and no version matrix to support. A self-contained executable that needs
  nothing installed (pex's scie output or PyInstaller embed the interpreter) is a packaging option
  to evaluate when penyero is distributed.
- Standard library only at runtime: `socket`, `selectors`, `subprocess`, `json`, `base64`,
  `argparse`, `tomllib`, `hashlib`, `tempfile`, `termios`, `pwd`, `secrets`. No
  third-party runtime dependencies. The inline prompt shows content escaped, not reformatted
  (SPEC.md §7); anything richer is the external pager or editor.
- **Rust is a deliberate later option, not now.** penyero's untrusted input is JSON over the
  channel, mostly passed through to subprocesses; the security-critical logic is process and
  permission handling, not parsing. The stdlib `json` scanner (and `base64`'s) is C, the one
  native component exposed to guest bytes; it is mature and widely exercised. If that ever needs
  removing, the stdlib's pure-Python scanner can be forced, at some cost in speed. A Rust rewrite
  fits as a "once the design stops changing" step.

## 10. Session holder (backlog)

A sketch for SPEC.md §8's session holder, a tool separate from the broker: it holds the process on
a pty in its own terminal, where the human sees it start and completes any interactive
authentication, and listens on a privileged-only Unix socket. The resource executable sends each
approved snippet there; the holder writes it to the session followed by a sentinel and returns
the output up to the sentinel. It also needs handling for output interleaved with prompts,
keepalive, and cancellation. Local clients such as psql would run confined, per SPEC.md §8's input
rules. Details are open (§12).

## 11. First iterations

Iterations are short and each one is usable; what comes after is chosen when the previous one
ships (SPEC.md §10).

1. **Environment** ([SETUP.md](SETUP.md)): [cayo](https://github.com/nandilugio/cayo) with one
   `trusted` guest and its egress, verified, and the tmux layout. Building the rest *inside* the
   real boundary surfaces the true frictions instead of guessing them.
2. **Core loop, minimal:** `serve` with configuration resolution, the channel and relay, one
   resource, `request`/`wait`/`retrieve` and the prompt with both confirmations, with requests
   running concurrently up to each resource's `concurrency`. Against a local dev database, with
   the Postgres example resource of SETUP.md §6 built and tested here. The channel parser and
   schema checks are fuzzed with malformed and adversarial input from this iteration on.
3. **Core loop, complete:** `list`, `describe`, `pending`, `cancel`, output review with the
   external pager and editor, timeouts and the running view, the audit log, a read-only cloud
   resource.
4. **Git and deploy flow** (cayo's git review flow, SETUP.md §7): clean clones, the `ext::`
   remote, push and deploy from a reviewed commit.
5. The rest of the backlog, in the order decided at the time.

## 12. Open questions

- The default caps in §6 (payload, outstanding requests, connections) are guesses to tune in use.
- Heartbeat and timeout defaults (10 s / 30 s) are guesses to tune in use.
- The sizes of the run id (6 base32 characters) and of the request part (10), tuned against real
  request and restart counts (§5).
- **(verify)** what the relay sees when `serve` is killed hard, per runtime (end-of-file or
  nothing). The heartbeat covers both, but it tells us how long orphans linger.
- Preview sizes (10 + 10 lines; whole requests up to 20), tuned in use (§5).
- Session holder: sentinel robustness, prompt noise, long-running statements, cancellation.
