# RADIANT BURNT

**A generative poster maker that sets type on fire.** Feed it a word, an image, or an SVG; it renders that shape as a persistent thermal fluid — advected, decayed, re-injected frame over frame — and finishes the result through an analog-scanner post chain of melt, bokeh, lens distortion, photocopy bleed and grain.

One file. No dependencies. No build step. No network calls.

```
git clone https://github.com/markdo27/Burnt-Noise.git
cd Burnt-Noise
python -m http.server 8000     # or: npx serve .
```

Then open <http://localhost:8000>. Opening `index.html` directly off the filesystem works too — you only need the server if you want to load custom fonts or images without hitting `file://` CORS restrictions.

Requires a browser with WebGL. Uses WebGL2 with half-float render targets where available and falls back to WebGL1 with 8-bit buffers (plus a sub-LSB dither to hide the quantised decay staircase). The status bar in the bottom-right tells you which path you're on.

---

## How it works

The whole effect comes from one idea: **the simulation reads its own previous output.** It isn't recomputed from scratch each frame, so detail and motion keep compounding — which is what makes it read as an actual fluid rather than a static drip.

```
[ TEX  ]  Hero mask      text (auto-fit) / image / SVG, rasterised on a 2D
                         canvas into a white-on-alpha coverage mask, then
                         optionally edge-blurred before it enters the sim
[ PASS A] Fluid sim      ping-pong FBO pair, PERSISTENT ACROSS FRAMES —
                         advect the previous state along a flow field
                         (moving brush + curl-ish turbulence), decay it,
                         re-inject the fresh mask on top
[ PASS M] Melt sim       persistent, time-integrated — heat accumulates,
                         material past its melt point yields and sags,
                         mass is transported and conserved
[ PASS B] Blur / glow    separable 9-tap Gaussian, run twice, over a
                         4-level pyramid below sim resolution
[ PASS C] Thermal map    core + glow drive a luminance lookup blending a
                         user 2-colour "cool" ramp into a hot fire ramp
[ TEX  ]  Poster type    metadata typography — kicker, date, line-up
                         block, serial — everything but the hero word
[ PASS 2] Post process   optics first (lens distortion, melt displacement,
                         jittered bokeh, chromatic fringing folded into the
                         same taps), then metadata composites, then the
                         emulsion (photocopy threshold, contrast, grain)
```

A few things worth knowing if you're reading the source:

**The type is a source term, not an obstacle.** There is no pressure solve, no divergence projection, and nothing flows *around* a letterform. Glyph coverage is density injected into the field, which advection then drags away. The turbulence is two decorrelated noise fields treated as a vector — not a true divergence-free curl, but it reads the same at this scale for a fraction of the cost.

**Sharp detail never passes through the simulation.** The crisp core is sampled straight from the full-resolution mask in the thermal pass and composited into a full-resolution scene buffer. Only the smooth glow field is ever upsampled — which is exactly where upsampling is invisible.

**Simulation resolution is decoupled from display resolution, and fixed across devices.** The sim is capped at 1400px on the long edge no matter what; the display buffer is CSS size × DPR with DPR capped at 1.5. That cap is deliberate — a 4200px export would otherwise allocate hundreds of MB, and pinning the sim size is what makes **an export match its preview** instead of resolving differently on a different machine.

**Everything is integrated against real time, not against frames rendered.** Decay is exponential in time (`persistence` raised to the power of the frame step), so a trail is the same length at 30, 60 and 144 Hz, and a backgrounded tab doesn't hand back one enormous delta that advects the whole poster at once.

---

## Controls

Six collapsible sections down the left. Everything is live; nothing needs re-rendering.

### Content Source
Text, image, or SVG. Text auto-fits the frame and accepts newlines. **Source Edge Blur** softens the mask before it reaches the simulation — this is the knob that controls how hard or how vaporous the boundary between ink and flow reads. Custom fonts upload for both the hero word and the metadata.

### Fluid & Interaction
`Manual` drives the brush with your pointer; `Auto` sweeps it for you across five presets — **sweep, drift, orbit, pulse, boil** — with speed, direction, wrap/ping-pong looping and easing.

| Control | What it does |
|---|---|
| Brush Radius | Size of the drag influence, and the feather on the content wipe |
| Smear Intensity | How far material moves per step |
| Persistence | Decay rate — how long the trail survives |
| Fluid Warp | Turbulence amplitude |
| Warp Scale | Turbulence frequency |

### Optical Effects
Cool ramp (two colours), burn handoff colour, ground, core colour. Plus **Exposure (Burn)**, which blends the user ramp toward the built-in fire ramp, RGB split, and glow softness.

### Post Process
Bokeh with a focus plane, focus band and edge falloff (`partial`) or a flat whole-frame defocus (`full`); lens distortion, fringing and highlight bloom; photocopy bleed, contrast and grain.

The **melt** block is its own small simulation — heat, melt point, hot-spot scale, gravity, viscosity, flow, spread and surface tension. Melt Amount is deliberately bounded by stroke width: the read offset has to stay within a fraction of the artwork's line weight, or every interior pixel samples something ten strokes away, the middle of each letter comes back as background, and all that survives is a chrome-looking rim.

**Lens Quality** defaults to `auto` — 48 bokeh taps, dropping to 16 only if frames are actually being missed.

### Poster Metadata
Optional flyer typography layered over the artwork: kicker and date at the head, a label above a stacked block, serial at the foot. Scale, tracking, leading, label gap, weight, case, mono, and a blend mode.

### Output
Aspect ratios 2:3, 3:4, 4:5, 1:1 and A-series. Exports:

- **PNG** at 1600 / 2400 / 3200 / 4200 px, clamped to the GPU's max texture size. Static exports fast-forward the simulation to a settled state first (up to 500 steps) and force the content fully visible, so you never catch it mid-fade.
- **GIF** at 320–720 px wide, 8–20 fps, 1–8 seconds.
- **WebM** video via MediaRecorder.

### Keyboard

| Key | Action |
|---|---|
| `Space` | Freeze / unfreeze animation |
| `R` | Randomize seed |
| `E` | Export PNG |

Also: **Randomize Seed** for a new composition from the same settings, **Randomize All** to reroll the parameters, and **Reset** to return to the tuned startup state.

---

## Project layout

```
index.html    the entire application — markup, CSS, JS, and every GLSL shader
```

That's it, and it's intentional. The file is heavily commented, and the comments carry the *reasoning* — most of them exist because something looked wrong once and the fix isn't obvious from the code alone. If you're changing the melt or the fluid step, read the comment block above it first; it will usually tell you what the naive version does and why it was abandoned.
