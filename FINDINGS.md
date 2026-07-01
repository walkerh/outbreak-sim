# Code Review Findings

Branch: `claude` vs `main`  
Reviewed: 2026-06-28  
Scope: preset dropdown + logarithmic μ slider (commits 8289fae–1bb4596)

---

## 1. Preset change does not call `init()` — mid-run parameter mixing (CONFIRMED)

**File:** `outbreak-sim.html` ~line 584  
**Severity:** High — silently corrupts simulation results

The preset `change` handler updates slider values but never calls `init()` or `stop()`. The Run button only calls `init()` when `isOver() && day > 0`; a paused-but-live simulation falls into the `else → start()` branch with no reset.

**Scenario:** User pauses a Measles outbreak mid-run, selects Smallpox, clicks Run. `stepModel` now applies Smallpox parameters (R0=6, μ=30%) against agents already distributed in a Measles outbreak state. The chart conflates both diseases; Smallpox outcomes are wrong.

**Fix:** In the preset handler, call `stop()` then `init()` before updating sliders (or after, before `start()`). Alternatively, call `init()` unconditionally when a preset is selected and the simulation is not at day 0.

---

## 2. Preset dropdown not reset when sliders are manually adjusted (CONFIRMED)

**File:** `outbreak-sim.html` ~line 569  
**Severity:** Medium — misleading UI state

`wireSlider` only updates the display `<span>`; it has no logic to reset `#s-preset`. After loading a preset, manually dragging any parameter slider leaves the dropdown showing the preset name even though the parameters no longer match.

**Scenario:** User loads Measles, drags R₀ to 3. Dropdown still shows "Measles". A screenshot or description of the setup incorrectly implies the simulation is using Measles parameters.

**Fix:** Add `$('s-preset').value = ''; $('preset-desc').textContent = '';` inside `wireSlider`'s `update` closure (or in separate `input` listeners on the four parameter sliders).

---

## 3. Measles preset: displayed μ (0.14%) disagrees with description text (~0.15%) (PLAUSIBLE)

**File:** `outbreak-sim.html` ~line 526  
**Severity:** Low-Medium — visible inconsistency on every Measles load

Log-scale round-trip: `sliderFromMu(0.0015)` → position 32 → `muFromSlider(32)` ≈ 0.00144 → `muDisplay` → `"0.14%"`. The preset `desc` string hardcodes `"CFR ~0.15%"`. The sidebar shows `μ = 0.14%` while the description directly below reads `CFR ~0.15%`.

**Fix (option A):** Adjust the Measles `mu` value to one that round-trips cleanly (e.g. `mu: 0.00144`), then update `desc` to match.  
**Fix (option B):** Generate the CFR string in the `desc` field dynamically from `muDisplay(p.mu)` rather than hardcoding it.

---

## 4. `sliderFromMu` returns 0 (zero mortality) for any `mu` below the scale floor (CONFIRMED — latent)

**File:** `outbreak-sim.html` ~line 554  
**Severity:** Medium — silent data corruption for future presets; safe today

For `0 < mu < ~9.6e-5` (~0.0096%), `Math.log10(mu) + 4 < 0`, the formula goes negative, `Math.round` gives ≤ 0, and `Math.max(0, …)` clamps to 0. `muFromSlider(0)` returns 0, silently eliminating mortality instead of clamping to the minimum non-zero position (s=1).

All four current presets are safely above the threshold (floor is flu_2009 at μ=0.0002). One new preset below ~0.01% CFR would trigger silent zeroing.

**Fix:** Change `Math.max(0, …)` to `mu > 0 ? Math.max(1, …) : 0` so a positive-but-tiny `mu` snaps to position 1 rather than 0.

```js
// current
return Math.max(0, Math.min(100, Math.round(1 + 99 * (Math.log10(mu) + 4) / 3.699)));
// fixed
return mu > 0 ? Math.max(1, Math.min(100, Math.round(1 + 99 * (Math.log10(mu) + 4) / 3.699))) : 0;
```

---

## 5. `select { background: #0a1828 }` — hardcoded color not in CSS variable system (CONFIRMED)

**File:** `outbreak-sim.html` ~line 149  
**Severity:** Low — CLAUDE.md convention violation

CLAUDE.md: *"All colors are defined as CSS custom properties on `:root`."* The value `#0a1828` matches no defined variable (`--bg` = `#06090f`, `--panel` = `#0c1220`, `--dim` = `#1e3448`). It is a novel one-off color invisible to any future theme change.

**Fix:** Add a new CSS variable (e.g. `--input: #0a1828`) in `:root` and use `var(--input)` here and in the `button { background: … }` rule that also uses `#0a1828`.

---

## 6. `select option { background: #0c1220 }` — should be `var(--panel)` (CONFIRMED)

**File:** `outbreak-sim.html` ~line 162  
**Severity:** Low — CLAUDE.md convention violation

`#0c1220` exactly equals `--panel` but bypasses the variable. A future change to `--panel` will not propagate to the select option background.

**Fix:** Replace `background: #0c1220` with `background: var(--panel)`.

---

## 7. `params` object built on every `requestAnimationFrame`, not per simulated day (CONFIRMED)

**File:** `outbreak-sim.html` ~line 869  
**Severity:** Low — wasted work on idle frames

The `params` object (five DOM `.value` reads + `muFromSlider` / `Math.pow`) is constructed unconditionally at the top of `loop()`, before the `while (tickAccum >= msPerDay)` check. At 60 fps with speed = 1 day/sec, ~59 of every 60 frames do this work for nothing.

**Fix:** Move the `params` block inside the `while` loop body, or guard it with `if (tickAccum >= msPerDay)`.

---

## 8. x-axis label loop produces NaN on very narrow viewports (PLAUSIBLE)

**File:** `outbreak-sim.html` ~line 787  
**Severity:** Low — degenerate edge case; canvas silently discards

`labelCount = Math.min(Math.floor(cw / 50), days - 1)`. When `cw < 50px`, `labelCount = 0`. The loop runs once (`i <= labelCount` with `i=0`) and computes `d = Math.round(0 / 0 * …) = NaN`. Canvas `fillText('dNaN', NaN, y)` is a no-op, so there's no crash — just a missing x-axis label.

**Fix:** `if (labelCount < 1) labelCount = 1;` after the `labelCount` assignment, or skip the label loop when `labelCount === 0`.

---

## Resolved / Refuted

- **Double-click Run race** — REFUTED. `start()` sets `running = true` synchronously before the first RAF call; the `if (running) return` guard in `start()` blocks any duplicate invocation from the 50 ms timeout.
