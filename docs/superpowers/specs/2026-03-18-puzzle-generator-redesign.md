# Puzzle Generator Redesign

**Date:** 2026-03-18
**Scope:** Redesign the difficulty levels, toggles, and game mechanic of the puzzle generator tab in `mathcat-helper/index.html`

---

## Goal

Replace the current 3-level difficulty system (Easy/Medium/Hard) with a 3-level find-many system that better scaffolds player skill. All levels share a common "find-many + reveal all" mechanic.

---

## Levels

### Easy
- **Operations:** `+` only by default; toggle to also allow `−`
- **Cards required:** Any non-empty subset (not all cards required)
- **Hand size:** 4 cards *(changed from current Easy=6)*
- **Goal:** Find as many solutions as possible, then reveal all

### Medium
- **Operations:** All four (`+ − × ÷`)
- **Cards required:** Toggle — default "any subset", can switch to "must use all 5 cards"
- **Hand size:** 5 cards *(unchanged)*
- **Goal:** Find as many solutions as possible, then reveal all

### Hard
- **Operations:** All four (`+ − × ÷`)
- **Cards required:** Must use all 6 cards
- **Complexity filter:** Solution expression must have `parenDepth ≥ 2` AND `multDivCount ≥ 2`
- **Hand size:** 6 cards *(changed from current Hard=4)*
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
4. Player taps **Reveal All** → app shows every valid solution + total count, sorted by expression length (shortest first)
5. If 0 solutions exist after filtering (should not occur due to generation guarantee, but as a fallback): show "😿 No solutions found." in place of the solution list

The **Reveal All** button replaces the current "Show Answer" / "Hide Answer" button in the same position within the puzzle output card (below the puzzle cards, centered).

- Before reveal: button reads "👁 Reveal All", solution area is hidden
- After reveal: button is hidden (removed from view); solution area shows all solutions + count. Solutions stay visible — there is no hide/toggle.
- If 0 solutions exist after filtering (fallback only — generator guarantees ≥1): button is hidden immediately; solution area shows "😿 No solutions found." instead of the list.
- Generating a new puzzle resets to the pre-reveal state (button visible, solution area hidden).

The solution list displayed after reveal is sorted by expression length shortest-first. The `solveHand` solver already returns solutions in this order; post-filtering preserves that order. No re-sort is needed after filtering.

---

## Rules Chips

Shown inside the puzzle output card, directly beneath the puzzle cards, after generation. They update each time a new puzzle is generated. They are not shown before the first puzzle is generated.

**Examples:**

| Situation | Chips shown |
|---|---|
| Easy, subtraction off | `+` · `any cards` |
| Easy, subtraction on | `+ −` · `any cards` |
| Medium, any subset | `+ − × ÷` · `any cards` |
| Medium, all cards | `+ − × ÷` · `all 5 cards` |
| Hard | `+ − × ÷` · `all 6 cards` · `complex` |

The `complex` chip renders with label text **"complex"** and a `title` tooltip: `"depth ≥ 2, uses 2+ × or ÷"`. This gives players who hover/long-press a concrete definition.

---

## Toggles

Toggles live inside the difficulty card (between difficulty selection and the Generate button). Only shown for the currently selected difficulty.

- **Easy:** checkbox `Include subtraction (−)`
- **Medium:** checkbox `Must use all 5 cards`
- **Hard:** *(none — rules are fixed)*

**Toggle persistence:** Toggle state persists for the duration of the session (i.e. the player's choice is remembered across puzzle generations). Toggle state resets to default when the player switches to a different difficulty level.

---

## Generator Algorithm Changes

### Easy
- Build a random 4-card hand
- Run `solveHand(hand, target)` with all ops (no solver changes needed)
- Post-filter solutions:
  - If subtraction toggle is **off**: keep only solutions where `!/[-]/.test(expr)` (U+002D hyphen-minus, the character produced by the solver)
  - If subtraction toggle is **on**: keep all solutions
- Require at least 1 solution passes the filter; retry up to 800 attempts on failure
- On failure after 800 attempts: show toast "😿 No Easy puzzle found for target N. Try a different target."

### Medium
- Build a random 5-card hand
- Run `solveHand(hand, target)` with all ops
- Post-filter solutions:
  - If "must use all cards" toggle is **on**: keep only solutions where `countNums(expr) === 5`
  - Otherwise: keep all solutions
- Require at least 1 solution passes the filter; retry up to 800 attempts on failure
- On failure after 800 attempts: show toast "😿 No Medium puzzle found for target N. Try different settings."

### Hard
- Build a random 6-card hand
- Run `solveHand(hand, target)` with all ops
- Post-filter solutions where ALL of:
  - `countNums(expr) === 6` (all cards used)
  - `maxParenDepth(expr) >= 2`
  - `countMultDiv(expr) >= 2`
- Require at least 1 solution passes the filter; retry up to 800 attempts on failure
- On failure after 800 attempts: show toast "😿 No Hard puzzle found for target N. Try a different target."

---

## Helper Functions

### `countNums(expr)`
Counts number tokens in an expression. Extracted as a standalone helper (previously embedded in `scoreExpr`).

```js
function countNums(expr) {
  return (expr.match(/\d+/g) || []).length;
}
```

### `countMultDiv(expr)`
Counts multiplication and division operators in an expression.

```js
function countMultDiv(expr) {
  return (expr.match(/[×÷]/g) || []).length;
}
```

### `maxParenDepth(expr)`
Already exists in the codebase. **Retained unchanged.**

---

## Changed

- **Hand size mapping inverted:** Easy 6→4, Hard 4→6. Medium stays at 5.
- **Difficulty labels:** same names (Easy / Medium / Hard), new meanings
- **Toggles added:** Easy gets subtraction toggle; Medium gets all-cards toggle
- **Game mechanic:** "Show Answer" → "Reveal All" (shows all solutions + count)
- **Rules chips:** new UI element shown after generation
- **Footer text:** update "Use any subset of cards in hand" to reflect that Hard requires using all cards (e.g. "Operations: + − × ÷ · Targets: 6–20")

## Removed

- `matchesDiff(expr, diff)` — replaced by per-level post-filter logic above
- `scoreExpr(expr)` — replaced by standalone `countNums` and `countMultDiv` helpers; `maxParenDepth` is retained
- "Show Answer" / "Hide Answer" button and its `toggleSolution()` function — replaced by "Reveal All"

## Unchanged

- Solver algorithm: `solveHand`, `searchNodes`, `combine`, `stripOuter` — no changes
- `maxParenDepth(expr)` — retained, used by Hard filter
- Target range: 6–20
- Bait card pool: 1, 2, 3, 5, 7, 9 (up to 6 copies each)
- Max hand size constant: 6 (Hard uses all 6; Easy and Medium use fewer)
- Tab structure (Solver / Generator tabs)
- Solver tab — no changes
- Toast notification system
