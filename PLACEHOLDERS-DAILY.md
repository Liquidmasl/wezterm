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
`wezterm-nightly` on Linux, brew on the Mac), remove the install below (on the
Mac: delete `WezTerm Daily.app` and the `(\.daily)?` in Karabiner) and the
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

`wezterm.executable_dir` is `…/WezTerm Daily.app/Contents/MacOS` on the Mac and
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

Status: set up 2026-10-14, **side by side** with the brew cask
`wezterm@nightly` (kept as the fallback, like apt on Linux). The marker file
holds the installed commit
(`cat ~/Applications/"WezTerm Daily.app"/Contents/MacOS/placeholders-daily`).

### One-time setup

1. Toolchain: Xcode command line tools and `brew install rustup` (keg-only,
   so `~/.zshrc.local` puts `/opt/homebrew/opt/rustup/bin` on PATH), then
   `rustup default stable`.
2. Clone the fork and add upstream for the PR ref:
   ```sh
   git clone -b placeholders-daily https://github.com/Liquidmasl/wezterm.git ~/repos/wezterm-daily
   cd ~/repos/wezterm-daily
   git remote rename origin fork
   git remote add origin https://github.com/wezterm/wezterm.git
   git submodule update --init --recursive
   ```
3. Build: `cargo build --release -p wezterm-gui -p wezterm -p wezterm-mux-server -p strip-ansi-escapes`
   (about 10 minutes cold on the M-series MacBook; a plain build is arm64-only,
   which `ci/deploy.sh` handles by copying instead of `lipo`).

### Building the app bundle

A bare `cargo build` gives binaries, but macOS launches WezTerm as an app
bundle (Spotlight, Dock, Raycast). This is the `darwin` case of
`ci/deploy.sh`, unsigned, plus our own name and bundle id so it doesn't clash
with the brew `WezTerm.app`:

```sh
out=~/Applications/"WezTerm Daily.app.new"; rm -rf "$out"
cp -r assets/macos/WezTerm.app "$out"
rm -f "$out"/*.dylib
mkdir -p "$out/Contents/MacOS" "$out/Contents/Resources"
cp -r assets/shell-integration/* "$out/Contents/Resources"
cp -r assets/shell-completion "$out/Contents/Resources"
tic -xe wezterm -o "$out/Contents/Resources/terminfo" termwiz/data/wezterm.terminfo
for bin in wezterm wezterm-mux-server wezterm-gui strip-ansi-escapes; do
  cp "target/release/$bin" "$out/Contents/MacOS/$bin"
done
git rev-parse HEAD > "$out/Contents/MacOS/placeholders-daily"   # marker for the env guard
pl="$out/Contents/Info.plist"
plutil -replace CFBundleIdentifier -string com.github.wez.wezterm.daily "$pl"
plutil -replace CFBundleName -string "WezTerm Daily" "$pl"
plutil -replace CFBundleDisplayName -string "WezTerm Daily" "$pl"
codesign --force --deep -s - "$out"   # ad-hoc sign; arm64 refuses unsigned code
rm -rf ~/Applications/"WezTerm Daily.app"; mv "$out" ~/Applications/"WezTerm Daily.app"
/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister -f ~/Applications/"WezTerm Daily.app"
```

Built locally, it has no quarantine attribute, so Gatekeeper won't complain.
Nothing in the wezterm code depends on the bundle id. Quit the Daily app
(Cmd+Q) before swapping in a rebuild.

### Things that key on the bundle id

The separate id (`com.github.wez.wezterm.daily`) keeps Spotlight, the Dock and
`brew upgrade` apart, but anything that matches the stock id needs the new one
too:

- **Karabiner** (dotfiles): the terminal exclusions match
  `^com\.github\.wez\.wezterm(\.daily)?$`. With the stock id only, Karabiner
  remaps Ctrl shortcuts in the Daily app (Ctrl+C/V and Ctrl+Alt splits break,
  zsh shows `execute:`).
- macOS privacy permissions (Full Disk Access etc.) are per app; grant them
  again if needed.

### CLI

No symlink: brew's `/opt/homebrew/bin/wezterm` stays the default. Each app
sets `WEZTERM_EXECUTABLE` in its panes, and `~/.zshrc.local` puts that app's
`Contents/MacOS` first on PATH, so `wezterm cli` in a pane always matches the
GUI it runs in. yakstack already calls `wezterm.executable_dir .. '/wezterm'`.

### Check it works

- In a Daily pane: `wezterm --version` shows this build's commit,
  `which wezterm` points into `WezTerm Daily.app`, and
  `echo $CLAUDE_CODE_FORCE_TERMINAL_IMAGES` prints `1`.
- Ctrl+C / Ctrl+V and the Ctrl+Alt split keys work (Karabiner, above).
- Repro script from sjuxax/wezterm#1 (`kitty_swap_test.py`): tests 1–6
  all switch to B / disappear.
- In Claude Code: images render inline.

## Later: one script for both

Plan: a `wezterm-daily` script in the dotfiles (chezmoi) that does the branch
update above, builds, installs for the current OS (Linux copy + symlinks, or
macOS bundle), and pushes `placeholders-daily` if anything changed. Not
written yet. Write it after the manual run on each machine has shown which
steps actually work.
