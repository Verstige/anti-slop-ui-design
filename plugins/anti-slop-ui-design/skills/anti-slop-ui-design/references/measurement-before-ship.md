# Measurement-Before-Ship — Visual Redesigns That Pass the Build But Fail the Eye

## The pattern

You ship a layout redesign (cards, ladders, grids, signal feeds). The
build passes, `tsc --noEmit` is clean, dev server compiles, commit is
on `main`. Within minutes the user comes back with: **"very compacted,"
"it's getting cut off," "needs to look cleaner and more gorgeous."**

You open the production screenshot and immediately see it: text
running into text, marks clipping at edges, dots overlapping labels,
numbers truncated. The bug was always visible — you just shipped
without looking.

## Why this keeps happening

The visual redesign workflow has three steps and one of them is
always skipped:

1. **Design** — pick a structure (grid, ladder, bands, tabs).
2. **Measure** — check that the structure fits the actual content
   widths + container width + edge constraints.
3. **Ship** — commit + push.

LLM agents default to step 1 → step 3. Step 2 is the one that takes
math, which agents skip because the math isn't required by the
type checker or the build. It only matters when a real human looks
at the rendered output.

## Concrete example 1 — Live signal card with per-rung prices

**Design:** 6 marks (SL, Entry, TP1, TP2, TP3, TP4) on a horizontal
rail at percentages `0% / 28% / 44% / 60% / 76% / 100%` with
`transform: translateX(-50%)` so each mark centers on its
position. Each mark renders the absolute price (`"84,499.58"` for
BTCUSD, `"4,266.75"` for XAUUSD).

**Measurement:** At a 870px-wide card with 16px container padding
the rail is 838px. Center-to-center spacing: 0% → 100% = 838px across
5 gaps = 168px each. Each price is ~70px wide in the chosen font.
With `min-width: 64px` per mark, the labels visually crowd each
other — adjacent 70px labels in 168px gaps look "fine" until you
notice the labels touch in the BTCUSD case (7-character prices).

**What I shipped:** All 5 prices on all 6 rungs. TP1 price
("84,499.58") ran into Entry price ("84,402.90") because the label
center at 28% (244px) and 44% (383px) put text edges only 39px
apart — barely the width of one character of padding.

**The user response:** "It's very compacted and needs to look
cleaner and more gorgeous." → the entire rung redesign followed.

**The fix that would have shipped on the first try:** drop per-rung
absolute price entirely. The header already shows the entry price.
The ladder's job is to show **progress through TPs**, not restate
the price ladder. One symbol-mark + label + pip-distance per rung
fits comfortably in any 168px gap.

## Concrete example 2 — Price ladder marks at `left: 0%` and `left: 100%`

**Design:** SL mark at `left: 0%`, TP4 mark at `left: 100%`, both
with `translateX(-50%)` (centers on the rail).

**Measurement:** `translateX(-50%)` on a mark of width N puts
the mark's CENTER at the rail position. The mark's outer edge is
N/2 on either side. At `left: 0%`, half of the mark extends to
`left: -N/2` — outside the card. Same for `left: 100%`.

**What I shipped:** SL clipped on the left card edge, TP4 clipped
on the right.

**The fix:** Pull edge marks inward (5% from each rail end). The
mark still feels "at the edge" but no text bleeds past the card
border. The rail itself also needs to span 5%–95% (not 0%–100%)
so it visually starts at the SL mark center and ends at the TP4
mark center — otherwise the rail extends past the marks and looks
disconnected.

## The pre-flight check (do this before every visual redesign)

```bash
# 1. List the container widths you'll see the design in
#    - Mobile (375px viewport, ~327px content width after 24px padding)
#    - Tablet (768px viewport, ~720px content width)
#    - Desktop (1280px+ viewport, 100%-240px sidebar)

# 2. For each container width, compute the gaps your marks/labels need to fit in:
#    - gap = (container_width - sum_of_label_widths) / num_gaps
#    - if gap < label_width / 2 → design is too tight
#    - if gap < 0 → labels overlap regardless of typography

# 3. For edge-anchored elements (translateX(-50%) at left:0%/100%):
#    - half-width of the element must fit in the padding, not the content
#    - or use a % inset (5%–95%) instead of 0%/100%
```

## The shipping decision tree

When you're about to commit a visual change, ask:

1. **Did I measure at the smallest container width?** (Phone
   viewports expose cramping that desktop hides.) If you don't
   know what the smallest container is, find it before you ship.
2. **Did I check the rendered output, not just the source?** Open
   the screenshot, the dev-server preview, or `curl -sL
   http://localhost:3000/<route> | head`. The bug is in the
   rendered pixels, not in the JSX.
3. **Did I count the labels and measure their widths at the
   container's font + size?** 5 labels × 70px = 350px in a 720px
   container looks fine on paper; with vertical stacks and mark
   offsets, the actual horizontal collision distance can be much
   smaller than the gap suggests.

If any answer is "no" or "I haven't checked" → open the page in
the browser before committing. Don't trust build-pass alone for
visual correctness.

## When "I measured but the math was off"

If you measured and shipped and it's still wrong, the most common
root cause is **assuming the same content widths at all screen
sizes**. BTCUSD prices (7 chars) don't fit where FX majors do
(6 chars). Add a content-width assumption table to your design doc:

| Pair    | Char count | Width at 0.7rem JetBrains Mono |
|---------|-----------|---------------------------------|
| XAUUSD  | 6-7       | 50-65px                          |
| BTCUSD  | 6-7       | 50-65px                          |
| EURUSD  | 6 (1.14101)| 50-60px                          |
| US30    | 5-6       | 45-55px                          |
| GBJPY   | 7         | 60-70px                          |

If the design doesn't fit the widest variant at the smallest
container, redesign. If it fits all variants at all containers,
ship.

## What the user actually wants in a "make it look gorgeous" pass

When the user comes back with compaction feedback, the right
response is rarely "add more decoration" or "make it more
complex." It's almost always:

1. **Remove what's redundant.** If the entry price is already in
   the header, don't repeat it on every ladder rung.
2. **Widen the spacing between marks.** 5%–95% instead of 0%–100%.
3. **Reduce per-mark content.** Less text per rung → more room for
   breathing.
4. **Test at the smallest container.** Phone viewports expose
   cramping first.

The "gorgeous" request is usually a symptom of a cramping problem,
not a separate aesthetic preference.

## Why this lesson complements `frontend-redesign.md`

`frontend-redesign.md` covers the **subagent delegation** failure
mode: build passes, visual fails because component defaults weren't
patched. This reference covers the **direct implementation** failure
mode: build passes, visual fails because measurement was skipped.

Both ship visually broken cards. The causes are different (default
component state vs. fit math) and the fixes are different (audit
shared components vs. measure at smallest viewport). The defense
in both cases is the same: **visually verify the rendered output
before committing, never trust exit code alone.**
