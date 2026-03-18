# Puzzle Generator Redesign Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the 3-level difficulty system in the puzzle generator with new Easy/Medium/Hard levels using find-many mechanics, toggles, and rules chips.

**Architecture:** All changes are in a single file (`mathcat-helper/index.html`). Logic changes (JS) are done before UI changes (HTML/CSS) so each commit is independently functional. No external dependencies added.

**Tech Stack:** Vanilla HTML/CSS/JS, no build tools.

---

## File Structure

Only one file is modified:
- **Modify:** `mathcat-helper/index.html`
  - JS section (~lines 528–635): difficulty logic replaced entirely
  - JS section (~lines 311–320): state vars updated
  - CSS section (~line 200): new chip/toggle styles appended
  - HTML generator tab (~lines 262–300): difficulty card + puzzle output replaced
  - HTML footer (~line 304): text updated

---

## Chunk 1: Logic Changes

### Task 1: Replace helper functions and add HAND_SIZES constant

**Files:**
- Modify: `mathcat-helper/index.html`

This task removes `scoreExpr` and `matchesDiff`, adds `countNums` and `countMultDiv` as standalone helpers, keeps `maxParenDepth` unchanged, and adds the `HAND_SIZES` constant to the constants block.

- [ ] **Step 1: Add `HAND_SIZES` constant to the constants block**

  Find (line ~315):
  ```js
  const TARGET_MAX = 20;
  ```
  Replace with:
  ```js
  const TARGET_MAX = 20;
  const HAND_SIZES  = { easy: 4, medium: 5, hard: 6 };
  ```

- [ ] **Step 2: Replace `scoreExpr`, `maxParenDepth`, and `matchesDiff` with new helpers**

  Find (lines ~538–562):
  ```js
  // Score a solution expression for difficulty classification.
  function scoreExpr(expr) {
    return {
      hasMultDiv : /[×÷]/.test(expr),
      numCount   : (expr.match(/\d+/g) || []).length,
      parenDepth : maxParenDepth(expr),
    };
  }

  function maxParenDepth(expr) {
    let depth = 0, max = 0;
    for (const ch of expr) {
      if (ch === '(') { depth++; max = Math.max(max, depth); }
      else if (ch === ')') depth--;
    }
    return max;
  }

  function matchesDiff(expr, diff) {
    const s = scoreExpr(expr);
    if (diff === 1) return !s.hasMultDiv;
    if (diff === 2) return s.hasMultDiv && s.numCount <= 4;
    if (diff === 3) return s.hasMultDiv && (s.numCount >= 4 || s.parenDepth >= 2);
    return true;
  }
  ```

  Replace with:
  ```js
  function countNums(expr) {
    return (expr.match(/\d+/g) || []).length;
  }

  function countMultDiv(expr) {
    return (expr.match(/[×÷]/g) || []).length;
  }

  function maxParenDepth(expr) {
    let depth = 0, max = 0;
    for (const ch of expr) {
      if (ch === '(') { depth++; max = Math.max(max, depth); }
      else if (ch === ')') depth--;
    }
    return max;
  }
  ```

- [ ] **Step 3: Verify syntax**

  Run:
  ```bash
  node -e "
  const fs = require('fs');
  const src = fs.readFileSync('mathcat-helper/index.html','utf8');
  const m = src.match(/<script>([\s\S]*?)<\/script>/);
  new Function(m[1]);
  console.log('syntax OK');
  "
  ```
  Expected output: `syntax OK`

- [ ] **Step 4: Commit**

  ```bash
  git add mathcat-helper/index.html
  git commit -m "refactor: replace scoreExpr/matchesDiff with countNums/countMultDiv helpers"
  ```

---

### Task 2: Replace state variables, add `selectLevel` and `filterSolutions`

**Files:**
- Modify: `mathcat-helper/index.html`

- [ ] **Step 1: Replace `selectedDiff` state variable**

  Find (line ~319):
  ```js
  let selectedDiff = 1;
  ```
  Replace with:
  ```js
  let selectedLevel      = 'easy';
  let easySubtraction    = false;
  let mediumAllCards     = false;
  let lastPuzzleSolutions = [];
  ```

- [ ] **Step 2: Replace `selectDiff` with `selectLevel` and add `filterSolutions`**

  Find (lines ~531–536):
  ```js
  function selectDiff(level) {
    selectedDiff = level;
    document.querySelectorAll('.diff-btn').forEach(b =>
      b.classList.toggle('selected', +b.dataset.diff === level)
    );
  }
  ```

  Replace with:
  ```js
  function selectLevel(level) {
    selectedLevel   = level;
    easySubtraction = false;
    mediumAllCards  = false;

    document.querySelectorAll('.diff-btn').forEach(b =>
      b.classList.toggle('selected', b.dataset.level === level)
    );
    document.querySelectorAll('.level-toggles').forEach(el =>
      el.style.display = el.dataset.level === level ? 'block' : 'none'
    );

    const st = document.getElementById('toggle-subtraction');
    const mt = document.getElementById('toggle-all-cards');
    if (st) st.checked = false;
    if (mt) mt.checked = false;
  }

  function filterSolutions(sols, handSize) {
    if (selectedLevel === 'easy') {
      if (!easySubtraction) return sols.filter(s => !/-/.test(s));
      return sols;
    }
    if (selectedLevel === 'medium') {
      if (mediumAllCards) return sols.filter(s => countNums(s) === handSize);
      return sols;
    }
    if (selectedLevel === 'hard') {
      return sols.filter(s =>
        countNums(s) === handSize &&
        maxParenDepth(s) >= 2     &&
        countMultDiv(s) >= 2
      );
    }
    return sols;
  }
  ```

- [ ] **Step 3: Verify syntax**

  Run:
  ```bash
  node -e "
  const fs = require('fs');
  const src = fs.readFileSync('mathcat-helper/index.html','utf8');
  const m = src.match(/<script>([\s\S]*?)<\/script>/);
  new Function(m[1]);
  console.log('syntax OK');
  "
  ```
  Expected output: `syntax OK`

- [ ] **Step 4: Commit**

  ```bash
  git add mathcat-helper/index.html
  git commit -m "refactor: replace selectedDiff state with selectedLevel + filterSolutions"
  ```

---

### Task 3: Rewrite generator functions

**Files:**
- Modify: `mathcat-helper/index.html`

This task replaces `runGenerator` and `toggleSolution` with `runGenerator`, `renderPuzzle`, `renderChips`, and `revealSolutions`.

- [ ] **Step 1: Replace `runGenerator` and `toggleSolution` with new functions**

  Find (lines ~581–635):
  ```js
  function runGenerator() {
    const btn = document.getElementById('gen-btn');
    btn.disabled = true;
    btn.innerHTML = '🎲 Generating<span class="spinner"></span>';

    document.getElementById('puzzle-output').style.display = 'none';
    document.getElementById('solution-area').style.display = 'none';
    document.getElementById('show-sol-btn').textContent = '👁 Show Answer';

    setTimeout(() => {
      // Difficulty → hand size (fewer cards = harder)
      const sizes = { 1: 6, 2: 5, 3: 4 };
      const size  = sizes[selectedDiff] || 5;

      let found = null;
      for (let attempt = 0; attempt < 800 && !found; attempt++) {
        const h    = randomHand(size);
        const sols = solveHand(h, genTarget);
        const ok   = sols.filter(s => matchesDiff(s, selectedDiff));
        if (ok.length > 0) found = { hand: h, solution: ok[0] };
      }

      btn.disabled = false;
      btn.innerHTML = '🎲 Generate Puzzle';

      if (!found) {
        toast(`😿 No ${['','Easy','Medium','Hard'][selectedDiff]} puzzle found for target ${genTarget}. Try a different difficulty or target.`);
        return;
      }

      // Render puzzle
      document.getElementById('puzzle-target-num').textContent = genTarget;

      const cardsEl = document.getElementById('puzzle-cards');
      cardsEl.innerHTML = '';
      [...found.hand].sort((a, b) => a - b).forEach(n => {
        const div = document.createElement('div');
        div.className = 'puzzle-card';
        div.textContent = n;
        cardsEl.appendChild(div);
      });

      document.getElementById('puzzle-solution').textContent = `${found.solution} = ${genTarget}`;
      document.getElementById('puzzle-output').style.display = 'block';
      document.getElementById('puzzle-output').scrollIntoView({ behavior: 'smooth', block: 'nearest' });
    }, 60);
  }

  function toggleSolution() {
    const area = document.getElementById('solution-area');
    const btn  = document.getElementById('show-sol-btn');
    const showing = area.style.display !== 'none';
    area.style.display = showing ? 'none' : 'block';
    btn.textContent = showing ? '👁 Show Answer' : '🙈 Hide Answer';
  }
  ```

  Replace with:
  ```js
  function runGenerator() {
    const btn = document.getElementById('gen-btn');
    btn.disabled = true;
    btn.innerHTML = '🎲 Generating<span class="spinner"></span>';
    document.getElementById('puzzle-output').style.display = 'none';

    setTimeout(() => {
      const size = HAND_SIZES[selectedLevel];
      const FAIL = {
        easy:   `😿 No Easy puzzle found for target ${genTarget}. Try a different target.`,
        medium: `😿 No Medium puzzle found for target ${genTarget}. Try different settings.`,
        hard:   `😿 No Hard puzzle found for target ${genTarget}. Try a different target.`,
      };

      let found = null;
      for (let attempt = 0; attempt < 800 && !found; attempt++) {
        const h    = randomHand(size);
        const sols = filterSolutions(solveHand(h, genTarget), size);
        if (sols.length > 0) found = { hand: h, solutions: sols };
      }

      btn.disabled = false;
      btn.innerHTML = '🎲 Generate Puzzle';

      if (!found) {
        toast(FAIL[selectedLevel]);
        return;
      }

      renderPuzzle(found.hand, found.solutions);
    }, 60);
  }

  function renderPuzzle(hand, solutions) {
    lastPuzzleSolutions = solutions;

    document.getElementById('puzzle-target-num').textContent = genTarget;

    const cardsEl = document.getElementById('puzzle-cards');
    cardsEl.innerHTML = '';
    [...hand].sort((a, b) => a - b).forEach(n => {
      const div = document.createElement('div');
      div.className = 'puzzle-card';
      div.textContent = n;
      cardsEl.appendChild(div);
    });

    renderChips();

    document.getElementById('reveal-btn').style.display = 'inline-block';
    const area = document.getElementById('solution-area');
    area.style.display = 'none';
    area.innerHTML = '';

    document.getElementById('puzzle-output').style.display = 'block';
    document.getElementById('puzzle-output').scrollIntoView({ behavior: 'smooth', block: 'nearest' });
  }

  function renderChips() {
    const chipsEl = document.getElementById('puzzle-chips');
    chipsEl.innerHTML = '';

    const ops   = selectedLevel === 'easy'
      ? (easySubtraction ? '+ −' : '+')
      : '+ − × ÷';
    const cards = selectedLevel === 'hard' || (selectedLevel === 'medium' && mediumAllCards)
      ? `all ${HAND_SIZES[selectedLevel]} cards`
      : 'any cards';

    [ops, cards, ...(selectedLevel === 'hard' ? ['complex'] : [])].forEach(text => {
      const span = document.createElement('span');
      span.className = 'rule-chip';
      span.textContent = text;
      if (text === 'complex') span.title = 'depth ≥ 2, uses 2+ × or ÷';
      chipsEl.appendChild(span);
    });
  }

  function revealSolutions() {
    document.getElementById('reveal-btn').style.display = 'none';
    const area = document.getElementById('solution-area');

    if (lastPuzzleSolutions.length === 0) {
      area.innerHTML = '<div class="no-solution">😿 No solutions found.</div>';
    } else {
      const hdr = document.createElement('div');
      hdr.className = 'result-header';
      hdr.textContent = `🎉 ${lastPuzzleSolutions.length} solution${lastPuzzleSolutions.length !== 1 ? 's' : ''}:`;

      const list = document.createElement('div');
      list.className = 'solution-list';
      lastPuzzleSolutions.forEach(s => {
        const chip = document.createElement('span');
        chip.className = 'solution-chip';
        chip.textContent = `${s} = ${genTarget}`;
        list.appendChild(chip);
      });

      area.innerHTML = '';
      area.appendChild(hdr);
      area.appendChild(list);
    }
    area.style.display = 'block';
  }
  ```

- [ ] **Step 2: Verify syntax**

  Run:
  ```bash
  node -e "
  const fs = require('fs');
  const src = fs.readFileSync('mathcat-helper/index.html','utf8');
  const m = src.match(/<script>([\s\S]*?)<\/script>/);
  new Function(m[1]);
  console.log('syntax OK');
  "
  ```
  Expected output: `syntax OK`

- [ ] **Step 3: Commit**

  ```bash
  git add mathcat-helper/index.html
  git commit -m "feat: rewrite generator with find-many mechanic and per-level filtering"
  ```

---

## Chunk 2: UI Changes

### Task 4: Add CSS for rule chips and toggles

**Files:**
- Modify: `mathcat-helper/index.html`

- [ ] **Step 1: Add new CSS rules after the existing footer style**

  Find (line ~200 — the last CSS rule in the `<style>` block, just before `</style>`):
  ```css
  footer { text-align: center; color: var(--muted); font-size: 0.78rem; padding: 24px; }
</style>
  ```

  Replace with:
  ```css
  footer { text-align: center; color: var(--muted); font-size: 0.78rem; padding: 24px; }

  /* ── RULE CHIPS ── */
  .rule-chip {
    background: #fff3e0; border: 1px solid #ffe0b2; border-radius: 20px;
    padding: 4px 12px; font-size: 0.8rem; font-weight: 700; color: var(--orange);
  }

  /* ── LEVEL TOGGLES ── */
  .level-toggles { margin-top: 14px; }
  .toggle-label {
    display: flex; align-items: center; gap: 8px; justify-content: center;
    font-size: 0.88rem; color: var(--muted); cursor: pointer;
  }
  .toggle-label input[type="checkbox"] {
    width: 16px; height: 16px; cursor: pointer; accent-color: var(--orange);
  }
  ```

- [ ] **Step 2: Verify CSS is well-formed**

  Open `mathcat-helper/index.html` in Chrome (File → Open). Open DevTools → Console. No CSS parse errors should appear.

- [ ] **Step 3: Commit**

  ```bash
  git add mathcat-helper/index.html
  git commit -m "feat: add CSS for rule chips and level toggles"
  ```

---

### Task 5: Replace difficulty card and puzzle output HTML

**Files:**
- Modify: `mathcat-helper/index.html`

- [ ] **Step 1: Replace the difficulty card HTML**

  Find (lines ~262–278):
  ```html
  <div class="card">
      <h2>🐾 Difficulty Level</h2>
      <div class="difficulty-btns">
        <button class="diff-btn selected" data-diff="1" onclick="selectDiff(1)">
          <div class="diff-label">😺 Easy</div>
          <div class="diff-desc">+ and − only</div>
        </button>
        <button class="diff-btn" data-diff="2" onclick="selectDiff(2)">
          <div class="diff-label">😼 Medium</div>
          <div class="diff-desc">Uses × or ÷</div>
        </button>
        <button class="diff-btn" data-diff="3" onclick="selectDiff(3)">
          <div class="diff-label">😈 Hard</div>
          <div class="diff-desc">Complex expressions</div>
        </button>
      </div>
    </div>
  ```

  Replace with:
  ```html
  <div class="card">
      <h2>🐾 Difficulty Level</h2>
      <div class="difficulty-btns">
        <button class="diff-btn selected" data-level="easy" onclick="selectLevel('easy')">
          <div class="diff-label">😺 Easy</div>
          <div class="diff-desc">Addition only · 4 cards</div>
        </button>
        <button class="diff-btn" data-level="medium" onclick="selectLevel('medium')">
          <div class="diff-label">😼 Medium</div>
          <div class="diff-desc">All ops · 5 cards</div>
        </button>
        <button class="diff-btn" data-level="hard" onclick="selectLevel('hard')">
          <div class="diff-label">😈 Hard</div>
          <div class="diff-desc">Complex · 6 cards</div>
        </button>
      </div>
      <div class="level-toggles" data-level="easy">
        <label class="toggle-label">
          <input type="checkbox" id="toggle-subtraction"
                 onchange="easySubtraction = this.checked">
          Include subtraction (−)
        </label>
      </div>
      <div class="level-toggles" data-level="medium" style="display:none">
        <label class="toggle-label">
          <input type="checkbox" id="toggle-all-cards"
                 onchange="mediumAllCards = this.checked">
          Must use all 5 cards
        </label>
      </div>
    </div>
  ```

- [ ] **Step 2: Replace the puzzle output card HTML**

  Find (lines ~284–300):
  ```html
  <div class="card" id="puzzle-output" style="display:none">
      <h2>🐾 Your Puzzle</h2>
      <div class="puzzle-box">
        <div class="puzzle-target" id="puzzle-target-num"></div>
        <div class="puzzle-target-label">🐱 Cat Card Target</div>
        <div class="puzzle-cards" id="puzzle-cards"></div>
      </div>
      <div style="text-align:center;margin-top:18px;">
        <button class="btn-primary btn-green" id="show-sol-btn" onclick="toggleSolution()">
          👁 Show Answer
        </button>
        <div id="solution-area" style="margin-top:14px;display:none">
          <div class="result-header" style="text-align:center;">One possible answer:</div>
          <span class="solution-text" id="puzzle-solution"></span>
        </div>
      </div>
    </div>
  ```

  Replace with:
  ```html
  <div class="card" id="puzzle-output" style="display:none">
      <h2>🐾 Your Puzzle</h2>
      <div class="puzzle-box">
        <div class="puzzle-target" id="puzzle-target-num"></div>
        <div class="puzzle-target-label">🐱 Cat Card Target</div>
        <div class="puzzle-cards" id="puzzle-cards"></div>
      </div>
      <div id="puzzle-chips"
           style="display:flex;flex-wrap:wrap;gap:6px;justify-content:center;margin-top:12px;">
      </div>
      <div style="text-align:center;margin-top:18px;">
        <button class="btn-primary btn-green" id="reveal-btn" onclick="revealSolutions()">
          👁 Reveal All
        </button>
        <div id="solution-area" style="margin-top:14px;display:none"></div>
      </div>
    </div>
  ```

- [ ] **Step 3: Verify the generator tab works end-to-end**

  > **Prerequisite:** Chunk 1 (Tasks 1–3) must be complete before this verification — the JS functions `selectLevel`, `filterSolutions`, `renderPuzzle`, `renderChips`, and `revealSolutions` must exist.

  Serve locally:
  ```bash
  python3 -m http.server 8080 --directory /Users/ashari/Documents/personal/arashari.github.io
  ```

  Open `http://localhost:8080/mathcat-helper/` → switch to **🎲 Puzzle Generator** tab.

  Check each scenario:

  | Action | Expected |
  |---|---|
  | Easy selected by default | "Include subtraction" checkbox visible |
  | Click Medium | "Must use all 5 cards" checkbox visible, subtraction checkbox hidden |
  | Click Hard | No toggles visible |
  | Click Easy again | "Include subtraction" checkbox reappears, unchecked |
  | Generate puzzle (Easy, no subtraction) | Puzzle appears. Chips show `+` · `any cards` |
  | Click "Reveal All" | Button disappears, solutions appear with count. All solutions contain only `+` |
  | Generate Easy with subtraction on | Chips show `+ −` · `any cards`. Solutions may include `-` |
  | Generate Medium with "must use all 5 cards" | Chips show `+ − × ÷` · `all 5 cards`. Each revealed solution uses exactly 5 number tokens |
  | Generate Hard | Chips show `+ − × ÷` · `all 6 cards` · `complex`. Hover `complex` chip → tooltip shows "depth ≥ 2, uses 2+ × or ÷" |
  | Generate new puzzle | Reveal button reappears, solution area is hidden |

- [ ] **Step 4: Commit**

  ```bash
  git add mathcat-helper/index.html
  git commit -m "feat: update generator HTML with level toggles and Reveal All button"
  ```

---

### Task 6: Update footer text

**Files:**
- Modify: `mathcat-helper/index.html`

- [ ] **Step 1: Update footer text**

  Find (line ~304):
  ```html
  <footer>🐾 Math Cat Helper &nbsp;·&nbsp; Operations: + − × ÷ &nbsp;·&nbsp; Use any subset of cards in hand</footer>
  ```

  Replace with:
  ```html
  <footer>🐾 Math Cat Helper &nbsp;·&nbsp; Operations: + − × ÷ &nbsp;·&nbsp; Targets: 6 – 20</footer>
  ```

- [ ] **Step 2: Verify footer renders correctly**

  In the browser, scroll to the bottom of the page. Footer should read: `🐾 Math Cat Helper · Operations: + − × ÷ · Targets: 6 – 20`

- [ ] **Step 3: Commit**

  ```bash
  git add mathcat-helper/index.html
  git commit -m "chore: update footer text to remove outdated rule"
  ```
