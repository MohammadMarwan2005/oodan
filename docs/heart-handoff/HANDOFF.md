# عوداً عوداً — Android (Jetpack Compose) port

Port the web prototype in `web-reference.html` to a native Android app in Kotlin + Jetpack Compose.
Open the HTML in a browser first; it is the source of truth for look, motion and behaviour.

The app shows an anatomical heart. The user throws **white** or **black** "nukta" (dots) at it. Each throw
fires an arrow from a random direction into a random spot on the heart; the dot blooms where it lands.
The user controls dot **size** and **intensity**, can **undo / redo**, and can browse a **history** and jump
back to any point. Inspired by the hadith in Sahih Muslim 144 (quoted in the UI).

---

## 1. Key decision: the lighting is pre-baked

The web version computes lighting with SVG filters (`feDiffuseLighting`, `feSpecularLighting`,
`feTurbulence`). Compose has no equivalent, and re-implementing them is not worth it because **those
layers never change** — only the dots change. So the static layers were rendered once at 4× and ship as
assets. Only dots, arrows, effects and UI are live.

Do **not** try to redraw the heart as Compose vector paths. Use the baked layers.

## 2. Assets (`assets/` → put in `res/drawable-nodpi/` or `assets/`)

All images are 1600 × 1840 px, transparent, and map 1:1 to a 400 × 460 logical **viewBox**.

| File | What it is | Draw with |
|---|---|---|
| `heart_base.webp` | Colour anatomy (muscle, vessels, fat, coronaries) + floor shadow | `SrcOver` |
| *(dots, live)* | White/black stains, clipped to heart silhouette | `SrcOver` |
| `heart_tint.webp` | Vessel/fat colour memory so details read on a white heart | **`Multiply`** |
| `heart_relief.webp` | Cast shadows, outlines, fibres, vessel edges | `SrcOver` |
| `heart_shade.webp` | Diffuse lighting from the height map | **`Multiply`** |
| `heart_spec.webp` | Wet specular highlights | **`Screen`** |
| `heart_paths.json` | SVG path data of every heart part, back→front (viewBox units) | clip + hit-test |
| `sfx_whoosh.wav`, `sfx_hit_white.wav`, `sfx_hit_black.wav` | Arrow flight + impact | `SoundPool` |

Draw order is exactly the table's top-to-bottom order. `reference-composite.png` shows the expected result
for clean / all-white / all-black, produced by recombining these exact files with these blend modes.

**The blend layers must composite inside an offscreen layer**, otherwise Multiply/Screen blend with the
screen background:

```kotlin
Canvas(
  Modifier
    .aspectRatio(400f / 460f)
    .graphicsLayer { compositingStrategy = CompositingStrategy.Offscreen }
) { ... }
```

Scale: `val s = size.width / 400f`; draw images into `Rect(0f, 0f, size.width, size.height)`; dot/arrow
coordinates are viewBox units × `s`.

## 3. Heart geometry

- Parse each `d` in `heart_paths.json` with `PathParser().parsePathString(d).toPath()`, then union them
  (`Path.combine(PathOperation.Union, …)`) into **one clip path** for the stains layer.
- **Hit-testing a random spot:** sample `x ∈ [80, 380]`, `y ∈ [4, 440]` until the point is inside the union.
  Easiest reliable test: decode `heart_shade.webp` once and accept when pixel alpha at `(x*4, y*4)` > 0.5
  (its alpha is the exact silhouette). Up to 400 tries, fallback `(220, 300)`.
- Stains may overlap the edge; the clip trims them. That is intended (vessel tips must get covered too).

## 4. Dots

Model:

```kotlin
data class Shot(val id: String, val color: Ink /* WHITE | BLACK */, val x: Float, val y: Float,
                val r: Float /* viewBox units, 3..40 */, val a: Float /* 0.10..1.00 */, val t: Long,
                val sample: Boolean = false)
```

Each dot is a circle of radius `r` filled with a radial gradient, then drawn with `alpha = a`:

| stop | white | black |
|---|---|---|
| 0.00 | `#FFFDF8` α1.00 | `#050304` α1.00 |
| 0.55 | `#FBF6EC` α0.96 | `#070405` α0.96 |
| 0.80 | `#F3ECE0` α0.55 | `#0D0709` α0.55 |
| 1.00 | `#F3ECE0` α0.00 | `#0D0709` α0.00 |

Performance: with hundreds of dots, do not redraw all of them every frame. Keep an `ImageBitmap`
(stain cache, same 1600×1840 or display size) and draw new dots into it incrementally; rebuild it fully
only on undo/redo/jump/clear. Draw the cache clipped to the union path.

## 5. Throw animation (timings from the web version)

All in viewBox units. Arrow geometry: tip at origin pointing +x; shaft from x=-150 to -8 (wood `#7A5A34`,
3.2 wide, highlight line `#C8A25E` 55%); steel head triangle `(0,0) (-16,-6) (-12,0) (-16,6)` fill `#D9DCDF`;
two fletchings near x=-128…-160, fill white `#F6F1E7` or black `#111` to match the shot; nock at x=-164.

1. **Pick target** `(x, y)` via hit-test. **Commit the shot to history immediately** (so undo works mid-flight).
2. **Origin:** angle θ uniform in [-170°, -10°] (from above / sides), distance 420 from target.
   Arrow rotation = direction origin→target.
3. **Flight 260 ms**, ease `k = t²(1.4 − 0.4t)`. Tip travels to target + 12 units past it (penetration).
   Faint trail line behind it (white or brass, 18% alpha, 90 units, shrinking). Play `sfx_whoosh`.
4. **Embed:** at 92% of flight, hide the tip inside a radius-14 circle around the target (clip that circle
   out of the arrow) so the head looks buried.
5. **Impact** (play `sfx_hit_white|black`, `HapticFeedbackType.LongPress` or a short `VibrationEffect`):
   - Heart flinch 320 ms: translate 5 units along arrow direction and scale 0.985, then back
     (`cubic-bezier(.2,.8,.3,1)`).
   - Stage shake 260 ms: 3 px along direction, then −1.5, then 0.
   - Dot bloom 420 ms: radius `r × easeOutBack(t)` with `c = 1.9`:
     `1 + (c+1)(t−1)³ + c(t−1)²`.
   - Ripple ring 520 ms: radius `r → r + (1.6r + 26)·(1−(1−t)³)`, stroke 2.5 → 0.3, alpha 0.8 → 0.
   - 8 splinters 380 ms: short lines flying back toward the shooter (±63° spread), length 6 → 0, fading.
6. **Quiver 650 ms:** rotation `+ sin(t·9π)·4°·(1−t)²`, then **fade 380 ms** and remove.
7. Respect *Remove animations* (`Settings.Global.ANIMATOR_DURATION_SCALE == 0`): skip straight to the final dot.

Rapid taps must work: each arrow is its own animation (e.g. a `SnapshotStateList<ArrowAnim>` driven by
`Animatable`s); undo during flight removes the dot without crashing the animation.

Idle **heartbeat** on the whole heart (graphicsLayer scale, pivot ≈ 53% / 58%), 1.6 s loop:
`0%→1.0, 12%→1.022, 24%→0.995, 34%→1.012, 60–100%→1.0`. Animating `graphicsLayer` scale is cheap
because the baked layers are bitmaps.

## 6. History, undo, persistence

- State = `entries: List<Shot>` + `cursor: Int` (number of active shots). Active = `entries.take(cursor)`.
- New shot: `entries = entries.take(cursor) + shot; cursor = entries.size`.
- Undo `cursor--`, redo `cursor++`, jump to item i → `cursor = i + 1`. Clear: two-step confirm in UI.
- History list: newest first, number, colour chip, `r`, intensity %, time `HH:mm:ss`; items past the cursor
  are dimmed and struck through; the current one is outlined. "مثال" tag on sample entries.
- Hardware keyboard: `Ctrl+Z` undo, `Ctrl+Shift+Z` / `Ctrl+Y` redo, `W` white, `B` black
  (`Modifier.onPreviewKeyEvent`).
- Persist `entries`, `cursor`, size, intensity, sound on/off with DataStore (JSON via kotlinx.serialization)
  or Room. First launch seeds 5 sample shots marked `sample = true`.
- Counters: white / black totals and a ratio bar (white share vs black share).

## 7. UI

- Arabic, RTL. Dark single theme ("night chamber"): bg `#100D0F`, panels `#181316`, line `#3A2F33`,
  text `#EDE4D8`, muted `#9A8C82`, accent brass `#C8A25E`, pale `#F6F1E7`, soot `#0A0809`.
- Fonts (Google Fonts, downloadable or bundled): **Amiri** for the hadith/title, **IBM Plex Sans Arabic** for UI.
- Phone layout: hadith → heart stage (rounded 18 dp, subtle radial bg `#2A1A1E → #100D0F`) with counters
  in its top corners → ratio bar → two big buttons (white "بيضاء / أنكرها", black "سوداء / أُشرِبها") →
  size slider (3–40) + intensity slider (10–100%, step 5) with a live preview swatch → history sheet
  (bottom sheet or second tab) with undo / redo / sound / clear.
- Tablet / landscape: controls column beside the stage.

## 8. Suggested structure

```
ui/HeartScreen.kt         screen layout, sliders, buttons
ui/HeartStage.kt          Canvas: layers + stain cache + arrows + effects
ui/HistoryList.kt
anim/ArrowAnimator.kt     per-shot animation state
model/Shot.kt, HeartViewModel.kt (StateFlow<UiState>, undo/redo/jump)
data/HeartRepository.kt   DataStore persistence
geom/HeartGeometry.kt     path parsing, union clip, alpha-mask hit test
audio/Sfx.kt              SoundPool
```

## 9. Done when

- Clean / all-white / all-black renders match `reference-composite.png` (details and lighting visible in all three).
- 300+ dots stay smooth (stain cache), rapid taps animate independently.
- Undo/redo/jump/clear and persistence survive process death.
- Works in RTL, with *Remove animations* on, and with a hardware keyboard.
