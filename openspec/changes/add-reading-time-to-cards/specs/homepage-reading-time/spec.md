## ADDED Requirements

### Requirement: Homepage cards display estimated reading time
Each article card on the homepage SHALL display an estimated reading time in the format "N min read", calculated at 250 words per minute with a minimum of 1 minute.

#### Scenario: Reading time appears on all card types
- **WHEN** a visitor views the homepage
- **THEN** every blog card (pinned, series episode, and standalone) shows a reading time estimate in its metadata area

#### Scenario: Short article shows 1 minute minimum
- **WHEN** an article has fewer than 250 words
- **THEN** the card displays "1 min read"

#### Scenario: Reading time matches the post page
- **WHEN** a visitor sees "3 min read" on a homepage card and clicks through
- **THEN** the individual post page also shows "3 min read"
