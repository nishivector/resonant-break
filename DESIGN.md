# Resonant Break — Design Document
Round 12

---

## Identity

**Game name:** Resonant Break
**Tagline:** The gap between almost and perfect is a frequency you learn to feel.

**What is the player:** A weapons technician aboard a silent reconnaissance vessel, firing a single resonance cannon at crystalline lifeforms that communicate — and die — in pure oscillation. You never move. You only tune.

**World feel:**
The field is absolute darkness pierced by receding planes of softly glowing enemies — geometric crystal clusters floating at three depth layers, each one silently broadcasting its frequency as a ripple of light and color. There is no explosion, no blood, no kinetics: when you get it right, a creature folds inward along its own geometry and ceases to exist, as though the universe simply revoked its permission to be solid.

**Emotional experience:** Surgical fluency

**Reference games:**
- *Thumper* — frequency is matched by eye and body, not by reading numbers; a mistimed release doesn't just miss, it punishes with an auditory wince that makes you feel like you've broken something delicate
- *Gran Turismo 3 (time trial)* — the difference between 94% and 100% is not incremental; it is the difference between acceptable and a completely different category of experience; the gap is enormous and crossing it feels like a revelation
- *Dead Space* — UI elements are embedded in the world, diegetic and exact; nothing is decorative; every readout means something

---

## Visual Spec

| Token | Value | Purpose |
|---|---|---|
| Background | `#1A0A2E` | Deep violet-black, space |
| Primary | `#E040FB` | Electric magenta — weapon glow, perfect shatter |
| Secondary | `#F5F5DC` | Warm cream — enemy crystal body, waveform lines |
| Accent | `#00FFB3` | Phosphor cyan-green — frequency match indicator, tuning HUD |

**Bloom:** Yes. Strength `1.8`, threshold `0.55`. Applied to all glowing elements: enemy waveform trails, weapon pulse ring, perfect shatter fragments. Do NOT bloom the background or cream crystal bodies — contrast is critical.

**Vignette:** Yes. Radial darkening at edges, opacity `0.45`, radius starts at `65%` of screen width. Pulls focus to the centre field.

**Camera:** Fixed orthographic, dead-on side view. The camera never moves. Enemies float in a 2D field occupying 80% of screen width, arranged across three depth layers (z-parallax achieved through scale only — front layer at `1.0×`, mid at `0.72×`, back at `0.52×`; no actual 3D transform). The player's weapon is a fixed off-screen-left emitter; only its UI (oscilloscope ring, frequency needle) is visible at screen left-centre.

**Player silhouette:** Thin annular oscilloscope ring

---

## Visual Language — Enemy Waveform Readout

This is the entire game's skill surface. Get this exactly right.

Each enemy emits two simultaneous signals, both visual, both readable without UI labels:

### Signal 1 — Frequency (what you tune to match)
A sinusoidal wave radiates outward from the enemy's body, drawn as a glowing cream (`#F5F5DC`) line. The wave's **spatial wavelength** (how tightly packed the cycles are) encodes the enemy's oscillation frequency. More tightly packed = higher frequency. The wave is drawn in screen space — it doesn't scroll or animate laterally. Its wavelength is redrawn every frame to reflect the current frequency. Range: `0.5 Hz` (very wide, slow cycles) to `4.0 Hz` (dense, tight cycles). The wave extends `120px` to the left and right of the enemy body.

### Signal 2 — Amplitude (when to release)
The enemy's crystal body pulses in brightness. At amplitude minimum, the body glows at `30%` of its base emission. At amplitude peak, it blazes to `100%`. This pulse is a **separate cycle** from the frequency wave — it runs at `0.5× the enemy's frequency`. So a `2.0 Hz` frequency enemy has an amplitude pulse every `1.0s`. The body color shifts from near-background (`#2A1040`) at minimum to full cream-magenta blend (`#F5C0F0`) at peak.

**Why two signals:** Frequency = what note. Amplitude = when. The player reads both simultaneously. This is the language.

---

## Sound Spec

**Music:**
140 BPM. Sparse, almost empty — a pulse drone in D minor built from sine waves only, no percussion except a sub-bass kick every 4 bars. The arrangement breathes around enemy frequencies; when no enemies are present, the track thins to a single faint pad. As enemies appear, thin high-frequency saw waves emerge (one per active enemy, each tuned to the enemy's actual frequency rendered as a musical pitch, `A3 = 1.0Hz` baseline, `+1 semitone per 0.1Hz delta`). The effect is that the enemy field IS the music. On perfect shatter, the enemy's saw wave cuts out cleanly, leaving audible silence where it was. **Near-win state:** The drone pitch shifts up `+2 semitones` and the BPM accelerates to `156` over 8 bars. **Game over:** Everything decays to silence over `3.0s`.

**Sound Effects:**

1. **Charge start** (`pointerdown`)
   `new Tone.Oscillator({ type: "sine", frequency: 220, volume: -18 }).start()` — a low sine hum that ramps from `220Hz` to the player's current tuned frequency over `0.3s`. Signals tuning mode is active.

2. **Frequency lock** (player's wave matches enemy within ±5%)
   `new Tone.Synth({ oscillator: { type: "sine" }, envelope: { attack: 0.01, decay: 0.05, sustain: 1.0, release: 0.1 } }).triggerAttackRelease("A5", 0.08)` — a brief magnetic click-tone at `880Hz`. Short enough to not be musical, long enough to feel physical. Plays each time match crosses the 5% threshold.

3. **Perfect strike** (≥95% combined match, strike lands)
   `new Tone.Synth({ oscillator: { type: "sine" }, envelope: { attack: 0.005, decay: 0.0, sustain: 1.0, release: 1.2 } }).triggerAttackRelease(enemyFreqAsNote, 1.0)` — a pure sine-wave ping at the enemy's exact musical frequency, sustains for `1.0s`, releases cleanly. This is the sound the player chases. It should be physiologically satisfying — tune it by feel.

4. **Partial strike** (70–94% combined match)
   Same as perfect strike but `triggerAttackRelease(enemyFreqAsNote, 0.3)` and immediately apply `new Tone.BitCrusher({ bits: 4 })` in the signal chain — the note plays then clips and degrades over `0.3s`. You hear the same pitch but it's been broken. Communicates "you were right, but not enough."

5. **Dissonant miss** (<70% combined match)
   `new Tone.Noise({ type: "white", volume: -6 }).start()` filtered through `new Tone.Filter({ frequency: 800, type: "bandpass", Q: 0.5 })`, duration `0.2s`, then hard stop. A brief burst of filtered noise — unpleasant but not ear-splitting. The auditory equivalent of a wrong note at high volume.

6. **Chain shatter** (conductor cascade — wave 6 only, see Level Design)
   6 perfect strike pings fire in sequence at intervals of `80ms`, each `+1 semitone` from the last, forming a rising arpeggio across `6` notes. Each is full volume, pure sine, `1.0s` sustain. They overlap. The result is a `6-note chord` building over `0.4s` — crystalline, overwhelming, earned.

---

## Mechanic Spec

### Core Loop
1. Enemy appears, begins oscillating (frequency + amplitude signals active)
2. Player holds `pointerdown` to enter tuning mode — weapon's oscilloscope ring activates
3. Player drags **horizontally** to adjust their tuned frequency — the weapon's ring waveform updates in real-time to reflect the current setting
4. Player watches the enemy's amplitude cycle and releases `pointerup` exactly `0.2s` before the amplitude peak they are targeting
5. Strike travels to enemy. `0.2s` after release, strike lands. Frequency match and timing match are calculated simultaneously.
6. Enemy shatters, takes damage, or absorbs — depending on combined match %
7. Score updates. Next enemy or wave begins.

### Primary Input
`pointerdown` / `pointerup` / `pointermove`

- `pointerdown` anywhere on screen: begin charging, show tuning ring, play charge hum
- `pointermove` while held: horizontal drag maps to frequency. Full screen width = full frequency range (`0.5Hz` to `4.0Hz`). Drag right = higher frequency. The mapping is linear. There is NO snap to enemy frequency — the player must find it manually.
- `pointerup`: fire. Strike launches. `0.2s` delay before impact.

**No multi-touch.** One strike at a time. If a second enemy appears mid-charge, the player must choose to abort (release without aiming) or commit to the current target.

### Key Timing Values

| Parameter | Value | Notes |
|---|---|---|
| Strike travel time | `200ms` | Constant across all levels |
| Amplitude cycle period | `0.5 × (1/enemyHz)` seconds | e.g. 2Hz enemy → amp peak every 0.5s |
| Timing window for "perfect" | ±`50ms` around peak (at landing) | Player must release at `peak − 200ms ± 50ms` |
| Timing window for "partial" | ±`150ms` around peak | |
| Frequency tolerance for "perfect" | ±`3%` of enemy frequency | |
| Frequency tolerance for "partial" | ±`15%` of enemy frequency | |
| Combined score formula | `freqMatch% × timingMatch%` | Both must be high to reach 95% |
| Dissonant immunity duration | `2.0s` | Enemy absorbs and glows red during this window |
| Enemy spawn delay | `1.5s` after previous enemy clears | First frame of new enemy shows waveform immediately |

### Frequency Drift
Enemies never hold a static frequency. They drift sinusoidally:

`currentHz = baseHz + (driftAmplitude × sin(2π × driftRate × t))`

| Level | driftAmplitude | driftRate |
|---|---|---|
| 1 | `0.00 Hz` (no drift) | n/a |
| 2 | `0.05 Hz` | `0.3 Hz` (slow sway) |
| 3 | `0.15 Hz` | `0.4 Hz` |
| 4 | `0.25 Hz` | `0.5 Hz` |
| 5 | `0.40 Hz` | `0.6 Hz` |

The player sees the drift as the enemy waveform's spatial frequency gradually tightening and loosening. There is no numerical readout. The eye reads it; the hand leads it.

### Prediction Window Shrink
The amplitude peak timing tolerance (`±50ms` for perfect, `±150ms` for partial) shrinks by `15ms` on the perfect threshold each wave:

| Wave | Perfect window | Partial window |
|---|---|---|
| 1 | ±65ms | ±165ms |
| 2 | ±50ms | ±150ms |
| 3 | ±35ms | ±135ms |
| 4 | ±20ms | ±120ms |
| 5 | ±10ms | ±110ms |
| 6 | ±10ms | ±100ms |

### Score System

- **Base score per enemy:** `100 × combinedMatch%` (e.g. 100% = 100pts, 78% = 78pts)
- **Resonant Multiplier:** If combined match ≥ 95%, score is multiplied by `3×` AND the current multiplier stack increments by `+1`
- **Multiplier stack:** Starts at `1`. Each consecutive ≥95% hit adds `+1`. A <95% hit (but still a hit) resets stack to `1`. A miss (<70%) resets stack to `1` and adds `−50pts` (can go negative). Maximum stack: `6×` (bonus visual: weapon ring turns full magenta at ×6)
- **Multiplier on display:** The oscilloscope ring's number of visible nodes corresponds to the current multiplier. At ×1: 1 node. At ×6: 6 nodes evenly spaced, ring blazes.

### Win / Lose

- **Win:** Complete all 6 waves (all enemies shattered or survived). Score tallied.
- **Lose:** Only happens if a surviving enemy reaches `100%` amplitude cycles without being shattered 3 times. Third cycle = enemy shatters your cannon instead. **Health** = 3 enemy escapes. Display as 3 amber rings around the player's oscilloscope — one goes dark per escape. At 0, game ends. Escaped enemies are ones that have been hit at <70% AND have cycled through 3 full amplitude peaks while still alive.

Actually: an enemy is "alive" until its total damage exceeds `100%`. Each ≥95% hit = `100%` (instant kill). Each 70–94% hit = `freqMatch% × timingMatch%` damage. Each <70% hit = `0%` damage. Enemy has `HP = 1.0`. Partial hits accumulate. An enemy that has survived `5 partial hits` with an average of 50% each is at `250%` damage and has been dead since hit 2 — kill it on that hit's final blow.

For simplicity: **each enemy requires exactly 1 clean hit ≥70% combined to clear** (unless below 70%, in which case 0 damage). If a player fires 3 hits below 70% on the same enemy without clearing it, that enemy counts as an escape at the 3rd miss and disappears from the field (player loses 1 health ring, forfeits that enemy's score). Maximum 3 such escapes before game over.

---

## Level Design

### Level 1 — "First Signal"
**What's new:** The entire mechanic, introduced implicitly. No tutorial text. One slow enemy. The first enemy's amplitude cycle is `2.0s` long (very slow). Its waveform is sparse (frequency `0.8Hz`). There is a subtle slow pulse on the enemy that makes the "peak is coming" visually obvious even to a new player.
**Parameters:** 1 enemy. `baseHz = 0.8`, `drift = 0`. Amplitude cycle `2.0s`. Perfect window `±65ms`. 2 enemies total in the wave, sequentially.
**Goal:** Player has time to observe 3–4 amplitude cycles before committing. A correct release on the first try is possible. A correct release on the third try is expected.
**Duration:** ~45 seconds.
**Emotional beat:** Self-discovery. The rule is revealed by doing it wrong once and right once.

### Level 2 — "Interference"
**What's new:** Two enemies on screen simultaneously, each at a different frequency. The waveforms overlap visually — the player must learn to isolate one from the other. Frequencies are chosen to be maximally distinct: Enemy A always `0.8Hz`, Enemy B always `2.4Hz` (3:1 ratio — visually very different wavelength).
**Parameters:** 2 simultaneous enemies. `drift = ±0.05Hz`, `driftRate = 0.3Hz`. Perfect window `±50ms`. 4 enemies total in the wave.
**Goal:** Player must decide which to target, tune to that frequency, fire, then retune for the second.
**Duration:** ~60 seconds.
**Emotional beat:** First sense of the frequency-as-language concept. Two signals, two solutions.

### Level 3 — "Drift"
**What's new:** Drift is now meaningful. Enemy frequency moves enough that a player who locks on and waits will find themselves mismatched by the time they fire. Leading the frequency becomes mandatory.
**Parameters:** 2 enemies max at a time. `drift = ±0.15Hz`, `driftRate = 0.4Hz`. `baseHz` range now `0.8–3.2Hz`. Perfect window `±35ms`. 6 enemies total.
**Goal:** First wave where waiting for a clean read will reliably fail. Player must predict where the frequency is going.
**Duration:** ~75 seconds.
**Emotional beat:** The first real frustration — and the first real mastery click when prediction works.

### Level 4 — "Noise Floor"
**What's new:** Three simultaneous enemies, waveforms overlapping in a dense visual field. The player must read three different frequencies at once and prioritize. Introduce one **armoured** enemy variant: armoured enemies have their crystal body tinted amber (`#FFB300`); they require TWO ≥70% hits to shatter (first hit cracks the shell visible as a permanent fracture line, second destroys). All other rules identical.
**Parameters:** Up to 3 simultaneous enemies, 1 may be armoured. `drift = ±0.25Hz`, `driftRate = 0.5Hz`. Perfect window `±20ms`. 8 enemies total including 2 armoured.
**Goal:** Chain ≥95% hits on 3 enemies before they cycle past the timing window.
**Duration:** ~90 seconds.
**Emotional beat:** The multiplier stack clicks into real power. A player hitting ×4 chain watches the score numbers become absurd.

### Level 5 — "Conductor" (wave 6 of the overall sequence)
**What's new:** The **Conductor** enemy. One unique entity that spawns at the start of this level. It is larger than normal (2.0× scale), floats at the centre depth layer, and broadcasts an oscillating signal that is visually distinct: instead of a radial wave, its signal is an expanding concentric ring every full amplitude cycle, rendered in primary magenta (`#E040FB`) with high bloom. This ring passes through every other enemy on screen. While the ring is passing through an enemy, that enemy's waveform **synchronises to the Conductor's frequency** for exactly the duration of the ring's passage. Standard enemies' own frequencies return afterward. There is absolutely NO text, tooltip, or visual indicator explaining what the ring does. The player discovers it.

**The chain mechanic:** If the player fires a ≥95% hit on the Conductor *while the Conductor's ring is overlapping at least 4 other enemies simultaneously* (ring position is at the midpoint of the field, passing through the cluster), the strike's harmonic resonance propagates along the active ring. Every enemy the ring currently overlaps shatters simultaneously with ≥95% scoring. This is the only time multiple simultaneous shatters can occur. The chain shatter SFX plays (6-note arpeggio). Slow-motion activates for `1.5s`. The field goes near-dark then erupts in cream-magenta fragment geometry.

**Parameters:** 1 Conductor + 5 simultaneous standard enemies. `drift = ±0.40Hz`, `driftRate = 0.6Hz`. Perfect window `±10ms`. Conductor `baseHz = 1.6Hz`, amplitude cycle `1.5s`. Standard enemies randomized `baseHz` within `0.8–3.5Hz`.
**Goal:** Either chip down all 6 enemies individually (brutal, hard, doable), OR read the ring timing and fire one perfect Conductor hit during peak overlap (almost impossibly elegant, massively rewarded).
**Duration:** ~2 minutes. This is the end.
**Emotional beat:** The chain shatter is the game's entire thesis in one moment. The player didn't know it was possible. It was always possible. Everything they learned about frequency, drift, and timing was preparation for a mechanic that was never explained.

---

## The Moment

On wave 6, the player fires a perfect tuned strike at the Conductor-class enemy at the exact moment its harmonic ring passes through all five remaining enemies — the screen goes dark for `0.08s`, then all six crystal forms fold inward along their geometry simultaneously, each one emitting a clean sine ping, the six tones stacking into a chord that builds over `0.4s` and resonates for `1.0s` while the score multiplier stack blazes at ×6, and then there is silence, and the field is empty, and the player sits there for a second not quite believing what just happened.

---

## Emotional Arc

**First 30 seconds:** One enemy. Slow pulse. You fire twice wrong — you hear the clipped note. On the third attempt you time it right, and a pure clean ping fills the silence. The rule is obvious. No one needed to explain it. You understood it with your hands.

**After 2 minutes:** Three enemies pulse in different rhythms. Their waveforms overlap on screen. You have stopped looking at the HUD. You are reading the enemies directly — tighter spacing = higher, you drag right; looser spacing = lower, you drag left. You are not translating. You are fluent. The frequency is a language and you speak it now.

**Near the win:** The Conductor's ring is approaching maximum overlap. Enemies are drifting. Your timing window is `±10ms`. You can see the peak coming. You fire `0.2s` early. The ring has four enemies. Five enemies. You hit it. The arpeggio builds. The field shatters. The multiplier blazes. You don't move for a second.

---

## Difficulty Curve Summary

| Wave | Enemies | Drift | Timing Window | Special |
|---|---|---|---|---|
| 1 | 2 (sequential) | None | ±65ms | Intro |
| 2 | 4 (2 at once) | ±0.05Hz | ±50ms | First interference |
| 3 | 6 (2 at once) | ±0.15Hz | ±35ms | Drift forces prediction |
| 4 | 8 (3 at once) | ±0.25Hz | ±20ms | Armoured enemies |
| 5 | 6 (1 Conductor + 5) | ±0.40Hz | ±10ms | Chain shatter moment |

---

## This game's identity in one line

**"This is the game where you learn to read a frequency with your eyes and release before you're ready."**

---

## HUD / UI Specification

All UI is diegetic — part of the weapon system, not floating labels.

**Player oscilloscope ring:**
- Centred at screen-left-centre, `x = 8%`, `y = 50%`
- Ring diameter: `120px`
- The ring's circumference displays a real-time waveform drawn along the ring's edge — the waveform amplitude shows the player's tuned frequency visually (same encoding as enemies — more cycles per ring = higher frequency). The ring shows `4` waveform cycles when at `1.0Hz`, `8` cycles when at `2.0Hz` etc. — this is how the player knows their current setting.
- Glow: `#E040FB` bloom, strength `1.8`
- Frequency lock indicator: when player's tuned Hz matches enemy's current Hz within ±5%, the ring briefly flashes accent cyan (`#00FFB3`) for `80ms`
- Multiplier stack nodes: small filled circles evenly spaced around the ring exterior, 1 per stack level. Each circle `6px` diameter, color `#00FFB3`. All dark at start. Illuminates as stack grows.
- Health rings: 3 amber rings (`#FFB300`, `2px` stroke, `10px` spacing) concentric outside the main ring. One darkens per escape.

**No score display during gameplay.** Score is shown only on the post-wave screen as a single large number against black.

**Frequency readout:** None. No Hz numbers anywhere on screen. The waveform IS the readout.

**Enemy health:** No health bar. Visual state only — intact crystal → hairline crack (first partial hit) → heavy fracture lines → shatter.

---

## Start Screen

### Idle Animation
Crystal fragments — 12 to 18 broken crystal shards, each one a polygon (5–8 sides, irregular) drawn in `#F5F5DC` at `40%` opacity — float slowly across the dark background (`#1A0A2E`). Each fragment:
- Moves in a direction chosen at spawn (`angle = random(0, 2π)`), speed `8–22px/s`
- Rotates slowly: `rotationSpeed = random(−0.3, 0.3) rad/s`
- Pulses in opacity: `opacity = 0.2 + 0.25 × sin(2π × pulseHz × t + phaseOffset)`, where `pulseHz = random(0.4, 1.8)` unique per fragment
- On each fragment, draw a short sinusoidal wave line across its longest axis in primary magenta (`#E040FB` at `60%` opacity), whose spatial frequency cycles slowly: `waveHz = 0.5 + 0.8 × sin(0.15 × t)` — the fragment appears to be resonating
- Fragments wrap at screen edges
- On the start screen, the background has a very faint oscilloscope sweep: a horizontal line at `y = 50%`, drawn in `#00FFB3` at `8%` opacity, with a sine wave riding it whose amplitude and frequency slowly shift (the "idle weapon scan") — subtly alive

### SVG Overlay

**Option A — Title glow (required):**
```svg
<svg xmlns="http://www.w3.org/2000/svg">
  <defs>
    <filter id="glow" x="-30%" y="-30%" width="160%" height="160%">
      <feGaussianBlur stdDeviation="6" result="blur"/>
      <feComposite in="SourceGraphic" in2="blur" operator="over"/>
    </filter>
  </defs>
  <text
    x="50%" y="44%"
    text-anchor="middle"
    font-family="'Courier New', monospace"
    font-size="52px"
    font-weight="700"
    fill="#E040FB"
    letter-spacing="0.15em"
    filter="url(#glow)"
    style="animation: fadeIn 1.8s ease-out forwards; opacity: 0;"
  >RESONANT BREAK</text>
  <text
    x="50%" y="54%"
    text-anchor="middle"
    font-family="'Courier New', monospace"
    font-size="16px"
    fill="#F5F5DC"
    letter-spacing="0.25em"
    opacity="0.7"
    style="animation: fadeIn 2.4s ease-out forwards; opacity: 0;"
  >TUNE · PREDICT · SHATTER</text>
</svg>
```
`@keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }`

**Option B — Crystal silhouette (80×80px), centred above title:**
An 8-sided irregular polygon suggesting a crystal cluster — not symmetric. Three overlapping crystal points, tallest one centred. Drawn as a single `<path>` in `#E040FB` at `85%` opacity, filter `url(#glow)`. The silhouette pulses in scale very slowly: `transform: scale(1.0 + 0.04 × sin(0.8t))`. No animation on the shape itself — the idle fragments handle the kinetic energy.

```svg
<g transform="translate(50%, 32%)">
  <path d="M0,-38 L10,-12 L32,-8 L14,8 L20,34 L0,20 L-20,34 L-14,8 L-32,-8 L-10,-12 Z"
    fill="#E040FB"
    opacity="0.85"
    filter="url(#glow)"
    style="animation: crystalPulse 3.2s ease-in-out infinite;"
  />
</g>
```
`@keyframes crystalPulse { 0%,100% { transform: scale(1.0); } 50% { transform: scale(1.04); } }`

**Start prompt:** `PRESS ANYWHERE TO BEGIN` — monospace, `13px`, `#00FFB3`, letter-spacing `0.3em`, blinking at `1.0s` interval (CSS `animation: blink 1s step-end infinite`), positioned at `y = 70%`.

---

## Implementation Notes for Programmer

1. **The drift equation must be calculated every frame** — do not quantize to frames. `currentHz = baseHz + (driftAmplitude × sin(2π × driftRate × elapsedSeconds))`. Store `elapsedSeconds` as a float, reset on new enemy spawn.

2. **The player's tuned frequency** is set by horizontal pointer position as a fraction of screen width: `playerHz = 0.5 + (pointerX / screenWidth) × 3.5`. Range `0.5–4.0Hz`.

3. **Strike timing calculation:** Record `releaseTimestamp = performance.now()` at `pointerup`. Strike lands at `releaseTimestamp + 200`. At landing, sample the enemy's amplitude: `amplitude = 0.5 + 0.5 × sin(2π × (1/(0.5/enemyHz)) × ((landingTimestamp − enemySpawnTimestamp) / 1000))`. Compare to peak (`1.0`). Timing delta = `|landingAmplitude − 1.0|` in amplitude units (0 = perfect peak, 1 = trough). Map to timing match %: `timingMatch = max(0, 1 − (|landingAmplitude − 1.0| / 0.5))` — this gives 100% at peak, 0% at trough.

4. **Conductor ring timing:** The ring emits every full amplitude cycle of the Conductor. Ring spawn = start of each Conductor amplitude cycle (when amplitude crosses from <0.5 to rising). Ring travels from Conductor position to screen edges at `400px/s`. An enemy is "overlapped" if the ring's current radius encompasses the enemy's position. Track `ringOverlapCount` at moment of strike landing.

5. **Slow-motion on perfect shatter:** set `gameSpeed = 0.15` for `1500ms` real time, then lerp back to `1.0` over `300ms`. Affect all enemy movement, drift, and animation speed. Do NOT affect audio playback speed — keep audio at normal tempo.

6. **Fragment shatter geometry:** On death, divide the enemy's polygon into 8–12 Voronoi-ish sub-polygons. Each fragment gets a randomized velocity (`100–280px/s`), rotational velocity (`1–4 rad/s`), and fades out over `0.8s` (opacity `1→0`). Fragments are drawn in `#F5F5DC` with a faint magenta glow. On chain shatter, fragments from all enemies explode outward simultaneously in slow-motion — the visual climax of the game.

7. **Tone.js:** Load via CDN. All audio synthesis client-side. No audio files needed. The music drone should start on first user gesture (browser autoplay policy).
