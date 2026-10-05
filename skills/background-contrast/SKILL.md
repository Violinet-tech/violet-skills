---
name: background-contrast
description: |
  Default opacity and scrim values for pages and app UI that sit on a
  photographic or video background. Use whenever a background image is "barely
  visible" or washed out, when text over artwork needs contrast, when adding a
  hero/plate image behind a layout, or when sidebars, panels and cards need to
  read against a busy backdrop. Carries the compounding-gradient trap and the
  arithmetic to check any stack.
---

# Backgrounds you can actually see

The failure this exists to prevent: buying a background, covering it to make
text legible, and shipping a page where the artwork is invisible. It has
happened twice in this workspace, both times because **stacked translucent
layers compound and the numbers were never worked out**.

## The arithmetic, which is the whole skill

Opacities multiply; they do not add. For a plate at opacity `p` under scrims of
opacity `s1, s2, …`:

```
visible fraction of the artwork = p x (1 - s1) x (1 - s2) x ...
```

Two real cases from this workspace:

```
radial .86 over linear .93          -> 1 - (0.14 x 0.07) = 99% opaque
plate .34 under a .86-.93 scrim     -> artwork visible at 2.4 - 6.8%
```

Both looked reasonable as individual numbers. Neither was.

**Always compute the product before shipping.** One line:

```python
p, scrims = 0.55, [0.58]
print(f"{p * math.prod(1 - s for s in scrims) * 100:.0f}% visible")
```

## Defaults

| | value | notes |
|---|---|---|
| **Artwork visible** | **25–45%** | the target the numbers above must land in |
| Full-page scrim | **0.28 top → 0.62 bottom** | one gradient, not two stacked |
| Plate opacity (app UI) | **0.55** | when the plate is a separate layer |
| Local pool behind bare text | 0.55–0.60 radial | spend darkness *here*, not everywhere |
| Panels / cards over artwork | **0.80–0.90** | they carry their own ground |
| Sticky nav / sidebar | 0.90 → 0.42 gradient + `backdrop-filter: blur(14px)` | |
| Text shadow, bare text only | `0 2px 26px rgba(bg,.85)` | display type; `0 1px 16px` for body |

## The principle

**Win contrast locally, not globally.** A full-page wash is the blunt fix: it
costs the artwork everywhere to solve a problem that exists in two places.

1. Keep the global scrim light.
2. Put a **soft radial pool** behind the one or two runs of text that sit on
   bare artwork (usually the hero copy).
3. Everything else already has a panel — raise *those* backgrounds instead.
4. Add a text-shadow to display type as the last 10%.

## Applying it

```css
/* the plate: a still or a looping video */
.plate{
  position:fixed; inset:0; z-index:-3; pointer-events:none;
  width:100%; height:100%; object-fit:cover;
  background:url('bg.webp') center/cover no-repeat;
}
/* ONE scrim. Darkness concentrated where bare text lands. */
.scrim{
  position:fixed; inset:0; z-index:-1; pointer-events:none;
  background:
    radial-gradient(75% 55% at 26% 34%, rgba(6,9,26,.58), transparent 72%),
    linear-gradient(180deg, rgba(6,9,26,.28) 0%, rgba(6,9,26,.40) 48%, rgba(6,9,26,.62) 100%);
}
```

That radial-over-linear pair compounds to 68% only where the hero copy sits and
28–62% everywhere else — which is the point of splitting them.

## Checking it

Contrast still has to pass: **4.5:1 for body text, 3:1 at 24px+** (5.5:1 is a
better floor in practice — see the stray-overlay story below, where 4.91:1 passed AA and was
still too dim at 11px). Over a background the worst case is one of the
artwork's two EXTREMES composited under the panel — which extreme depends on
which way the palette runs, and checking only one is a trap:

- **Light text on a dark panel** loses contrast where the artwork is
  **brightest** (a light source bleeding through drags the surface up toward
  the text).
- **Dark text on a light panel** loses it where the artwork is **darkest**
  (a dark pixel drags the surface down toward the text).

A check written for one direction and never revisited becomes silently wrong
the moment a palette flips — it would pass the exact failure it exists to
catch. Check both extremes (composite pure white AND pure black under the
panel at its real alpha) and take whichever is worse; don't assume.

A good place to put this is a small script run in CI or `npm run check:contrast`
that covers token-on-panel pairs: a direction-agnostic worst-case check, plus a
literal-rgba assertion so a `.panel` rule can't silently drift from its own token.
It does **not** cover text sitting
directly on bare backdrop with no panel under it — do that arithmetic by hand.

## Before touching the palette, rule out a stray overlay

**"No opacity" is not always a contrast problem.** One app's dark palette was
rebuilt, failed, and reverted to light four times before the real bug was
found: `#modal { display: flex }` outranked `.hidden { display: none }` (an ID
selector beats a class one), so the approval overlay stayed painted over the
whole app on every one of those runs — a 70%-black wash greying every panel,
which looks *identical* to a genuine value-separation failure whether or not
the palette underneath is sound. Four rounds of retuning scrim and panel
numbers never touched the actual bug, because it wasn't in the palette.

A token-pair contrast check cannot see this class of bug — it only reads CSS
custom properties, not runtime state, a stray full-screen element, or a
hardcoded fill that never used a token at all. Two other real examples from
this workspace: `calc()` inside `color-mix()` computed correctly under
`getComputedStyle` and painted nothing, and `backdrop-filter` does the same
under software compositing — both leave a panel fully transparent while every
tool reports the palette as applied. Before redesigning contrast values in
response to a "washed out" or "no opacity" report, rule out: a modal/overlay
with a specificity bug, a hardcoded (non-token) fill, and a CSS feature that
silently no-ops under the current renderer. Only redo the arithmetic once
you've confirmed the palette is what's actually on screen.

Related: `seamless-loop-video` for when the plate is a video. Remove any CSS
drift from a plate that moves on its own.
