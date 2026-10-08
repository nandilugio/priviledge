# penyero — reference setup (draft)

Status: draft. This describes **one** way to deploy penyero: on
[cayo](https://github.com/nandilugio/cayo), which provides the isolated workspaces (guests in
dedicated colima VMs, egress through allow-list proxies, the git review flow), with penyero's own
pieces around it. penyero itself does not depend on cayo; it only needs the deployment contract in
[SPEC.md §4](SPEC.md#4-deployment-contract); how penyero reaches a guest is in
[DESIGN.md §3](DESIGN.md#3-channel-and-relay). Other setups are sketched in §8. Items marked
**(verify)** must be checked on the real machine. Every command here changes the machine's
configuration, so the human reviews and runs it.

Terms (guest, profile, privileged side, broker) are as defined in
[SPEC.md §3](SPEC.md#3-threat-model).

## 1. cayo as the deployment

cayo started as this document's reference deployment and became its own project; its README covers
installing it, its VMs, guests, egress, image, the git review flow, dev services, host exposure and
verification. It meets the deployment contract (SPEC.md §4):

1. **Separation.** Guests run in colima VMs that mount only their exchange directories and,
   read-only (enforced by the host), the human's dotfiles and whatever `CAYO_VM_MOUNTS` adds
   (clean clones for compose). Nothing of `~/.penyero` is visible to them.
2. **A way to run a program inside a guest:** `docker --context colima-cayo exec -i
   cayo-<profile>-<name>`, the `guest_exec` of §3.
3. **No path back.** Guests get no runtime socket, no capabilities, no privilege escalation and no
   terminal device of the host's (cayo's README).
4. **One guest per trust domain:** a cayo guest per project and profile.
5. **Profiles are real.** cayo's profiles are SPEC.md §3's reference set (`trusted`, `public`,
   `hostile-web`, `hostile-sample`), with their mounts and egress; `cayo verify` checks a guest
   against them. `hostile-*` guests run in a VM of their own and get no broker.

## 2. The client in the guest image

penyero is one program (DESIGN.md §1), installed the same way everywhere; in the guest only its
client and relay subcommands are used. It goes into the guest image through one of cayo's image
layers (`~/.cayo/image/Dockerfile`, for every profile), installed system-wide: `guest_exec` runs
it without a login shell, so it must be on the default `PATH`. Its integrity in the guest doesn't
matter (SPEC.md §3): it only has to speak a protocol version the broker accepts (DESIGN.md §4).
The broker's copy is what matters: install it on the host from a privileged-owned clean clone at
a reviewed tag, never from a guest.

## 3. Configuration

```toml
# ~/.penyero/config.toml
[profiles.trusted]
guest_exec = ["docker", "--context", "colima-cayo", "exec", "-i", "cayo-{profile}-{name}"]

[profiles.public]
guest_exec = ["docker", "--context", "colima-cayo", "exec", "-i", "cayo-{profile}-{name}"]
```

`--context` selects cayo's VM explicitly, since the broker runs with a fixed environment and the
machine's default Docker context is not cayo's. The relay runs as the guest's user, `cayo`, and
listens in its home volume (DESIGN.md §3).

## 4. The approval pane

Each guest's broker runs in a terminal pane of its own, next to the panes that are in the guest:

```sh
tmux split-window "penyero serve trusted shop"
```

- **The approval pane in more than one window.** tmux can't show one pane in two windows. To
  reach the same prompt from several, run the broker under `dtach`, which multiplexes its
  terminal: `dtach -A ~/.penyero/run/trusted-shop.serve -r winch penyero serve trusted shop` in
  one window (the directory created first), `dtach -a ~/.penyero/run/trusted-shop.serve -r
  winch` in another; input from any attached window reaches the prompt, and output goes to all.
  The socket is privileged-owned and never visible to guests. One limit **(verify)**: the
  broker's terminal has one size, that of the window attached last, so the full-screen pager or
  editor renders correctly only in windows of that size; the line-oriented prompt is unaffected.
  `abduco` is the alternative with the same shape.
- New prompts ring the terminal bell (`notify`, SPEC.md §8); `set -g monitor-bell on` and
  `set -g bell-action other` make tmux flag the window.
- The broker's connection to the guest has no terminal (it starts `guest_exec` detached,
  DESIGN.md §3), and the pane shows agent-written text escaped (SPEC.md §7). cayo's README
  covers the terminal itself, clipboard reads denied included.

## 5. Review tools (privileged side)

The approval prompt shows escaped plain text; any richer view comes from the pager and editor the
human configures (SPEC.md §7). They run as the privileged user over agent-authored content, so the
choice matters:

- They are the `[review]` argv lists in `~/.penyero/config.toml`, run without a shell and with
  a fixed environment, so an exported `LESS=-R` (escape sequences passed to the terminal) or
  `LESSOPEN` (a lesspipe script running other programs over the file) never reaches them.
  Tool-specific settings go in the list itself, e.g. `["env", "LESSHISTFILE=-", "less", ...]`.
- The default pager is `less -+r -+R --no-lessopen`; the `-+` options reset raw control
  characters whatever a lesskey file says.
- The default editor, `vi -u NONE -i NONE -n -c "set nocompatible nomodeline"`, loads nothing:
  no vimrc, no plugins, no syntax colouring. `-u NONE` starts vim in compatible mode, where
  modelines are off when the file is read; `-c` then restores normal vim editing and keeps
  modelines off. Checked on Debian 13 and Ubuntu 24.04 (vim-tiny), Fedora 44 (vim-minimal) and
  macOS 15: a modeline that fires when forced on is ignored. On all three Linux systems the
  minimal package provides `vi` but no `vim`, hence the name.
- With full vim, colour comes from a privileged-owned vimrc that leaves the human's `~/.vim` out
  of the runtime path (`vim --clean` doesn't do: it leaves compatible mode after `--cmd` runs,
  which turns modelines back on). Checked on macOS's vim 9.1:

  ```toml
  [review]
  editor = ["vim", "-u", "~/.penyero/review.vim", "-i", "NONE", "--noplugin"]
  ```

  ```vim
  " ~/.penyero/review.vim
  set nocompatible
  set runtimepath=$VIMRUNTIME packpath=
  set nomodeline noswapfile noundofile viminfo=
  syntax on
  ```
- A minimal nvim serves as both, the pager in read-only mode:

  ```toml
  [review]
  pager = ["nvim", "--clean", "-u", "~/.penyero/review.lua", "-R"]
  editor = ["nvim", "--clean", "-u", "~/.penyero/review.lua"]
  ```

  ```lua
  -- ~/.penyero/review.lua
  vim.o.modeline = false   -- the content must not set options
  vim.o.swapfile = false   -- no copy of the reviewed content (unredacted output, say)
  vim.o.undofile = false   --   outlives the review
  vim.cmd("syntax on")     -- colouring from nvim's bundled syntax files
  ```

  `--clean` starts nvim without the human's configuration, plugins or shada, and `-u` then loads
  only this file. Checked on nvim 0.12.5: the file is loaded, no script from the human's
  configuration or data directories is, a modeline in the content is ignored, and bundled syntax
  colouring works. No plugin, no LSP, no treesitter parser beyond the few nvim bundles.
- Answering a human resource (SPEC.md §8) is an edit of its output in the same editor.
- Diff review before a push (§7) is the human's own tooling, outside penyero, under the same
  rule: no tool that runs project code.

## 6. Resources (privileged side)

Resource executables are the human's, written to the contract and the three properties in
SPEC.md §8. The repository will carry complete, tested examples under `examples/resources/`, to
copy into `~/.penyero/resources/` and adapt; the two below are for the resources in
SPEC.md's example configuration.

```sh
#!/bin/sh
# ~/.penyero/resources/aws-readonly
# Only allow-listed operations; global options go after them. No host files, no other endpoint.
case "$1 $2" in
  "logs filter-log-events"|"logs describe-log-groups"|"ecs describe-services") ;;
  *) echo "not allowed: aws $1 $2"; exit 1 ;;
esac
for a in "$@"; do
  case $a in
    *file://*|*fileb://*|--endpoint-url*) echo "not allowed: $a"; exit 1 ;;
  esac
done
# Only this profile's files. The broker's fixed environment (SPEC.md §8) keeps any AWS_* the
# human exported out of here.
AWS_CONFIG_FILE="$HOME/.penyero/aws/readonly.config" \
AWS_SHARED_CREDENTIALS_FILE="$HOME/.penyero/aws/readonly.credentials" \
  exec aws "$@"
```

```sh
#!/bin/sh
# ~/.penyero/resources/prod-db-ro
# `sqlquery` is the human's driver-based script: one SQL statement in on stdin, sent with the
# extended query protocol (which refuses a second one, so a SET can't lift the timeout), CSV
# out, and the driver's own errors on stderr, for the human.
PGPASSWORD="$(security find-generic-password -s prod-db-ro -w)" \
  exec "$HOME/.penyero/bin/sqlquery" \
  "host=... user=app_ro dbname=... options='-c statement_timeout=60s'"
```

- `sqlquery` is a single-file Python script run by uv, with its database driver pinned inline
  (`#!/usr/bin/env -S uv run --script`): the standard library has no Postgres driver, and the
  dependency belongs to this resource, not to penyero. It reads one statement from stdin, sends
  it as a prepared statement (the extended query protocol, which refuses a second statement, so a
  `SET` can't lift the timeout), writes CSV to stdout, forwards the server's own error message to
  stdout (a syntax error or an unknown column is useful to the agent and names no host), and keeps
  connection errors on stderr. It is built and tested in the core-loop iteration (DESIGN.md §11).
- Both check the shape of their input, not its meaning: which operations, not which data. What the
  credential can read is still exposed (SPEC.md §5, condition 2).
- For tools with a large input surface, running them confined is the stronger option: a disposable
  container that holds only this resource's credential and mounts nothing from the host
  (SPEC.md §8).
- **(verify)** that the aws CLI accepts global options after the operation, and that a running
  Postgres statement can't change its own timeout.

## 7. Agent and git

- The agent runs in the guest (cayo's README). Its permission settings should let the `penyero`
  commands (SPEC.md §6) run without asking: they are how the agent asks, and the human answers in
  the approval pane, not in the agent's. Any agent that runs shell commands works.
- The git read-only token (SPEC.md §5) is the only native token, in a `trusted` guest's git
  credential store. Other read-only access (code host, error tracker, issue tracker) goes through
  penyero resources.
- Review and push follow cayo's git flow: a clean clone on the host, the guest as an `ext::`
  remote, a comprehension pass in the guest and an authoritative one in the clean clone, for
  example with `git log -p` piped into the review nvim of §5.

**Short-lived fetch tokens (optional).** Instead of a stored token, a git credential helper in the
guest asks penyero for one on each use. A `git-token` resource mints a short-lived, read-only token
(for example a one-hour app installation token) and prints it in git's credential format;
auto-approved and released unreviewed in `trusted`. The token still enters the guest (SPEC.md
§5), but it expires, and every use is in the audit log. **(verify)**

```sh
#!/bin/sh
# ~/.local/bin/git-credential-penyero, in the guest: git config credential.helper penyero
[ "$1" = get ] || exit 0
id=$(penyero request git-token -r "git credential for a fetch" </dev/null) &&
  penyero wait "$id" && exec penyero retrieve "$id"
```

## 8. Other setups

The same penyero configuration works with a different `guest_exec` template per profile:

| Setup | `guest_exec` | Notes |
|---|---|---|
| cayo (this document) | `docker --context colima-cayo exec -i cayo-{profile}-{name}` | Dedicated VMs that see only their mounts; read-only enforced by the host |
| Docker Desktop, Podman machine | `docker exec -i <container>` (or `podman`) | One VM shared with all the machine's containers: what an escape reaches is its file sharing, by default the whole home |
| Apple `container` (macOS 26) | `container exec -i <container>` **(verify)** | One lightweight VM per container: a VM boundary between guests too. Same OCI images |
| Lima VM | `limactl shell <vm>` or `ssh` **(verify)** | Full VM per guest |
| Separate OS user | `ssh <user>@127.0.0.1` or `sudo -u <user>` | See below |

**Separate OS user** (the only option with no runtime): one user per guest; close the privileged
home to them (`chmod o-rwx ~`; they are not in the privileged user's group), and give each its own
toolchains. For interactive panes, prefer `ssh` to loopback over `sudo -u`/`su`: on macOS `su` does
not allocate a new terminal, so the guest shell shares the privileged pane's terminal device. The
broker's connection has no terminal (it starts `guest_exec` detached, DESIGN.md §3), so `sudo -u` is
fine there; for the same reason sudo can't ask for a password, so it needs a `NOPASSWD` rule that
lets the privileged user run commands as the guest user (never the other way round). For ssh:
key-only logins restricted to `AllowUsers <user>@127.0.0.1 <user>@::1`, no forwarding of any kind,
and a dedicated key installed with `restrict,pty` (on macOS, launchd starts sshd on all interfaces
and may ignore `ListenAddress` **(verify)**). Never give such a user sudo to the privileged account,
including for Homebrew: sudo would authenticate the guest's own password, which other guest
processes can capture, and brew run as its owner with guest-influenced input is equivalent to a
shell as the owner. Its weak point is egress: restricting it per user needs packet-filter rules
keyed on uid (`pf` on macOS, nftables on Linux) **(verify)**, so until those exist it cannot make a
`public` profile real (SPEC.md §4, item 5).

## 9. Verification

- `cayo verify <profile> <name>` checks the guest against its profile (cayo's README).
- Once the core loop exists: `penyero list` in the guest shows only this profile's resources, and
  stopping the guest's `penyero serve` makes `request` fail fast.
- Nothing in the guest can read `~/.penyero`: it is neither mounted into the guest nor into its VM.
