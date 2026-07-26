## Context

The homepage shows blog posts as cards grouped by series. Each card displays the title, date, and topic categories. Individual post pages already show reading time, but the homepage does not.

## Goals / Non-Goals

**Goals:**
- Show reading time on every homepage blog card
- Keep it visually consistent with how it already appears on post pages

**Non-Goals:**
- Changing the reading speed formula
- Adding reading time to the search overlay
- Creating a shared component (too small to justify)

## Decisions

- **Reuse the same calculation already on post pages** — keeps the numbers consistent everywhere. No new logic needed.
- **Place reading time inside each card's metadata area, after topic tags** — this matches reader expectations and mirrors the post page layout.
- **Duplicate the calculation in each card type rather than extracting a partial** — the logic is only 3 lines and unlikely to change. A shared file adds overhead for no real benefit at this scale.

## Risks / Trade-offs

- Minor repetition across three card types — acceptable for simplicity
- No performance concern — the site has only a handful of articles
