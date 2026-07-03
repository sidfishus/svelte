# Issue #9483 — Working notes

> Investigation notes for [Allow `role="separator"` as an interactive ARIA role](https://github.com/sveltejs/svelte/issues/9483).
> Intended for the PR description / reviewer context. **Not meant to merge upstream** — copy what you need, then delete or gitignore locally.

**Issue:** https://github.com/sveltejs/svelte/issues/9483  
**Reporter repro (StackBlitz):** https://stackblitz.com/edit/vitejs-vite-pfawto?file=Separator.svelte  
**Branch:** `9483_separator_a11y_WIP`

---

## 1. Problem statement

The reporter is building a **resizable window splitter** — the draggable bar between two panels. Their markup:

```svelte
<span role="separator" tabindex="0" onkeydown={handleKeys}></span>
```

Svelte emits two compile-time a11y warnings:

| Code | Message (summary) |
|------|-------------------|
| `a11y_no_noninteractive_tabindex` | noninteractive element cannot have nonnegative tabIndex value |
| `a11y_no_noninteractive_element_interactions` | Non-interactive element should not be assigned mouse or keyboard event listeners |

The reporter's claim: **this combination is valid** per ARIA / accessibility standards. The only workaround today is `<!-- svelte-ignore ... -->` on both rules.

**Working conclusion (2026-07-03):** The reporter is correct. See §2.1 and §2.7.

---

## 2. Why the reported combination is valid (spec research)

### 2.1 Two types of `separator` — static vs interactive

[MDN — separator role](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles/separator_role) states explicitly:

> There are **two types of separators**: a static structure that provides a visible boundary, identical to the HTML `<hr>` element, and a **focusable, moveable widget**.

| Type | Purpose | Focusable? | Keyboard? | Example |
|------|---------|------------|-----------|---------|
| **Static separator** | Visual / structural divider only | No | No | `<hr>`, decorative rule between blog posts |
| **Interactive separator** | Human-operable widget — user resizes adjacent sections | **Yes** | **Yes** | Window splitter (what the reporter is building) |

The reporter is building the **interactive** type. That is a documented, first-class use of `role="separator"` — not a hack or spec stretch.

The implicit role of `<hr>` is `separator`, but that is the **static** case. The same role name is reused when authors intentionally build an **interactive** splitter.

**Key insight:** `role="separator"` is not inherently non-interactive. Whether it behaves as structure or widget depends on whether it is focusable and operable.

---

### 2.2 ARIA 1.1: focusable separator behaves as a widget

[W3C APG — Window Splitter Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/windowsplitter/) opens with:

> **NOTE:** ARIA 1.1 introduced changes to the separator role so it **behaves as a widget when focusable**.

This is the authoritative pattern for the UI the reporter is building. It describes a "moveable separator between two sections, or panes, of a window that enables users to change the relative size of the panes."

---

### 2.3 Window splitter pattern — required role and keyboard interaction

From the same APG page:

**WAI-ARIA roles, states, and properties:**

- The element that serves as the **focusable splitter** has **`role="separator"`**
- `aria-valuenow` — current position (required when focusable)
- `aria-valuemin` / `aria-valuemax` — range (typically 0 and 100)
- `aria-controls` — references the primary pane
- Accessible name via `aria-label` or `aria-labelledby`

**Keyboard interaction** (why `onkeydown` is expected):

| Key | Action |
|-----|--------|
| Arrow keys | Move splitter (direction depends on orientation) |
| Enter | Collapse / restore primary pane |
| Home / End (optional) | Min / max position |
| F6 (optional) | Cycle panes |

A focusable separator **must** be in the tab order and **must** respond to keyboard input. That is not optional decoration — it is how the pattern works.

---

### 2.4 Mapping the reporter's attributes to the spec

| Attribute | Reporter's value | Spec expectation | Valid? |
|-----------|------------------|------------------|--------|
| `role` | `"separator"` | APG: "The element that serves as the focusable splitter has role separator" | ✅ |
| `tabindex` | `"0"` | Focusable widget must be reachable via keyboard (tab sequence). APG examples and tests use `tabindex="0"` for operable splitters; `tabindex="-1"` when disabled | ✅ |
| `onkeydown` | handler | APG defines arrow-key and Enter interaction on the splitter | ✅ |

**Conclusion:** The reporter's *combination* of `role="separator"` + nonnegative tabindex + keyboard handler matches the focusable / widget interpretation of the role. Svelte is applying rules meant for **static** non-interactive elements to a **valid widget** pattern.

---

### 2.5 What a production splitter also needs (context only)

A complete window splitter implementation would also set:

- `aria-valuenow`, `aria-valuemin`, `aria-valuemax`
- `aria-controls` pointing at the primary pane
- `aria-label` or `aria-labelledby` (required when multiple focusable separators exist)
- `aria-orientation` when not horizontal (default is horizontal per MDN)

[MDN — Focusable separator](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles/separator_role#focusable_separator):

> If the separator is focusable … the value of **aria-valuenow must be set** to a number reflecting the current position of the separator and the value must be updated when it changes.

Those are separate a11y rules (`a11y_role_has_required_aria_props`, etc.). **This issue is not about missing aria-valuenow** — it is about Svelte incorrectly treating the element as non-interactive in the first place.

---

### 2.6 Contrast: static `<hr>` is still non-interactive

A plain thematic break:

```html
<hr />
<!-- implicit role: separator, static, not focusable -->
```

Should **not** have `tabindex="0"` or keyboard handlers. That remains correct to warn on.

The bug is Svelte not distinguishing:

- `<hr>` — static separator (non-interactive)
- `<div role="separator" tabindex="0" onkeydown>` — focusable widget (interactive)

---

### 2.7 Conclusion — Svelte's warning is categorically wrong for this case

**Agreed framing:**

1. **`separator` has two types** — static (structure only) and interactive (human-operable widget). The reporter is building the interactive type (window splitter).

2. **Per MDN, the reporter's markup is valid.** For a focusable separator, MDN explicitly requires tabindex-like focusability, `aria-valuenow`, and keyboard-driven resizing. The reporter's `role="separator"` + `tabindex="0"` + `onkeydown` is exactly how you implement that pattern. *(Note: "valid" here means valid accessibility authoring per MDN/ARIA — not that `<span>` is a special HTML element, but that applying this role and these behaviours together is spec-correct.)*

3. **Svelte's warnings are false positives.** Both warnings assume the element is non-interactive:
   - `a11y_no_noninteractive_tabindex` — "noninteractive element cannot have nonnegative tabIndex"
   - `a11y_no_noninteractive_element_interactions` — "Non-interactive element … should not be assigned … keyboard event listeners"

   For an interactive separator, those premises are wrong.

4. **Root cause (confirmed in §4):** Svelte treats `separator` as **always static / non-interactive**, with no branch for the focusable widget case. `tabpanel` has a manual exception; `separator` does not.

**What we are NOT claiming:** that every `role="separator"` is interactive. Static `<div role="separator">` without focus or keyboard handlers should remain valid and silent. *(Note: the tabpanel-style fix has an `<hr>` side effect — see §4.7 / §5.3.)*

---

## 3. Reproduction (TDD)

### 3.0 Approach

Using **TDD**: add failing tests first, then fix the compiler until they pass.

Each validator sample tests **one specific a11y warning rule**. For `separator`, we spread cases across three files by **behaviour**, not by splitting tabindex and keyboard across two files (see §3.5).

| File | Rule under test |
|------|-----------------|
| `a11y-no-noninteractive-element-interactions/input.svelte` | mouse/keyboard handlers on non-interactive elements — **#9483 valid cases live here** |
| `a11y-interactive-supports-focus/input.svelte` | interactive role + handlers but **no** tabindex |
| `a11y-no-noninteractive-tabindex/input.svelte` | nonnegative `tabindex` on non-interactive elements — `tabpanel` precedent only; no `separator` cases |

Each `input.svelte` has two sections:

- **Valid** — markup that should **not** warn for that rule
- **Invalid** — markup that **should** warn

`warnings.json` lists only the **invalid** cases (with exact line/column positions).

Run affected tests from repo root:

```bash
pnpm vitest run packages/svelte/tests/validator/test.ts -t "a11y-no-noninteractive-element-interactions|a11y-interactive-supports-focus|a11y-no-noninteractive-tabindex"
```

**TDD loop:** red (§3.2) → fix compiler (§5) → green (§3.6).

### 3.1 Test cases — final layout

We test **both or neither** for `separator`, not half-interactive combinations:

| Behaviour | Markup | File | Section |
|-----------|--------|------|---------|
| **Static** | `<div role="separator"></div>` | `a11y-no-noninteractive-element-interactions` | valid |
| **Interactive** (window splitter) | `<div role="separator" tabindex="0" on:keydown={() => {}}></div>` | `a11y-no-noninteractive-element-interactions` | valid |
| **Broken** (handlers without focus) | `<div role="separator" on:keydown={() => {}}></div>` | `a11y-interactive-supports-focus` | invalid |

**Why not tabindex-only or keydown-only in the #9483 tests?**

- **tabindex only** — redundant once the fix is in; the full interactive case already proves tabindex doesn't warn.
- **keydown only** — correctly triggers `a11y_interactive_supports_focus` after the fix (separator is interactive-capable). That rule is already covered generically for `button` / `menuitem`; we added an explicit `separator` invalid case in `a11y-interactive-supports-focus` to document the contract (§3.5).

**`a11y-no-noninteractive-tabindex`** — no `separator` lines. `tabpanel` remains as the existing precedent for a focusable structure role:

```svelte
<div role="tabpanel" tabindex='0'></div>
```

### 3.2 Initial status — tests failed (red)

Before the fix, the interactions test failed on the interactive separator case:

| Test | Failed on | Passed (static case) |
|------|-----------|----------------------|
| `a11y-no-noninteractive-element-interactions` | `a11y_no_noninteractive_element_interactions` on separator + keydown | static `<div role="separator">` silent |

(Earlier we also had separator cases in `a11y-no-noninteractive-tabindex`; those were removed in §3.5.)

### 3.3 Comparison: `tabpanel` — already passes

```svelte
<div role="tabpanel" tabindex='0'></div>
```

No warnings. Svelte already excludes `tabpanel` from `non_interactive_roles`. We applied the same pattern to `separator` (§5).

### 3.4 Regression guards

| Case | In test files? | Expected behaviour |
|------|----------------|-------------------|
| Static `<div role="separator">` (no tabindex, no handlers) | ✅ interactions valid | no warnings |
| Interactive separator (tabindex + keydown) | ✅ interactions valid | no warnings — proves #9483 fix |
| Separator + keydown only (no tabindex) | ✅ interactive-supports-focus invalid | `a11y_interactive_supports_focus` |
| Static `<hr tabindex="0">` (implicit separator) | not in tests | **see §5.3 — side effect of tabpanel approach** |

### 3.5 Test consolidation (2026-07-03)

After implementing the fix, we simplified the test layout:

1. **Removed** both separator lines from `a11y-no-noninteractive-tabindex/input.svelte`; reverted `warnings.json` line numbers (invalid cases back to lines 13–16).
2. **Kept** static + full interactive cases in `a11y-no-noninteractive-element-interactions/input.svelte` only.
3. **Added** invalid separator case to `a11y-interactive-supports-focus/input.svelte`:

```svelte
<div role="separator" on:keydown={() => {}}></div>
```

This documents the third state: interactive-capable role with handlers but no tabindex → wrong.

### 3.6 Current status — tests pass (green)

```bash
pnpm vitest run packages/svelte/tests/validator/test.ts -t "a11y-no-noninteractive-element-interactions|a11y-interactive-supports-focus|a11y-no-noninteractive-tabindex"
```

All three pass.

### Checklist

- [x] Add failing validator tests (TDD red phase)
- [x] Add static separator cases (regression guard)
- [x] Compare with `role="tabpanel"` (known exception)
- [x] Fix compiler; tests green
- [x] Consolidate separator tests (both-or-neither strategy)
- [x] Add `interactive_supports_focus` invalid case for separator
- [ ] Resolve `<hr>` side effect (§5.3) — may need follow-up or accept as tabpanel tradeoff

---

## 4. How Svelte classifies roles (investigation)

### 4.1 Trace method

1. Grep the warning code → `packages/svelte/src/compiler/phases/2-analyze/visitors/shared/a11y/index.js`
2. Read the `if` condition that calls `w.a11y_no_noninteractive_*`
3. Follow helpers (`is_interactive_roles`, `is_non_interactive_roles`) → `constants.js`
4. Verify role lists with a node script from `packages/svelte/`:

```bash
node -e "
import { non_interactive_roles, interactive_roles } from './src/compiler/phases/2-analyze/visitors/shared/a11y/constants.js';
for (const r of ['separator', 'tabpanel']) {
  console.log(r, { non_interactive: non_interactive_roles.includes(r), interactive: interactive_roles.includes(r) });
}
"
```

### 4.2 Call chain

```
compile() → 2-analyze → RegularElement.js → check_element() → a11y/index.js
```

### 4.3 Where warnings fire

**`a11y_no_noninteractive_tabindex`** (index.js ~314–321) — fires when element is not interactive AND role is not in `interactive_roles`, and tabindex ≥ 0.

**`a11y_no_noninteractive_element_interactions`** (index.js ~339–354) — fires when role is in `non_interactive_roles` and element has keyboard/mouse handlers.

### 4.4 Role classification (`constants.js`)

Roles come from `aria-query`. `non_interactive_roles` excludes roles whose `superClass` includes `widget` or `window`, **plus manual exceptions**:

```js
!['toolbar', 'tabpanel', 'separator', 'generic', 'cell'].includes(name)
```

| Role | `aria-query` superClass | In `non_interactive_roles`? | In `interactive_roles`? |
|------|-------------------------|----------------------------|------------------------|
| `separator` (before fix) | `structure` | **yes** | no |
| `separator` (after fix) | `structure` | no (manual exception) | **yes** |
| `tabpanel` | `structure`, `section` | no (manual exception) | **yes** |

### 4.5 Why `tabpanel` passes but `separator` failed (before fix)

Both are on a static `<div>`. The difference was purely role classification:

- **`tabpanel`** — explicitly excluded from `non_interactive_roles`. Treated as interactive-capable → no false positives on tabindex/handlers.
- **`separator`** — classified as non-interactive structure → both #9483 warnings fired.

**Root cause confirmed:** Svelte had no equivalent of the `tabpanel` exception for `separator`.

### 4.6 `is_interactive` vs `is_interactive_roles` (tabindex rule walkthrough)

For `<div role="separator" tabindex="0">`, warning #1 checks three things:

1. **`!is_interactive`** — true. `is_interactive` comes from `element_interactivity()` (native HTML tag only). A plain `<div>` is `Static`; tabindex is not considered here.
2. **`!is_interactive_roles('separator')`** — was true before fix (separator in `non_interactive_roles`); false after fix.
3. **non-negative tabindex** — true.

The bug was step 2, not step 1. Adding `tabindex` does not flip `is_interactive`.

**`is_interactive_roles` is a static per-role lookup** — it cannot express "can be interactive". Excluding a role from `non_interactive_roles` means the compiler treats it as **interactive-capable for all rules that use the list**, not "always interactive in the ARIA sense".

### 4.7 Pitfall: `<hr>` implicit role is also `separator`

`tabpanel` has no native HTML element with implicit tabpanel role. **`separator` does** — `<hr>` maps to `separator` via aria-query.

After adding `separator` to the interactive-capable list, aria-query rebuilds element schemas: `<hr>` moves from `non_interactive_element_role_schemas` to `interactive_element_role_schemas`. Side effects:

| Markup | Before fix | After fix |
|--------|------------|-----------|
| `<hr tabindex="0">` | warned (`a11y_no_noninteractive_tabindex`) | **no warning** |
| `<hr role="tab">` | warned (`a11y_no_noninteractive_element_to_interactive_role`) | **no warning** |

The §2.6 goal ("static `<hr>` with tabindex should still warn") is **not met** by the tabpanel-style fix. Accept as tradeoff or address in a follow-up.

### Checklist

- [x] Where are warnings emitted?
- [x] How does Svelte decide interactive vs non-interactive?
- [x] Why does `tabpanel` not warn but `separator` does?
- [x] Pitfall: `<hr>` implicit role is also `separator`

---

## 5. Fix (implemented)

### 5.1 Approach

Follow the existing **`tabpanel` exception** in `constants.js`: add `separator` to the manual exclusion list so it is not in `non_interactive_roles`, hence `is_interactive_roles('separator')` returns true.

**File:** `packages/svelte/src/compiler/phases/2-analyze/visitors/shared/a11y/constants.js`

```js
!['toolbar', 'tabpanel', 'separator', 'generic', 'cell'].includes(name)
```

Comment added alongside `tabpanel` explaining:

- `tabpanel` and `separator` are structure roles that **can** be made focusable
- Excluding them from `non_interactive_roles` does **not** mean they are always interactive in the ARIA sense — only that valid focusable instances should not trigger non-interactive tabindex/interaction warnings
- Same rationale as `tabpanel`; see GitHub #9483

### 5.2 What this fixes

| Reporter markup | Warnings before | After |
|-----------------|-----------------|-------|
| `role="separator"` + `tabindex="0"` | `a11y_no_noninteractive_tabindex` | none |
| `role="separator"` + `on:keydown` (+ `tabindex`) | `a11y_no_noninteractive_element_interactions` | none |
| `role="separator"` + `on:keydown` only | N/A (was interactions false positive) | `a11y_interactive_supports_focus` (correct) |

### 5.3 Known side effect — `<hr>`

See §4.7. One existing validator test may fail: `a11y-no-noninteractive-element-to-interactive-role` expects `<hr role="tab">` to warn; after the fix it does not. Needs decision before PR: accept, update test expectations, or pursue a more targeted fix.

### 5.4 Files changed (fix + tests)

| File | Change |
|------|--------|
| `constants.js` | Add `separator` to exception list + comment |
| `a11y-no-noninteractive-element-interactions/input.svelte` | Valid: static separator + interactive (tabindex + keydown) |
| `a11y-no-noninteractive-tabindex/input.svelte` | Removed separator cases |
| `a11y-no-noninteractive-tabindex/warnings.json` | Line numbers 13–16 (was 15–18) |
| `a11y-interactive-supports-focus/input.svelte` | Invalid: separator + keydown, no tabindex |
| `a11y-interactive-supports-focus/warnings.json` | New entry for separator (line 32) |

---

## 6. References

| Source | URL |
|--------|-----|
| GitHub issue | https://github.com/sveltejs/svelte/issues/9483 |
| MDN — separator role | https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles/separator_role |
| MDN — Focusable separator section | https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles/separator_role#focusable_separator |
| W3C APG — Window Splitter Pattern | https://www.w3.org/WAI/ARIA/apg/patterns/windowsplitter/ |
| WAI-ARIA 1.2 — separator | https://www.w3.org/TR/wai-aria-1.2/#separator |
| Reporter StackBlitz | https://stackblitz.com/edit/vitejs-vite-pfawto?file=Separator.svelte |

---

## Session log

| Date | What we did |
|------|-------------|
| 2026-07-03 | Read MDN + W3C APG docs; documented why reporter markup is spec-valid (sections 1–2) |
| 2026-07-03 | Agreed conclusion: two separator types; reporter builds interactive type; Svelte falsely assumes always static (§2.7) |
| 2026-07-03 | Confirmed both validator tests fail (red); command in §3.2 |
| 2026-07-03 | Added static separator regression cases; re-ran tests (2 fail, static cases pass) |
| 2026-07-03 | Traced warnings to `a11y/index.js` + `constants.js`; root cause confirmed (§4) |
| 2026-07-03 | Walkthrough: `is_interactive` vs `is_interactive_roles`; tabpanel = always interactive-capable in compiler (§4.6) |
| 2026-07-03 | Fix: `separator` added to same exception list as `tabpanel` in `constants.js` (§5) |
| 2026-07-03 | Consolidated tests: both-or-neither strategy; removed separator from tabindex sample; added invalid case to interactive-supports-focus (§3.5) |
| 2026-07-03 | Tests green for interactions, interactive-supports-focus, tabindex samples |
| 2026-07-03 | Noted `<hr>` side effect — implicit separator now interactive-capable via aria-query schemas (§4.7, §5.3) |
