---
name: concentric-corner-spacing
description: Use when nesting a rounded element inside a rounded container (a button/icon in a card, a row in a list, a toggle thumb in a track, a pill inside a capsule, a Dynamic-Island-style content swap) and the padding needs to read as visually even all the way around, including at the corners — not just on the straight edges.
license: MIT
---

# Concentric Corner Spacing

## Overview

When a rounded shape sits inside another rounded shape, the padding can be numerically identical on all four sides and *still* look wrong, because the gap changes as it travels around the corner. Fixing it means shrinking the inner radius by exactly the padding:

```
inner_radius = outer_radius − padding
```

That makes the two corner arcs **share one center point** — which is what "concentric" means, and why the gap between them stays constant in every direction.

```
   ╭──────────────────────────────────
  ╱   ╭──────────────────────────────      outer arc:  radius R
 │   ╱                                     inner arc:  radius R − p
 │  │
 │  │        · C   ←  ONE shared center for both arcs
 │  │
 │  │        every ray out of C crosses the inner arc, travels
 │  │        R − (R − p) = p, then crosses the outer arc.
 │  │        The gap is p on the flat edges AND around the curve.
```

## The Three Cases

Top-left corner of a container with `R = 20`, `padding = 16`. The flat-edge gap is always 16 — the only question is what the gap does at the corner apex (↖).

```
 ✗  r = 0  (square)          ✓  r = R − p = 4            ✗  r = R = 20 (reused)
    ╭──────────────             ╭──────────────              ╭──────────────
   ╱   ↓16                     ╱   ↓16                      ╱   ↓16
  │  ┌───────────             │   ╭───────────             │       ╭────────
  │  │                        │  ╱                         │     ╱
  │  │  ↖ 14.3                │ │   ↖ 16.0                 │   │    ↖ 22.6
  │→16│                       │→16│                        │→16│
       PINCHED                     CONSTANT                      BALLOONED
   corner gap is TIGHTER       gap holds all the way        corner gap is LOOSER
   than the edge gap           round — this is the goal     than the edge gap
```

Both failure modes read as "the padding looks off at the corners" but they fail in **opposite directions**, and that tells you which way to correct:

| Inner radius is… | Corner gap | Looks like |
|---|---|---|
| too small (`< R − p`) | shrinks below `p` | inner corner poking out toward the container's curve |
| exactly `R − p` | equals `p` | even, calm, intentional |
| too large (`> R − p`) | grows above `p` | a wedge of dead space bulging at each corner |

Reusing the container's radius on the inner element — the single most common mistake — is the **right-hand** case: it balloons.

## When to Use

- A card, panel, or screen bezel has rounded corners, and something (a button, thumbnail, list row, icon chip) is inset from its edge
- A toggle/slider thumb or knob needs consistent clearance from its track's edge, including where the track curves
- Multiple stacked rows live inside a rounded container, and the first/last row's outer corners must blend into the container's corners
- A capsule/pill (Dynamic-Island-style) morphs its content (icon → text → icon) and the content must stay inset by a constant margin regardless of shape
- Corners look bulged or pinched even though the padding value is identical on every side

**Not needed** when the container's corners are square, or when nothing rounded is nested inside anything rounded.

## The Formula

```
inner_radius = outer_radius − padding
```

Solve for whichever variable you're missing:

| Known | Unknown | Formula |
|---|---|---|
| outer radius, padding | inner radius | `inner = outer − padding` |
| inner radius, padding | required outer radius | `outer = inner + padding` |
| inner radius, outer radius | padding that makes them concentric | `padding = outer − inner` |

**Degenerate case:** if `outer − padding <= 0`, the padding has eaten the whole curve and no radius can be concentric — a square inner corner (radius 0) is the correct, honest answer here. It will still pinch slightly at the apex; that's unavoidable, and it's why deeply-padded elements inside gently-rounded containers look fine with square corners.

**Cap:** an inner element can't be more than half-round. If `outer − padding >= inner_height / 2`, clamp to `inner_height / 2` — it's a stadium/pill.

## Edge Cases

- **Non-uniform padding:** each corner's radius is driven by the paddings meeting *at that corner*. If they differ (e.g. `padding-top: 8`, `padding-left: 16`), no single radius is concentric on both axes — use the **smaller** padding so the shape never overshoots the guide on the tighter side, or accept an elliptical corner.
- **Stacked rows in a rounded card:** only the corners that actually touch the container's curve need the formula.

```
   ╭──────────────────────────────╮   card:  R = 16, padding = 8
   │ ╭──────────────────────────╮ │   ← top row's OUTER corners hug the card's
   │ │  row 1                   │ │      curve → r = 16 − 8 = 8
   │ ╰──────────────────────────╯ │
   │ ╭──────────────────────────╮ │   ← interior rows never touch the card's
   │ │  row 2                   │ │      curve → any small radius is fine
   │ ╰──────────────────────────╯ │
   ╰──────────────────────────────╯
```

- **Content-swap capsules (Dynamic Island):** the outer capsule is a full stadium (`R = height / 2`). Recompute the inset radius per content type — an icon, a text run, and a progress bar have different effective heights, so a single hardcoded inner radius will only be concentric for one of them.

## Implementation

**CSS** — derive it, never hardcode it:
```css
.card {
  --r: 24px;
  --pad: 12px;
  border-radius: var(--r);
  padding: var(--pad);
}
.card > .inset-el {
  border-radius: calc(var(--r) - var(--pad));  /* clamps naturally: 0 if pad >= r */
}
```

**Tailwind** — same idea via arbitrary values: `rounded-[calc(var(--r)-var(--pad))]`.

**SwiftUI** — `ContainerRelativeShape` does this for you in contexts that support it (widgets, Dynamic Island). Elsewhere, apply the formula by hand.

**Figma / no-formula tools** — compute per component and store the *padding* as the token, deriving radius from it. Storing the inner radius as its own token breaks the moment the container's radius changes.

## Common Mistakes

| Mistake | What it does to the corner | Fix |
|---|---|---|
| Inner element reuses the container's radius | Gap **balloons** at the corner (`p√2` instead of `p`) — a visible wedge of dead space at each corner | `inner = outer − padding` |
| Inner element left square inside a heavily rounded container | Gap **pinches** at the corner — the sharp corner crowds the container's curve | Give it the computed radius |
| Measuring padding only on the straight edges | Edge padding can be perfect while corner padding is 40% off — and the eye reads curves first | Check the gap at the corner apex, not just top/left/right/bottom |
| Storing the inner radius as a design token | Breaks concentricity the moment the outer radius changes | Store padding; derive radius with `calc()` |
| Applying the formula to an inner element that doesn't touch the curve | Wasted effort, and interior elements end up with weirdly specific radii | Only corners that hug the container's curve need it |

## Reference Frames

`references/` holds five stills from a design-talk walkthrough of this technique — useful when explaining it to a human or spot-checking a design:

- `corner-concentricity-formula.png` — single corner, guide line at constant offset, inner corner sitting exactly on it
- `list-rows-corner-match.png` — stacked rows in a rounded card, top row's outer corner matched to the card
- `thumb-equal-padding.png` — circular thumb in a pill track; tick marks prove constant clearance around the curve
- `capsule-inset.png` — pill-shaped inset inside a larger capsule
- `dynamic-island-content-swap.png` — capsule content swapping icon/text while holding a consistent inset

`references/grids/` has the full source set (158 frames as seven 25-per-sheet contact sheets, `grid_01.png`–`grid_07.png`) if you need the whole progression — the complete Dynamic Island icon→text morph, or every intermediate step of the thumb rounding a corner — rather than just the end-state stills.
