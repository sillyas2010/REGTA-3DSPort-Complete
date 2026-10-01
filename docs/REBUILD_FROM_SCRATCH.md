# Rebuilding all three games from scratch

Step-by-step runbook for going from **zero** (no toolchain, no game data, empty
SD card) to three working installs — GTA III, Vice City and Liberty City
Stories — including CIA packaging and delivery to the console either by direct
SD-card access or over FTP.

Every command here was executed end-to-end on **WSL2 (Ubuntu) against a
Windows host**, which is the environment this document assumes. Adapt paths if
you run on native Linux.

> **Repository:** this checkout is the fork
> [`sillyas2010/REGTA-3DSPort-Complete`](https://github.com/sillyas2010/REGTA-3DSPort-Complete)
> of upstream
> [`Epic0522/REGTA-3DSPort-Complete`](https://github.com/Epic0522/REGTA-3DSPort-Complete).
> Commit and push documentation and build changes to the fork, not upstream.

---

## 0. Parameters — choose your folders

Everything below is driven by these variables. Set them once per shell.

```sh
export REPO=/home/$USER/documents/projects/REGTA-3DSPort-Complete
export TCHAIN=$REPO/toolchains              # toolchain root (makefile-pinned location)
export TOOLS_BIN=$TCHAIN/tools/bin          # host build tools land here

# --- asset source folders (your legally owned game data) ---
export SRC_RE3="/mnt/c/gta/re3/game"              # GTA III PC install
export SRC_RE3_AUDIO="/mnt/c/gta/re3/install/audio_backup/Audio"  # only if game/audio lacks streams
export SRC_REVC="/mnt/c/gta/revc/game"            # Vice City PC install
export SRC_RELCS="/mnt/c/gta/relcs_out"           # CONVERTED PS2 data (see §4.3)

# --- result folder (where CIAs/3DSX packages are written) ---
export OUTDIR=$REPO/packaging/production_cia/output

# --- delivery target: pick ONE ---
export FTP_HOST=192.168.10.11:5000         # 3DS running ftpd homebrew
export STAGING=$HOME/sdcard/3ds            # FTP workflow: local folder mirroring /3ds
# or, for a locally mounted SD card:
export SD=/mnt/f/3ds                       # mounted <SD>:/3ds folder
```

Notes on the folders:

| Variable | What goes in it |
| --- | --- |
| `SRC_RE3` / `SRC_REVC` | Stock PC installs of GTA III / Vice City. Read-only sources; nothing modifies them. |
| `SRC_RELCS` | **Output of the reLCS Asset Converter** (§4.3), not a raw PS2 ISO extract. |
| `OUTDIR` | Also accepted by the official packaging script via the same env var name. |
| `FTP_HOST` | The 3DS must be running an FTP server (e.g. `ftpd` homebrew) and be on the same network. |

---

## 1. Host prerequisites

```sh
sudo apt-get install -y --no-install-recommends \
    g++ make rsync md5sum zsh curl lftp ffmpeg genisoimage p7zip-full
```

* `g++` / `make` — host builds; the repo's makefiles also need `md5sum`
  (GNU coreutils) for their build fingerprint.
* `zsh` — the official CIA script is `#!/bin/zsh`.
* `ffmpeg` — only needed for the LCS audio conversion (§4.3).
* `genisoimage` — only if you must rebuild an ISO for the LCS converter.
* `lftp` — recursive FTP upload (§7.2).

**WSL2 specifics**

* Windows drives appear under `/mnt/c`, … but only for drives present at WSL
  start. A later-plugged drive must be mounted first; passwordless `sudo` is
  usually unavailable, so mount through root interop instead:

  ```sh
  wsl.exe -u root -e sh -c 'mkdir -p /mnt/f && mount -t drvfs F: /mnt/f'
  ```

  Unmount the same way before ejecting: `wsl.exe -u root -e sh -c 'umount /mnt/f'`.
* `robocopy.exe`, `tasklist.exe`, `taskkill.exe` etc. are callable directly
  from WSL (interop). Windows-native copies are both faster and more reliable
  than writing through `/mnt/*` — see §8, pitfall P1.

---

## 2. Toolchain — the exact pinned stack

The game makefiles **hard-fail** unless the toolchain sits at
`$REPO/toolchains/devkitARM` and is devkitARM **r55 / GCC 10.2**:

```make
REVC_DEVKITARM_R55 := $(abspath ../../toolchains/devkitARM)   # relative to miami/build → <repo>/toolchains/devkitARM
ifneq ($(realpath $(DEVKITARM)),$(realpath $(REVC_DEVKITARM_R55)))
$(error ...)
```

> ⚠ The path resolves from each game's `build/` dir **two levels up**, i.e. the
> repository root — *not* `miami/toolchains`. A newer system devkitPro install
> cannot be substituted: the vendored libctru/newlib ABI is locked to r55.

### 2.1 devkitARM r55 + runtime libs

The devkitPro archive (`wii.leseratte10.de`) hosts old releases. Download and
extract — both packages unpack to `opt/devkitpro/devkitARM`, so extract to a
temp dir and move:

```sh
mkdir -p /tmp/tc && cd /tmp/tc
curl -LO "https://wii.leseratte10.de/devkitPro/devkitARM/r55%20%282020-07-25%29/devkitARM-r55-1-linux_x86_64.pkg.tar.xz"
curl -LO "https://wii.leseratte10.de/devkitPro/devkitARM/devkitarm-rules/devkitarm-crtls-1.0.2-1-any.pkg.tar.xz"
mkdir x1 x2 && tar -xf devkitARM-r55-1-linux_x86_64.pkg.tar.xz -C x1
tar -xf devkitarm-crtls-1.0.2-1-any.pkg.tar.xz -C x2
mkdir -p "$TCHAIN"
cp -r x1/opt/devkitpro/devkitARM "$TCHAIN/devkitARM"
cp -r x2/opt/devkitpro/devkitARM/arm-none-eabi/lib/. "$TCHAIN/devkitARM/arm-none-eabi/lib/"
# verify the crt0 for the 3DS multilib actually landed:
find "$TCHAIN/devkitARM" -name 3dsx_crt0.o   # must print .../lib/armv6k/fpu/3dsx_crt0.o
"$TCHAIN/devkitARM/bin/arm-none-eabi-gcc" --version   # must print 10.2.0, release 55
```

* The **crtls** package provides `3dsx.specs`, `3dsx.ld` and
  `armv6k/fpu/3dsx_crt0.o` — without them linking fails with
  `cannot find 3dsx_crt0.o` (or `-specs=3dsx.specs` not found).
* ⚠ Pitfall P5: merging with `mv lib/* dest/lib/` silently *nests* or loses
  files when subdirs already exist. Always use `cp -r src/. dst/` and verify.

### 2.2 Host build tools

Three small devkitPro tools must be compiled from source into
`$TOOLS_BIN` (the makefiles find them on `PATH`):

| Tool | Source repo | Notes |
| --- | --- | --- |
| `3dsxtool`, `smdhtool` | `github.com/devkitPro/3dstools` | `3dsxtool` = ELF→3DSX; `smdhtool` = icon metadata |
| `bin2s` | `github.com/devkitPro/general-tools` | generates headers for embedded `.bin` data (fonts, touch maps) |
| `picasso` | `github.com/devkitPro/picasso` | PICA shader assembler for the `.shlist` pipeline |

```sh
git clone --depth 1 https://github.com/devkitPro/3dstools /tmp/3dstools
g++ -std=c++17 -O2 -o "$TOOLS_BIN/3dsxtool" /tmp/3dstools/src/3dsxtool.cpp \
    /tmp/3dstools/src/romfs.cpp /tmp/3dstools/src/lodepng/lodepng.cpp
g++ -std=c++17 -O2 -o "$TOOLS_BIN/smdhtool" /tmp/3dstools/src/smdhtool.cpp \
    /tmp/3dstools/src/lodepng/lodepng.cpp

git clone --depth 1 https://github.com/devkitPro/general-tools /tmp/gt
gcc -O2 -DPACKAGE_STRING='"bin2s 1.0"' -o "$TOOLS_BIN/bin2s" /tmp/gt/bin2s.c

git clone --depth 1 https://github.com/devkitPro/picasso /tmp/picasso
g++ -std=c++17 -O2 -DPACKAGE_STRING='"picasso 1.4"' \
    -o "$TOOLS_BIN/picasso" /tmp/picasso/source/*.cpp
```

(`-DPACKAGE_STRING` is required — both tools reference it unconditionally.)

### 2.3 Environment

```sh
export DEVKITPRO=$TCHAIN
export DEVKITARM=$TCHAIN/devkitARM
export PATH=$DEVKITARM/bin:$TOOLS_BIN:$PATH
```

`scripts/build.sh` honors these; `scripts/setup-game.sh` needs `rsync` (used
on non-WSL systems; on WSL prefer the robocopy method of §4).

---

## 3. Repo checks and one generated file

```sh
"$REPO/scripts/verify-layout.sh"     # all vendor symlink targets must resolve
```

The makefiles never define `g_GIT_SHA1` (upstream CMake generates it from
`src/extras/GitSHA1.cpp.in`). Generate it for **each** tree or linking fails
with `undefined reference to g_GIT_SHA1`:

```sh
SHA=$(git -C "$REPO" rev-parse --short HEAD)
for G in III miami stories; do
  printf '#define GIT_SHA1 "%s"\nconst char* g_GIT_SHA1 = GIT_SHA1;\n' "$SHA" \
    > "$REPO/$G/src/extras/GitSHA1.cpp"
done
```

---

## 4. Game data → SD layout

The runtime contract: every executable `chdir`s into its data folder and reads
everything relative to it — `/3ds/re3`, `/3ds/miami`, `/3ds/relcs`. Letter
case is documented (`audio` for III, `Audio` for VC, `AUDIO` for LCS); the
port opens files case-insensitively, but keep the documented case.

Replicate `scripts/setup-game.sh` behaviour:
**rsync everything except desktop binaries and junk, then apply the curated
overrides from `gamefiles/<game>`.**

### 4.1 Copy method (WSL)

⚠ Pitfall P1: **rsync writes to `/mnt/*` drvfs mounts fail with
`Operation not permitted` on every file** (`mkstemp` of its temp copies).
Plain `cp`, `touch` and Windows-native tools work fine. Two working options:

* **Preferred:** robocopy through Windows interop (below) — fast and exempt
  from the 9p layer.
* Alternative: run `setup-game.sh` with a **local staging dir** as
  destination, then `cp -r` staging → SD.

Exclude lists replicate the setup script exactly:

```sh
# GTA III (also Vice City — same list)
robocopy.exe "$(wslpath -w "$SRC_RE3")"  'F:\3ds\re3'  /E /R:2 /W:2 /NFL /NDL /NP \
  /XD userfiles 'userfiles_*' /XF '*.exe' '*.dll' '*.dmp' '*.log' .DS_Store '._*'
robocopy.exe "$(wslpath -w "$SRC_REVC")" 'F:\3ds\miami' /E /R:2 /W:2 /NFL /NDL /NP \
  /XD userfiles 'userfiles_*' /XF '*.exe' '*.dll' '*.dmp' '*.log' .DS_Store '._*'
```

Read-only PC files keep the Windows read-only attribute; `chmod` fails on
drvfs. Overwrite them with `cp -f` (it replaces the file) — verified working.

### 4.2 Per-game notes and overrides

**GTA III**

* The install may lack `audio/` streams (only `sfx.SDT`/`sfx.RAW` present).
  If so, copy the stream set from your backup over the same folder:
  `robocopy.exe "$(wslpath -w "$SRC_RE3_AUDIO")" 'F:\3ds\re3\audio' /E /R:2 /W:2`
* Required by the code: `models/gta3.img`+`.dir` (hardcoded), `data/gta3.dat`,
  `main.scm`, `paths/flight*.dat`, `paths/spath0.dat`, **`paths/tracks*.dat`**
  (GTA III *does* use trains — do not expect the VC situation), `anim/ped.ifp`,
  `anim/cuts.img`+`.dir`, `TEXT/american.gxt`, individual `txd/loadsc*.TXD`
  (there is no combined `LOADSCS.TXD` requirement).
* Overrides (`gamefiles/re3`): `TEXT/american.gxt`, `data/PARTICLE.CFG`,
  `data/main_d.scm`, `data/main_freeroam.scm`, `audio/music/PUSH_*.WAV`,
  `movies/*.3mv|*.pcm` (create `movies/` if absent — the 3DS build never reads
  the PC `.mpg`/`.bik` movies).

**Vice City**

* A stock install has everything, including all **9** radio `.adf` streams and
  the `audio/sfx.sdt`+`sfx.raw` pair. Validate `data/gta_vc.dat` exists.
* Overrides (`gamefiles/revc`): `TEXT/american.gxt`, `data/particle.cfg`,
  `Audio/music/SELF_FM.WAV`+`SELF_LOOP.WAV` (capital-A `Audio`), `movies/`.

**Liberty City Stories** — see next section; different pipeline.

### 4.3 LCS: PS2 → converted → 3DS

LCS has no PC version. Two conversion stages are required:

**Stage 1 — disc assets → reLCS format.** Run the official
[`reLCSAssetConverter.exe`](https://github.com/knackers4/res/releases/tag/relcs)
against your extracted PS2 disc. It is a **GUI wizard** (input: the extracted
disc folder or image; output: an empty target folder — it ignores CLI args).
Validate the result before continuing:

```text
DATA/gta_lcs.DAT          AUDIO/sfx.RAW + sfx.sdt
models/gta3.img           AUDIO/MUSIC/*.VB   (19 files)
                          AUDIO/NEWS/*.VB    AUDIO/CUTSCENE/*.VB
```

**Stage 2 — reLCS format → 3DS audio formats.** Copy the tree **without** the
VB streams and split banks, then convert:

```sh
# excludes replicate setup-game.sh: keep sfx bank, drop VB + SET banks
robocopy.exe "$(wslpath -w "$SRC_RELCS")" 'F:\3ds\relcs' /E /R:2 /W:2 /NFL /NDL /NP \
  /XD userfiles 'userfiles_*' SET0 SET1 SET2 SET3 SET4 SET5 SET6 \
  /XF '*.exe' '*.dll' '*.VB' '*.vb' .DS_Store '._*'

# ⚠ Pitfall P6: these scripts are '#!/bin/sh' but use bash's 'read -d'.
# Under dash they silently do NOTHING and still exit 0. Run with bash:
bash "$REPO/stories/tools/convert_lcs_music_adpcm_3ds.sh" \
     "$(wslpath "$SRC_RELCS")/AUDIO" /mnt/f/3ds/relcs/AUDIO
bash "$REPO/stories/tools/convert_lcs_streams_3ds.sh" \
     "$(wslpath "$SRC_RELCS")/AUDIO" /mnt/f/3ds/relcs/AUDIO
```

Expect `AUDIO/MUSIC/*.WAV` (19), `AUDIO/NEWS/*.MP3`, `AUDIO/CUTSCENE/*.MP3`.

Overrides (`gamefiles/relcs`): `AUDIO/MUSIC/CHASE_FM.WAV`+`CHASE_LOOP.WAV`,
`movies/*.3mv|*.pcm` (create the folder — converter output has none), and
`txd/LOADSC0.TXD` (`cp -f` — the converter ships its own copy).

### 4.4 Verify before building

```sh
# example: VC; repeat per game with its file list
for f in models/gta3.img models/gta3.dir data/gta_vc.dat data/main.scm \
         data/particle.cfg TEXT/american.gxt anim/ped.ifp anim/cuts.img \
         audio/sfx.SDT audio/sfx.RAW Audio/music/SELF_FM.WAV \
         movies/Logo.3mv movies/Logo.pcm; do
  [ -f "$SD/miami/$f" ] && echo "OK   $f" || echo "MISS $f"
done
```

---

## 5. Builds

```sh
"$REPO/scripts/build.sh" re3      # → III/build/re3.elf + re3.3dsx
"$REPO/scripts/build.sh" revc     # → miami/build/miami.elf + miami.3dsx (OPTIMIZED_BUILD=1)
"$REPO/scripts/build.sh" relcs    # → stories/build/relcs.elf + relcs.3dsx
```

Clean rebuild (per the main README — never mix object sets):

```sh
export DEVKITPRO=$TCHAIN DEVKITARM=$DEVKITARM PATH=$DEVKITARM/bin:$TOOLS_BIN:$PATH
make -C "$REPO/miami/build" -f GNUmakefile clean    # ⚠ needs DEVKITARM exported,
make -C "$REPO/III/build"   -f GNUmakefile clean    #   else: 'DEVKITARM not in
make -C "$REPO/stories/build" -f GNUmakefile clean  #   your environment'
"$REPO/scripts/build.sh" revc                        # then rebuild
```

* First build populates `obj/` from nothing — expect several minutes per game.
* Build noise that is **not** an error: `-fstrict-aliasing` notes in vendored
  libctru `ndsp.c`; `Ignoring assembly file` messages.
* The `%.bin.o` and `%.shbin.o` rules mask tool failures through pipes
  (`bin2s … | as`). If a data/shader header is ever reported missing
  (`default_font_bin.h`, `bottom_touch_bin.h`, `VSH_FVEC_*`), delete the
  matching stale objects — `find <game>/build/obj -name '*.bin.o*' -delete`
  and `-name '*.shbin*' -o -name '*_shbin.h'` — and rebuild.

---

## 6. CIA packaging

### 6.1 Tools

```sh
# makerom — official Project_CTR release
curl -LO https://github.com/3DSGuy/Project_CTR/releases/download/makerom-v0.19.0/makerom-v0.19.0-ubuntu_x86_64.zip
# bannertool — original Steveice10 repo is gone; the diasurgical fork ships
# the same v1.2.0 with Linux binaries in its release zip
curl -LO https://github.com/diasurgical/bannertool/releases/download/1.2.0/bannertool.zip
sudo cp makerom bannertool /usr/local/bin/ && sudo chmod +x /usr/local/bin/{makerom,bannertool}
# 3dsxtool — already at $TOOLS_BIN (§2.2), which is on PATH
```

### 6.2 Per-game packaging

`packaging/production_cia/build_production.sh` packages **all three** games and
requires all three ELFs + `zsh`. For a single game, run the same steps it
does, with `$OUTDIR` as the output folder:

```sh
A="$REPO/packaging/prebuilt"; O="$OUTDIR"; mkdir -p "$O"

# ---- GTA III ----
bannertool makebanner -ci $A/re3.cgfx   -ca $A/re3.bcwav   -o $O/re3.bnr
bannertool makesmdh -s 'GTA3 For Nintendo 3DS' -l 'Grand Theft Auto III' \
  -p 'Epic' -i $A/re3-icon.png -f visible,extendedbanner -r regionfree -o $O/gta3.smdh
3dsxtool "$REPO/III/build/re3.elf" "$O/GTA3 For Nintendo 3DS.3dsx" --smdh=$O/gta3.smdh
makerom -f cia -o "$O/GTA3 For Nintendo 3DS.cia" \
  -rsf "$REPO/packaging/production_cia/container.rsf" -target t \
  -elf "$REPO/III/build/re3.elf" -icon $O/gta3.smdh -banner $O/re3.bnr \
  -DAPP_TITLE='GTA3 For' -DAPP_PRODUCT_CODE='CTR-P-0RE3' -DAPP_UNIQUE_ID=0x2F60

# ---- Vice City ----  (miami.elf; strings 'GTAVC Fo'/'CTR-P-REVC'/0x2F61)
# ---- Liberty City Stories ----  (relcs.elf; 'GTALCS F'/'CTR-P-RLCS'/0x2F62)
```

| Game | Title ID | Product code | Banner asset |
| --- | --- | --- | --- |
| GTA III | `00040000002F6000` | `CTR-P-0RE3` | `re3.cgfx` + `re3.bcwav` + `re3-icon.png` |
| Vice City | `00040000002F6100` | `CTR-P-REVC` | `revc.*` |
| LCS | `00040000002F6200` | `CTR-P-RLCS` | `relcs.*` |

### 6.3 Verifying a CIA

Check the title ID bytes appear in both the ticket and the TMD (offsets are
stable for makerom output):

```sh
python3 - <<'EOF'
import struct, sys
d = open(sys.argv[1], 'rb').read()
tid = 0x00040000002F6100   # ← expected title ID
needle = struct.pack('>Q', tid)
hits = [hex(i) for i in range(len(d)) if d[i:i+8] == needle][:3]
print('title id bytes at:', hits)
assert len(hits) >= 2, 'ticket+TMD must both carry the title id'
EOF
```

⚠ Two CIAs from identical inputs have **different SHA-256 hashes** — makerom
randomizes signatures per run. Compare sizes and title IDs, not hashes.

---

## 7. Delivery to the console

The console reads data from `/3ds/re3`, `/3ds/miami`, `/3ds/relcs`; CIAs are
installed from anywhere on the card (conventionally `/3ds/cias`). CIA and 3DSX
builds share the same folders and saves.

### 7.0 Naming deployed builds

Postfix every deployed executable and CIA with its **build** date-time so
consecutive test builds stay distinguishable on the card. Take the stamp from
the artifact's mtime, not the upload time:

```sh
TS=$(date -r "$REPO/miami/build/miami.3dsx" +%Y%m%d-%H%M)
# → deploy as revc-$TS.3dsx, e.g. revc-20261001-1725.3dsx
TSC=$(date -r "$OUTDIR/GTAVC For Nintendo 3DS.cia" +%Y%m%d-%H%M)
# → deploy as "GTAVC For Nintendo 3DS-$TSC.cia"
```

Upload the postfixed names, then delete the previous build's files: two CIAs
with the same title ID both appear in FBI, and stale 3DSX files accumulate.
Both names are free-form — the Homebrew Launcher lists whatever `*.3dsx` it
finds, and FBI reads the title from the CIA metadata. In the examples below,
substitute the postfixed names for `revc.3dsx` and
`GTAVC For Nintendo 3DS.cia`.

### 7.1 Direct SD-card access

```sh
wsl.exe -u root -e sh -c 'mkdir -p /mnt/f && mount -t drvfs F: /mnt/f'
"$REPO/scripts/install-3dsx.sh" revc /mnt/f/3ds     # writes /3ds/revc.3dsx
cp "$OUTDIR/GTAVC For Nintendo 3DS.cia" /mnt/f/3ds/cias/
cmp "$REPO/miami/build/miami.3dsx" /mnt/f/3ds/revc.3dsx     # verify copy
sync
wsl.exe -u root -e sh -c 'umount /mnt/f'             # before ejecting
```

(`install-3dsx.sh` renames `miami.3dsx`→`revc.3dsx` etc.; its `cp -p` prints a
harmless `preserving times` warning on drvfs.)

### 7.2 FTP (no card removal)

With the 3DS running `ftpd` at `$FTP_HOST` (anonymous, passive mode), point
your data-prep commands (§4) at a **local staging folder** instead of the card
— e.g. `STAGING=$HOME/sdcard` used in place of `/mnt/f/3ds` throughout — then
push everything:

```sh
# single files (build artifacts — quick):
curl --ftp-pasv -T "$OUTDIR/GTAVC For Nintendo 3DS.cia" "ftp://$FTP_HOST/3ds/cias/"
curl --ftp-pasv -T "$REPO/miami/build/miami.3dsx" "ftp://$FTP_HOST/3ds/revc.3dsx"

# whole trees (recursive mirror upload):
lftp -u anonymous, -e "set ftp:passive-mode on; \
  mirror -R $STAGING/re3   /3ds/re3;   \
  mirror -R $STAGING/miami /3ds/miami; \
  mirror -R $STAGING/relcs /3ds/relcs; \
  bye" $FTP_HOST
```

* Speed is Wi-Fi-bound (roughly 1 MB/s) — budget time for ~4 GB of data.
  The small per-game build artifacts (`.3dsx`, `.cia`) are quick; bulk data
  folders are the slow part.
* `mirror -R` uploads only new/changed files, so re-runs are incremental.
* Filenames with spaces (all three CIA names have them): upload to the
  **directory** URL and let curl use the local basename, e.g.
  `curl --ftp-pasv -T "$OUTDIR/GTAVC For Nintendo 3DS.cia" "ftp://$FTP_HOST/3ds/cias/"`.
  Appending the name to the URL needs `%20` encoding; curl decodes it before
  sending, so the server stores the real space-named file (no stray `%20`
  file is created — this was verified against ftpd). Note the directory-URL
  trick can only upload the **local basename** — when the remote name must
  differ (the §7.0 date postfix), you have to use the `%20`-encoded full URL.
  `lftp put -O` also handles spaces natively. Verify results with
  `lftp ls -l` of the directory itself (not a globbed path — P16), since
  curl's bare listing omits sizes and dates. For **single-file** size checks,
  `curl -sI --ftp-pasv "ftp://$FTP_HOST/<path>"` works against the 3DS ftpd
  and reports the size as `Content-Length` — the easiest way to diff deployed
  data overrides against local `stat` sizes in a loop.
* When only `src/` changed (no `gamefiles/` commit), redeploying just the
  executable is enough — the SD data overrides have never changed since the
  repository import, and the console's startup integrity check will reject
  the launch if they ever go stale.
* Verified working: `lftp … mirror -R …` + `curl -T` against `ftpd` on port
  5000. Keep `ftp:passive-mode on` — most 3DS servers are passive-only.

---

## 8. Pitfall index (symptom → cause → fix)

| # | Symptom | Cause | Fix |
| --- | --- | --- | --- |
| P1 | rsync to `/mnt/*`: `mkstemp … Operation not permitted` on every file | WSL 9p/drvfs rejects rsync's temp-file pattern (background copies especially) | Use robocopy via interop, or stage locally then `cp -r` |
| P2 | `make: *** "DEVKITARM not in your environment"` even for `clean` | clean target also includes the makefile's env check | Export `DEVKITPRO`/`DEVKITARM`/`PATH` first |
| P3 | `reVC requires bundled devkitARM r55: <repo>/toolchains/devkitARM` | toolchain placed elsewhere (e.g. `miami/toolchains`) | Path is relative to each game's `build/` → repo root `/toolchains/devkitARM` |
| P4 | link: `cannot find 3dsx_crt0.o` / `-specs=3dsx.specs` missing | crtls package not installed, or lost in a `mv` merge | Re-extract crtls, `cp -r lib/. …`, verify `find -name 3dsx_crt0.o` |
| P5 | silently missing files after merging toolchain dirs | `mv src/* dst/` does not merge existing subdirs reliably | Use `cp -r src/. dst/`; verify key files afterwards |
| P6 | LCS audio conversion produces nothing, exits 0 | scripts are `#!/bin/sh` but use bash-only `read -d` | Run with `bash script.sh …` |
| P7 | `undefined reference to g_GIT_SHA1` | `GitSHA1.cpp` is CMake-generated; makefiles don't | Generate for each tree (§3) |
| P8 | `default_font_bin.h` / `bottom_touch_bin.h` / `VSH_FVEC_*` not declared | `bin2s`/`picasso` were absent when `.bin.o`/`.shbin.o` "built" (pipe masks failure) | Install tools, delete stale `*.bin.o*`, `*.shbin*`, rebuild |
| P9 | `cp: cannot create regular file … Permission denied` on overwrite | PC files carry the Windows read-only attribute; `chmod` fails on drvfs | `cp -f` (replaces instead of opening for write) |
| P10 | `GetWindowText` returns empty for a GUI field you just set | cross-process reads need `WM_GETTEXT` via `SendMessage` | Use `SendMessage(h, 0x000D, …)` (relevant when driving GUI converters) |
| P11 | CIA hashes differ between identical builds | makerom randomizes signatures | Expected — compare size + title ID instead |
| P12 | `GetTouchInputInfo`-style strings, `Next >>` wizard | `reLCSAssetConverter.exe` is a GUI installer, ignores argv | Run it interactively; validate its output folder (§4.3) |
| P13 | `md5sum: command not found` during make | fingerprint step needs GNU coreutils | `apt-get install coreutils` (present on Ubuntu by default) |
| P14 | `curl: (3) URL using bad/illegal format` on an FTP upload | curl parses the URL before connecting; literal spaces are illegal in it | Upload to the directory URL and let curl take the local basename (§7.2), or percent-encode spaces as `%20` (curl decodes the path before sending) |
| P15 | "REGTA requires devkitARM r55" immediately after "setting" the env | one-line `export REPO=… TCHAIN=$REPO/toolchains …` — `$REPO` is expanded before its own assignment takes effect, so `DEVKITARM` becomes `/devkitARM` and the version probe runs a nonexistent binary | Put each assignment on its own line (as in §0), or use absolute paths in the same line |
| P16 | `lftp … ls -l /3ds/cias/GTA3*`: `550 No such file or directory` even though the file is there | the 3DS ftpd does not expand shell globs in LIST arguments — the literal `*` is sent to the server | List the directory itself (`ls -l /3ds/cias`) and eyeball/grep the output locally |

## 9. First launch (per game)

* Long first load is **normal** — the console generates the native texture
  cache (`models/txd.img` + `txd.dir`) and capability data; do not interrupt.
* Launch CIA from HOME Menu, 3DSX via the Homebrew Launcher; saves are shared.
* Install CIAs with FBI (`/3ds/cias`). Custom firmware required for CIAs;
  New 3DS family hardware required for all three games.
