# ocean-skills

Agent skills for design and frontend engineering, built for the [open agent skills ecosystem](https://agentskills.io).

## Install

```bash
npx skills add risingtides-dev/ocean-skills
```

Install a single skill:

```bash
npx skills add risingtides-dev/ocean-skills --skill concentric-corner-spacing
```

Or try one without installing:

```bash
npx skills use risingtides-dev/ocean-skills@concentric-corner-spacing | claude
```

## Skills

| Skill | Use when |
|---|---|
| [`concentric-corner-spacing`](skills/concentric-corner-spacing) | Nesting a rounded element inside a rounded container — a thumbnail in a card, a thumb in a toggle track, rows in a list, content inside a Dynamic-Island-style capsule — and the padding needs to read as visually even around the corners, not just on the straight edges. |
| [`prove-the-test-can-fail`](skills/prove-the-test-can-fail) | Trusting a test suite, mutation gate, or CI check as evidence — before calling work done, porting code between languages, or claiming coverage. Catches checks that report success while proving nothing. |

## Why "concentric corners" matters

Padding can be numerically identical on all four sides and still look wrong, because the gap between two rounded shapes changes as it travels around the corner. With an outer radius of 20 and 16px of padding:

| Inner radius | Gap at the corner (flat-edge gap is 16) | Result |
|---|---|---|
| `0` — square | 14.3 | pinched |
| `4` — `outer − padding` | 16.0 | **correct** |
| `20` — reuses the container's radius | 22.6 | ballooned |

Reusing the container's radius on the inner element is the most common version of this bug, and it fails in the *opposite* direction from leaving the corner square — so knowing which way it's off tells you which way to fix it. The skill carries the formula, the geometry, the edge cases (non-uniform padding, stacked rows, content-swap capsules), and CSS/Tailwind/SwiftUI implementations.

## Structure

Skills follow the [Agent Skills specification](https://agentskills.io/specification):

```
skills/
  <skill-name>/
    SKILL.md        # required: frontmatter + instructions
    references/     # optional: supporting material loaded on demand
```

## License

MIT
