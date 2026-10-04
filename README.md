# Orbit Dash — M4 Prototype

One-tap timing game. A ball orbits a ring; tap exactly when it's inside the green gate. Each hit: score, direction reverses, speed up, gate shrinks. Tap outside the gate — or drift past it — and you're dead. Tap to retry instantly.

## Scoring

- **Normal hit** (anywhere in the gate): **+1**
- **PERFECT hit** (inside the gold core, the middle 34% of the gate): **+2**, and
  each consecutive perfect adds one more — +2, +3, +4, +5, +6, capped at +6.
  One non-perfect hit resets the streak.
- Coins earned = points scored, so precision pays for skins too.
- Best perfect streak is saved and shown next to your high score.

A perfect hit sends a **shockwave along the ring** rather than a detached
particle puff: two crisp near-white pulses race outward around the circumference
from the hit point, and sparks shear off along the tangent. They travel further
with longer streaks.

Deliberately **no translucent overlays** on perfects — no spreading wash, no
swelling halo, no bloom under the pulses, no full-screen tint. Those hazy layers
muddied the ring instead of reading as energy. Solid geometry only.

## Palette

Within any single frame the gate is still **single-hue escalation** — edge →
base → near-white core — so "better" is communicated by brightness, not by a
second hue competing for attention. The earlier lime-green + gold pairing fought
both each other and the cool background.

What's new is that the hue **travels with speed**. `speed_shown` (1.0 → 3.2)
drives a continuous sweep through four anchors, so how deep you are in a run is
legible from the ring itself without reading the score:

| Tier | Speed | Edge | Base | Core |
| --- | --- | --- | --- | --- |
| Teal | ×1.00 | `#17A392` | `#2DD4BF` | `#EAFFFA` |
| Cyan | ×1.55 | `#1799B8` | `#22D3EE` | `#E6FBFF` |
| Violet | ×2.15 | `#6D5BE8` | `#8B7CFF` | `#EFEAFF` |
| Magenta | ×2.75 | `#C44AE0` | `#E15CFF` | `#FBE6FF` |

Only one hue is ever on screen at a time — this is a slow drift with depth, not
two hues coexisting. Interpolation is continuous, so the sweep never visibly
"switches" tiers, and it's keyed to `speed_shown` rather than `speed_mult` so the
colour eases on the same 0.25s ramp the ball does.

The ramp deliberately stops short of red: `#FF3B5C` stays reserved for death, and
every tier's core sits ≥2.9:1 against it. Measured against the background, using
the 3:1 bar for graphical objects (these are arcs, not text): edges ≥3.9:1,
bases ≥5.8:1, cores ≥16.3:1.

Background `#0B0F14`, ring `#1E2A33`, death `#FF3B5C`, coin `#FFD166`.

Tune the whole thing via `GATE_RAMP` in `main.gd`.

The ball carries a **dark rim** so it stays readable against the gate, sweet spot
and shockwave no matter which skin is equipped — several skin hues sit close to
the gate's brightness and would otherwise blend at exactly the moment the timing
has to be judged. Because the gate hue now travels, it can land directly on top
of a skin (the violet tier vs the default Azure ball is only 1.13:1), so the rim
widens 3px → 4.5px above `RIM_WIDEN_ABOVE` (×1.9).

## Gate shape

The gate is drawn as **34 segments of one continuous arc** that ramp in *both*
colour and thickness to a peak at the centre — 8px at the edges, 28px at the
middle, with **rounded caps** at each end. The lens shape is what gives the
target physical form; the colour ramp is what stops the perfect zone reading as
a white block pasted onto a flat band, which is how the old two-`draw_arc`
version looked.

Thickness follows a **cosine dome** (`_gate_profile`), not a `pow()` falloff —
`pow` left a visible crease either side of the core. The ends need explicit
`draw_circle` caps because `draw_arc` has butt caps; without them the band
terminates in two square corners.

The **glow follows that same profile**, drawn as two wider, dimmer passes of the
same band. It used to be a single flat-width `draw_arc`, which stuck out past
the tapering gate at both ends and read as a hard rectangular shelf behind it.
The glow needs *more* segments than the gate (30 vs 34 for a much shorter visual
arc) because its width is scaled ~2.6×, which multiplies the per-segment
thickness step and staircases badly at low counts.

A **continuous train of crests flows outward** from the perfect core, brightening
and swelling the band as each passes. Two dead ends got us here:

- Breathing in alpha only read as dead — the shape never changed, so there was
  nothing for the eye to track.
- A *single* crest sweeping centre → edge read as a loop, not a flow: it
  vanished at the edge and popped back to the centre. Sampling the field, that
  reset was a full-amplitude (1.0) discontinuity once per cycle, with moments of
  zero crests in flight.

`_gate_shimmer` is now a smooth function of position and time with no modulo on
the amplitude, so nothing is ever reset — largest frame-to-frame change drops to
0.025, and there are always 1–2 crests in transit. Crests dim as they travel
(`GATE_SHIMMER_FADE`) so they die out before the edge rather than being cut off
by it.

Tune with `GATE_SHIMMER_SPEED` (crests per second), `GATE_SHIMMER_WAVES` (how
many are in flight at once), `GATE_SHIMMER_SHARP` (higher = tighter pulses,
lower = a broad sine swell) and `GATE_SHIMMER_FADE`.

Because the ramp is soft, the PERFECT boundary is marked by two **crisp radial
ticks** at ±`PERFECT_ZONE`, plus a third at dead centre — the bonus window stays
literally readable even though the fill is a gradient. Tick length follows the
gate's *local* thickness at that angle, not the core width, or it overshoots the
tapering band and reads as a stray whisker.

Tune the shape with `GATE_SEGMENTS`, `GATE_EDGE_W`, `GATE_CORE_W`.

The perfect window stays hittable at top speed (~52 ms, about 3 frames at
60 fps) — demanding but fair. Tune with `PERFECT_ZONE` and `PERFECT_MAX_MULT`
in `main.gd`.

## Run it

1. Download Godot 4.2+ (standard version): https://godotengine.org/download
2. Open Godot → Import → select this folder's `project.godot`
3. Press **F5** (or the ▶ button)

Click = tap (touch emulation is enabled).

## Audio

All sound is synthesized in code at startup (`scripts/audio_service.gd`) — no
audio files, no licensing, ~545 KB of RAM and nothing added to the APK.

- **Backing loop**: 4-bar Am–F–C–G progression at 110 BPM (bass, kick, arpeggio),
  generated one frame after launch so startup never hitches.
- **Musical taps**: every gate plays the next note up the A-minor-pentatonic
  scale, so a run builds an ascending phrase that's always in key with the
  backing track. Wraps every 14 notes into a new phrase. 4 voices so fast taps
  overlap instead of cutting each other off.
- **Perfect hits ring a bell**, and each streak level gets its own interval —
  3rd, 5th, 6th, octave, then octave+3rd with a second shimmer bell at max.
  So consecutive perfects each sound different. All intervals are pentatonic,
  so they stay in key whenever they land.
- **♪ ON/OFF** toggle top-left (shown outside of play only), persisted in the save.

Verified: no clipping (peak 0.84), clean loop seam, no aliasing.

## Layout

All UI is positioned at runtime from the real viewport size (`_layout_ui()` and
`_layout_shop()` in `main.gd`), never from the 720×1280 design size — phone
aspect ratios run from ~16:9 to ~21:9 and `stretch/aspect = expand` means the
viewport height changes per device. Buttons are pinned to the bottom edge, the
header to the top, and the ring is sized to fit the gap between them. The skins
grid scrolls, so it can't overflow a short screen. Both re-run on resize/rotate.

Verified overlap-free from 4:3 through 21:9.

## Files

- `scenes/main.tscn` — main scene (single node; everything is code-drawn)
- `scripts/main.gd` — all game logic; tuning constants at the top
- `project.godot` — 720×1280 portrait, mobile renderer

## Tuning

Edit the constants at the top of `main.gd`:
`BASE_SPEED`, `SPEED_GAIN`, `SPEED_CAP`, `SPEED_RAMP_TIME`, `GATE_HALF_START`, `GATE_HALF_MIN`, `GATE_SHRINK`.

Speed curve: +5.5% compounding per point, capped at 3.2x (reached ~score 22).
A full lap goes from 3.5s at the start to 1.2s by score 20. Gate spawn distance
scales with speed so you always get at least ~0.55s to react — raise `MIN_REACT`
in `_spawn_gate()` if it feels unfair, lower it to make it brutal.

Goal per the design doc: average player dies at 5–10 on first tries, reaches 25–40 after a few sessions.

## M2 — juice (done)

Particle bursts on score/death, screen shake on death, generated ding with rising pitch per combo (no audio assets needed), low thud on death, ball trail, gate spawn pop + breathing glow, ring pulse, score label pop, breather gate every 15 points.

## M3 — meta (done)

- **Coins**: 1 per gate passed, banked on death, shown top-right
- **Skins shop**: 12 ball/trail color skins (50–300 coins), SKINS button on the ready/death screens
- **Daily reward**: escalating 25 → 200 coins for consecutive days, popup on launch
- Persistence in `scripts/save_data.gd` (`user://save.cfg`) — old high scores carry over

## M4 — services (flow done, simulated)

- **Continue (ad)**: once per run, resumes with your score after a rewarded ad
- **2x coins (ad)**: once per run on the death screen
- **Interstitials**: max 1 per 3 deaths, never in the first 5 runs of a session
- **Remove ads**: stub $2.99 button in the shop (disables interstitials, keeps rewarded)
- Ads are SIMULATED in-editor (`scripts/ad_service.gd`) so the whole flow is testable now.
  Real AdMob / Play Games leaderboard / Play Billing wiring: see **M4-SETUP.md**.

## Next (M5 — ship)

Icons, screenshots, store listing, closed testing, release. Real-service wiring per M4-SETUP.md.
# orbit-dash
