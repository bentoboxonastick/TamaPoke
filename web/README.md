# TamaPoke web installer

A one-click page that flashes the firmware and loads the sprites from the browser
(Chrome/Edge), with no Arduino or drivers. It uses
[ESP Web Tools](https://esphome.github.io/esp-web-tools/) to flash and **Web
Serial** to push the sprites to the SD with the firmware's `PUT` protocol (the
same one as `tools/send_sd.py`).

## Contents

- `index.html` — the page (flashing + sprite loader).
- `manifest.json` — ESP Web Tools config (points at the firmware).
- `firmware/bootloader.bin` `partitions.bin` `boot_app0.bin` `app.bin` — what the
  installer actually writes, each at its own offset. **This is deliberate**: a
  single image at `0x0` pads the gaps with `0xFF` and so blanks the NVS
  partition at `0x9000`, which is the player's save. `tools/check_installer.py`
  fails the build if anything the manifest ships would land on it.
- `firmware/tamapoke.bin` — the same four merged into one image for flashing a
  **blank** board from the command line. Not in the manifest, because installing
  it over an existing game erases the pet.
- `sprites-kanto.pak`, `sprites-johto.pak`, `sprites-hoenn.pak`,
  `sprites-sinnoh.pak` — the sprites bundled (TPAK) **one file per region**, so
  the page sends a region in one click. **Generated** by
  `tools/pack_bundle.py`, which derives the region list from `dex_data.py`, and
  **committed** — see *Hosting the sprites* below for why they have to be.

## Regenerate

After changing the firmware or the sprites:

```bash
bash tools/build_web.sh        # recompiles -> firmware/tamapoke.bin AND rebuilds the region .paks
```

## Test locally

Web Serial and ESP Web Tools need a **secure context**: `https://` or
`http://localhost`. To test:

```bash
cd web && python3 -m http.server 8000
# open http://localhost:8000 in Chrome/Edge
```

## End-user flow

1. **Install TamaPoke** → flashes the firmware (pick the USB port; tick "Erase
   device" for a fresh board).
2. **Connect board** + a region button → downloads that region's `.pak` and copies it to
   the microSD over USB (progress bar, ~8–10 min). Close the step-1 install tab
   first: only one program can use the port at a time.
3. Restart (PWR button) → choose your starter and play.

A hidden "pick them manually" option lets advanced users send their own `.bin`.

## Hosting the sprites

**The `.pak` files ARE committed, and that is deliberate.** They have to be
served **same-origin** from GitHub Pages, because **GitHub release assets send
no CORS headers at all** — a browser `fetch()` of one is blocked, however much
nicer it would be to keep 140 MB out of the repo. Verified with an `Origin`
header: the asset returns `200` and no `access-control-allow-origin`, while
Pages sends `access-control-allow-origin: *`.

This page said the opposite for a long time — "gitignored", "not committed",
"serves Access-Control-Allow-Origin" — while `.gitignore` carried the real
reason and the files were tracked. Acting on this file rather than on the code
untracks them and silently breaks every download button. It is written down
here now so the next person does not have to find out the same way.

```bash
bash tools/build_web.sh   # rebuilds the firmware, the manifest and every .pak
git add web/sprites-*.pak # yes, really
```

The page tries **same-origin first**, then `PAK_RELEASE`. Same-origin is the
path that actually works in a browser; the release fallback is a convenience for
people downloading a bundle by hand, and for local testing you can leave the
`.pak` files in `web/` and run `python3 -m http.server`.

`PAK_RELEASE` at the top of the script block in `index.html` points at the repo;
change it if you fork.

**Why one file per region and not one big one:** all four together come to about
140 MB, and GitHub's hard per-file limit is 100 MB — a single bundle would be
uncommittable. Splitting also means most people can take Kanto (~40 MB) and
stop, and add a region later without re-sending what is already on the card.
A region whose `.pak` is not on the card shows as locked in the Pokedex chooser
and is kept out of the egg pool, so a partial install is a supported state
rather than a broken one.

All sprites are from PMD SpriteCollab, CC BY-NC (non-commercial sharing with
attribution is allowed); see [`../CREDITS.md`](../CREDITS.md).

## Deploy (GitHub Pages)

### Option A: GitHub Actions (Recommended)

1. Repo settings → **Pages** → **Build and deployment** → Source: select **GitHub Actions**.
2. Push to `main` or manually trigger the **Deploy Web Installer to GitHub Pages** workflow under the **Actions** tab.
3. The workflow automatically:
   - Sets up `arduino-cli` with the ESP32 core and required libraries (`GFX Library for Arduino`, `SensorLib`, `XPowersLib`).
   - Runs `tools/build_web.sh` to compile `TamaPoke.ino` and regenerate the binaries in `web/firmware/` (`app.bin`, `bootloader.bin`, `partitions.bin`, `boot_app0.bin`, `tamapoke.bin`).
   - Validates that the manifest cannot overwrite NVS save memory (`tools/check_installer.py`).
   - Deploys the freshly built installer directly to GitHub Pages at `https://<user>.github.io/<repo>/`.

> **Note:** Because the action compiles the firmware automatically in CI, you do not need to install `arduino-cli` locally or manually commit recompiled `.bin` files when updating sketch code.

### Option B: Deploy from branch

1. Repo settings → **Pages** → Source: **Deploy from a branch** → Branch: `main`, folder `/web`.
2. Pages serves `/web` directly and provides HTTPS automatically.
   *(Note: This serves whatever static binaries are pre-committed in `web/firmware/`; it will not compile sketch updates automatically).*

> **Pages on private repos** needs GitHub Pro/Team. If you make the repo
> **public** to use Pages for free, decide about the sprites first (see above and
> CREDITS).

## Serial Debugging & Diagnostics

The firmware provides real-time logging and debug commands over USB Serial (115200 baud), accessible via Chrome/Edge Web Serial, the Arduino IDE Serial Monitor, or `arduino-cli monitor`:

- **Boot & Progress Logs**:
  - `[BADGE] Loaded: Kanto=0x... Johto=0x...` logs all regional badge masks restored from NVS on boot.
  - `[BADGE] Win: rg=<n> (<Region>) idx=<n> hard=<0|1> mask=0x...` logs when a gym badge is awarded.
  - `[BATTLE] Won against trainer ...` confirms badge awards or notes already-owned badges upon victory.

- **Helpful Console Commands**:
  - `STATS` — Dumps full pet stats (species, level, care, IVs, training) and badge bitmasks across all regions.
  - `BADGES` — Displays current easy and hard mode badge bitmasks for every region.
  - `BADGE <rg> <idx> [hard]` — Manually awards a badge (e.g. `BADGE 0 0` for Brock) to verify persistence across reboots without battling.
  - `EXPORT` / `IMPORT` — Dumps and restores full NVS save backups.

## Limitations

- Desktop **Chrome/Edge** only (Web Serial isn't in Firefox/Safari).
