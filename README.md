# fix-steamstub

Fixes `Application load error 6:0000065432` for SteamStub-wrapped games under
[SLSsteam](https://github.com/swwayps/slsteam-moon).

SLSsteam strips a wrapped game only when its Steam launch options carry
`--steamless`. Without the token, SLSsteam restores the wrapped executable at
every launch and the error persists. This tool plants the token and
pre-strips the executable.

Built for [LuaTools-moon](https://github.com/swwayps/luatools-moon) libraries;
nothing in it is LuaTools-specific.

## What it does

1. Locates the game across all Steam libraries.
2. Detects wrapped executables (SteamStub v2 `VLV` magic / v3 `.bind`
   section).
3. Plants `--steamless` into the app's LaunchOptions in a dormant user
   profile. Existing options (`-dx11`, etc.) are preserved; the token is
   appended.
4. Pre-strips the executable via SLSsteam's `run-steamless.sh`. The original
   is kept as `<exe>.original.exe`; a `.steamless_done` marker records the
   state.

Re-running is safe: planting is a no-op when the token is set, and stripped
executables with a fresh marker are skipped.

## Requirements

- Python 3
- Steam installed natively with your package manager (apt, pacman, dnf,
  etc.; no Flatpak or Snap)
- [LuaTools-moon](https://github.com/swwayps/luatools-moon) recommended; it
  installs the `steamstub-bypass` kit this tool depends on
- The `steamstub-bypass` kit at
  `~/.local/share/SLSsteam/steamstub-bypass/`, containing `run-steamless.sh`
  and `steamless-bin/`
- Wine or Proton for the strip step; any Proton install in Steam covers it

Run the tool while the target game is closed. The strip helper terminates
stale `wineserver` processes.

## Install

### 1. Fetch

```bash
git clone https://github.com/pepino13216/fix-steamstub
cd fix-steamstub
```

Alternatively, fetch the single file:

```bash
curl -fsSLO https://raw.githubusercontent.com/pepino13216/fix-steamstub/main/fix-steamstub
chmod +x fix-steamstub
```

### 2. Install next to the kit

```bash
install -m755 fix-steamstub ~/.local/share/SLSsteam/steamstub-bypass/fix-steamstub
```

### 3. Optional: add to PATH

```bash
mkdir -p ~/.local/bin
ln -sf ~/.local/share/SLSsteam/steamstub-bypass/fix-steamstub ~/.local/bin/fix-steamstub
```

### 4. Verify

```bash
fix-steamstub --scan
```

Without the PATH symlink, invoke by full path:

```bash
~/.local/share/SLSsteam/steamstub-bypass/fix-steamstub --scan
```

### Update

```bash
git pull
install -m755 fix-steamstub ~/.local/share/SLSsteam/steamstub-bypass/fix-steamstub
```

## Usage

```
fix-steamstub <appid> [<appid> ...]      plant token + strip (fix now)
fix-steamstub --no-strip <appid> [...]   plant token only
fix-steamstub --check <appid> [...]      report state, change nothing
fix-steamstub --scan                     list installed apps with wrapped exes
fix-steamstub --scan --fix               fix every app found
```

The appid is the number in the store URL (`store.steampowered.com/app/361420/`):

```bash
fix-steamstub 361420
```

Output:

```
[fix-steamstub] appid 361420: planted --steamless in ~/.steam/steam/userdata/658562770/config/localconfig.vdf (new appid block; oldest profile (dormant, live client won't rewrite it))
[fix-steamstub] stripping Astro-Win64-Shipping.exe (SteamStub v3) via run-steamless.sh
[fix-steamstub] unpacked: Astro-Win64-Shipping.exe (original kept at Astro-Win64-Shipping.exe.original.exe)
```

Exit codes: `0` ok / nothing to do, `1` failure, `2` usage error.

## States

One row here is a trap: the last one looks fixed and re-breaks at the next
launch. `--scan` exists to catch it.

| on disk | token | at next launch |
|---|---|---|
| wrapped exe | missing | error `6:0000065432` persists |
| wrapped exe | set | SLSsteam strips, then launches |
| stripped exe + fresh marker | set | launches directly |
| stripped exe + fresh marker | missing | SLSsteam restores the wrapped exe |

## Notes

- Steam updates rewrite executables; the marker goes stale and the game
  shows up in `--scan` again. Re-run to fix.
- On single-profile machines the token lives in that profile, which the
  client can rewrite. If it goes missing, close Steam and re-run.
- Tested against The Binding of Isaac: Rebirth, Euro Truck Simulator 2,
  Astroneer.

## Development

Lint and format with [ruff](https://github.com/astral-sh/ruff):

```
ruff check fix-steamstub
ruff format fix-steamstub
```

## License

AGPL-3.0-only.

The LaunchOptions parsing and `-steamless` matching are a Python port of
[`steamstub_launchopt.hpp`](https://github.com/swwayps/slsteam-moon/blob/slsteam-moon/src/feats/steamstub_launchopt.hpp)
from SLSsteam (AGPL-3.0-only). Unpacking is done by
[Steamless](https://github.com/atom0s/Steamless) via SLSsteam's
`run-steamless.sh`.
