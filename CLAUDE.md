# provoke_tools

Personal dev tools, installed onto remote (mostly Amazon Linux) boxes by a companion
installer repo.

## The two repos

| local path | remote | branch | role |
|---|---|---|---|
| `~/Developer/code/misc/provoke_tools` | `git@github.com:provoke/tools.git` | `master` | payload — bare executables at repo root |
| `~/code/misc/provoke_install` | `git@github.com:provoke/install.git` | `main` | bootstrap `install.sh` |

`~/code` is a symlink to `~/Developer/code`, so both paths reach the same directory.

Install one-liner:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/provoke/install/main/install.sh)"
```

It curls the tools tarball from `master`, drops everything into `~/.local/bin`, relocates
`tmux.conf` to `~/.tmux.conf`, and wires an aliases source line into shell rc files. Both
URLs are public (verified 200 unauthenticated) — anything committed here is world-readable.

## Conventions

- Tools are bare executables at repo root: no extension, mode 755, shebang'd.
- Everything at repo root ships to every machine. There is no per-environment mechanism,
  so env-specific tools self-guard — fail with a clear message when off their target env.
- Commits use conventional style (`feat:`, `fix:`, `build(push)`).
- `.idea/` stays untracked in both repos.
- Verify installer behaviour empirically with an overridden `HOME` in the scratchpad and a
  stub payload; never against the real `$HOME`.

## Current status (2026-08-04)

Committed, pushed, and tested. Both remotes confirmed via `ls-remote`:

- provoke_tools `master` — `31560d4 feat: dump-substruct`
- provoke_install `main` — `41b3356 fix: anchor installer to $HOME and survive a failed download`

`dump-substruct` was installed and run on the staging box and reported working
(2026-08-04). This work is complete; nothing is in flight.

## Open items

No active work. One thing left dangling, not blocking: the four known-broken payload items
below, if they ever become annoying enough to fix.

`CLAUDE.md` is committed deliberately, so it survives a fresh clone. It does get installed
to `~/.local/bin/CLAUDE.md` on every machine — accepted. Infrastructure and local-workflow
detail is deliberately kept out of this file and lives in auto-memory instead.

## dump-substruct

Two-pass mysqldump of `com_yes_substruct` from staging Aurora, with
`raw_track_play_instances` and `track_play_instances` as structure-only.

Decisions already made, with reasons — don't re-litigate:

- **Hardcoded** host/db/table-list rather than a generic `db-dump --db X`. The source spec
  said to lean hardcoded unless the repo already had a per-env config convention; it doesn't.
- **Installs to `~/.local/bin`, not `/usr/local/bin`.** The spec named `/usr/local/bin` but
  also said to match the repo's existing layout, and the installer only writes to
  `~/.local/bin`. For a system-wide name:
  `sudo ln -s ~/.local/bin/dump-substruct /usr/local/bin/dump-substruct`.
- **Credentials via `--defaults-extra-file`** (default `~/.my.cnf`, override with
  `DUMP_SUBSTRUCT_CNF`), never argv. Refuses a missing or group/world-readable file.
- **`--defaults-extra-file` must remain the first mysqldump argument** — MySQL requires it.
- **`--column-statistics=0`** is added only when the client advertises it, covering 8.0
  client vs Aurora 5.7 server skew.
- **Temp file then `mv` on success**, so a failed second pass can't leave a truncated dump
  that looks complete. Exits non-zero on failure. Output lands mode 600.
- **The passes were deliberately NOT inverted.** Each mysqldump pass emits
  `SET FOREIGN_KEY_CHECKS=0` in its header, so appending the two `CREATE TABLE`s at the end
  restores cleanly. The staging run reported no problems. If a future schema change ever
  does trip it, invert: `--no-data` for the whole DB first, then `--no-create-info` with the
  ignore flags for the data.
- **The hardcoded Aurora endpoint being public is fine — reviewed and accepted, don't
  re-raise.** No credentials live in the repo.

The source spec `substruct-dump-task.md` still shouldn't be committed, not for secrecy but
because the installer would ship it to `~/.local/bin` on every machine. Already removed from
the working tree.

## Known-broken payload (none fixed — out of scope so far)

Fix only if asked; all four were raised and deliberately deferred.

1. **`~/.local/bin` is never added to `PATH`** by the installer. Check
   `command -v dump-substruct` right after installing.
2. **`git-mark-resolved` and `git-resolve-theirs` don't work.** Their `sed` strips a leading
   `#` from `git status`, which is the pre-1.8 format; modern git emits a tab, so the entire
   line gets handed to `git add` / `git checkout`. `git-resolve-theirs` also has `--their`,
   missing the `s`. Modern replacement: `git diff --name-only --diff-filter=U`.
3. **`tmux.conf` targets tmux ~1.9.** Local tmux is 3.6a; `window-status-{bg,fg,attr}`,
   `pane-border-{bg,fg}`, `message-attr` and friends were removed in 2.9 in favour of
   `-style` options. Sourcing it today produces unknown-option errors.
4. **`zellij` is a 17 MB Linux x86-64 binary committed to git.** Dead on macOS
   (`exec format error`) and the reason the tarball is ~6.8 MB.

## Environment assumptions

- The payload is Linux/bash-oriented (`install_codedeploy` is yum-based), but the installer
  wires both `.bashrc` and `.zshrc`, since a zsh login shell never reads `.bashrc`.
