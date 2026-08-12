# Shipyard — shadcn v4 / Tailwind v4 Theme Export

Harbor Amber theme export for implementation. **Status: finalized 2026-08-12.**

## Files

| File | Purpose |
|---|---|
| `globals.css` | Complete shadcn v4 theme scaffold — paste into your app's global CSS (replaces the default `:root` / `.dark` blocks) |
| `components.json` | shadcn CLI config template. Run `npx shadcn@latest init` in the app repo and answer the prompts — the CLI generates the canonical version; adjust `css`/`aliases` to your app layout |

## Token mapping (ds-* → shadcn variables)

### Light (`:root`)

| shadcn var | ds token | Value |
|---|---|---|
| `--background` | `ds-bg` | `#F4F3EF` |
| `--foreground` | `ds-text` | `#171717` |
| `--card` / `--popover` | `ds-surface` | `#FFFFFF` |
| `--primary` | `ds-brand` | `#B45309` |
| `--primary-foreground` | — (button default) | `#FFFFFF` |
| `--secondary` / `--muted` | neutral quiet surface | `#F0EFEB` |
| `--muted-foreground` | `ds-text-muted` | `#6C6861` |
| `--accent` | `ds-brand-soft` | `#FFF4DB` |
| `--accent-foreground` | `ds-brand` | `#B45309` |
| `--destructive` | `ds-danger` | `#B42318` |
| `--border` / `--input` | `ds-border` | `#DEDCD5` |
| `--ring` | `ds-focus` | `#F59E0B` |
| `--sidebar` | `ds-sidebar` | `#EEEDE8` |
| `--sidebar-accent` | active nav (canvas) | `#FFFFFF` |
| `--chart-1..5` | brand, accent, success, info, warning | `#B45309` `#F59E0B` `#2E7D5B` `#2563EB` `#A65F00` |

### Dark (`.dark`)

| shadcn var | Canvas Dark Foundation | Value |
|---|---|---|
| `--background` / `--sidebar` | `ds-sidebar-dark` (tokenized) | `#161512` |
| `--foreground` | `ds-text-dark` | `#F7F4ED` |
| `--card` / `--popover` / `--muted` | `ds-surface-dark` | `#1B1916` |
| `--primary` / `--ring` | `ds-accent` | `#F59E0B` |
| `--primary-foreground` | dark on amber | `#161512` |
| `--secondary` / `--accent` / `--border` / `--input` | `ds-accent-soft-dark` | `#332513` |
| `--accent-foreground` | amber on warm tint | `#F59E0B` |
| `--muted-foreground` | `ds-text-muted-dark` | `#AAA39A` |

## Decisions & notes

- **Radius:** `--radius-sm/md/lg/xl` are fixed to the exact ds scale (4/8/12/16px); the extended scale continues the 4px rhythm (2xl 20, 3xl 24, 4xl 28). Pills use `rounded-full` (999px).
- **Fonts:** `--font-sans` = Inter, `--font-mono` = Geist Mono. Load the fonts in the app shell (next/font, @fontsource, or link tags) — globals.css only declares the stacks.
- **ds-* utilities:** a bonus `@theme` block exposes the full palette as utilities (`bg-ds-surface`, `text-ds-muted`, `border-ds-border-strong`, …) for one-off design-system usage.
- **Dark tokens:** only `ds-sidebar-dark` is tokenized in `shipyard.pen` today; the other dark values are approved canvas samples. Tokenize them in the canvas when dark-mode implementation starts (design.md §15 tracks this).
- **Charts:** `--chart-1..5` reuse the semantic family (brand, accent, success, info, warning) so dashboards stay inside the system.
- **Animations:** the `@import "shadcn/tailwind.css"` line is the current shadcn v4 default; if your CLI version doesn't emit it, use `@import "tw-animate-css";` instead.
