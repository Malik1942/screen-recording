# Screen-Recording Capture Protocol

Records product footage from a user-provided script/shot list — **native apps (iOS Simulator), websites, or Figma prototypes** — then derives auto-zoom camera plans and cursor navigation from measured interaction times. Compositing tools live in the companion [product-film](https://github.com/Malik1942/product-film) skill's `scripts/` — this skill produces their inputs.

The script the user provides must pin down, per scene: the start state, the interaction to show, the payoff to read, and the data that must exist (product-film's `references/writing-the-script.md` has the full six-block template). If it doesn't, get that first.

## 0. Pick the capture surface

| Surface | Drive with | Record with | Composite as |
|---|---|---|---|
| Native iOS app | simctl + simulator taps | `simctl io recordVideo` | device mockup (mobile path) |
| Desktop website / desktop app | your agent's browser automation (navigate, locate targets, click, fill forms) — or AppleScript-driven Chrome | `screencapture -v` region capture of the viewport | `FILL=1` full-bleed (desktop path) |
| Figma prototype | open the `figma.com/proto/...` link in the driven browser; drive hotspots as normal clicks | same `screencapture -v` route | phone-shaped proto → device frame or `FRAMELESS=1` card (mobile path); landscape proto → `FILL=1` (desktop path) |

**The path fork is decided by form factor:** portrait/phone content → mobile path; landscape/desktop content → desktop path. The desktop path's composite cover-crops the capture to fill the canvas, so **desktop captures MUST match the canvas aspect** (16:9 for 2560×1440): size the browser window so the recorded viewport region is exactly 16:9 (e.g. 2560×1440 or 2048×1152 pixels) and tighten `-R` to the viewport, excluding ALL browser chrome. A tall or odd-aspect capture forces the cover-crop to eat UI at the edges. Verify the region with a single `screencapture -R` still before rolling.

**Website / Figma-proto recording mechanics:**
```bash
# window bounds of the target app/window (points; multiply by backingScaleFactor for pixels)
osascript -e 'tell application "System Events" to get {position, size} of front window of (first process whose frontmost is true)'
# region recording — SIGINT to stop, same discipline as simctl
screencapture -v -R "x,y,w,h" ~/Desktop/<proj>/takeN.mov & echo $! > /tmp/rec.pid
```
- `screencapture -v` needs the host terminal to have Screen Recording permission (System Settings → Privacy) — if the file comes out black, that's why; the user must grant it once.
- **Resolution maths (verify BEFORE rolling):** `screencapture -R` takes POINTS; native pixels = points × `devicePixelRatio`. On a 2x HiDPI display a 1280×720-point viewport captures at exactly 2560×1440 — zero scaling, zero crop. This REQUIRES the display in HiDPI mode; if it drops to 1x (dpr 1) true 2560×1440 capture is impossible. Check `system_profiler SPDisplaysDataType` and the page's `devicePixelRatio` first. Verify the region by injecting a full-viewport solid-colour overlay and locating its largest solid run in a still — the raw bbox alone is inflated by stray same-colour pixels.
- Crop browser chrome either by tightening the -R region to the viewport or later via composite trim (`cropTop`).
- **Decide the pointer question before rolling.** `screencapture -v` records the real macOS cursor, and some sites draw their own custom cursor on top. Either park the OS pointer outside the region (synthetic browser clicks don't move it) and let the compositor supply a synthetic one, OR accept the recorded pointer and composite with `CURSOR_STYLE=none` so only click rings are added. Two pointers on screen is an instant reject — check a frame of the take and pick one.
- Figma prototypes: set the proto to "Fit" scaling, wait for the flow's first frame to fully load before rolling, and note that hotspot hints flash on stray clicks — a stray flash contaminates the take (discard, re-roll).

Set `DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer` for all simctl calls (works without full xcode-select).

## 1. Parse the script into a shot list

For each scene extract: start screen, ordered actions (tap/swipe/type + target), required end state, and which moments the viewer must *read* (those get zoom holds). If any scene needs data that doesn't exist in the app (populated lists, images, links, old timestamps), seed it FIRST — through the real UI when possible, sqlite3 store surgery when not.

## 2. Stage the surface (all of it, before the first take)

**Websites:** clean profile state — dismiss cookie/consent banners off-camera, log in beforehand, kill toasts/onboarding tooltips, set viewport to the script's aspect (resize_window), disable animations-reducing OS settings if they fight the footage. **Figma protos:** presentation mode, correct starting frame, "Fit" zoom.

**Simulator:**

```bash
U=<udid>
xcrun simctl boot $U
xcrun simctl spawn $U defaults write .GlobalPreferences AppleLanguages -array "en-US"
xcrun simctl spawn $U defaults write .GlobalPreferences AppleLocale "en_US"
xcrun simctl shutdown $U && xcrun simctl boot $U          # locale needs reboot
xcrun simctl status_bar $U override --time "9:41" --batteryState charged --batteryLevel 100 --cellularBars 4 --wifiBars 3
xcrun simctl privacy $U grant all <bundle-id>             # KILLS a running app — do before launch
```
Then launch the app once and burn through every one-time surprise OFF-camera: mic/speech permission dialogs, rating prompts, first-run sheets. Only then record.

## 3. Take protocol (per scene)

1. **Dry-run the flow once** without recording: screenshot each screen, note tap targets in points (screenshot px ÷ scale; e.g. 1206-wide capture on a 402pt device → ÷3... use the image's stated multiplier). A cheap dry run beats a garbage take.
2. Navigate to the scene's start state. Relaunching resets tabs/segments — re-verify with a screenshot; don't assume.
3. Roll detached, into a DURABLE directory (never /tmp — it gets wiped):
   ```bash
   nohup xcrun simctl io $U recordVideo --codec h264 --force ~/Desktop/<proj>/takeN.mp4 > /tmp/rec.log 2>&1 &
   echo $! > /tmp/rec.pid
   ```
4. Drive the actions with deliberate pacing (1.5–3s holds where the viewer must read). Text fields: tap to focus, screenshot to confirm the cursor, then type — typing into an unfocused field silently no-ops. Never type HTML entities.
5. Stop with `kill -INT $(cat /tmp/rec.pid)`; wait 2s for finalize; check the file is >0 bytes.
6. **Verify the end frame** with a screenshot before moving on. A take is not done until its last frame is confirmed.

If ANY dialog/toast appears mid-take: stop, discard, dismiss/fix the cause, re-roll. Contaminated takes are never salvaged in the edit.

## 4. Measure the take

Run product-film's `diffscan.swift` (mobile) or **`diffscan2.swift` (desktop — aspect-correct, and it reports the centroid of each change so you get the responding region as fx/fy)** on the take → real screen-change windows. Tap-injection latency through any driving tool is unpredictable (a planned 15s take can run 60s with 13s holds); planned timings are fiction. The transitions plus your action order give each interaction's true time T and target (fx, fy = fraction of screen width/height from top-left).

## 5. Auto-zoom + cursor derivation

Rule (encoded in `scripts/autoplan.py`): for each interaction at time T targeting (fx, fy):

- **Camera (mobile)**: wide (scale 1.02) until T−1.2 → punch to (fx, fy, scale 1.35–1.5) arriving T−0.3 → hold through the response until the next beat → move to next target or return wide. Camera moves ONLY around interactions and readable payoffs; no idle drift.
- **Camera (desktop, `--desktop` → FILL=1)**: wide = exactly 1.0 (the page cover-fills the canvas, edge-flush); punches are deeper (1.5–1.8, up to 2.2 for small controls) and always expand toward the cursor's interaction point; consecutive targets < ~0.35 apart get a constant-zoom follow-pan instead of a wide return (Screen Studio grammar). The composite clamps every move to content bounds — the page fills the canvas in every frame, so corner targets are safe to request.
- **Cursor**: fade in at T−1.4 at the previous point (or screen center for the first), eased glide arriving T−0.25, press-dip, ripple at T−0.15. **The ripple must complete before the UI transition starts.** Mobile = touch dot; desktop = arrow + click ring (automatic under FILL=1).

```bash
python3 scripts/autoplan.py taps.json             # mobile: device-mockup camera grammar
python3 scripts/autoplan.py taps.json --desktop   # desktop: FILL=1 full-bleed grammar
# taps.json: [{"t": 12.6, "fx": 0.27, "fy": 0.80}, ...]
# → prints the composite2.swift invocation (camera plan string + cursor/tap args)
```
Feed the output to product-film's `composite2.swift` (with the trim you chose from diffscan). For a scene with several clicks, pass them all as `CUES="t fx fy; ..."` — one persistent cursor glides between every target, which is what a real pointer does; a drag is its handle-start cue plus a `click=0` waypoint at the end position.

## 6. Failure modes (each cost a real retake)

| Symptom | Cause |
|---|---|
| Take shows a permission dialog / "Enjoying app?" prompt | Staging skipped; one-time surprises not burned off-camera |
| Typed text never appears | Field wasn't focused (no cursor) — or text was sent to the wrong booted device (pass udid explicitly) |
| Take is 0 bytes | recordVideo finalize needs SIGINT + a sandbox-writable path |
| Take 4× longer than planned | Tap latency — harmless IF you measure; fatal if you trim by plan |
| UI in wrong language | Locale set without reboot, or fresh install re-seeded demo data |
| App on wrong tab/segment after relaunch | Launch state resets — verify with screenshot before rolling |
| Ripple/zoom lands on the wrong screen | Cue timed after the transition started — cues complete BEFORE T |
| Desktop film shows black margins / floating card / UI eaten at edges | Capture wasn't canvas-aspect, or mobile card grammar used on landscape content — 16:9 viewport region + `FILL=1` + `autoplan.py --desktop` |
