---
title: Stitch Design Workflow
tags: [stitch, design, mcp, workflow]
created: 2026-04-07
updated: 2026-04-07
status: active
related:
  - ../projects/the-bridge.md
---

# Stitch Design Workflow

How to use the Stitch MCP to generate designs and apply them to a codebase.

## Step 1: Generate designs in Stitch

Give Stitch a detailed text description of the app — purpose, layout, pages, content types, aesthetic direction. The more specific, the better. Include:
- Page names and what they show
- Layout structure (sidebar? tabs? grid?)
- Content types (prose, cards, stats, lists)
- Aesthetic references ("like Linear", "like Arc browser")

Stitch generates a full design system (colours, typography, spacing, components) plus individual screen HTML files.

## Step 2: Retrieve via MCP

```
mcp__stitch__list_projects          → find the project by title
mcp__stitch__list_screens           → get screen IDs + download URLs
```

Each screen has:
- `htmlCode.downloadUrl` — full self-contained HTML with Tailwind CDN
- `screenshot.downloadUrl` — PNG preview

## Step 3: Download and serve for preview

```bash
mkdir stitch-designs
curl -sL "<downloadUrl>" -o stitch-designs/page-name.html
python3 -m http.server 8090 --bind 0.0.0.0 &
# Preview at http://100.90.11.37:8090/page-name.html
```

## Step 4: Extract design tokens

The HTML files contain a `tailwind.config` script block with the full colour palette, font families, and border radius values. The `<style>` block has glass effects, scrollbar styles, and Material Symbols settings.

Key things to extract:
- **Colour tokens** from `tailwind.config.theme.extend.colors` → put in `globals.css` as `@theme inline` variables
- **Font families** → import from Google Fonts in `layout.tsx`
- **Glass/scrollbar CSS** → copy to `globals.css`
- **Component patterns** → extract Tailwind classes from the HTML structure

## Step 5: Apply to codebase

Work through each component, matching the Stitch HTML structure and classes to your React components. The HTML is the source of truth for:
- Exact Tailwind classes and colour tokens
- Layout structure (flex, grid, spacing)
- Typography scale (which font, weight, size for each element)
- Interactive states (hover, active, focus)

## Gotchas

- Stitch uses Tailwind CDN (v3 syntax) — you may need to adapt for Tailwind v4 (`@theme inline` instead of `tailwind.config.js`)
- Stitch generates placeholder content — replace with real data bindings
- Material Symbols need a Google Fonts stylesheet link (not an npm package)
- The design system markdown (`designMd` field in the project) contains detailed Do's/Don'ts — read it before implementing
