# CSS Pitfalls Reference

## Flexbox Height Collapse (Critical)

**Symptom:** A panel, sidebar, or tab content area renders with zero height or appears "cut off" at the top — content may still be accessible by scrolling, but the container has no visible bounds.

**Root Cause:** In a `display: flex; flex-direction: column` layout, a flex child that is itself a flex parent with `overflow: hidden` or `overflow-y: auto` will collapse to `height: 0` unless `min-height: 0` is set. Default `min-height: auto` on flex children prevents them from shrinking below their content size, breaking the intended flex distribution.

**The Fix:** Add `min-height: 0` to every intermediate flex container in the chain:

```css
/* The outer shell that constrains height */
.page-root {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

/* Intermediate flex parent — THE CULPRIT when missing */
.content-area {
  flex: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  min-height: 0;       /* ← REQUIRED */
}

/* The inner scrollable container */
.tab-pane {
  height: 100%;
  overflow-y: auto;
  min-height: 0;       /* ← REQUIRED if also a flex parent */
}

/* Any OS/panel root that lives inside the tab-pane */
.os-root {
  display: flex;
  flex-direction: column;
  height: 100%;
  min-height: 0;      /* ← REQUIRED */
  overflow: hidden;
}
```

**Chain Rule:** If A → B → C are all `display: flex` with `height: 100%` or `flex: 1` in a column layout, every one needs `min-height: 0`. Missing it on any link causes collapse.

---

## Mobile Drawer: `position: fixed` Must Be Scoped to Mobile

**Symptom:** Sidebar works on mobile but causes layout shifts on desktop — content "pushes down" or sidebar disappears.

**Root Cause:** `position: fixed` applied to a sidebar globally (no media query). Fixed removes the element from document flow, so siblings reflow as if it doesn't exist.

**Wrong:**
```css
/* Global — breaks desktop layout */
.sidebar {
  position: fixed;
  transform: translateX(-110%);
  z-index: 50;
}
.sidebar--open { transform: translateX(0); }
```

**Correct:**
```css
/* Base: sidebar in normal document flow for desktop */
.sidebar {
  width: 220px;
  flex-shrink: 0;
  /* No position: fixed here */
}

/* Mobile only: fixed off-canvas drawer */
@media (max-width: 768px) {
  .sidebar {
    position: fixed;
    top: 0; left: 0; bottom: 0;
    width: 260px;
    transform: translateX(-110%);
    z-index: 50;
    transition: transform 0.25s cubic-bezier(0.4, 0, 0.2, 1);
  }
  .sidebar--open { transform: translateX(0); }

  .mobile-only-header { display: flex; }
  .mobile-menu-btn    { display: flex; }
}

/* Mobile-only elements: hidden on desktop by default */
.mobile-only-header { display: none; }
.mobile-menu-btn    { display: none; }
```

---

## Multiple CSS Rules with Same Selector

**Symptom:** A style change has no effect even though the code is correct.

**Root Cause:** The same CSS class is defined multiple times. The cascade picks the last definition.

**Diagnosis:**
```bash
grep -n "\.class-name {" stylesheet.css
```

**Example from trading.css:**
```css
/* Line 344 — first definition */
.trade-content { padding-bottom: 160px; }

/* Line 401 — second definition OVERRIDES the first */
.trade-content { width: 100%; }
```

Fix: merge into a single rule. Delete the duplicate.

---

## `overflow: hidden` Does Not Equal Scroll Containment

**Symptom:** Inner content overflows despite `overflow: hidden` on the parent.

**Root Cause:** `overflow: hidden` on a flex child only clips content overflowing that child's own box — not the content of its children.

**Correct pattern for scrollable tab content:**
```css
.tab-content {
  flex: 1;
  min-height: 0;
  overflow: hidden;     /* prevents child from pushing out */
  display: flex;
  flex-direction: column;
}

.tab-pane {
  height: 100%;
  overflow-y: auto;    /* the actual scroll container */
  min-height: 0;
}
```

Scroll lives at the innermost level, not the containment boundary.
