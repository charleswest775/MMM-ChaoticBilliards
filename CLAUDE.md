# MMM-ChaoticBilliards — context for Claude sessions

Charles's MagicMirror² module: chaotic billiards, an elliptical table above Bunimovich's stadium,
for his hallway mirror, as one page in a rotation of pages. Split out of MMM-ChaosTheory on
2026-09-28 with its history (it was the `billiards` simulation of that module's chaos page).

## Files

- `MMM-ChaoticBilliards.js` — module shell: one canvas plus an HTML caption (equations + live
  readout, updated 2×/s). A new start every `cycleSeconds` and on each `resume()`. Loop:
  `setTimeout` until a frame is due, then one `requestAnimationFrame`. `suspend()` stops it; a
  sim with `resting = true` is polled only every 500 ms (billiards never rests); while
  MagicMirror fades the module out (`hidden` is set at the start, `suspend()` comes after),
  frames draw nothing. `turns: { of, at }`: only every nth showing; otherwise the wrapper gets
  `display: none` and nothing starts.
- `simulations/billiards.js` — the `billiards` sim on `window.BilliardsSimulations`. Ellipse
  (semi-axes 1, ½) above a stadium of the same size (straight sides 1, ends of radius ½); three
  balls in each from the same point, 10⁻⁶ rad apart, in a random direction. Bounces found
  exactly (line–wall intersection, reflect in the normal). Long exposure with `lighter`
  compositing; each frame draws one table's new paths only, in turn, so a frame's changes stay
  in one half of the canvas.
- `simulations/common.js` (`window.BilliardsCommon`) — shared helpers (`sci` for the readout).
- UMD-style, so the physics runs in Node: `tests/billiards.test.js` (`node --test`, no
  dependencies): reflection law and unit speed on both tables, the ellipse's invariant (product
  of the angular momenta about the foci), stadium balls parting while the ellipse's don't, one
  stadium ball reaching every part of the table.
- `node_helper.js` — the stats panel (`statsPanel: true`): CPU of Electron and cage, per core,
  temperature, from `/proc`, only while shown.
- `dev/preview.html` — runs the module in a desktop browser (`python3 -m http.server` in the repo,
  then `/dev/preview.html?fps=30`).

MMM-ChaosTheory has the same simulation (`billiards`, among its eight): a fix to the physics or
the drawing belongs in both.

The shell (`MMM-ChaoticBilliards.js`, `node_helper.js`'s stats panel, `dev/preview.html`) is
shared in spirit with the sibling modules (MMM-ChaosTheory, MMM-LorenzAttractor,
MMM-DoublePendulum, MMM-FractalBasins, MMM-LogisticMap, MMM-SymmetricIcons, MMM-ThreeBody,
MMM-Rule30, MMM-Atom, MMM-FractalZoom, MMM-Chladni, MMM-SacredGeometry, MMM-Tilings,
MMM-PlanetsDance, MMM-SnowCrystal, MMM-NightSky, MMM-PhotoDeck): a fix there probably belongs in
the siblings too.

## Measured cost on the Pi

900², 20 fps, Electron + cage, over a 60 s showing (measured in MMM-ChaosTheory): 58% of a core,
22 fps achieved. Hidden: 0.3% (baseline 0.2%).

## Performance findings on the Pi (measured)

- A frame that changes the canvas costs ~2%/fps fixed; beyond that, cost scales with the
  **bounding box of everything changed in the frame**. Full redraws of a 900² canvas at 20 fps
  saturate the pipeline (~150%). JS is never the bottleneck (<3 ms/frame).
- So: draw incrementally (long-exposure trails), keep each frame's changes spatially compact,
  and rest when the picture is static. Line width, opacity, `rAF` vs timer made no difference.
- MagicMirror applies `electronSwitches` after app ready, so `remote-debugging-port` can't be set
  that way; use `debugStats: true` and a `grim` screenshot to see fps on the Pi.

## Hard constraints: the target device

- **Raspberry Pi 3 B+, 905 MB RAM, 64-bit Debian 13.** Mirror runs Electron 42 in a cage
  Wayland kiosk.
- **No GPU acceleration, and it can't be enabled**: the Pi 3's VideoCore IV only does GLES 2.0,
  Chromium needs ES 3.0 (tested). All canvas drawing is CPU. **No WebGL / three.js.**
- Screen will be **portrait 1200×1920** once mounted (Dell U2413, rotated). Design for portrait.
- Electron baseline is ~0.5% of one core. **Measure, don't guess**: on the Pi,
  `~/.cache/mm-sample.sh 60` prints Electron CPU% and RSS over 60 s. Record before/after numbers
  in the README.
- The mirror rotates pages every 15-30 s (MMM-pages, which hides/shows modules). `suspend()` and
  `resume()` must fire on page changes, or the loop burns CPU 24/7.

## Deploying and testing

- This repo is public so the Pi can `git clone`/`git pull` without credentials. It is meant for
  modules.magicmirror.builders: keep `screenshot.png` and the README's Installation / Update /
  Configuration sections, which the list's checks look for.
- Pi access: `ssh fatherson@raspberrypi.local` (key auth). Module path:
  `~/MagicMirror/modules/MMM-ChaoticBilliards`. Restart: `pm2 restart MagicMirror`
  (pm2 is in `~/.npm-global/bin`). Logs: `pm2 logs MagicMirror`.
- The mirror's **config.js lives in a separate private repo**, `charleswest775/magicmirror-setup`
  (cloned at `~/dev/magicmirror-setup`). Add the module's config block there, then
  `./deploy.sh diff` and `./deploy.sh push` (push validates config before restarting).
  Don't hand-edit config.js on the Pi without `./deploy.sh pull` afterwards.
- Faster iteration: run it in a desktop browser (`dev/preview.html`), then confirm performance
  on the Pi.
- Commit as Charles's GitHub noreply address (set in this repo's git config).
