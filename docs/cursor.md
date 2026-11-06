# Cursor guide

min-cursor turns a Minimal session into a Remote-SSH host named
`min-cursor.NAME` and opens Cursor on `/workbench` in it. Everything Cursor
does after that (installing its server in the box, terminals, extensions,
port forwarding, its agent) is the behaviour of Cursor's own Remote-SSH
extension.

## Requirements

- Cursor with the `cursor` shell command (Command Palette, "Shell Command:
  Install 'cursor' command in PATH"). If it is installed but not found, run
  `mkdir -p ~/.local/bin && ln -s /Applications/Cursor.app/Contents/Resources/app/bin/cursor
  ~/.local/bin/cursor`, and make sure `~/.local/bin` is on your `PATH`.
- Cursor's own Remote-SSH extension, `anysphere.remote-ssh` (bundled with
  current Cursor; `cursor --install-extension anysphere.remote-ssh`).
  Microsoft's `ms-vscode-remote.remote-ssh` does not run in Cursor and is not
  what min-cursor checks for.
- `min-cursor ssh-config` installed (`just install` does it): Cursor's ssh
  reads `~/.ssh/config`, which then includes `~/.ssh/min-cursor.conf`.
- The box: glibc ≥ 2.28 (older needs Cursor's `linux-legacy` server), bash,
  tar, curl (all present; checked 2026-10-02).

`min-cursor doctor` checks each of these.

## Opening a session

| Command | Session | Window |
|---|---|---|
| `min-cursor open [PATH]` | the live session for PATH if there is one, else a new one | yes |
| `min-cursor open PATH -n NAME` | NAME if it exists (for PATH), else a new one named NAME | yes |
| `min-cursor new [PATH]` | always a new one | yes |
| `min-cursor attach [NAME]` | an existing one (no NAME: a numbered picker) | yes |

New sessions are activated with `--loadout cursor`. Each command prepares
the box (pins its host key, pushes the public key, refreshes the stanza),
refreshes the settings block, then runs:

    cursor --remote ssh-remote+min-cursor.NAME /workbench

The command prints `min-cursor.NAME`, so `ssh "$(min-cursor open)"` works too.

Recent folders and the Remote Explorer reconnect the same way: every
connection goes through `min-cursor _proxy NAME`, which prepares the box
again first, so a session recreated under the same name heals itself.

## How Cursor's Remote-SSH differs from VS Code's

Read from `anysphere.remote-ssh` 1.1.16 (Cursor 3.23.12) on 2026-10-16:

- **Same setting names, fewer of them.** It reads `remote.SSH.remotePlatform`
  and `remote.SSH.serverInstallPath` under the same keys as Microsoft's
  extension, both looked up by the exact host name (no wildcard keys). It has
  **no `remote.SSH.useLocalServer`**; min-cursor does not write one.
- **Platform detection.** With no `remotePlatform` entry Cursor runs its
  installer and detects the platform with `uname`; it asks only when that
  install fails. The per-session entry skips the guess and the fallback
  prompt.
- **No exec server, no separate CLI.** One server per commit, at
  `~/.cursor-server/bin/linux-<arch>/<commit>/` (arch `x64` or `arm64`),
  started from `bin/cursor-server`; data and extensions in
  `~/.cursor-server/`. 2026-10-02 (Cursor 3.21) put 442 MB there.
- **One ssh per window,** `ssh -T -D <port> min-cursor.NAME` (or `-L` to a
  socket with `remote.SSH.remoteServerListenOnSocket`), with
  `-o RemoteCommand=none` unless `remote.SSH.enableRemoteCommand` is on.
- **Agent forwarding.** `remote.SSH.enableAgentForwarding` (default on) only
  fixes up the box environment when ssh actually forwards the agent; the
  stanza says `ForwardAgent no`, and the box's sshd refuses it anyway.
- **GPG forwarding** (`remote.SSH.enableGPGAgentForwarding`, default off)
  runs a cleanup ssh with `-o StrictHostKeyChecking=accept-new`. The key is
  already pinned in the stanza's `UserKnownHostsFile`, so a changed key is
  still refused; leave it off unless you need it.
- **Installer lock** `cursor-remote-lock.<md5 of the server dir>` in
  `$XDG_RUNTIME_DIR` or `/tmp`, with a heartbeat; the commit directory is
  created before the lock is taken.

## The "Run on" menu

The **Run on** menu in Cursor's Agents window has no top-level Minimal entry,
and min-cursor cannot add one. A box can appear there in two ways: as an
ordinary remote workspace, or as a
[My Machines](https://cursor.com/docs/cloud-agent/self-hosted/my-machines)
worker (see [below](#a-box-as-a-my-machines-worker)). To run Cursor's agent in
a box with nothing extra in it, open the box from the Minimal view in IDE mode
and use the agent in that window.

Read from the Cursor 3.27.22 app files on 2026-11-05, not seen on screen.
Cursor changes these labels often. This version carries two Agents window
UIs, Glass and Molten, and Cursor picks which one you get; what follows is
Glass unless it says otherwise.

- **What a box looks like.** Under **Remote Machine**: the path `/workbench`,
  with `min-cursor.NAME` below it and a generic server icon. Molten's row
  shows the folder name (`workbench`) with `min-cursor.NAME` as its
  description. While the box is not connected, the description adds a status
  such as `Not connected` and the icon is a plug. Once a box is selected, the
  menu's trigger shows `min-cursor.NAME` alone.
- **Which boxes are listed.** Rows come from workspaces Cursor already knows.
  `~/.ssh/config` feeds the Connect dialog, not this list. So a box probably
  shows only once it has been opened, and only beside the selected project
  (inferred from the code, not traced to the end).
- **No route for an extension.** The menu's groups are fixed in Cursor's
  code, and its variable rows are remote workspaces, plus pools and workers
  from Cursor's backend. Extension points are the stock VS Code ones, and the
  [extension API](https://cursor.com/docs/extension-api) documents only `mcp`
  and `plugins`. The Agents window also filters third-party extensions out,
  which is why the Minimal view needs IDE mode. That filter was read in Glass
  only; nothing suggests Molten loads extensions either.
- **No route through a deep link.** Cursor's
  [prompt link](https://cursor.com/docs/integrations/deeplinks) only fills in
  a prompt. It does not start the agent or choose a machine.
- **Daytona.** The app has no built-in Daytona entry. Daytona connects through
  [Self-Hosted Pools](https://cursor.com/docs/cloud-agent/self-hosted/pool),
  which need Cursor Enterprise and a service-account key and run the agent
  loop through Cursor's cloud. A pool's row shows the name its operator gave
  it.

A top-level entry needs a change in Cursor; see
[#12](https://github.com/norrietaylor/cursor/issues/12).

### A box as a My Machines worker

A Cursor worker running in a box puts a row in the menu, under
**Remote Machine > My Machines**, and an agent started from that row runs its
tool calls in the box. The agent loop still runs in Cursor's cloud. Tested by
hand on 2026-11-06 with Cursor CLI `2026.11.05-86047df` on an aarch64 box and a
personal Cursor account:

```sh
curl https://cursor.com/install -fsS | bash
export PATH="$HOME/.local/bin:$PATH"
agent login
agent worker --name Minimal --worker-dir /workbench start
```

A box made from a folder has it at `/workbench`. If the repository is somewhere
else in the box, use that path; the test used a clone under `/workbench`.

- **The row.** Its label is the worker folder's name, with `--name` in grey
  beside it, so `/workbench` shows as `workbench`.
- **Git.** A worker in a folder with no git repository could not be selected;
  the same worker in a clone with an `origin` remote could. That was one
  before-and-after test, and no repository was picked in the first attempt, so
  treat this as likely, not proven. Cursor's docs don't require a repository,
  but they do say a machine's repos come from its folders' git remotes.
- **Plan.** No Enterprise plan was needed. A team account may need an admin to
  turn on **Allow Personal Machines**; that was not tested.
- **Lifetime.** The row is there only while the worker runs. It does not create
  boxes on demand, and nothing restarts it after a box reboots. Cursor's docs
  say a worker stays connected until you stop it, but this CLI's help gives
  `--idle-release-timeout` a default of 3600 seconds. The test ran for 22
  minutes, so it did not settle which applies; pass `0` to turn the timeout off.
- **Data.** Cursor's
  [docs](https://cursor.com/docs/cloud-agent/self-hosted) say the checkout stays
  on the machine, but the worker sends the content the agent needs, such as file
  contents, terminal output and diffs. Artifacts such as screenshots and log
  references upload to Cursor-managed storage by default.
- **Credentials.** `agent login` keeps a Cursor credential in the box, so
  anything running there can use it. `agent logout` removes it. Without GitHub
  credentials in the box an agent can commit but not push, and Cursor's minted
  GitHub tokens are for pool workers only.

## Settings (`min-cursor config`)

min-cursor keeps one `remote.SSH.remotePlatform` entry per session in
Cursor's user settings:

```jsonc
{
  // min-cursor:begin (managed by `min-cursor config`; `min-cursor config --remove` removes it)
  "remote.SSH.remotePlatform": {
    "min-cursor.myproject-3f9c": "linux"
  }
  // min-cursor:end
  ...your settings...
}
```

- Entries are added by `open`, `attach` and `ssh`, and dropped by
  `destroy`, `gc` and any later refresh once min no longer lists the session.
- If you already have `remote.SSH.remotePlatform` (Cursor writes it when you
  pick a platform for any host), only that object's value is rewritten,
  keeping every host that does not start with `min-cursor.`; there is then no
  marked block. That object comes back re-indented as plain JSON (comments
  inside it are lost); everything else in the file keeps its bytes.
- `--remove` restores the file byte for byte when the block was ours: the
  original framing after `{` (the end marker says `(joined)` when it had to
  add a line break), CRLF line endings and a UTF-8 BOM are all kept.
- When `min ls` fails, `config` leaves the existing `min-cursor.*` entries
  as they are (a partial list would prune live ones), and doctor says it
  could not check them.
- `remote.SSH.serverInstallPath` is left alone: the server goes to
  `/home/.cursor-server`, which persists with the session.
- Before each write the old file is copied to
  `~/.local/state/min-cursor/backups/settings.json.<time>`. A symlinked
  settings.json is written through.
- `min-cursor config --print` shows the result without writing;
  `min-cursor config --remove` takes everything out (and deletes the file if
  min-cursor created it and it is empty again).

The file is `~/Library/Application Support/Cursor/User/settings.json`, or
`MIN_CURSOR_SETTINGS`.

## Troubleshooting

- **Cursor asks for the platform.** Its install failed first; see the
  Remote-SSH output panel. `min-cursor config` (once) adds the entry so the
  retry does not ask. `min-cursor doctor` shows missing or stale entries.
- **"Could not establish connection".** `min-cursor doctor`, then
  `ssh -v min-cursor.NAME true`, then `~/.local/state/min-cursor/proxy.log`
  (one line per connection attempt). The Remote-SSH output panel shows the
  ProxyCommand's stderr.
- **"no min-cursor ssh stanza for this min-cursor.\* host".** The session is
  gone, or was never prepared by min-cursor: `min-cursor status`, then
  `min-cursor attach NAME`.
- **Host key changed after recreating a session.** Reconnect: `_proxy`
  re-pins before sshd starts.
- **Claude login fails in a box.** The browser authorizes, then
  `localhost:PORT/callback` fails with `ERR_CONNECTION_REFUSED`. Workaround:
  trust `claude.com` and all its subdomains. This is a workaround; the cause
  is not confirmed. Each box needs its own sign-in.
- **Window opens but the terminal is in `/home`.** The `min:workbench` rc
  line moves interactive shells that start in `$HOME`; sessions without the
  loadout get it on the next connect.
