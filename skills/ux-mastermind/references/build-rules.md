# Build Rules — ux-mastermind

House rules for building wireframe-level, fully prototyped UX inside a user's existing
Figma file. This file is WHAT to build and to what standard. For HOW to call the Plugin
API, load `figma-use` (mandatory before every `use_figma` call). For HOW to construct a new
component, load `figma-generate-library` alongside it. Neither is duplicated here.

## Table of contents

1. [Reuse before create](#1-reuse-before-create)
2. [Atomic design](#2-atomic-design)
3. [New component standard](#3-new-component-standard)
4. [Variables, tokens, styles](#4-variables-tokens-styles)
5. [Auto layout & responsiveness](#5-auto-layout--responsiveness)
6. [Wireframe fidelity — no design decisions](#6-wireframe-fidelity--no-design-decisions)
7. [File hygiene](#7-file-hygiene)
8. [Definition of done per screen](#8-definition-of-done-per-screen)

---

## 1. Reuse before create

The whole point of prototyping inside the user's file is that a shadcn/ui system is
already there — treat every screen as an assembly problem, not a drawing problem. Before
touching Figma for a given piece of UI:

1. Check `.ux-prototype/DESIGN-SYSTEM.md` and `.ux-prototype/NEW-COMPONENTS.md` first —
   these are the running inventories this skill maintains (see §3, §7). They're cheaper to
   read than the file and stay in sync between sessions.
2. Search the file's **local** components first, then any enabled libraries, by name AND by
   synonym — shadcn naming and everyday UX vocabulary diverge often enough that a plain-text
   search will miss the match otherwise. The team's files keep the design system as *local*
   components, variables and styles — usually **not published as a library** — so
   `get_libraries` / `search_design_system` (which look at published libraries) will often
   return nothing or claim there is no design system. That is not evidence that nothing
   exists. Enumerate local components with the Plugin API instead (read-only script):

   ```js
   // Local component inventory — run once per page, or over figma.root for small files
   await figma.loadAllPagesAsync();
   const sets = figma.root.findAllWithCriteria({ types: ['COMPONENT_SET', 'COMPONENT'] })
     .filter(n => n.type === 'COMPONENT_SET' || !n.parent || n.parent.type !== 'COMPONENT_SET');
   return sets.map(n => ({
     id: n.id, key: n.key, name: n.name, page: n.parent && (n.parent.type === 'PAGE' ? n.parent.name : (n.parent.parent && n.parent.parent.name)),
     props: n.type === 'COMPONENT_SET' ? Object.keys(n.componentPropertyDefinitions) : undefined
   }));
   ```

   Only conclude "nothing fits" after both this local scan and a library search come back
   empty. Same for tokens: `figma.variables.getLocalVariableCollectionsAsync()`,
   `getLocalVariablesAsync()`, `figma.getLocalTextStylesAsync()`, `getLocalEffectStylesAsync()`
   see local definitions that `get_variable_defs` on a published library would miss.

| If you're thinking… | Also search for… |
|---|---|
| Modal | Dialog, Alert Dialog |
| Drawer | Sheet |
| Dropdown | Select, Dropdown Menu, Combobox |
| Toast, snackbar | Sonner |
| Tooltip, popover, hover card | Popover, Tooltip, Hover Card |
| Autocomplete | Combobox |
| Multi-step form | Tabs, Stepper, Form |
| Nav bar, top bar | Navigation Menu, Header, App Bar |
| Side nav | Sidebar |
| Spinner, loader | Skeleton, Progress |
| Chip, tag, pill | Badge |
| Search box / command palette | Command, Input |
| Table with filters/search/bulk actions | Data Table (Table + Toolbar + Pagination) |

3. Once you've found the right component, insert an **instance** — for a local component,
   `(await figma.getNodeByIdAsync(id))` then `createInstance()` on the `ComponentNode` (for a
   set, pick the variant via `set.defaultVariant` or `set.findChild`); use
   `importComponentByKeyAsync` / `importComponentSetByKeyAsync` **only** for components that
   come from a published library — it fails for unpublished local components — never detach it, never copy its
   layers into a new frame, and never draw a rectangle/text stack that recreates what a
   component already renders. A detached or hand-drawn copy stops tracking the design
   system, so any later restyle misses every screen that used it.
4. Configure the instance through its own API surface: variant properties (`setProperties`),
   boolean/text/instance-swap component properties, and text overrides on its children. If
   the instance's property model can't express what the screen needs, that's a signal to
   re-read §3 (new component standard) rather than fight the instance.

### Cheat-sheet: shadcn component → typical UX use

| Component | Typical use |
|---|---|
| Button | Primary/secondary actions, form submit, icon-only actions |
| Input | Single-line text entry, search fields |
| Label | Field labels, paired with Input/Select/Checkbox |
| Form field | Label + control + helper text + error message, as one molecule |
| Select | Choosing one value from a short, known list |
| Combobox | Choosing one value from a long or searchable list |
| Checkbox | Independent on/off, multi-select in a list |
| Radio group | Mutually exclusive choice among a few visible options |
| Switch | Immediate on/off setting (no explicit save step) |
| Textarea | Multi-line free text |
| Card | Grouping related content/actions as a discrete unit |
| Dialog | Blocking, in-flow confirmation or short focused task |
| Alert dialog | Destructive/irreversible action confirmation |
| Sheet | Side-anchored panel for secondary content or a task, non-blocking to the rest of the page |
| Drawer | Bottom-anchored panel, mobile-style task or menu |
| Popover | Small contextual panel anchored to a trigger, dismiss on outside click |
| Dropdown menu | List of actions/commands anchored to a trigger |
| Tooltip | Short label on hover/focus, no interaction inside |
| Tabs | Switching between sibling views in the same context |
| Accordion | Progressive disclosure of stacked sections |
| Table | Static or lightly-interactive tabular data |
| Data table | Tabular data with sorting, filtering, selection, pagination |
| Pagination | Paging through a long list or table |
| Breadcrumb | Showing/ navigating hierarchical location |
| Navigation menu | Top-level site/app navigation with possible submenus |
| Sidebar | Persistent app-level navigation, often collapsible |
| Command | Keyboard-driven search/command palette |
| Badge | Status, count, or category label |
| Avatar | User/account representation |
| Alert | Inline, non-blocking status message on a page |
| Sonner / Toast | Transient, non-blocking notification after an action |
| Skeleton | Loading placeholder for content still fetching |
| Progress | Determinate/indeterminate progress of a task |
| Separator | Visual division between sections |
| Calendar / Date picker | Selecting a date or date range |
| Empty state | Zero-data condition for a list, table, or search result |

---

## 2. Atomic design

This is how the team names and nests things. Match these definitions even if the file uses
different words for them — but if the file already has its own atomic-design convention
(page names, section names, layer-naming pattern), follow that instead of the below.

- **Atoms** — the design system's own primitives, used as-is: Button, Input, Badge, Avatar,
  Checkbox. You do not build atoms in this skill; you insert instances of the ones the
  design system already ships (§1).
- **Molecules** — a small, reusable cluster of atoms with one job. Example: a *form field*
  = Label + Input + helper text + error message. A molecule is still generic — it doesn't
  know what page it's on.
- **Organisms** — a self-contained section of UI made of molecules and atoms, doing one
  meaningful piece of product work. Examples: a login form (form fields + button + links),
  a data table with its toolbar (search + filters + Data Table + Pagination), an app header
  (logo + Navigation Menu + Avatar menu).
- **Templates** — a page-level layout skeleton with named slots, no real content yet.
  Example: an *app shell* = Sidebar + header + a content slot. Templates express structure,
  not a specific flow.
- **Pages** — one instance of a template, filled with real organisms and realistic content
  for one specific screen in the flow (e.g., "Settings → Billing" built from the app shell
  template, with a billing-history organism dropped into the content slot).

**Nesting rule:** each level is composed of *instances* of the level below it. A page
should contain no raw layers of its own except layout frames (auto-layout wrappers used
purely for arrangement, or one-off inline compositions of instances — see below) — every piece of actual UI on a page is an instance of a molecule,
organism, or atom, not a hand-built substitute.

**Build bottom-up.** Before starting level *N*, confirm the level *N-1* pieces it depends on
already exist — as design-system atoms, or as molecules/organisms you've already built and
recorded in `NEW-COMPONENTS.md`. Building a login organism before its form-field molecule
exists produces a one-off you'll have to tear apart later.

**Naming convention for new components:** prefix with the atomic level and group with a
slash, e.g. `Molecule/Form Field`, `Organism/Login Form`, `Template/App Shell`. This is a
proposal — if the file already has its own naming convention (even a different scheme),
follow the file's convention instead of imposing this one. Discover it first, per `figma-use`
§9.

**Where new components live: on the prototype page itself.** The whole prototype — new
components and all screens — goes on one page (proposed name `🧠 UX Prototype`, or the
page the user points at). Reviewers want to see the pieces and the screens together, and
teammates shouldn't have to hunt across pages to find what a screen is made of. Lay the
page out as horizontal bands of Sections, top to bottom: `Components` (one sub-section per
atomic level: Molecules / Organisms / Templates), then one Section per flow with its screens
left-to-right. Don't create extra pages for components; if the file already has its own
convention for generated components, follow that instead and note it in
`DESIGN-SYSTEM.md`. Leave the design-system pages untouched either way.

**Create only what earns its keep.** Every new component is something a teammate has to
learn, maintain and check — a prototype drowning in `Organism/…` sets is as hard to read as
one made of raw layers. Before creating one, ask:

- Does an existing component already do this with a property or variant? Use it.
- Will it appear more than once (across screens or states), or must it carry state
  interactions of its own? If not, compose it *inline* from atom/molecule instances inside
  a plain, well-named auto-layout frame on the screen. One-off page content (a specific
  hero, a specific settings section, a one-time confirmation message) is inline, not a
  component.
- Is it really a new thing, or a variant of one you already made? Extend the existing set.
- Is it an overlay (dialog, sheet, menu, popover, tooltip, toast)? Then it is a single
  top-level frame in the `Overlays` sub-section holding an instance of the design-system
  component, opened via an `OVERLAY` reaction — not a slot on every screen, and never a
  reason to duplicate a screen. See `references/prototyping.md` "Open overlay".

As a rule of thumb, a flow of 5–8 screens usually needs 0–2 molecules and 1–3 organisms
plus at most one template; if the plan lists more, prune it. Templates are only worth
making when 3+ screens share the same skeleton. The nesting rule above still holds for
whatever *is* a component: it's built from instances of the level below.

---

## 3. New component standard

Reach for a new component only after §1's search comes back empty and no instance
configuration can express what the screen needs. When you do build one, load
`figma-generate-library` for the construction mechanics (variant creation, property
linking, variable binding) — this section is the acceptance bar those mechanics must hit.

A new component is done only when all of the following hold:

- **It's a component set with variants for every state the UX actually needs** — not just
  the states that look different at rest. At minimum, consider: default, hover, focus,
  pressed, disabled, loading, error, empty, filled, and selected/open, including only the
  ones that apply to this component. A form-field molecule needs error and filled at
  minimum; a menu-trigger organism needs open/closed.
- **Component properties are used, not variant explosion** — TEXT properties for copy,
  BOOLEAN properties for optional parts (e.g. a dismiss icon, a helper-text row),
  INSTANCE_SWAP for slots that accept an arbitrary child (e.g. a leading icon, an avatar
  slot). Reserve variants for states that change structure or visual treatment; use
  properties for everything else. See `figma-generate-library` §3 and
  `component-patterns.md` for the API.
- **It's built from existing atoms**, not raw shapes — a new molecule/organism composes
  instances of design-system atoms the same way a hand-coded component would compose
  shadcn primitives.
- **Every value is bound to a variable** — no hardcoded fill, stroke, padding, gap, radius,
  or font value anywhere in the new component (§4 has the exact rule and a verification
  snippet).
- **It uses auto layout** throughout, with deliberate fill/hug choices (§5).
- **The component (or component set) has a filled-in `description`** — what it is and where
  to use it. Set `description` only on `COMPONENT`/`COMPONENT_SET`, per `figma-use` Rule 3a.
- **State interactions are wired on the main component** so the prototype is actually
  clickable/testable, not just a static mockup — see `references/prototyping.md`.
- **It's recorded in `.ux-prototype/NEW-COMPONENTS.md`** the moment it's created — name,
  atomic level, node ID, states covered, and which screens use it. This keeps §1's reuse
  check fast for the next screen and the next agent.

---

## 4. Variables, tokens, styles

No hardcoded hex, pixel spacing, radius, or font value in anything this skill creates.
Every fill and stroke binds to a color variable, every padding/gap/radius binds to a number
variable, every run of text uses a text style, every shadow uses an effect style. This
matters because the whole reason to prototype inside the user's file is that a restyle of
the design system should ripple through every screen automatically — a hardcoded value
silently opts a node out of that.

**Discover before inventing.** Enumerate the local variable collections and styles
(`figma.variables.getLocalVariableCollectionsAsync()` / `getLocalVariablesAsync()`,
`figma.getLocalTextStylesAsync()`, `getLocalEffectStylesAsync()`, plus `get_variable_defs` on
a node that uses them) before creating anything — the team's tokens are local to the file,
not a published library, so a tool that reports "no variables/library" is looking in the
wrong place. shadcn files typically expose semantic names —
`background`, `foreground`, `primary`, `secondary`, `muted`, `accent`, `border`, `ring`,
`destructive`, `radius`, plus a spacing scale. Use exactly what's there; match casing and
grouping conventions.

**If a value you need doesn't exist yet:**

1. Add it to the collection it structurally belongs to, following that collection's
   existing naming scheme — don't start a new collection for one value.
2. Set a value for every mode the collection defines (typically Light/Dark) — a variable
   with only one mode populated breaks the moment the file switches modes.
3. Alias semantic → primitive where the file already does that (e.g. `color/bg/primary`
   aliasing `blue/50`) rather than duplicating the raw value at the semantic layer.
4. Set `scopes` explicitly — never leave `ALL_SCOPES`. See `variable-patterns.md` for the
   scope-per-property-type table.
5. Log it in `.ux-prototype/DESIGN-SYSTEM.md` under a "Tokens added" section — name,
   collection, value(s) per mode, why it was needed.

**Verification snippet** — walk a subtree and report anything still unbound:

```js
const root = await figma.getNodeByIdAsync(ROOT_ID); // scope to the screen/frame you just built
const unbound = [];

root.findAllWithCriteria({ types: ['FRAME', 'RECTANGLE', 'ELLIPSE', 'TEXT', 'COMPONENT', 'INSTANCE'] })
  .forEach(n => {
    const bindings = n.boundVariables || {};
    if ('fills' in n && Array.isArray(n.fills) && n.fills.length && !bindings.fills) {
      unbound.push({ id: n.id, name: n.name, issue: 'unbound fill' });
    }
    if ('strokes' in n && Array.isArray(n.strokes) && n.strokes.length && !bindings.strokes) {
      unbound.push({ id: n.id, name: n.name, issue: 'unbound stroke' });
    }
    if ('paddingLeft' in n && n.layoutMode !== 'NONE') {
      ['paddingLeft','paddingRight','paddingTop','paddingBottom','itemSpacing'].forEach(k => {
        if (!bindings[k]) unbound.push({ id: n.id, name: n.name, issue: `unbound ${k}` });
      });
    }
    if ('cornerRadius' in n && typeof n.cornerRadius === 'number' && !bindings.topLeftRadius && !bindings.cornerRadius) {
      unbound.push({ id: n.id, name: n.name, issue: 'unbound cornerRadius' });
    }
    if (n.type === 'TEXT' && !n.textStyleId) {
      unbound.push({ id: n.id, name: n.name, issue: 'text without a text style' });
    }
  });

return { checked: root.name, count: unbound.length, unbound };
```

Run this after building each screen, before moving on — fix hits before they compound
across the next screen you copy patterns from.

---

## 5. Auto layout & responsiveness

Every frame that has children uses auto layout (`figma.createAutoLayout()`, per `figma-use`
Rule 12a) — absolute x/y positioning is reserved for true overlays (badges pinned to a
corner, floating action buttons) and nothing else. This is what keeps the prototype from
silently breaking when copy length or content count changes, which happens constantly once
realistic content (§6) is in play.

- Choose `FILL` vs `HUG` deliberately, not by default: content areas (a card's body, a
  table's row) fill their container so they grow with content; controls (a button, a badge,
  a checkbox) hug their content so they don't stretch into oversized hit targets.
- Set min/max width on content columns (form columns, reading-width text blocks) so they
  don't go edge-to-edge on a wide frame or crush on a narrow one.
- Use `layoutWrap: 'WRAP'` for card grids and tag/chip rows so they reflow instead of
  overflowing.
- Set constraints on the screen frame itself (its own children's constraints, not the
  screen's) so panels anchor correctly when the frame resizes.
- **Screen frames are 1512px wide** desktop frames (team default) with height set to hug
  content — never a fixed height that clips a long screen.
- **Resize-test every screen**: resize the screen frame to ~1280 and ~1728, screenshot at
  each width, and confirm nothing clips, overlaps, or reflows badly. Restore the frame to
  1512 width when done — the resize is a test, not the final state.

**Verification snippet** — frames with children but no auto layout:

```js
const root = await figma.getNodeByIdAsync(ROOT_ID);
const offenders = root.findAllWithCriteria({ types: ['FRAME', 'COMPONENT', 'INSTANCE'] })
  .filter(n => n.children && n.children.length > 0 && n.layoutMode === 'NONE')
  .map(n => ({ id: n.id, name: n.name, childCount: n.children.length }));

return { checked: root.name, count: offenders.length, offenders };
```

A hit here is not automatically wrong (a true absolute-overlay container is a legitimate
exception) — but every hit should be a deliberate exception, not an oversight.

---

## 6. Wireframe fidelity — no design decisions

This skill produces UX structure for review, not visual design — that's a separate,
later decision the team makes on purpose, not one this skill should make by accident while
assembling screens. Concretely:

- Use shadcn defaults exactly as the file already has them configured — default variant,
  default color mapping, default spacing scale. Don't introduce a new brand color, a custom
  illustration, stock imagery, a decorative gradient/shadow, or a typographic flourish the
  design system doesn't already define. If a screen "needs" one of these to feel finished,
  that's a sign the review should happen before visual polish, not that this skill should
  supply it.
- Images are neutral placeholders (design-system placeholder fill, or a generic image
  component if the library has one) — never a specific photo, illustration, or logo chosen
  for its look.
- **Do** use realistic content everywhere: real field labels, plausible sample data (names,
  dates, amounts that look like they came from the product, not "Lorem ipsum" or "Text
  here"), real error messages ("Card number is invalid" not "Error"), real empty-state copy
  ("No invoices yet — they'll appear here once you're billed" not "Empty"). Copy is UX —
  vague placeholder text hides exactly the problems a wireframe review exists to catch
  (message length, tone, what an empty/error state actually needs to say).

---

## 7. File hygiene

- Work only in the user's file, and only on the pages the orchestrator assigned — this
  skill is one contributor among possibly several agents/sessions touching the same file.
- Never delete or restyle an existing design-system component — if something about it seems
  wrong for the UX, flag it in `DESIGN-SYSTEM.md` rather than changing it in place; a shared
  component change affects every screen that uses it, including ones outside this task.
- Keep instances named exactly as their component — never set `.name` on an instance
  (`Button`, `Input`, `Dialog` stay `Button`, `Input`, `Dialog`). Figma shows the component
  name on an instance by default, and that is how designers, Dev Mode, Code Connect and the
  next agent recognise what it is; a renamed instance looks like a custom layer and breaks
  the trace back to the design system. Express purpose through the *containing* frame and
  through text/property values instead — a `Login Form` frame holding an `Input` whose
  label reads "Email" is clear; an instance renamed `Email Field` is not.
- Name the frames *you* create meaningfully — `Frame 47` tells the next agent (or
  `use_figma` call) nothing; `Login Form`, `Header`, `Content` does. This applies only to
  plain frames, sections and screens you draw, never to instances.
- Also leave the internal layers of an instance alone — never rename, reorder or detach
  nodes inside an instance; change them only via component properties and overrides of
  text/visibility/instance-swap.
- One page for the whole prototype (§2): a `Components` Section at the top, then one
  Section per flow below it — don't spread flows or components across pages.
- Lay screens out left-to-right in flow order with consistent spacing inside their flow's
  Section — this makes the page readable as a flow at a glance instead of a grid of frames.
- Work in small, incremental `use_figma` scripts, each returning a summary of created node
  IDs (per `figma-use` Rule 15) so the orchestrator can record them — one giant script is
  much harder to debug and recover from when a step fails partway through. When you're in
  new-component territory, `figma-generate-library` also requires sequential (never
  parallel) calls — follow that even inside an otherwise screen-building task.
- Screenshot after each screen to self-check before moving on — catching a clipped label or
  overlap right after building it is far cheaper than catching it three screens later.

---

## 8. Definition of done per screen

A screen is done only when all of these are true:

- [ ] Every piece of UI is an instance of an existing or newly-recorded atom/molecule/
      organism — no raw layers except layout frames (§1, §2)
- [ ] Anything genuinely new was built to the §3 standard (variants for every needed state,
      properties not variant explosion, bound to variables, description filled in,
      interactions wired) and logged in `NEW-COMPONENTS.md`
- [ ] No hardcoded fill/stroke/padding/gap/radius/font — verification snippet in §4 returns
      empty (or every hit is explained)
- [ ] Every frame with children uses auto layout except deliberate absolute overlays —
      verification snippet in §5 returns empty (or every hit is explained)
- [ ] Screen frame is 1512px wide, height hugs content, and it doesn't clip/overlap at
      ~1280 and ~1728 widths
- [ ] Content is realistic (real labels, plausible data, real error/empty copy) and visuals
      are wireframe-neutral (design-system defaults only, no new brand/imagery/decoration)
- [ ] Every frame Claude created is meaningfully named; every instance still carries its
      original component name (no renamed instances, nothing renamed inside instances)
- [ ] Screen sits in flow order, in the right Section, on the assigned page
- [ ] Screen has no overlay slot and is not a duplicate of another screen with an overlay
      on top; overlays it opens are shared frames in the `Overlays` sub-section
- [ ] `use_figma` calls that built it each returned their created/mutated node IDs, and
      those IDs are recorded for the orchestrator
- [ ] A screenshot was taken and reviewed after building
