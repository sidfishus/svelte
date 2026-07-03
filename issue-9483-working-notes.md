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

4. **Root cause (hypothesis, to verify in §4):** Svelte treats `separator` as **always static / non-interactive**, with no branch for the focusable widget case. That one-size-fits-all assumption contradicts MDN and ARIA 1.1+.

**What we are NOT claiming:** that every `role="separator"` is interactive. Static `<hr>` without focus or keyboard handlers should still be non-interactive and should still warn if given tabindex/handlers. The fix must preserve that.

---

## 3. Reproduction (TDD)

### 3.0 Approach

Using **TDD**: add failing tests first, then fix the compiler until they pass.

Each validator sample tests **one specific a11y warning rule**:

| File | Rule under test |
|------|-----------------|
| `a11y-no-noninteractive-tabindex/input.svelte` | nonnegative `tabindex` on non-interactive elements |
| `a11y-no-noninteractive-element-interactions/input.svelte` | mouse/keyboard handlers on non-interactive elements |

Each `input.svelte` has two sections:

- **Valid** — markup that should **not** warn for that rule (includes existing cases like `tabpanel`, plus our new `separator` cases)
- **Invalid** — markup that **should** warn

`warnings.json` lists only the **invalid** cases (with exact line/column positions). The separator lines are intentionally absent — they belong in the valid section and should not affect the expected warnings once fixed.

**Files changed for tests:**

| File | Change |
|------|--------|
| both `input.svelte` | one separator line added in the valid section |
| both `warnings.json` | line numbers incremented by 1 only (invalid cases shifted down one line); no new entries |

Run:

```bash
pnpm test -- validator a11y-no-noninteractive-tabindex a11y-no-noninteractive-element-interactions
```

**TDD loop:** red (now) → fix compiler → green (same command, no `warnings.json` changes needed).

### 3.1 Test cases added (valid — should produce **no** warnings after fix)

**`a11y-no-noninteractive-tabindex/input.svelte`** (alongside existing `tabpanel` exception):

```svelte
<div role="separator" tabindex='0'></div>
```

**`a11y-no-noninteractive-element-interactions/input.svelte`** (keyboard handlers only — no tabindex; that's covered by the other test):

```svelte
<div role="separator" on:keydown={() => {}}></div>
```

`warnings.json` in each sample lists only the **invalid** cases; the separator lines are intentionally absent.

### 3.2 Current status — tests fail (red)

**Both tests currently fail.** Run from repo root:

```bash
pnpm test -- validator a11y-no-noninteractive-tabindex a11y-no-noninteractive-element-interactions
```

Expect: `Tests  2 failed` (plus many others passing). Each failure shows **Expected** vs **Received** — the `+` lines are extra warnings from the separator cases in the valid section.

| Test | Extra warning (line in `input.svelte`) |
|------|----------------------------------------|
| `a11y-no-noninteractive-tabindex` | `a11y_no_noninteractive_tabindex` on line 11 — `<div role="separator" tabindex='0'>` |
| `a11y-no-noninteractive-element-interactions` | `a11y_no_noninteractive_element_interactions` on line 9 — `<div role="separator" on:keydown={...}>` |

This matches the issue report exactly. After the fix, re-run the same command — both should pass with no `warnings.json` changes.

### 3.3 Comparison: `tabpanel` — already passes in same file

```svelte
<div role="tabpanel" tabindex='0'></div>
```

No warnings. Svelte already has a special case for `tabpanel` (also focusable but not a widget role in aria-query). `separator` is treated differently — that asymmetry is the bug.

### 3.4 Regression guard: static `<hr>` — still warns (correct)

Not in these test files yet; verify during fix that `<hr tabindex="0">` still warns (implicit static separator).

### Checklist

- [x] Add failing validator tests (TDD red phase)
- [x] Compare with `role="tabpanel"` (known exception in same test file)
- [ ] Confirm `<hr tabindex="0">` still warns after fix (regression guard)

---

## 4. How Svelte classifies roles (investigation)

<!-- Fill in while tracing compiler -->

- [ ] Where are `a11y_no_noninteractive_tabindex` and `a11y_no_noninteractive_element_interactions` emitted?
- [ ] How does Svelte decide an element/role is "interactive" vs "non-interactive"?
- [ ] Why does `tabpanel` not warn but `separator` does?
- [ ] Pitfall: `<hr>` implicit role is also `separator` — fix must not break hr checks

---

## 5. Proposed fix

<!-- Fill in after investigation. Stashed WIP exists: `git stash list` → `9483 separator a11y fix (WIP - for later)` -->

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
| | Stashed prior AI fix; working tree clean for iterative investigation |
