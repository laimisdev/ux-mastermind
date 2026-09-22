# UX Research Procedure

Reference for a subagent researching UX best practice for ONE feature (e.g. "multi-step
onboarding", "data table filtering", "checkout", "password reset") before Figma work starts.
Research informs structure, flow, and states — not visual style. Output is a single cached
note other sessions can reuse.

## Table of contents

1. [Purpose & budget](#1-purpose--budget)
2. [Mobbin procedure](#2-mobbin-procedure)
3. [Web procedure](#3-web-procedure)
4. [Always cover](#4-always-cover)
5. [Output](#5-output)
6. [Honesty rules](#6-honesty-rules)

## 1. Purpose & budget

- The goal is a **structural** brief: what screens/steps exist, what states each needs, what
  components are implied, what a11y rules apply. Not colors, fonts, or brand — that belongs to
  the design system, not this research.
- **Check the cache first.** Look for `.ux-prototype/research/<feature-slug>.md`. If a note for
  this feature already exists, read it and stop — do not repeat research. Research is cached so
  other sessions never redo the same query.
- **Timebox**: ≤ ~8 Mobbin queries and ≤ ~5 web pages per feature. If you hit the budget before
  you have enough to write the note, write it anyway with what you have and mark gaps as open
  questions — do not silently keep searching past budget.
- Prefer breadth-then-depth: a handful of strong examples beats exhaustively cataloguing every
  result.

## 2. Mobbin procedure

Mobbin is the primary source for real-world screen/flow references — it's how you ground the
prototype in what shipped products actually do, not just written guidance.

1. **Confirm availability first.** Mobbin's tools are added to a session via
   `claude mcp add mobbin --scope user --transport http https://api.mobbin.com/mcp` and require
   OAuth authentication (browser sign-in) plus a paid Mobbin plan (Pro/Team/Enterprise). Look for
   tools prefixed `mcp__mobbin__` — use `ToolSearch` with query `"mobbin"` to discover what's
   actually registered in this session. Do not assume tool names in advance; confirm them at
   runtime.
   - **If no `mcp__mobbin__` tools are found**: do not silently skip this step. Report back to
     the orchestrator that the user must run the `claude mcp add mobbin ...` command above and
     authenticate via `/mcp`, then continue with web research only, and say so explicitly in the
     output note (see §6).
2. **What to search for.** As of this writing Mobbin's MCP exposes natural-language search over
   three surfaces (confirm exact names/params via `ToolSearch` — they may have changed):
   - screens — individual UI screens
   - flows — multi-step user flows (e.g. onboarding, checkout) — usually the most useful surface
     for this skill, since it shows step sequencing
   - sections — page sections (e.g. pricing pages, footers) — mainly useful for web/marketing
     features
   Treat these as intent categories, not a guaranteed API: search "flow" style when you want step
   sequences, "screen" style when you want a single state (e.g. an empty state or error toast),
   "section" style for page-level web patterns.
3. **Scope the query.** Search for the feature on the platform relevant to the project (default:
   **web**, since this team builds desktop-first products, but check project context — mobile
   apps need mobile examples). Bias queries toward products in a similar domain to the project
   (e.g. B2B SaaS, marketplace, fintech) when you can — the orchestrator's project context should
   tell you the domain.
4. **Collect 4–6 strong examples** from different products, not 20 mediocre ones. For each:
   - the step sequence (for flows) or the single screen's purpose
   - what's on each screen (key elements, primary/secondary actions, hierarchy)
   - which states are shown (empty / loading / error / success) — note gaps, i.e. states Mobbin's
     screenshots don't cover, since you'll need to infer those from written guidance instead
   - keep the image URL / link and app name for each — image URLs expire after ~30 days, so the
     cached note should record the Mobbin app/flow name and any permalink, not rely on the image
     surviving
5. **Look for convergence vs. outliers.** Note what most examples agree on (this is the safe
   default to recommend) and call out anything only one product does (mention it, but don't build
   the recommended flow around an outlier).

## 3. Web procedure

Use web search/fetch to fill in guidance Mobbin can't show — usability rationale, accessibility
requirements, and edge-case handling that a screenshot doesn't convey.

- Prefer, in this order:
  1. Nielsen Norman Group (nngroup.com) — general usability
  2. Baymard Institute (baymard.com) — e-commerce/checkout specifically
  3. GOV.UK Design System (design-system.service.gov.uk) — forms, accessibility, plain patterns
  4. Material Design (m3.material.io) / Apple HIG (developer.apple.com/design) — platform
     conventions when relevant
  5. W3C WAI/WCAG (w3.org/WAI) — accessibility requirements, this is the source of truth, not a
     blog's summary of it
  6. Smashing Magazine, UX Collective — secondary only, use to fill gaps the above don't cover,
     and verify anything load-bearing against a primary source above if possible
- Extract **concrete, checkable guidelines** — things you could verify a design against — not
  platitudes. "Error messages appear inline next to the field, not only in a summary banner" is
  checkable. "Make errors clear and helpful" is not — skip guidance that can't be turned into a
  screen requirement.
- Every guideline you keep must have a source link you actually opened (see §6).

## 4. Always cover

Regardless of feature, the resulting flow and state list must account for:

- **Happy path** — the numbered steps a successful user takes, screen by screen.
- **All states per screen**, as applicable: empty, loading, partial (partial data / partial
  completion), error, success, disabled, permission-denied, offline.
- **Validation & error recovery** — when validation fires (inline/on-blur/on-submit), how errors
  are announced, how the user gets back to a valid state.
- **Edge cases**: long text (truncation/wrap), zero items, exactly one item, many items
  (pagination/virtualization), first-time user vs. returning user (different defaults/empty
  states).
- **Accessibility**: focus order, form labels, full keyboard operability, minimum target size
  (44x44pt / equivalent), how errors are announced to assistive tech (e.g. `aria-live`,
  associated error text, not color alone).
- **Desktop considerations at 1512px** — this team's default canvas width. Note where layout
  should reflow vs. stay fixed-width, and where dense/desktop-only affordances (hover states,
  multi-column layouts, keyboard shortcuts) apply that wouldn't exist on mobile.

## 5. Output

Write the cached note to `.ux-prototype/research/<feature-slug>.md`. Keep it to **~120 lines or
less** — this is a working brief, not a report. Use this exact section structure:

```markdown
# <Feature name>

## Feature
One sentence: what this feature is and its scope boundary (what's included/excluded).

## Project context used
What you knew about the project (domain, platform, existing patterns) that shaped the choices
below. If none was available, say so.

## Recommended flow
1. Step/screen name — one-line purpose
2. Step/screen name — one-line purpose
...

## Required states per screen
- Screen name: empty, loading, error, success, ... (only list states that actually apply)

## Components implied
- Shadcn component name (or closest equivalent) — where/why it's used
(Only name shadcn components where the mapping is obvious; otherwise describe the UI element.)

## Guidelines
- Guideline text. [Source](https://...)
- Guideline text. [Source](https://...)

## Mobbin references
- App name — flow/screen name — link or ID — what to borrow from it
(If Mobbin was unavailable, write: "Mobbin unavailable this session — see §6 note below.")

## Anti-patterns to avoid
- Pattern to avoid and why.

## Open questions for the user
- Anything the research couldn't resolve and needs a product decision.
```

After writing the file, return this to the orchestrator:

- The path written (`.ux-prototype/research/<feature-slug>.md`)
- A 5-bullet digest of the most important findings
- The open questions list (even if empty, say so explicitly)

## 6. Honesty rules

- **Cite only sources you actually opened.** Never invent a URL, a Mobbin app name, or a
  guideline attributed to a source you didn't read.
- **Mark inference as inference.** If you filled a gap using judgment rather than a source
  (Mobbin or web), label it inline, e.g. "(inference — no source covered this)".
- **If Mobbin was unavailable** (tools not registered, not authenticated, or budget exhausted
  with no results), say so plainly in the note's Mobbin references section and in the digest back
  to the orchestrator — do not present web-only research as if it were verified against real
  products.
- **If web research was unavailable or inconclusive** for a claim, say so rather than presenting
  it as settled guidance.
