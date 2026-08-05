---
name: screen-recording
version: 1.0.0
description: Use when screen-recording a product demo from a script or shot list — a native app in the iOS Simulator, a website in the browser, or a Figma prototype — or when footage needs auto-zoom on interactions and a visible cursor navigating the UI. Also when takes come out contaminated (permission dialogs, wrong language, wrong tab), empty (0 bytes), or mysteriously longer than planned.
---

# screen-recording

Script-driven product capture — **iOS Simulator, website, or Figma prototype** — with auto-zoom and cursor navigation **derived from measured interaction times**, never hand-guessed. The synthetic cursor comes from the compositor, so it works identically on every surface.

**The user's script must specify, per scene: start state, the interaction to show, the payoff to read, and the data that must exist.** If it doesn't, get those four fields per scene before capturing anything (the companion product-film skill ships a full fill-in template in its `references/writing-the-script.md`).

**Read `references/capture-protocol.md` before rolling anything.** It has the staging checklist, the per-take protocol, and the derivation rule.

**COMPANION for editing/compositing:** the [product-film](https://github.com/Malik1942/product-film) skill supplies the downstream tools (`diffscan.swift`/`diffscan2.swift` for measuring takes, `composite2.swift` for rendering zoom + cursor). This skill produces their inputs; capture itself works standalone.

## The loop

1. Parse the user's script → shot list (start state, actions, end state, readable moments). Seed any missing product data first.
2. Pick the capture surface (protocol §0: simulator / website / Figma proto — each has its drive+record pair) AND the presentation path by form factor: portrait/phone content → mobile path (mockup, dot cursor); landscape/desktop content → desktop path (`FILL=1` full-bleed, arrow cursor, **capture must be canvas-aspect 16:9**). Stage it: sim = locale+status-bar+privacy+burned dialogs; web = logged in, banners dismissed, viewport sized to the aspect; proto = presentation mode, Fit zoom.
3. Per scene: dry-run for coordinates → roll detached to a durable dir → drive with deliberate pacing → SIGINT → **verify the end frame**.
4. Measure with product-film's diffscan (mobile) / diffscan2 (desktop) → real interaction times.
5. `python3 scripts/autoplan.py taps.json` (add `--desktop` on the desktop path) → composite2 camera plan + cursor/ripple args (zoom punches arrive 0.3s before each tap; cues complete **before** the UI transition).

## Non-negotiables

- No take before staging is complete and one-time dialogs are burned.
- Dry-run before every recorded flow; verify field focus before typing.
- Record to durable storage; a take without a verified end frame isn't done.
- Trim only from measured times — tap latency makes plans fiction.
- A contaminated take (dialog, toast, wrong state) is discarded and re-rolled, never rescued in the edit.
- Pass the udid explicitly on every drive command when more than one simulator exists.
