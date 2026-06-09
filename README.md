# aura-effects

> **Deprecated:** This repository has been migrated to the [aura-prototype](https://github.com/aaronsilinskas/aura-prototype) mono-repo for faster iteration. Please use that repository going forward.

---

A lightweight animation library for RGB LED strips, designed for spell and
elemental effects on constrained hardware such as CircuitPython microcontrollers.
It separates **what an effect looks like** (shapes, palettes) from **how it
behaves over time** (steps), so visual effects are easy to compose and reuse
without allocating objects on every frame.

---

## Concepts

### Level

Every effect is parameterized by a **level** in `[1, 10]`. Level 1 is the
quietest, most subtle expression of an effect; level 10 is the most intense.
The level drives all the `level_lerp` calls inside each element builder — you
do not need to touch individual parameters to tune intensity.

### Effect and Steps

An `Effect` pairs a **shape** (a function `float → float` mapping strip
position to a `[0.0, 1.0]` value) with an ordered list of **steps**. During each frame:

- `Effect.update(state, timer)` — advances the active step. A step returns
  `True` when it is done, handing control to the next step in the sequence,
  which wraps around.
- `Effect.value(state, position)` — samples the effect at a strip position in
  `[0.0, 1.0]`. All steps get a chance to shift the position
  (`adjust_position`) and then modify the resulting value
  (`adjust_value`), in order. This value is typically used with a palette lookup
  or to create grayscale color.

The same `Effect` instance can drive multiple independent animations at once
because all mutable state lives in an `EffectState` object that you own.

### EffectState and EffectTimer

- **`EffectState`** — holds every piece of mutable per-frame data: which step
  is active, per-step working data, and any shared state that steps
  communicate through (e.g. velocity). Create one per animation and pass it
  through every `EffectRenderer.update` / `render` call. When switching to a new effect at
  runtime, always create a fresh `EffectState` to avoid stale data.
- **`EffectTimer`** — tracks elapsed time and overall progress for a duration.
  Create one globally, advance it with `timer.update(elapsed_seconds)` once
  per frame, and pass it unchanged to every `update` call.

### Shapes

`Shape` is a collection of static factory methods that each return an
`EffectShapeFunc` (`float → float`). Built-in shapes include:

| Shape                                 | Description                                                |
| ------------------------------------- | ---------------------------------------------------------- |
| `Shape.none()`                        | Always `0.0` — invisible baseline                          |
| `Shape.gradient()`                    | Gamma-corrected ramp, dim at position `0` to bright at `1` |
| `Shape.centered_gradient()`           | Peaks at the center, fades to both edges                   |
| `Shape.padded(padding, inner)`        | Dead zones at each end; inner shape fills the middle       |
| `Shape.reverse(inner)`                | Mirrors any shape left-to-right                            |
| `Shape.sine(frequency)`               | Sine wave oscillating between `0` and `1`                  |
| `Shape.checkers(value, count, width)` | Repeating flat-top segments with fade tails                |

Shapes can be composed by wrapping: `Shape.padded(0.2, Shape.reverse(Shape.gradient()))`.

### Palettes

A `Palette` maps a normalized value float to a packed 24-bit RGB integer.
`PaletteLUT256` is the standard implementation: you supply a compact
`bytes` definition of color stops (`[index, r, g, b, ...]`) and it expands
them into a 256-entry lookup table at construction time, making per-pixel
lookups a single array index.

### Built-in Steps

Steps live in `effects/steps/` and are created via factory functions:

| Factory                                           | What it does                                                     |
| ------------------------------------------------- | ---------------------------------------------------------------- |
| `hide(duration)`                                  | Suppresses output for `duration` seconds, then advances          |
| `call(callback)`                                  | Fires a callback once on activation, then immediately advances   |
| `duration(duration, steps, persist_steps)`        | Runs a timed sub-sequence of steps                               |
| `rotate(rps)`                                     | Continuously shifts sampling position at `rps` rotations/second  |
| `accelerate(start, end, direction)`               | Ramps rotational speed from `start` to `end` over the timer      |
| `face_forward()`                                  | Mirrors the sampling direction when moving in reverse            |
| `set_position(position)`                          | Shifts sampling position by a fixed or dynamic offset each frame |
| `multiplier(start, end)`                          | Scales output brightness from `start` to `end` over the timer    |
| `flame(spark_count, heat_rate, ...)`              | Overlays a heat-diffusion flame simulation                       |
| `sparkle(sparkle_count, ...)`                     | Overlays randomly spawning sparkles that fade in and out         |
| `drift_noise(resolution, drift_speed, amplitude)` | Overlays slowly drifting random noise                            |

### Renderers

`EffectRenderer` pairs an `Effect` with a `Palette` and exposes the two calls
you make in the render loop:

- `renderer.update(state, timer)` — once per frame
- `renderer.render(state, position)` → packed RGB int — once per pixel

`AdditiveMergeRenderer` and `AverageMergeRenderer` combine multiple renderers
into one, summing or averaging their RGB output per pixel respectively.

### RendererConfig

`RendererConfig` bundles the settings passed to element builder functions:

| Field         | Description                                                        |
| ------------- | ------------------------------------------------------------------ |
| `level`       | Effect intensity `[1, 10]`, clamped automatically                  |
| `pixel_count` | Number of physical LEDs; used to size buffers that align to pixels |
| `resolution`  | Number of internal simulation cells (higher = smoother)            |
| `listeners`   | Optional callbacks invoked when the effect fires named events      |

### Dynamic Values

Many step parameters accept a `DynamicValue`: either a plain `float` or a
zero-argument callable returning a `float`. Use `ValueGenerator` to create
them:

| Helper                   | Returns                                                              |
| ------------------------ | -------------------------------------------------------------------- |
| `VG.random(min, max)`    | A callable that samples `uniform(min, max)` each time it is resolved |
| `VG.random_choice(list)` | A callable that picks randomly from a list                           |
| `VG.resolve(value)`      | Evaluates either form — call this when you need the concrete float   |

Dynamic values are **resolved** as needed by the step, which can happen
at step initialization, or during a frame. Using `VG.random(...)` for a step parameter
means the value is randomized each time the step needs a new value.

---

## Built-in Elements

Ten ready-to-use elemental effects are provided in `effects/elements/`, each
with visual character that varies with level:

| Element     | Appearance                                                  |
| ----------- | ----------------------------------------------------------- |
| `air`       | Green-white breezes that sweep and dissolve                 |
| `dark`      | Sparse deep-red sparks flickering against black             |
| `earth`     | Slow, broad smolder in golds with earthy greens             |
| `fire`      | Turbulent flame in red, orange, and golden yellow           |
| `gravity`   | Deep-space nebula with twinkling white stars                |
| `ice`       | Cold, rotating flame from dark teal to cyan and white       |
| `light`     | Tight, rapid white flickers like an overdriven flash        |
| `lightning` | Blinding orange flashes striking at random positions        |
| `time`      | Drifting amber-brown sand overlaid with rotating tick marks |
| `water`     | Blue-to-cyan flame that periodically reverses direction     |

Use the registry to look up any element by name:

```python
from effects.elements.registry import get_element_builder, list_element_names
```

`get_element_builder(name)` returns the builder callable, or raises
`ValueError` for unknown names. `list_element_names()` returns all valid keys.

---

## The Render Loop

The fundamental per-frame pattern is:

1. Compute `elapsed` seconds since the last frame.
2. Call `timer.update(elapsed)` once.
3. Call `renderer.update(state, timer)` once.
4. For each LED at index `i`, compute `position = i / num_leds`, call
   `renderer.render(state, position)`, and write the returned packed RGB int
   to the pixel buffer.
5. Flush the pixel buffer to the hardware.

When switching effects or levels at runtime, reset `EffectState` before
constructing the new renderer, and run `gc.collect()` on CircuitPython to
reclaim memory from the previous effect's allocations.

---

## Development

Install development dependencies:

```bash
uv sync --group dev
```

Run tests:

```bash
uv run pytest
```

Run tests with coverage:

```bash
uv run pytest --cov=effects
```

Run a specific test file:

```bash
uv run pytest tests/test_smoke.py
```
