# Puzzle Generator Redesign

**Date:** 2026-03-18
**Scope:** Redesign the difficulty levels, toggles, and game mechanic of the puzzle generator tab in `mathcat-helper/index.html`

---

## Goal

Replace the current 3-level difficulty system (Easy/Medium/Hard) with a 3-level find-many system that better scaffolds player skill. All levels now share a common "find-many + reveal all" mechanic.

---

## Levels

### Easy
- **Operations:** `+` only by default; toggle to also allow `−`
- **Cards required:** Any non-empty subset (not all cards required)
- **Hand size:** 4 cards
- **Goal:** Find as many solutions as possible, then reveal all

### Medium
- **Operations:** All four (`+ − × ÷`)
- **Cards required:** Toggle — default "any subset", can switch to "must use all 5 cards"
- **Hand size:** 5 cards
- **Goal:** Find as many solutions as possible, then reveal all

### Hard
- **Operations:** All four (`+ − × ÷`)
- **Cards required:** Must use all 6 cards
- **Complexity filter:** Solution expression must have `parenDepth ≥ 2` AND at least 2 instances of `×` or `÷`
- **Hand size:** 6 cards
- **Goal:** Find as many solutions as possible (typically very few — the challenge is finding even one)
- **No toggles** — rules are fixed

---

## Find-Many Mechanic (all levels)

1. Player configures level + toggles, picks target, taps **Generate Puzzle**
2. App generates and displays:
   - Target number
   - Hand of cards
   - **Rules chips** (see below)
3. Player tries to find solutions mentally or on paper
4. Player taps **Reveal All** → app shows every valid solution + total count
   - Solutions are sorted by expression length (shortest first), same as solver

The "Reveal All" button replaces the current "Show Answer" / "Hide Answer" toggle. Once revealed, solutions stay visible (no re-hiding needed).

---

## Rules Chips

Shown directly beneath the puzzle cards after generation. Small, non-interactive chips summarising the constraints that were applied.

**Examples:**

| Situation | Chips shown |
|---|---|
| Easy, subtraction off | `+` · `any cards` |
| Easy, subtraction on | `+ −` · `any cards` |
| Medium, any subset | `+ − × ÷` · `any cards` |
| Medium, all cards | `+ − × ÷` · `all 5 cards` |
| Hard | `+ − × ÷` · `all 6 cards` · `complex` |

---

## Toggles

Toggles live inside the difficulty card (between difficulty selection and the Generate button). Only shown for the currently selected difficulty.

- **Easy:** checkbox `Include subtraction (−)`
- **Medium:** checkbox `Must use all 5 cards`
- **Hard:** *(none — rules are fixed)*

---

## Generator Algorithm Changes

### Easy
- Build a random 4-card hand
- Run `solveHand(hand, target)` with ops restricted to `+` and optionally `−`
- Filter: at least 1 solution exists
- If toggle-subtraction is off: filter solutions by `!/[−]/.test(expr)`

### Medium
- Build a random 5-card hand
- Run `solveHand(hand, target)` with all ops
- If "must use all cards" toggle is on: filter solutions where card count in expr === 5
- Require at least 1 solution passes the filter

### Hard
- Build a random 6-card hand
- Run `solveHand(hand, target)` with all ops
- Filter solutions where: all 6 cards used AND `parenDepth ≥ 2` AND `multDivCount ≥ 2`
- Require at least 1 solution passes the filter
- Max 800 generation attempts (same as current)

### Solver restriction for Easy (addition/subtraction only)
The existing `solveHand` tries all 4 operators. For Easy, pass an `allowedOps` parameter (or post-filter) to restrict to `['+']` or `['+', '-']`. Post-filtering is simpler: run the full solver, then filter solutions by regex.

---

## "All cards used" check

For Medium (when toggle is on) and Hard, a solution uses all cards when the count of number tokens in the expression equals the hand size.

```js
function countNums(expr) {
  return (expr.match(/\d+/g) || []).length;
}
```

Already exists as part of `scoreExpr` (`numCount`). Reuse it.

---

## Removed

- `matchesDiff(expr, diff)` — replaced by per-level filters above
- `scoreExpr` difficulty scoring — replaced by explicit constraints
- "Show Answer" / "Hide Answer" toggle — replaced by "Reveal All" (one-way)

---

## Unchanged

- Solver algorithm (`solveHand`, `searchNodes`, `combine`) — no changes
- Target range: 6–20
- Bait card pool: 1, 2, 3, 5, 7, 9 (up to 6 copies each)
- Max hand size: 6 (Hard uses the max; Easy and Medium use fewer)
- Tab structure (Solver / Generator) — Generator tab only changes
- Solver tab — no changes
