# Accessibility Standards (WCAG 2.1 AA)

**Web Content Accessibility Guidelines (WCAG)** exist so websites and web applications are usable by people with disabilities — including users who rely on screen readers, keyboard-only navigation, magnification, high contrast, or reduced motion.

ThriveHR targets **WCAG 2.1 Level AA** for all product UI (`thrivehr-ui`). This is a **non-negotiable** quality bar alongside responsive layout and light/dark themes.

---

## What WCAG Means for ThriveHR

| Principle | Meaning | Examples in our stack |
|-----------|---------|------------------------|
| **Perceivable** | Users can see or hear content | Labels on inputs, alt text, sufficient contrast, captions for video |
| **Operable** | Users can navigate and interact | Keyboard access, visible focus, no keyboard traps, 44px touch targets |
| **Understandable** | UI is predictable and readable | Consistent nav, error messages tied to fields, `lang` on `<html>` |
| **Robust** | Works with assistive tech | Semantic HTML, Radix/shadcn ARIA, valid roles and names |

**Level AA** (our minimum) includes contrast ratios, resize text to 200% without loss, focus visible, and error identification on forms.

---

## Tooling (required in `thrivehr-ui`)

| Tool | Purpose |
|------|---------|
| **eslint-plugin-jsx-a11y** | Catch missing labels, invalid ARIA, click handlers on non-interactive elements at lint time |
| **vitest-axe** + **jest-axe** matchers | Automated WCAG rule checks in component tests |
| **@axe-core/playwright** | E2E accessibility scans on critical flows |
| **eslint-plugin-testing-library** | Prefer `getByRole` / `getByLabelText` over test IDs |

Full install and config: [assets/accessibility-setup-template.md](../assets/accessibility-setup-template.md).

---

## Document & Page Shell

### `index.html` baseline

- `<html lang="en-IN">` (or product locale when i18n ships)
- `<title>` unique per route (update via `react-helmet-async` or router `document.title`)
- **Skip link** as first focusable element — see [assets/skip-to-content-template.md](../assets/skip-to-content-template.md)
- Viewport allows zoom: `width=device-width` — do **not** use `user-scalable=no` or `maximum-scale=1`

### Landmark structure (`AppShell`)

Every authenticated layout must expose landmarks:

```tsx
<body>
  <a href="#main-content" className="skip-link">Skip to main content</a>
  <header role="banner">…</header>
  <nav aria-label="Primary">…</nav>
  <main id="main-content" tabIndex={-1}>…</main>
  <footer role="contentinfo">…</footer>
</body>
```

- **One** `<main>` per page
- Sidebar nav: `aria-label="Primary navigation"` (or "Admin", "Recruiter", etc. if role-specific)
- Current page: `aria-current="page"` on active nav link

### Page titles

Update on route change so screen reader users know where they are:

```typescript
// Example in route layout or page component
useEffect(() => {
  document.title = `Job listings · ThriveHR`;
}, []);
```

Format: `{Page name} · ThriveHR`.

---

## Color & Contrast (Axis burgundy)

WCAG 2.1 AA contrast minimums:

| Content | Ratio |
|---------|-------|
| Normal text (&lt; 18px / 14px bold) | **4.5:1** |
| Large text (≥ 18px / 14px bold) | **3:1** |
| UI components & graphical objects | **3:1** |
| Focus indicator | **3:1** against adjacent colors |

### Approved combinations (light mode)

| Use | Foreground | Background | Notes |
|-----|------------|------------|-------|
| Primary button | `#FFFFFF` | `#971237` (burgundy) | Default CTA — passes AA |
| Body text | `#1A1A1A` (charcoal) | `#FFFFFF` | Body copy |
| Secondary text | `#4A4A4A` (slate) | `#FFFFFF` | Captions, hints |
| Links (inline) | burgundy + underline | white | Do not rely on color alone |
| Focus ring | `--ring` (burgundy) | page bg | `focus-visible:ring-2 ring-ring ring-offset-2` |

### Rules

- **Never** use burgundy (`#971237`) for paragraph-length body text on white
- **Never** convey state by color alone — add icon, text, or `aria-label`
- Verify **both** `:root` and `.dark` tokens in `globals.css` before merge
- Destructive/success/warning: pair icon + text label

### High contrast / forced colors

Do not remove focus outlines globally. shadcn focus styles must remain visible. Test with Windows **High contrast** or browser forced-colors mode for critical flows.

---

## Keyboard & Focus

### Requirements

- All interactive controls reachable and operable via **Tab** / **Shift+Tab**
- **Enter** / **Space** activate buttons and toggles
- **Escape** closes dialogs, sheets, dropdowns, popovers (Radix default — do not override)
- No **keyboard trap** except intentional focus trap in modals (Radix `Dialog` handles this)
- **Focus order** matches visual order (avoid positive `tabIndex` except `tabIndex={-1}` for programmatic focus)

### Focus visibility

Use Tailwind `focus-visible:` (not `focus:` only) so mouse users do not see redundant rings:

```tsx
<Button className="focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2" />
```

### Focus management patterns

| Scenario | Pattern |
|----------|---------|
| Open dialog | Radix moves focus to dialog; keep `DialogTitle` + `DialogDescription` |
| Close dialog | Return focus to trigger (Radix default) |
| Route change | Move focus to `<main>` or page `<h1>`: `mainRef.current?.focus()` |
| Submit success | Announce via `aria-live` or toast with `role="status"` (sonner) |
| Delete confirm | Focus primary safe action ("Cancel") or explicit destructive label |

### Skip link

First tab stop on every page load. Visible on focus only. See asset template.

---

## Forms (react-hook-form + shadcn)

### Labels

- Every input has visible `<FormLabel>` linked via `htmlFor` / `id`
- Placeholder is **not** a label
- Required fields: `(required)` in label or `aria-required="true"` + visual indicator
- Optional fields: mark as optional in label when most fields are required

### Errors

- Associate errors with fields: `aria-invalid="true"` + `aria-describedby` pointing to error id
- shadcn `FormMessage` renders error text — ensure id linkage:

```tsx
<FormItem>
  <FormLabel htmlFor="email">Email</FormLabel>
  <FormControl>
    <Input id="email" aria-invalid={!!fieldState.error} aria-describedby={fieldState.error ? 'email-error' : undefined} {...field} />
  </FormControl>
  <FormMessage id="email-error" />
</FormItem>
```

- On submit with errors: focus **first invalid field** and optionally announce summary region with `role="alert"`

### Groups

- Related radios/checkboxes: `<fieldset>` + `<legend>` or `role="group"` + `aria-labelledby`
- Select: use shadcn `Select` (Radix) — not unstyled native unless styled and labeled

---

## Buttons, Links, and Icon-Only Controls

| Control | Requirement |
|---------|-------------|
| `<Button>` | Visible text or `aria-label` / `aria-labelledby` |
| Icon-only | **Mandatory** `aria-label` (e.g. `aria-label="Close dialog"`) |
| Link vs button | Navigation → `<Link>` or `<a href>`; actions → `<button type="button">` |
| Loading button | `aria-busy="true"` + disable; label like "Saving…" not spinner alone |
| Toggle | `aria-pressed` or Radix `Switch` with label |

Theme toggle example:

```tsx
<Button variant="ghost" size="icon" aria-label={theme === 'dark' ? 'Switch to light mode' : 'Switch to dark mode'}>
  {theme === 'dark' ? <Sun /> : <Moon />}
</Button>
```

---

## Tables (@tanstack/react-table)

- Wrap in `<table>` with `<caption>` (visually hidden if redundant with page title) or `aria-label` on table
- Column headers: `<th scope="col">`; row headers: `<th scope="row">` when applicable
- Sortable columns: `aria-sort="ascending" | "descending" | "none"` on active header
- Actions column: header text "Actions" (not empty)
- Mobile card layout: each card exposes same information with headings, not color-only status

---

## Dialogs, Sheets, and Menus

shadcn/Radix primitives provide baseline ARIA. **Do not** remove:

- `DialogTitle` — required (can use `VisuallyHidden` if design hides title)
- `DialogDescription` when content needs context
- `SheetTitle` / `AlertDialogTitle` for confirmations

Destructive actions:

```tsx
<AlertDialogTitle>Delete job listing?</AlertDialogTitle>
<AlertDialogDescription>
  This cannot be undone. Applications in progress will remain.
</AlertDialogDescription>
```

---

## Images, Icons, and Media

| Type | Rule |
|------|------|
| Informative image | `alt` describes purpose (not filename) |
| Decorative image | `alt=""` or `aria-hidden="true"` on SVG wrapper |
| lucide icons next to text | `aria-hidden="true"` on icon; text provides name |
| Icon-only | Parent has `aria-label` |
| Charts | Text summary or `aria-label` / table fallback for data |
| Video | Captions + transcript when product includes video |

---

## Dynamic Content & Toasts

| Pattern | Implementation |
|---------|----------------|
| Loading | `aria-busy="true"` on region or Skeleton with `aria-label="Loading jobs"` |
| Empty state | Heading + description; action button labeled |
| Toasts (sonner) | Library sets live region; use clear messages ("Job saved") not "Success" |
| Async updates | `aria-live="polite"` for non-critical; `aria-live="assertive"` for errors |
| Route loading | `Suspense` fallback with accessible loading text, not blank screen |

---

## Motion & Animation

Respect `prefers-reduced-motion`:

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

- No auto-playing carousels without pause control
- Avoid flashing content (&gt; 3 flashes per second)

---

## Responsive & Touch (overlap with `frontend.md`)

- Minimum **44×44 CSS px** touch target for buttons and links
- Pinch-zoom not disabled
- Tables: horizontal scroll region with `tabIndex={0}` and visible focus if scrollable

---

## Radix / shadcn Checklist

Radix primitives ship accessible behavior. Common regressions to avoid:

- [ ] Removing `DialogTitle` because design lacks a title
- [ ] Custom `div` with `onClick` instead of `Button`
- [ ] `pointer-events-none` on focusable elements
- [ ] Truncating focus ring with `overflow-hidden` on parents without testing
- [ ] Duplicate `id` attributes across portals
- [ ] Autocomplete off on fields that benefit from `autoComplete` (names, email)

---

## Testing Accessibility

### Unit / component (Vitest + RTL + axe)

```typescript
import { axe } from 'vitest-axe';
import { render } from '@testing-library/react';

test('JobForm has no axe violations', async () => {
  const { container } = render(<JobForm />);
  expect(await axe(container)).toHaveNoViolations();
});
```

Prefer queries that mirror assistive tech:

```typescript
screen.getByRole('button', { name: /create job/i });
screen.getByLabelText(/email/i);
// Avoid: getByTestId unless no semantic alternative
```

### E2E (Playwright + axe)

```typescript
import AxeBuilder from '@axe-core/playwright';

test('dashboard meets WCAG 2.1 AA', async ({ page }) => {
  await page.goto('/dashboard');
  const results = await new AxeBuilder({ page }).withTags(['wcag2a', 'wcag2aa']).analyze();
  expect(results.violations).toEqual([]);
});
```

Run axe on **critical flows** in CI (login, job list, apply, upload). Track violations; fail build on new critical issues.

### Manual smoke (per feature PR)

- [ ] Tab through entire flow without mouse
- [ ] Focus always visible
- [ ] Screen reader spot-check (NVDA on Windows or VoiceOver on macOS): page title, landmarks, form errors
- [ ] 200% browser zoom — no clipped primary actions
- [ ] Light and dark mode contrast check

---

## PR / Definition of Done

Every UI PR must satisfy [frontend.md usability checklist](frontend.md#usability-checklist-per-feature-pr) **plus**:

- [ ] Keyboard-complete primary flow
- [ ] Icon-only controls have accessible names
- [ ] Form errors linked to fields; focus first error on submit
- [ ] axe clean (or documented exceptions with ticket)
- [ ] Tested in light and dark mode
- [ ] No WCAG contrast regressions on burgundy primary actions

---

## Exceptions & Documentation

If a third-party widget cannot meet AA:

1. Document in PR with ticket and mitigation (alternative path, support contact)
2. Do not block entire app — isolate and plan replacement
3. Never exempt auth/login flows

---

## Reference Links

- [WCAG 2.1 Quick Reference](https://www.w3.org/WAI/WCAG21/quickref/?levels=aa)
- [WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/)
- [Radix UI Accessibility](https://www.radix-ui.com/primitives/docs/overview/accessibility)
- [Deque axe rules](https://github.com/dequelabs/axe-core/blob/develop/doc/rule-descriptions.md)
