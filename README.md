# wynston

Host a Wynston chat engine from the command line.

A Wynston chat can be answered by a coding agent on your own machine instead of
by a server-side assistant. The Wynston Mac app can host one, and
`wynston engine serve` does the same job headless, on a Linux box or a server:
- it opens one chat per folder you configure;
- when someone types in the chat, it runs Claude Code in that folder;
- it streams the reply back into the chat.

**Status: early.** Sign-in works (`login`, `whoami`, `logout`), and so does
`wynston engine serve`, with a dashboard in a terminal where you approve the
asks the policy doesn't allow, and it runs as a service.

## Install

Each release has static binaries for Linux and macOS, on amd64 and arm64, with
no dependencies.

```sh
brew install --cask lbpdevcodes/wynston/wynston     # macOS, or Linux with Homebrew
```

Or download `wynston_<version>_<os>_<arch>.tar.gz` from the
[releases](https://github.com/lbpdevcodes/homebrew-wynston/releases), check it
against `checksums.txt`, and put `wynston` on your `PATH`:

```sh
tar -xzf wynston_0.1.0_linux_amd64.tar.gz
sudo install wynston /usr/local/bin/
wynston --version
```

## Sign in

```sh
wynston login
```

In a terminal, `login` asks two things. Press Enter to accept a default.
1. **Server:** the Wynston to sign in to. Leave it empty for production.
2. **This machine:** the name you'll see when you approve it, and later under
   Access keys. It defaults to the hostname.

`login` then shows a code and a link. Open the link in a browser where you're
signed in to Wynston, and approve the code. `login` saves the key and says who
you're signed in as. Ctrl+C cancels at any point.

Without a terminal, such as a pipe, a script or `ssh` with no tty, `login` asks
nothing. It prints the code and the link as plain lines and waits. `--host` and
`--name` give it the answers instead.

Once you're signed in:

```sh
wynston whoami    # who this machine is signed in as; also checks the key still works
wynston logout    # forgets the key on this machine
```

`logout` deletes only the local copy. The key keeps working until you revoke it
under Access keys, and `logout` prints the link.

## Answer chats

Before `serve` can answer anything:
1. **Claude Code is signed in on this machine.** `serve` runs
   [Claude Code](https://claude.com/claude-code), and uses its sign-in.
2. **Your Wynston workspace has a published default assistant.** It never
   answers these chats, but a chat can't be opened without one.
3. **You've run `wynston login` once**, at a terminal, and approved the
   device. A service can't do this for you.

```sh
wynston engine add ~/code/my-app            # answer a chat for this folder
wynston engine allow 'Bash(git status:*)'   # let Claude use a tool without asking
wynston engine list                         # the folders, their chats, and the policy
wynston engine serve                        # answer until stopped (Ctrl+C)
wynston engine remove ~/code/my-app         # stop answering for a folder
```

- **One chat per folder.** `serve` opens a chat for each folder that doesn't
  have one yet, titled `<machine> · <folder>`, and remembers it, so each folder
  keeps its chat between runs. `list` shows `not opened yet` until then.
- **Claude runs in the folder** when someone types in that folder's chat, and
  the reply streams back into the chat.
- **Changes to folders and rules apply from the next `serve`.** Restart `serve`
  after `add`, `remove` or `allow`.
- **The policy.** Allow rules use Claude Code's permission syntax, as in
  `Bash(git status:*)`, and a tool they cover runs without asking. Every other
  permission ask goes to the dashboard; without one, it is declined at once.
  Asks are only ever answered on this machine, never from the web.

### On a terminal

Run in a terminal, `serve` shows a dashboard: a line per folder with its chat
and how its turn stands, then the turn going on, its steps and words as they
come. When Claude asks for a tool the policy doesn't allow, the ask shows at
the bottom: press `y` to allow it or `n` to deny it. Press `q` or `Ctrl+C` to
stop. Turns still running end as failed, and `serve` waits for Claude to exit.

Without a terminal, as under a service manager, `serve` writes log lines
instead (JSON) and declines every ask outside the policy.

### As a service

Each release archive has a `contrib/` folder with a service file for each
system. Homebrew keeps it in `$(brew --prefix)/Caskroom/wynston/<version>/contrib/`.
Each file says how to install it.
- **Linux:** `wynston-engine.service` is a systemd user unit. It runs as you,
  restarts after an error, and logs to the journal
  (`journalctl --user -u wynston-engine -f`). Run `loginctl enable-linger` to
  keep it running while you're logged out.
- **macOS:** `ai.wynston.engine.plist` is a launchd agent that logs to
  `~/Library/Logs/wynston-engine.log`.

A service's `PATH` is minimal, so set `"claude"` in `engine.json` to its full
path, as in `"claude": "/usr/local/bin/claude"`. Both files stop `serve` with
SIGTERM: turns still running fail, and it waits up to 30 s for Claude to exit.

## Configuration

| Setting | What it does |
| --- | --- |
| `--host`, `WYNSTON_HOST` | The server `login` signs in to. The default is `https://hooks.wynston.ai`. A bare name gets `https://`; `localhost`, `127.*`, `*.local` and `*.test` get `http://`. |
| `--name` | The machine name `login` shows. The default is the hostname. |
| `WYNSTON_TOKEN` | A key to use instead of the saved one. It goes to `WYNSTON_HOST` if that's set, and to production otherwise. Use it on servers and in CI. |
| `XDG_CONFIG_HOME` | Where the key is saved: `$XDG_CONFIG_HOME/wynston/credentials.json`, or `~/.config/wynston/credentials.json` when it's unset. The file is mode 0600. The engine's config, `engine.json`, sits beside it. |
| `XDG_STATE_HOME` | Where `serve` remembers each folder's chat: `$XDG_STATE_HOME/wynston/chats.json`, or `~/.local/state/wynston/chats.json` when it's unset. Under the dashboard, `serve` logs to `serve.log` beside it. |
| `engine.json` | Written by `add`, `remove` and `allow`. Edit it by hand for the rest: `"deny"` rules, `"claude"` for a binary that isn't on `PATH`, and `"concurrency"` for how many turns run at once (default 3). |

`WYNSTON_HOST` on its own never sends the saved key to a different server.

Production is `https://hooks.wynston.ai`, the name Wynston keeps for machines.
The approval link points there too. If you point `wynston` at `wynston.ai`, the
error says where to go instead.
