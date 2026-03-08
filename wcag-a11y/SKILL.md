---
name: wcag-a11y
description: >
  Apply WCAG 2.2 accessibility standards when building UI components, forms, navigation,
  modals, interactive elements, or any frontend/UX code. Use this skill whenever the user
  asks you to build, modify, review, or audit any UI element, component, layout, or user
  interaction — even if they don't mention "accessibility" or "a11y" explicitly. Also
  trigger for tasks like building buttons, inputs, dialogs, menus, tables, carousels,
  animations, color choices, or anything a user will see or interact with. Always keep
  WCAG 2.2 AA as the minimum bar, and call out AAA improvements when feasible.
---

# WCAG 2.2 Accessibility Skill

This skill ensures that any UI/UX code Claude produces or reviews meets **WCAG 2.2 AA** standards — the globally accepted legal compliance level — and nudges toward AAA where practical.

**Target level:** WCAG 2.2 AA (56 criteria: all Level A + all Level AA)  
**Reference:** [WCAG 2.2 Quick Reference](https://www.w3.org/WAI/WCAG22/quickref/)  
**Full criteria list:** See `references/wcag-22-criteria.md`

---

## Core Workflow

When writing or reviewing UI code, always:

1. **Apply the POUR checklist** (see below) to the component being built
2. **Annotate your code** with inline comments explaining a11y choices (e.g., `// aria-label required — icon-only button`)
3. **Flag any WCAG gaps** at the end of your response with the criterion number and a fix
4. **Suggest AAA improvements** when they're low-effort

---

## The POUR Principles — Quick Checklist

### 🔍 Perceivable
Users must be able to perceive all content and UI.

- [ ] All images/icons have meaningful `alt` text (or `alt=""` if decorative) — **1.1.1 A**
- [ ] Color is never the *only* way to convey information — **1.4.1 A**
- [ ] Text contrast ≥ 4.5:1 (normal text), ≥ 3:1 (large text/18px+ or bold 14px+) — **1.4.3 AA**
- [ ] UI component contrast ≥ 3:1 (borders, icons, focus rings vs background) — **1.4.11 AA**
- [ ] Content reflows at 400% zoom without horizontal scrolling — **1.4.10 AA**
- [ ] Text can be resized to 200% without loss of functionality — **1.4.4 AA**
- [ ] Avoid images of text; use real text — **1.4.5 AA**
- [ ] Hover/focus tooltips: dismissable (Escape), hoverable, persistent — **1.4.13 AA**
- [ ] Text spacing can be adjusted without content loss — **1.4.12 AA**
- [ ] Structural info conveyed in code (not just visually) — **1.3.1 A**
- [ ] Reading order in DOM matches visual order — **1.3.2 A**
- [ ] Don't rely on shape/size/position/color alone as instructions — **1.3.3 A**
- [ ] UI works in both portrait and landscape — **1.3.4 AA**
- [ ] Form fields have `autocomplete` attributes where applicable — **1.3.5 AA**

### ⌨️ Operable
Users must be able to operate all UI with a variety of input methods.

- [ ] All functionality accessible by keyboard alone — **2.1.1 A**
- [ ] No keyboard traps — **2.1.2 A**
- [ ] Single-character keyboard shortcuts can be turned off or remapped — **2.1.4 A**
- [ ] Focus order is logical and meaningful — **2.4.3 A**
- [ ] Focus indicator is clearly visible — **2.4.7 AA**
- [ ] Focused element not completely hidden (e.g. by sticky header/footer) — **2.4.11 AA** *(new in 2.2)*
- [ ] Skip links provided to bypass repeated blocks — **2.4.1 A**
- [ ] Links have descriptive text (not "click here") — **2.4.4 A**
- [ ] Headings and labels describe content — **2.4.6 AA**
- [ ] Touch/pointer targets ≥ 24×24 CSS pixels — **2.5.8 AA** *(new in 2.2)*
- [ ] Multi-touch gestures have single-pointer alternatives — **2.5.1 A**
- [ ] Drag-and-drop has single-pointer alternative — **2.5.7 AA** *(new in 2.2)*
- [ ] Visible label text is part of accessible name — **2.5.3 A**
- [ ] Motion-activated actions have alternatives; can be disabled — **2.5.4 A**
- [ ] Avoid content that flashes >3 times/sec — **2.3.1 A**
- [ ] Provide a way to pause/stop moving or auto-updating content — **2.2.2 A**
- [ ] Animations from interactions can be disabled (respect `prefers-reduced-motion`) — **2.3.3 AAA** *(low-effort, always do this)*

### 🧠 Understandable
Users must be able to understand the content and how the interface works.

- [ ] Page language set in `<html lang="...">` — **3.1.1 A**
- [ ] No context changes on focus alone — **3.2.1 A**
- [ ] No context changes on input alone (or user is warned) — **3.2.2 A**
- [ ] Navigation is consistent across pages — **3.2.3 AA**
- [ ] Components with same function are identified consistently — **3.2.4 AA**
- [ ] Help (if present) appears in a consistent location — **3.2.6 A** *(new in 2.2)*
- [ ] Form errors are automatically identified and described in text — **3.3.1 A**
- [ ] Labels or instructions given for user inputs — **3.3.2 A**
- [ ] Error suggestions provided where possible — **3.3.3 AA**
- [ ] Previously entered information is not required again in the same session — **3.3.7 A** *(new in 2.2)*
- [ ] Authentication doesn't require cognitive function tests (e.g. CAPTCHAs without alternatives) — **3.3.8 AA** *(new in 2.2)*

### 🤖 Robust
Content must work with current and future assistive technologies.

- [ ] All UI components have proper `name`, `role`, and `value` — **4.1.2 A**
- [ ] Status messages (success/error toasts, live regions) announced to AT without focus — **4.1.3 AA**
- [ ] Use semantic HTML first; reach for ARIA only to supplement, never replace semantics

---

## ARIA Usage Rules

Follow the **First Rule of ARIA**: if a native HTML element exists, use it.

```
✅ <button>Submit</button>
❌ <div role="button" tabindex="0">Submit</div>  ← use only if unavoidable
```

When ARIA is necessary:
- `aria-label` — for elements with no visible text (icon buttons, landmarks)
- `aria-labelledby` — reference visible text as label
- `aria-describedby` — supplementary description (hint text, error messages)
- `aria-expanded` / `aria-haspopup` — for disclosure/menu patterns
- `aria-live="polite"` / `aria-live="assertive"` — for dynamic content updates
- `role="dialog"` + `aria-modal="true"` + focus trap — for modals
- `aria-required`, `aria-invalid`, `aria-errormessage` — for form validation

Never use `aria-hidden="true"` on focusable elements.

---

## Common Component Patterns

### Buttons
```html
<!-- Icon-only button MUST have aria-label -->
<button aria-label="Close dialog">
  <svg aria-hidden="true">...</svg>
</button>

<!-- Loading state -->
<button aria-disabled="true" aria-busy="true">
  <span aria-hidden="true">⏳</span> Saving...
</button>
```

### Forms
```html
<label for="email">Email address</label>
<input
  id="email"
  type="email"
  autocomplete="email"
  aria-describedby="email-hint email-error"
  aria-invalid="true"   <!-- when in error state -->
  required
/>
<div id="email-hint">We'll never share your email.</div>
<div id="email-error" role="alert">Enter a valid email address.</div>
```

### Modal / Dialog
```html
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="dialog-title"
  aria-describedby="dialog-desc"
>
  <h2 id="dialog-title">Confirm delete</h2>
  <p id="dialog-desc">This action cannot be undone.</p>
  <!-- Focus trap: Tab/Shift+Tab cycle within; Escape closes -->
</div>
```

### Navigation
```html
<nav aria-label="Main navigation">
  <ul>
    <li><a href="/" aria-current="page">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>
```

---

## Focus Management Rules

- **Modals/drawers**: move focus to first interactive element (or the dialog itself) on open; return focus to trigger on close
- **Route changes (SPAs)**: announce new page title and move focus to `<main>` or an `<h1>`
- **Toast notifications**: use `role="status"` or `role="alert"` — do NOT steal keyboard focus
- **Expandable content**: focus stays on trigger; `aria-expanded` reflects state

---

## Color Contrast Quick Reference

| Text type | Minimum (AA) | Enhanced (AAA) |
|---|---|---|
| Normal text (<18px regular, <14px bold) | 4.5:1 | 7:1 |
| Large text (≥18px regular, ≥14px bold) | 3:1 | 4.5:1 |
| UI components & graphics | 3:1 | — |
| Decorative / inactive controls | No requirement | — |

Recommended tools: [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/), browser DevTools color picker.

---

## CSS: Accessible Patterns to Always Include

```css
/* 1. Respect reduced motion */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}

/* 2. Visible focus ring — never just remove outline */
:focus-visible {
  outline: 2px solid #005fcc;
  outline-offset: 2px;
}
/* Remove outline only for pointer users */
:focus:not(:focus-visible) {
  outline: none;
}

/* 3. Skip link */
.skip-link {
  position: absolute;
  top: -100%;
  left: 0;
}
.skip-link:focus {
  top: 0;
}
```

---

## What to Say at the End of Every UI Response

After writing any UI code, append an **Accessibility Notes** section:

```
## Accessibility Notes

✅ Applied: [list key a11y choices made]
⚠️ Watch out for: [any gaps the user should handle, e.g. dynamic content, routing]
💡 AAA improvement: [optional enhancements]
```

---

## Reference Files

- `references/wcag-22-criteria.md` — Full list of all 87 WCAG 2.2 criteria with levels
  - Read this when you need to check a specific criterion number or find criteria by principle
