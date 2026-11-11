# Frontend Redesign Delegation — Critical Lesson

## The Problem
When delegating a visual redesign (e.g., light-theme overhaul of an existing site), a subagent can:
1. Pass the build cleanly
2. Apply many patches to page.tsx
3. Still miss the root cause — the **design system component defaults** weren't updated

The build passes. The visual spec fails silently.

## Root Cause
Pages in Next.js apps often use shared UI components (`Section`, `Modal`, `Button`, `Card`). These components have `background` and `padding` defaults. A subagent patching `page.tsx` will use the component but the component's defaults may be hardcoded for the old theme. Patching every section in page.tsx doesn't fix the component's internal defaults.

## Real Example
- Goal: convert the health client from dark theme to white/light theme
- Subagent updated `globals.css` (new colors) and `page.tsx` (section backgrounds swapped from `dark`/`darker` to `white`/`gray`)
- **But** `Section.tsx` component had `background = 'dark'` as its default
- Result: build passes but dark backgrounds still render because component defaults weren't patched
- Fix: also patch `Section.tsx` — add `white` and `gray` background options + update padding map

## Protocol for Design System Overhauls
When the task is a theme/color-system change on an existing site with shared components:

1. **Before delegating**: Identify ALL shared UI components that might carry theme state:
   `Section`, `Modal`, `Card`, `Button` — read their source, check defaults
2. **Delegate with full context**: Include component file paths and specific default values needing change
3. **After subagent completes**: Run build, then **visually verify** via browser — don't trust exit code for visual correctness
4. **If visual mismatch**: Subagent likely fixed page.tsx but missed component defaults. Read component source directly.