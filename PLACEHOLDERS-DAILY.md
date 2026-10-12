# placeholders-daily

My daily-driver WezTerm build until wezterm/wezterm#7924 is merged upstream:
the Unicode placeholder PR plus our fixes for it (sjuxax/wezterm#1), so Claude
Code images and video work inline.

**This branch is personal. Never open a PR from it.** PR work happens on
`fix-placeholder-retransmit`, which is the head of sjuxax/wezterm#1.

## Branches in `Liquidmasl/wezterm`

| Branch | Purpose | Updated by |
|---|---|---|
| `main` | Untouched mirror of upstream, so new PRs start clean | GitHub "Sync fork" |
| `fix-placeholder-retransmit` | Our 3 fixes on top of the #7924 head (= sjuxax/wezterm#1) | Only for PR review changes |
| `placeholders-daily` | What I run: latest #7924 + our fixes + this file | **Merges only**, never reset or force-pushed, so both machines can update it |

Updating `placeholders-daily` (on either machine):

```sh
git fetch origin pull/7924/head:refs/remotes/pr/7924   # origin = wezterm/wezterm
git fetch fork                                          # fork   = Liquidmasl/wezterm
git switch placeholders-daily
git merge --no-edit fork/placeholders-daily     # what the other machine pushed
git merge --no-edit pr/7924                     # latest PR head
git merge --no-edit fork/fix-placeholder-retransmit
# once sjuxax merges #1 the last merge reports "Already up to date"
git push fork placeholders-daily
```

Merge conflict: stop and resolve by hand. Don't let a script guess.

## Retiring it

Once #7924 lands in the nightly: switch back to the packaged nightly (apt
`wezterm-nightly` on Linux, brew on the Mac), remove the install below and the
`wezterm.lua` guard, delete this branch, and ask Claude Code to allowlist WezTerm
so `CLAUDE_CODE_FORCE_TERMINAL_IMAGES` isn't needed.

## Claude Code env var

Claude Code only draws images in terminals on its allowlist (checked by
XTVERSION name: kitty, Ghostty). WezTerm isn't on it even with placeholder
support, so it needs `CLAUDE_CODE_FORCE_TERMINAL_IMAGES=1`.

**Only set it when running this build.** In stock WezTerm, Claude Code would
send placeholders the terminal can't draw, and images turn into garbage
characters. The install step drops a marker file `placeholders-daily` next to the
binaries, and the tracked `wezterm.lua` (dotfiles) sets the variable only if
that file exists:

```lua
-- in dotfiles wezterm.lua (tracked, harmless on machines without this build)
local marker = wezterm.executable_dir .. '/placeholders-daily'
local f = io.open(marker, 'r')
if f then
  f:close()
  config.set_environment_variables = config.set_environment_variables or {}
  config.set_environment_variables.CLAUDE_CODE_FORCE_TERMINAL_IMAGES = '1'
end
```

`wezterm.executable_dir` is `…/WezTerm.app/Contents/MacOS` on the Mac and
`~/.local/opt/wezterm-placeholders` on Linux, so the same file works on both.

## Linux (Pop!_OS desktop)

Status: set up 2026-10-13. The guard is in the dotfiles `wezterm.lua`, and the
marker file holds the installed commit (`cat ~/.local/opt/wezterm-placeholders/placeholders-daily`).

- Worktree `~/repos/wezterm-daily` on this branch. `~/repos/wezterm-pr7924`
  stays on `fix-placeholder-retransmit` for PR work.
- Build: `cargo build --release -p wezterm-gui -p wezterm -p wezterm-mux-server`
  (here with `CARGO_TARGET_DIR=~/repos/wezterm-pr7924/target`, shared with
  the PR checkout so builds stay incremental; safe because we copy the binaries)
- Install: **copy** (don't link `target/`, or a rebuild or `cargo clean` pulls
  the binary out from under the running terminal) `wezterm`, `wezterm-gui`,
  `wezterm-mux-server` to a fresh `~/.local/opt/wezterm-placeholders.new/`, add
  the marker file, then swap it into `~/.local/opt/wezterm-placeholders/`.
- Symlink those three into `~/.local/bin/`. It's first on PATH, including the
  COSMIC session, so Super+T (`wezterm start --cwd .`) and the system
  `.desktop` entry (`Exec=wezterm start …`) both pick it up. No `.desktop`
  override needed.
- apt's `wezterm-nightly` stays installed as a fallback in `/usr/bin`.
- After switching, quit every running WezTerm once. `wezterm start` hands
  new windows to an already running GUI process.

## macOS (MacBook)

Status: to do. WezTerm is probably the brew cask there; check with
`brew list --cask | grep -i wezterm`.

### One-time setup

1. Toolchain: Xcode command line tools (`xcode-select --install`) and Rust
   via rustup.
2. Clone the fork and add upstream for the PR ref:
   ```sh
   git clone -b placeholders-daily https://github.com/Liquidmasl/wezterm.git ~/repos/wezterm-daily
   cd ~/repos/wezterm-daily
   git remote rename origin fork
   git remote add origin https://github.com/wezterm/wezterm.git
   git submodule update --init --recursive
   ```
3. Build: `cargo build --release -p wezterm-gui -p wezterm -p wezterm-mux-server -p strip-ansi-escapes`

### Building the app bundle

A bare `cargo build` gives binaries, but macOS launches WezTerm as
`WezTerm.app` (Spotlight, Dock, Raycast). `ci/deploy.sh` (the `darwin` case)
assembles the bundle; the local, unsigned version of that:

```sh
out=$(mktemp -d)/WezTerm.app
cp -r assets/macos/WezTerm.app "$out"
rm -f "$out"/*.dylib
mkdir -p "$out/Contents/MacOS" "$out/Contents/Resources"
cp -r assets/shell-integration/* "$out/Contents/Resources"
cp -r assets/shell-completion "$out/Contents/Resources"
tic -xe wezterm -o "$out/Contents/Resources/terminfo" termwiz/data/wezterm.terminfo
for bin in wezterm wezterm-mux-server wezterm-gui strip-ansi-escapes; do
  cp "target/release/$bin" "$out/Contents/MacOS/$bin"
done
touch "$out/Contents/MacOS/placeholders-daily"   # marker for the env guard
codesign --force --deep -s - "$out"              # ad-hoc sign; arm64 refuses unsigned code
```

### Installing it

- Uninstall the brew cask (`brew uninstall --cask wezterm` or
  `wezterm@nightly`). Two apps with the same bundle id confuse Spotlight and
  the Dock, and `brew upgrade` would overwrite an app in `/Applications`.
- Swap the bundle into `~/Applications/WezTerm.app`. Built locally, it has no
  quarantine attribute, so Gatekeeper won't complain.
- CLI: symlink `~/Applications/WezTerm.app/Contents/MacOS/wezterm` into a PATH
  directory (the cask did that via `/opt/homebrew/bin`, and uninstalling it
  removes that link).
- Quit WezTerm fully (Cmd+Q) before relaunching.

### Check it works

- `wezterm --version` shows a version from this build, not the brew one.
- `echo $CLAUDE_CODE_FORCE_TERMINAL_IMAGES` in a new pane prints `1`.
- Repro script from sjuxax/wezterm#1 (`kitty_swap_test.py`): tests 1–6
  all switch to B / disappear.
- In Claude Code: a show-your-work video plays and the before/after toggle
  switches images.

## Later: one script for both

Plan: a `wezterm-daily` script in the dotfiles (chezmoi) that does the branch
update above, builds, installs for the current OS (Linux copy + symlinks, or
macOS bundle), and pushes `placeholders-daily` if anything changed. Not
written yet. Write it after the manual run on each machine has shown which
steps actually work.
