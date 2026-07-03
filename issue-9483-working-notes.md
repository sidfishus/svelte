# Issue #9483 — Working notes

> Investigation notes for [Allow `role="separator"` as an interactive ARIA role](https://github.com/sveltejs/svelte/issues/9483).
> Intended for an issue comment, PR discussion, or maintainer context. **Not meant to merge upstream** — copy what you need, then delete or gitignore locally.

**Issue:** https://github.com/sveltejs/svelte/issues/9483  
**Reporter repro (StackBlitz):** https://stackblitz.com/edit/vitejs-vite-pfawto?file=Separator.svelte  
**Branch:** `9483_separator_a11y_WIP`

**Outcome of this investigation:** We agree the reporter is correct and Svelte's warnings are false positives for a valid window-splitter pattern. We do **not** think the issue should be fixed with a one-line role exception or a targeted `separator` special-case. Both approaches have unacceptable tradeoffs. In our view, the right fix is a broader rework of how the a11y visitor infers interactiveness.

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

**Conclusion:** The reporter is correct. See §2.

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

This is the authoritative pattern for the UI the reporter is building.

---

### 2.3 Mapping the reporter's attributes to the spec

| Attribute | Reporter's value | Spec expectation | Valid? |
|-----------|------------------|------------------|--------|
| `role` | `"separator"` | APG: focusable splitter has role separator | ✅ |
| `tabindex` | `"0"` | Focusable widget must be in tab order | ✅ |
| `onkeydown` | handler | APG defines arrow-key and Enter interaction | ✅ |

**Conclusion:** The reporter's combination matches the focusable / widget interpretation of the role. Svelte is applying rules meant for **static** non-interactive elements to a **valid widget** pattern.

---

### 2.4 Contrast: static `<hr>` must stay non-interactive

```html
<hr />
<!-- implicit role: separator, static, not focusable -->
```

Should **not** have `tabindex="0"` or keyboard handlers. Svelte should continue to warn on those.

The bug is Svelte not distinguishing:

- `<hr>` — static separator (non-interactive)
- `<div role="separator" tabindex="0" onkeydown>` — focusable widget (interactive)

---

### 2.5 What we are NOT claiming

- Not every `role="separator"` is interactive. Static `<div role="separator">` without focus or keyboard handlers should remain valid and silent.
- This issue is not about missing `aria-valuenow` etc. — separate rules cover that. This is about Svelte incorrectly treating a valid widget as non-interactive in the first place.

---

## 3. Root cause — how Svelte decides interactiveness today

### 3.1 Call chain

```
compile() → 2-analyze → RegularElement.js → check_element() → a11y/index.js
```

Warnings fire in `packages/svelte/src/compiler/phases/2-analyze/visitors/shared/a11y/index.js`. Role lists come from `constants.js`.

### 3.2 Two separate signals

Svelte uses **role name** as the primary signal, with a secondary signal from **native HTML tag**:

| Signal | Source | Used for |
|--------|--------|----------|
| `is_interactive` / `is_non_interactive` / `is_static` | `element_interactivity()` — aria-query element → role schemas | Native tag classification (`<button>`, `<hr>`, `<div>`) |
| `is_interactive_roles()` / `is_non_interactive_roles()` | Static role lists built from aria-query + manual exceptions | Explicit `role="..."` attribute |

For `<div role="separator" tabindex="0">`:

1. **`is_interactive`** — false. A plain `<div>` is `Static`; tabindex is not considered here.
2. **`is_interactive_roles('separator')`** — false before any fix. `separator` is in `non_interactive_roles`.
3. Non-negative tabindex — true.

→ `a11y_no_noninteractive_tabindex` fires. The bug is step 2: **role alone says "non-interactive"**, with no branch for the focusable widget case.

### 3.3 Manual exceptions already exist

`non_interactive_roles` in `constants.js` excludes certain structure roles from the non-interactive list:

```js
!['toolbar', 'tabpanel', 'generic', 'cell'].includes(name)
```

`tabpanel` has no native HTML element with an implicit tabpanel role, so this exception is safe. **`separator` is not in the list**, which is why #9483 fires.

### 3.4 `is_interactive_roles` cannot express "can be interactive"

Excluding a role from `non_interactive_roles` means the compiler treats it as **interactive-capable for all rules that use the list** — not "interactive only when focusable and operable". This distinction matters for `separator` (see §5).

---

## 4. Reproduction

We reproduced with validator samples (TDD). Before any fix, the interactive separator case fails:

```svelte
<div role="separator" tabindex="0" on:keydown={() => {}}></div>
```

Expected: no warnings. Actual: `a11y_no_noninteractive_element_interactions` (and tabindex warning if tested separately).

Static case passes throughout:

```svelte
<div role="separator"></div>
```

Run affected tests from repo root:

```bash
pnpm vitest run packages/svelte/tests/validator/test.ts -t "a11y-no-noninteractive-element-interactions|a11y-interactive-supports-focus|a11y-no-noninteractive-tabindex"
```

Comparison — `tabpanel` already passes with tabindex alone:

```svelte
<div role="tabpanel" tabindex='0'></div>
```

No warnings. This is the precedent we tried to follow (§5.1).

---

## 5. Approaches tried — and why we rejected them

We explored two fixes. Both resolve #9483 in the happy path. Both are, in our view, **hacks** — special-casing one awkward role rather than fixing the underlying model. We do not recommend shipping either.

### 5.1 Approach A — tabpanel-style role exception (rejected)

**Change:** Add `separator` to the manual exclusion list in `constants.js`, same as `tabpanel`:

```js
!['toolbar', 'tabpanel', 'separator', 'generic', 'cell'].includes(name)
```

**Why it fixes #9483:** `is_interactive_roles('separator')` returns true → no false positives on tabindex or keyboard handlers for the reporter's markup.

**Why we rejected it — the `<hr>` problem:**

`tabpanel` has no native HTML element with an implicit tabpanel role. **`separator` does** — `<hr>` maps to `separator` via aria-query.

Adding `separator` to the interactive-capable list also changes aria-query element schemas: `<hr>` moves from `non_interactive_element_role_schemas` to `interactive_element_role_schemas`. Observed regressions:

| Markup | Before fix | After tabpanel-style fix |
|--------|------------|--------------------------|
| `<hr tabindex="0">` | warned (`a11y_no_noninteractive_tabindex`) | **no warning** ❌ |
| `<hr on:keydown>` | warned (`a11y_no_noninteractive_element_interactions`) | **no warning** ❌ |
| `<hr role="tab">` | warned (`a11y_no_noninteractive_element_to_interactive_role`) | **no warning** ❌ |

**Verdict:** We can't mark `separator` as globally interactive because `<hr>` is always a `separator`, and `<hr>` must stay non-interactive — so the simple fix creates false negatives on real mistakes.

---

### 5.2 Approach B — targeted context special-case for `separator` (rejected)

**Change:** Keep `separator` in `non_interactive_roles`. Add helpers in `index.js` that treat explicit `role="separator"` as interactive only when specific attributes/handlers are present — e.g. both non-negative `tabindex` AND keyboard handlers.

Rough shape (~60 lines in `index.js`):

```js
function is_dual_nature_role_interactive(role, attribute_map, handlers) {
  if (role !== 'separator') return false;
  return has_non_negative_tabindex(attribute_map) && has_recommended_interactive_handlers(handlers);
}
```

Wire this into the four rules that currently use `is_interactive_roles()` / `is_non_interactive_roles()`.

**Why it fixes #9483:** Reporter's markup (tabindex + keydown) passes. Static `<div role="separator">` stays silent. `<hr>` stays non-interactive because the special-case checks explicit `role="separator"`, not implicit role from `<hr>`.

**Why we rejected it:**

1. **Still a hack.** Hardcoded `if (role !== 'separator')` in multiple code paths. Same structural problem as the `tabpanel` exception, just moved and made more complex.

2. **Inconsistent rules for one role.** We ended up requiring BOTH tabindex and handlers for `separator` to count as interactive, while `tabpanel` still accepts tabindex alone. That inconsistency exists only because we're patching around the `<hr>` collision, not because the spec treats these roles differently in that way.

3. **Awkward half-states.** Tabindex-only on explicit `role="separator"` fires `a11y_no_noninteractive_tabindex` ("noninteractive element") — technically correct under the patch, but the messaging is misleading. Handlers-only fires `a11y_interactive_supports_focus`. These are compensating for a model that can't express "this role is interactive *in this context*."

4. **Doesn't generalise.** The next dual-nature role with an implicit HTML element will hit the same wall. We'd add another `if (role !== '...')` block.

**Verdict:** Fixes #9483 and preserves `<hr>`, but encodes ad-hoc knowledge about one role. Not a principled fix.

---

## 6. Our recommendation — broader rework needed

### 6.1 The underlying issue

**Inferring interactiveness purely from ARIA role name has drawbacks.**

The current model assumes each role is in one bucket: interactive, non-interactive, or (via manual exception) "interactive-capable". That works for roles that are always one thing (`button`, `article`, `heading`). It breaks for **context-dependent roles** where the same role name covers both structure and widget depending on attributes and behaviour.

`separator` is the clearest example today, but the pattern is general:

| Role | Static case | Widget case | Native element with implicit role? |
|------|-------------|-------------|-------------------------------------|
| `separator` | `<hr>`, decorative divider | Window splitter | **Yes** (`<hr>`) |
| `tabpanel` | — | Focusable panel content | No |
| (future) | … | … | … |

The `tabpanel` exception works only because there is no `<tabpanel>` tag. **`separator` exposes the flaw in the model itself.**

### 6.2 What a proper fix would look like (out of scope for a minimal PR)

A rework would centralise interactiveness into something like:

```
element_a11y_interactivity(node, role, attributes, handlers)
  → Static | NonInteractive | Interactive
```

Where:

- **Default** comes from role + native element (as today)
- **Override** when context signals widget behaviour: non-negative tabindex, keyboard/mouse handlers, required ARIA widget properties, etc.
- **Explicit vs implicit role** is a first-class input — so `<hr>` (implicit static separator) is never treated the same as `<div role="separator" tabindex="0" onkeydown>` (explicit interactive widget)
- Manual exception lists like `tabpanel` in `constants.js` get revisited under the same rules

This touches multiple rules in `index.js`, likely `constants.js`, and would need a careful pass over the ~50 a11y validator samples. It is a **maintainer-scale change**, not a one-file patch.

### 6.3 What we are proposing for #9483

1. **Acknowledge the bug.** The reporter's markup is spec-valid; Svelte's warnings are false positives.
2. **Do not merge Approach A or B.** Both are special-case hacks with tradeoffs we find unacceptable.
3. **Track as needing architectural work.** Either as a follow-up issue ("context-aware a11y interactiveness") or as part of a broader a11y refactor.
4. **Short-term for users:** `<!-- svelte-ignore a11y_no_noninteractive_tabindex -->` and `<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->` remain the correct workaround until the model is fixed.

---

## 7. Investigation artefacts (local WIP, not for upstream)

We have local branch work demonstrating Approach B (compiler changes + validator tests including `<hr>` regression cases). This was useful for understanding the problem space. **We do not intend to open a PR with this code.**

| File | What it contains |
|------|------------------|
| `a11y/index.js` | `is_dual_nature_role_interactive`, `is_dual_nature_role_with_handlers`, rule wiring |
| `a11y/constants.js` | Comment noting separator handled separately; no tabpanel-style exception |
| `a11y-no-noninteractive-element-interactions/` | Valid: static + full interactive separator; invalid: `<hr on:keydown>` |
| `a11y-no-noninteractive-tabindex/` | Valid: `<hr />`; invalid: `<hr tabindex="0">`, `<div role="separator" tabindex="0">` |
| `a11y-interactive-supports-focus/` | Invalid: `<div role="separator" on:keydown>` (handlers, no tabindex) |

---

## 8. References

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
| 2026-07-03 | Spec research (MDN, W3C APG); confirmed reporter markup is valid |
| 2026-07-03 | Traced warnings to `a11y/index.js` + `constants.js`; root cause = role-only classification |
| 2026-07-03 | TDD: failing tests for interactive separator; static case passes throughout |
| 2026-07-03 | **Approach A:** tabpanel-style exception — fixes #9483, breaks `<hr>` |
| 2026-07-03 | **Approach B:** context special-case for `separator` — fixes #9483, preserves `<hr>`, but is ad-hoc |
| 2026-07-03 | Added `<hr>` regression tests; explored both-or-neither contract for separator half-states |
| 2026-07-03 | **Decision:** neither approach is shippable; issue needs broader interactiveness rework (§6) |
| 2026-07-03 | Wrote up findings for issue / maintainer discussion (this document) |
