# Flows & screens

<!-- Plan each user flow, the screens it contains, and how they wire together.
     A screen = a top-level frame where the user lands somewhere new. Tabs, filters, empty/error/
     loading, hover and selection are VARIANTS of a component on that screen, listed in its row,
     not extra screens. Overlays are listed separately. Keep this file structured so
     "what's left?" and "how many screens?" are one read. -->

## Summary

| Flow | Unique screens | Overlays | Status |
|------|----------------|----------|--------|
| (example — delete) | 4 | 2 | approved |

## Flow: (name)

<!-- Fill in for each flow: status, where it lives, where it starts, and research notes if any -->

| Field | Value |
|-------|-------|
| Status | (planned / in progress / awaiting user review / approved) |
| Figma page & section | (e.g., page "Flows", section "Auth") |
| Flow starting point name | (e.g., "Login") |
| Research note path | (e.g., "research/login.md") |
| Open items in NEEDED-INFO.md | (IDs, or "none") |

### Steps and branching paths

<!-- Steps in order. Branches listed separately; mark which non-obvious ones the user asked to show. Skip self-evident states. -->

1. (step)
- Branch: (e.g., "payment declined") — show? (yes / no / asked)

### Screens

| # | Screen | Node ID | Template used | Organisms used | Variants on this screen | Status |
|---|--------|---------|----------------|----------------|-------------------------|--------|
| 1 | (example — delete) | (ID) | (template component name) | (child components) | (e.g., "tabs: all/open/closed; empty; error") | (todo / in progress / review / done) |

### Overlays

| Overlay | Node ID | Opened from | Wired on (main component / screen instance) | Position to set by hand |
|---------|---------|-------------|---------------------------------------------|-------------------------|
| (example — delete) | (ID) | (trigger) | (where) | (e.g., "anchor right") |

### Prototype wiring

| From (screen › element) | Trigger | Action | To | Done? |
|--------------------------|---------|--------|-----|-------|
| (example — delete) › Button | on click | navigate | (target screen name) | Y/N |
