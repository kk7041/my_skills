---
name: design-system-theming
version: 1.0.0
description: |
  Build a reusable, multi-theme design system where structure and contracts are
  shared but each site/brand has a different look. Use when the user wants to:
  - Set up a design system that must support several visual styles
  - Add or swap a visual theme (brand, dark mode, sub-product skin)
  - Refactor hardcoded colors/fonts into reusable design tokens
  - Apply one component library across differently-styled surfaces
  Core idea: reuse the structure and token contract, isolate "style" into the
  only replaceable layer (a per-theme set of variable values).
---

# Design system theming

Reuse the mechanism, not the colors. Different sites look different because they
fill the *same contract* with different values, not because they each rewrite a
new visual system. Components, semantic token names, utility classes, and
density rules stay constant across themes.

## When to use

- The project needs more than one visual style (multi-brand, landing vs app, light/dark, client skins).
- You are about to hardcode hex/oklch/fonts in components.
- You want a new project to inherit an existing design-system approach.

## When not to use

- A single fixed style with no foreseeable variants. Plain shadcn defaults are enough.
- Pure one-off styling with no reuse intent.

## Core principle: stable contract + replaceable skin

All style differences collapse into "a raw palette + a set of theme knobs."
Everything else is shared.

Four layers, from most to least stable. Reuse the top three; each new style only
replaces the bottom layer.

| Layer | What | Reuse |
| --- | --- | --- |
| Spec docs | density / typography / spacing discipline | ~90% inherited |
| Components | the UI component library (e.g. shadcn `ui/*`) | ~100%, never bind colors |
| Semantic tokens | `--primary` / `--background` / `--radius` ... | name is the contract, unchanged |
| Raw palette | concrete hex/oklch, fonts, radius values | 100% replaced per theme |

## Step 1: Identify the target stack

The mechanism is "CSS variables + a bridge that maps variables to utilities."
Only the bridge changes per stack.

| Stack | Adaptation |
| --- | --- |
| Tailwind v4 + shadcn | Use `@theme inline` to map `--color-*` to `var(--*)`. Direct reuse. |
| Tailwind v3 + shadcn | Map via `tailwind.config` `theme.extend.colors` referencing CSS vars (shadcn v3 default). |
| Plain CSS / no Tailwind | Keep primitive+semantic vars; components consume `var(--primary)` directly, skip the utility bridge. |
| CSS-in-JS | Drive theme through CSS vars; scope via ThemeProvider or a root class. |

## Step 2: Token three-layer model

Do not let components touch raw values. Three tiers:

```css
/* 1. primitive: raw palette, defined per theme, only used inside that theme */
--c-brand-500: #D8A060;
--c-neutral-900: #333333;

/* 2. semantic: role names, fixed across themes -- this IS the reuse contract */
--primary: var(--c-brand-500);
--foreground: var(--c-neutral-900);

/* 3. component bridge: utilities consume semantic only, never primitive */
@theme inline {        /* Tailwind v4; v3 uses tailwind.config */
  --color-primary: var(--primary);
  --color-foreground: var(--foreground);
}
```

Naming:
- primitive: `--c-{family}-{step}` (e.g. `--c-brand-500`). Pure palette fact, no meaning.
- semantic: reuse shadcn names (`--background`, `--foreground`, `--primary`, `--muted`, `--accent`, `--border`, `--ring`, `--radius`, `--sidebar-*`, `--chart-*`). Do not invent new ones unless a genuinely new role exists.
- brand raw colors for highly visual components: `--brand-{name}` (e.g. `--brand-cream`).

## Step 3: Set up the contract skeleton (one time)

In the global stylesheet, define the `:root` semantic tokens and the bridge
(`@theme inline` for v4). This skeleton is written once and serves every theme.

## Step 4: Fill the first theme

Copy `templates/theme-contract.css` and fill one set of primitive + semantic
values. Mount it on the root scope.

## Step 5: Add more themes (the payoff)

For each extra style, copy the contract template, fill values, and add one scope
class, e.g. `.theme-<name> { ... }`. Mount it on the outer container:

```tsx
<div className="theme-<name> bg-background min-h-screen">...</div>
```

All `ui/*` components and token utilities inside automatically re-skin. The same
`<Button>` looks different in different scopes with zero component changes.

Per-theme knobs to fill (the "theme contract"):

| Knob | Examples |
| --- | --- |
| palette | warm cream / cool slate / high-contrast mono |
| `--radius` | `0rem` (brutalist) / `0.5rem` / `0.75rem` (friendly) |
| fonts (`--font-sans/--font-mono/--font-heading`) | IBM Plex / Nunito / Inter / pixel |
| shadow strength | none / `shadow-xs` / layered / `glow` |
| density | compact (`h-8`) / comfortable (`h-10`) |
| texture / motion | dot-grid / noise / glitch / none |

If a theme needs fonts, inject `--font-*` via the framework's font loader
(e.g. `next/font` in `layout.tsx`), then reference them in the theme. Put
theme-specific textures/animations behind scoped utilities (e.g. `.theme-x .dot-grid-bg`).

## Step 6: Lock the rules

- Components use semantic token utilities only (`bg-background`, `text-primary`, `border-border`, `rounded-lg`). No hardcoded hex/oklch.
- Reuse the component library; never invent a per-page visual system.
- Inherit a density/typography baseline (4/8px grid, control heights `h-8/9/10`, icons `size-4`, `text-sm leading-5` / `text-xs leading-4`). Override per theme only where needed, always scoped.

## Reference files

- `templates/theme-contract.css` — fill-in CSS skeleton (primitive + semantic + scope).
- `templates/strategy.md` — full methodology, four-layer model, filled examples, checklist.

## Verify it worked

- Switching only the scope class changes the whole look, with no component edits.
- `grep` the components for raw color literals (`#`, `oklch(`, `rgb(`): there should be none outside the token/theme layer and intentionally-visual components.
- Each new theme touched only primitive + semantic mappings.
- Semantic token names are unchanged across themes.

## What not to do

- Do not hardcode colors in components to bypass tokens.
- Do not duplicate the component library to make a new theme; theme diffs live only in the variable layer.
- Do not redefine the component bridge (`@theme inline` / config map) per theme; the bridge is a global contract.
- Do not override `--font-*` locally to a non-theme font unless explicitly asked.
- Do not nest cards inside cards or use ad-hoc spacing.
