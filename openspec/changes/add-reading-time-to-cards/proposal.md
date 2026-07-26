## Why

Homepage article cards show title, date, and categories but no reading time. Readers can't gauge article length before clicking. Adding reading time helps them pick articles that fit their available time.

## What Changes

- Show estimated reading time (e.g., "3 min read") inside each blog card on the homepage
- Apply to all card types: pinned series index cards, series episode cards, and standalone cards
- Reuse the same 250 words-per-minute calculation already used on individual post pages

## Capabilities

### New Capabilities
- `homepage-reading-time`: Display estimated reading time on each homepage article card

### Modified Capabilities
<!-- None -->

## Impact

- `_layouts/home.html` — homepage template where article cards are rendered
- No new dependencies — uses built-in Jekyll Liquid filters
