---
inclusion: fileMatch
fileMatchPattern: ['app/views/**', 'app/components/**', 'app/javascript/**', 'app/assets/tailwind/**', 'config/locales/**']
---

# UI, components and design system

Full guidance: [design system](../../docs/llm-guides/design-system.md) and
[UI guide](../../docs/llm-guides/ui.md). Reviewers escalate DS reuse and
repeated-shape violations to close/rewrite; token, icon/SVG, localization and
scale violations are request-changes.

## Design tokens

- Use functional tokens from `app/assets/tailwind/sure-design-system.css`:
  `text-primary`, `bg-container`, `border-primary`, `bg-warning/10`,
  `text-destructive`, etc.
- **No raw Tailwind palette classes and no hex literals.**
- Use scale tokens instead of arbitrary pixel values when a scale token fits.
- Adding styles to `sure-design-system.css` or `application.css` requires
  **explicit permission**.

## Components

- Check `app/components/DS/` first — use `DS::Alert`, `DS::Button`, `DS::Disclosure`,
  `DS::Dialog`, `DS::Menu`, `DS::Table` and other existing primitives before building
  an alert, badge, button, disclosure, dialog, input or table shape.
- If the same hand-built shape appears at least twice in a diff with no DS equivalent,
  propose a new `DS::*` primitive before the second copy lands.
- Render data tables with `DS::Table`, not hand-written `<table>` markup.
- ViewComponent for reusable/complex/interactive/configurable UI; partials for
  mostly static, simple, context-specific content. Shared partials in
  `app/views/shared/` (underscore prefix).
- Follow the `DS::*` namespace; do not use the obsolete `ButtonComponent`/
  `DialogComponent` examples.

## Icons and SVG

- Use the `icon` helper from `ApplicationHelper`, **never `lucide_icon` directly**.
- No raw SVG outside DS primitives.

## Hotwire and Stimulus

- Prefer semantic native HTML (`<dialog>`, `<details><summary>`) via DS wrappers.
- Keep display state in URL query parameters before local storage or sessions.
- Format money/numbers/dates server-side; pass display values to Stimulus.
- Declare Stimulus actions in HTML (`data-action="click->toggle#toggle"`); pass data
  via `data-*-value`; use targets over `getElementById`. Aim for fewer than seven targets.
- Keep domain logic out of ERB.

## Localization

- Use `t()` for all user-facing strings; update `config/locales/`.
- Descriptive hierarchical keys (`accounts.index.title`). Use interpolation and
  Rails pluralization for dynamic text.
