# Prototyping reference

Wiring Figma's prototype layer (Reactions, Navigation, overlays, scroll, flows) on top of nodes the `figma-use` skill already knows how to build. `figma-use` covers node/component/variable mechanics; this file covers making things clickable and connected.

## Contents

1. [Principle: component behaviour lives on the component](#1-principle-component-behaviour-lives-on-the-component)
2. [Principle: screen navigation lives on the instance inside the screen](#2-principle-screen-navigation-lives-on-the-instance-inside-the-screen)
3. [Working snippets](#3-working-snippets)
   - [Read existing reactions](#read-existing-reactions)
   - [setReactionsAsync basics](#setreactionsasync-basics)
   - [ON_CLICK navigate to another screen](#on_click-navigate-to-another-screen)
   - [ON_HOVER / MOUSE_ENTER · MOUSE_LEAVE change-to variant](#on_hover--mouse_enter--mouse_leave-change-to-variant)
   - [ON_PRESS](#on_press)
   - [AFTER_TIMEOUT (loading → loaded)](#after_timeout-loading--loaded)
   - [Open overlay / close overlay](#open-overlay--close-overlay)
   - [BACK](#back)
   - [SCROLL_TO](#scroll_to)
   - [Transitions: instant vs dissolve/smart-animate](#transitions-instant-vs-dissolvesmart-animate)
   - [flowStartingPoints — one per user flow](#flowstartingpoints--one-per-user-flow)
   - [Scroll overflow and fixed (sticky) children](#scroll-overflow-and-fixed-sticky-children)
4. [Wiring checklist per flow](#4-wiring-checklist-per-flow)
5. [Verification script](#5-verification-script)
6. [Common failure modes and fixes](#6-common-failure-modes-and-fixes)

---

## 1. Principle: component behaviour lives on the component

Any interaction that expresses what a component *does by itself* — hover, pressed, focus, open/closed, checked, tab switch, accordion expand/collapse, input filled/error, dropdown open — is wired exactly once, on the variants inside the component's `COMPONENT_SET`, using `navigation: 'CHANGE_TO'`.

Why:

- **Single source of truth.** The reaction lives on the master variant. Every instance dropped anywhere in the file inherits it automatically — there is nothing to re-wire per instance, and nothing for a future session (human or agent) to forget.
- **CHANGE_TO requires same-set variants.** `navigation: 'CHANGE_TO'` only works when `destinationId` is another `COMPONENT` inside the *same* `COMPONENT_SET` as the trigger node. It swaps the active variant in place without navigating anywhere — this is how Figma represents "hover state," "pressed state," "checked," etc.
- **Never re-wire this per instance.** If you find yourself setting a hover reaction on a `Button` instance out on a screen, stop — that reaction belongs on the `Default` variant inside the `Button` component set, pointing at the `Hover` variant. Fix it once at the source.

Concretely: open (or `getNodeByIdAsync`) the `COMPONENT_SET`, iterate its variant `COMPONENT` children, and call `setReactionsAsync` on each variant with `CHANGE_TO` reactions pointing at sibling variants (`ON_HOVER` → hover variant, `ON_PRESS`/`MOUSE_DOWN` → pressed variant, etc.). See [ON_HOVER / MOUSE_ENTER](#on_hover--mouse_enter--mouse_leave-change-to-variant) below for the exact call.

## 2. Principle: screen navigation lives on the instance inside the screen

Anything that moves the user between screens — screen → screen, open overlay, close overlay, back, scroll-to — is wired on the specific instance/layer sitting inside that screen frame, not on the component definition. Destination screens are screen-specific: the "Continue" button on the Sign Up screen goes to a different place than the "Continue" button on the Checkout screen, even though both are the same `Button` component.

- **Target the instance, not the master component.** `setReactionsAsync` is called on the `InstanceNode` (or a nested node inside it) that lives inside the top-level screen `FrameNode`, with `destinationId` pointing at another top-level frame on the same page.
- **Targeting a nested node inside an instance.** Reactions can be set on any node that has `ReactionMixin` — including nodes nested inside an instance (e.g. a "Back" icon-button nested inside a header inside a card instance), as long as you can resolve that nested node's ID (`instance.findAllWithCriteria({ types: [...] }).find(...)`). Instance children still carry their own reactions independent of the master.
- **Fallback if per-instance nesting isn't practical.** If the interactive sub-element is buried too deep to address reliably (e.g. inside a locked/complex nested instance), wire the reaction on the outer instance itself, or wrap the target region in a thin `FRAME` with its own reaction and stack it over the clickable area. Prefer the direct nested-node approach first — it keeps the hit target visually accurate.
- **NAVIGATE / SWAP / OVERLAY / SCROLL_TO destinations must be top-level frames on the same page.** They cannot target a frame on a different page or an arbitrary nested layer (SCROLL_TO's target frame must itself be scrollable, see below).

## 3. Working snippets

All snippets assume the calling script already resolved `figma.currentPage` per the `figma-use` page rules (set it at most once per script), and that `nodeId` values are captured `string` IDs from prior calls (per `gotchas.md` → "Prefer returned IDs for workflow state").

### Read existing reactions

```js
const node = await figma.getNodeByIdAsync(NODE_ID)
if (!node || !('reactions' in node)) {
  return { error: 'Node not found or has no ReactionMixin', nodeId: NODE_ID }
}
return { nodeId: node.id, reactions: node.reactions }
```

`reactions` is read-only — Figma docs say to `JSON.parse(JSON.stringify(node.reactions))` (or any deep clone) before mutating, then pass the clone to `setReactionsAsync`. Never mutate `node.reactions` in place and expect it to persist.

### setReactionsAsync basics

```js
const node = await figma.getNodeByIdAsync(NODE_ID)
await node.setReactionsAsync([
  {
    trigger: { type: 'ON_CLICK' },
    actions: [
      {
        type: 'NODE',
        destinationId: DESTINATION_FRAME_ID,
        navigation: 'NAVIGATE',
        transition: null, // instant — see transitions section
      },
    ],
  },
])
return { nodeId: node.id, reactionCount: node.reactions.length }
```

Notes from the type signature (`Reaction = { trigger: Trigger | null, actions?: Action[] }`):

- `setReactionsAsync` **replaces** the full reaction list on that node — always pass every reaction you want the node to have, not just the new one. Read first, append/modify, then write the full array back.
- `setReactionsAsync` is the only way to write reactions when the plugin manifest uses `"documentAccess": "dynamic-page"` (the default for `use_figma`). Do not attempt `node.reactions = [...]` — that throws.
- Batch independent nodes with `Promise.all` — each `setReactionsAsync` call is its own round trip.

### ON_CLICK navigate to another screen

```js
const button = await figma.getNodeByIdAsync(BUTTON_INSTANCE_ID) // lives inside the screen frame
await button.setReactionsAsync([
  {
    trigger: { type: 'ON_CLICK' },
    actions: [
      {
        type: 'NODE',
        destinationId: NEXT_SCREEN_FRAME_ID,
        navigation: 'NAVIGATE',
        transition: { type: 'DISSOLVE', easing: { type: 'EASE_OUT' }, duration: 0.2 },
        resetScrollPosition: true,
      },
    ],
  },
])
return { nodeId: button.id }
```

### ON_HOVER / MOUSE_ENTER · MOUSE_LEAVE change-to variant

Wired on the **variant inside the component set** (see principle 1), not on screen instances.

```js
const defaultVariant = await figma.getNodeByIdAsync(BUTTON_DEFAULT_VARIANT_ID)
const hoverVariant = await figma.getNodeByIdAsync(BUTTON_HOVER_VARIANT_ID) // sibling in same COMPONENT_SET
await defaultVariant.setReactionsAsync([
  {
    trigger: { type: 'ON_HOVER' },
    actions: [
      { type: 'NODE', destinationId: hoverVariant.id, navigation: 'CHANGE_TO', transition: null },
    ],
  },
])
// Mirror the return trip so the hover state reverts on mouse-leave
await hoverVariant.setReactionsAsync([
  {
    trigger: { type: 'ON_HOVER' },
    actions: [
      { type: 'NODE', destinationId: defaultVariant.id, navigation: 'CHANGE_TO', transition: null },
    ],
  },
])
return { defaultVariantId: defaultVariant.id, hoverVariantId: hoverVariant.id }
```

`ON_HOVER` alone models "while hovering, look like X"; Figma's own hover behaviour treats it as a two-way toggle in the prototype player. `MOUSE_ENTER` / `MOUSE_LEAVE` triggers exist too (each carries `delay: number` and `deprecatedVersion: boolean`, e.g. `{ type: 'MOUSE_ENTER', delay: 0, deprecatedVersion: false }`) for enter/leave asymmetry or a delay — prefer plain `ON_HOVER` unless you need that.

### ON_PRESS

```js
// Pressed state — also lives on the component variant, alongside hover
await defaultVariant.setReactionsAsync([
  {
    trigger: { type: 'ON_PRESS' },
    actions: [{ type: 'NODE', destinationId: pressedVariant.id, navigation: 'CHANGE_TO', transition: null }],
  },
])
```

`ON_PRESS`, `ON_CLICK`, `ON_HOVER`, `ON_DRAG` share the same trigger shape (`{ type }` only, no extra fields). `MOUSE_DOWN` / `MOUSE_UP` are the delay-bearing siblings of press if finer timing control is needed (`{ type: 'MOUSE_DOWN', delay: number }`).

### AFTER_TIMEOUT (loading → loaded)

Wired on the loading-state screen frame itself (or a loading variant), navigating forward automatically.

```js
const loadingScreen = await figma.getNodeByIdAsync(LOADING_SCREEN_FRAME_ID)
await loadingScreen.setReactionsAsync([
  {
    trigger: { type: 'AFTER_TIMEOUT', timeout: 1.2 }, // seconds
    actions: [
      {
        type: 'NODE',
        destinationId: LOADED_SCREEN_FRAME_ID,
        navigation: 'NAVIGATE',
        transition: { type: 'DISSOLVE', easing: { type: 'EASE_OUT' }, duration: 0.2 },
      },
    ],
  },
])
return { nodeId: loadingScreen.id }
```

### Open overlay / close overlay

Overlays are opened with `navigation: 'OVERLAY'` from the trigger element, and closed with a `CLOSE` action from inside the overlay content (typically a close (X) icon-button or scrim tap).

```js
// Open — wired on the trigger element inside the calling screen
const trigger = await figma.getNodeByIdAsync(TRIGGER_INSTANCE_ID)
await trigger.setReactionsAsync([
  {
    trigger: { type: 'ON_CLICK' },
    actions: [
      {
        type: 'NODE',
        destinationId: OVERLAY_FRAME_ID, // a top-level frame on the same page
        navigation: 'OVERLAY',
        transition: { type: 'DISSOLVE', easing: { type: 'EASE_OUT' }, duration: 0.2 },
      },
    ],
  },
])

// Close — wired on the close button inside the overlay frame
const closeButton = await figma.getNodeByIdAsync(CLOSE_BUTTON_ID)
await closeButton.setReactionsAsync([
  {
    trigger: { type: 'ON_CLICK' },
    actions: [{ type: 'CLOSE' }],
  },
])
```

**Overlays are single shared frames, never duplicated screens.** Each dialog, sheet, dropdown menu, popover, tooltip, toast or command palette exists exactly once, as its own top-level frame on the prototype page (an `Overlays` sub-section of the `Components` Section), containing one instance of the relevant design-system component (`Dialog`, `Sheet`, `Dropdown Menu`, …) with its real content. Screens open it with `OVERLAY` and it closes with `CLOSE`. Do **not** add an "overlay slot" to screens, and do not duplicate a screen to show it with an overlay on top — that multiplies screens (Settings, Settings + delete dialog, Settings + saved toast…) and every later change has to be made N times. A base screen plus one overlay frame is one screen in the file and one screen in the reviewer's head.

**Wire the overlay from the main component whenever the overlay belongs to the component.** `OVERLAY` reactions set on a node inside a main component (or a variant inside a component set) are inherited by every instance, exactly like `CHANGE_TO` reactions — and because the whole prototype lives on one page, the "destination must be a top-level frame on the same page" rule is satisfied. So:

- A component-owned overlay — the account menu opened from the app header's avatar, the options list of a `Select`/`Combobox`, a `Tooltip` on an icon button, the `Date Picker` popover of a date input, the `Command` palette from the search field — is wired **once, on the main component** (organism or atom), pointing at the single overlay frame. Every screen that uses the header then opens the same menu without any per-screen wiring.
- A screen-specific overlay — the "Delete project?" confirm dialog from *this* screen's delete button, the "Changes saved" toast after *this* form's submit — is wired on the instance inside the screen, still pointing at the single shared overlay frame for that dialog/toast.
- Closing is wired once, inside the overlay frame (`CLOSE` on its close/cancel button; a primary action that should also move on gets `CLOSE` followed by `NAVIGATE` in the same `actions` array, or just `NAVIGATE`, which dismisses the overlay). Because the overlay frame contains an instance of the component, put close reactions on the nodes *inside that instance* — or better, if the overlay component itself owns a close button, wire `CLOSE` on the main component's close button so every overlay built from it closes without extra work.

```js
// Component-owned overlay: wire on the MAIN component so all instances inherit it
const header = await figma.getNodeByIdAsync(APP_HEADER_MAIN_COMPONENT_ID) // COMPONENT, not an instance
const avatarTrigger = header.findOne(n => n.name === 'Avatar')
await avatarTrigger.setReactionsAsync([{
  trigger: { type: 'ON_CLICK' },
  actions: [{ type: 'NODE', destinationId: ACCOUNT_MENU_OVERLAY_FRAME_ID, navigation: 'OVERLAY',
              transition: { type: 'DISSOLVE', easing: { type: 'EASE_OUT' }, duration: 0.2 } }],
}])
```

**Overlay position/background are read-only on the frame in the Plugin API.** `FramePrototypingMixin.overlayPositionType`, `.overlayBackground`, and `.overlayBackgroundInteraction` are all declared `readonly` — there is no setter in this d.ts. They can only be read, not written; changing "centered modal" vs "bottom sheet" placement, background dim, or click-outside-to-dismiss must be done by hand in Figma's Prototype panel. The one overlay-related field the **action** *can* set is `overlayRelativePosition` (a `Vector`) on the `NODE` action, and only when the destination's `overlayPositionType` is already `'MANUAL'`. A frame first used as an overlay gets Figma's defaults (centered, no background dim, close on click outside), which is fine for dialogs and acceptable as a wireframe for menus and sheets. This limitation is **not** a reason to avoid overlays or to fake them by duplicating screens — use the overlay and list, in the checkpoint summary, the overlay frames whose placement/background the user should adjust by hand (e.g. "Sheet → anchor right", "Dropdown → manual position under trigger").

### BACK

```js
const backButton = await figma.getNodeByIdAsync(BACK_BUTTON_ID)
await backButton.setReactionsAsync([
  {
    trigger: { type: 'ON_CLICK' },
    actions: [{ type: 'BACK' }],
  },
])
```

`BACK` returns to the previous frame in the prototype's navigation history (whatever frame the user actually came from), not a fixed destination — use it for "Back" chevrons and Cancel buttons on modals opened from multiple places.

### SCROLL_TO

```js
const jumpLink = await figma.getNodeByIdAsync(JUMP_LINK_ID) // e.g. a "Jump to reviews" link
await jumpLink.setReactionsAsync([
  {
    trigger: { type: 'ON_CLICK' },
    actions: [
      {
        type: 'NODE',
        destinationId: REVIEWS_SECTION_FRAME_ID, // must be a scrollable frame, or a node within one
        navigation: 'SCROLL_TO',
        transition: { type: 'SMART_ANIMATE', easing: { type: 'EASE_IN_AND_OUT' }, duration: 0.3 },
      },
    ],
  },
])
```

`SCROLL_TO` scrolls the current screen's scroll container to bring the destination node into view — it does not navigate to a different top-level frame. The destination only needs to be inside a frame with a non-`'NONE'` `overflowDirection` (see [Scroll overflow](#scroll-overflow-and-fixed-sticky-children)) for the scroll to actually move.

### Transitions: instant vs dissolve/smart-animate

This skill builds wireframe-level UX, not motion design — keep transitions simple and legible. Two options cover nearly everything:

```js
// Instant — for state changes the user expects to feel immediate (toggles, tab switches)
const instant = null

// Dissolve — cheap, safe default for screen-to-screen navigation
const dissolve = { type: 'DISSOLVE', easing: { type: 'EASE_OUT' }, duration: 0.2 }

// Smart animate — for continuity between screens that share matching layer names/structure
// (e.g. a card expanding into a detail screen)
const smartAnimate = { type: 'SMART_ANIMATE', easing: { type: 'EASE_OUT' }, duration: 0.2 }
```

Avoid `DirectionalTransition` (`MOVE_IN`/`MOVE_OUT`/`PUSH`/`SLIDE_IN`/`SLIDE_OUT`, which need a `direction` and `matchLayers`) and exotic `Easing` types (`BOUNCY`, `CUSTOM_SPRING`, etc.) unless the user explicitly asks for a specific motion treatment — they add configuration surface a wireframe-level prototype doesn't need. `null` (instant) or ~200ms dissolve/smart-animate covers the default case.

### flowStartingPoints — one per user flow

`flowStartingPoints` lives on the `PageNode`, is a plain read/write array (`ReadonlyArray<{ nodeId: string; name: string }>` typed, reassign the whole array to change it), and lists the named entry points shown in Presentation view's flow picker. Give each user flow ("Onboarding", "Checkout", "Password reset") exactly one starting point on its first screen.

```js
const page = figma.currentPage // set via setCurrentPageAsync earlier in this script, per figma-use page rules
const existing = page.flowStartingPoints.map(fp => ({ ...fp }))
const already = existing.some(fp => fp.nodeId === FIRST_SCREEN_FRAME_ID)
if (!already) {
  page.flowStartingPoints = existing.concat([{ nodeId: FIRST_SCREEN_FRAME_ID, name: 'Onboarding' }])
}
return { flowStartingPoints: page.flowStartingPoints }
```

`nodeId` should point at a top-level `FrameNode` (or `COMPONENT`/`INSTANCE`/`GroupNode` per the type union documented on `prototypeStartNode`) that is the true first screen of the flow — not a shared shell or a screen also used as an entry point for another flow, since flows are meant to be independently launchable and named.

### Scroll overflow and fixed (sticky) children

Both live on `FramePrototypingMixin`, so only on frame-shaped nodes (`FrameNode`, `ComponentNode`, `ComponentSetNode` via `DefaultFrameMixin`) — not on groups or shapes.

```js
const screen = await figma.getNodeByIdAsync(SCREEN_FRAME_ID)
screen.overflowDirection = 'VERTICAL' // 'NONE' | 'HORIZONTAL' | 'VERTICAL' | 'BOTH'
return { nodeId: screen.id, overflowDirection: screen.overflowDirection }
```

`numberOfFixedChildren` pins the *first N children in z-order* (not an arbitrary subset) as sticky headers/nav bars that stay fixed while the rest of the frame scrolls underneath. Order the header/nav bar as the frame's first child(ren) before setting this:

```js
const screen = await figma.getNodeByIdAsync(SCREEN_FRAME_ID)
const header = await figma.getNodeByIdAsync(HEADER_NODE_ID)
screen.insertChild(0, header) // fixed children must be first in z-order
screen.numberOfFixedChildren = 1 // pins children[0] (the header) while the rest scrolls
return { nodeId: screen.id, numberOfFixedChildren: screen.numberOfFixedChildren }
```

If both a sticky header and a sticky bottom tab bar are needed, both must be among the first `numberOfFixedChildren` children in the layer order — reorder both to the top before setting the count.

## 4. Wiring checklist per flow

Run this checklist for every user flow before calling it done:

- [ ] **Every interactive element has a reaction, or a documented reason it's a dead end.** Buttons, links, tabs, list rows, icon-buttons, chips — each either has a `reactions` entry or is intentionally inert (e.g. a disabled state, a placeholder "coming soon" affordance) and that reason is noted in the build report, not just silently skipped.
- [ ] **Every screen is reachable from the flow's `flowStartingPoints` entry.** No orphaned screens that only exist as unreferenced frames on the page.
- [ ] **Every screen has a way back or out** — a `BACK` action, an explicit `NAVIGATE` to a known previous/parent screen, or (for the flow's true entry screen) it legitimately has no "back."
- [ ] **Error, empty, loading, and success states are reachable**, not just visually designed. If a screen has an `Error` variant or a companion `[Screen] — Error` frame, some trigger in the flow must actually navigate/change-to it (even if only for demo purposes, e.g. a "Simulate error" affordance) — an unreachable error state isn't prototyped, it's just drawn.
- [ ] **One frame per overlay.** No screen contains an "overlay slot", and no two top-level frames are the same screen differing only by an overlay on top. Component-owned overlays are wired on the main component, not per screen.
- [ ] **Overlays close.** Every `OVERLAY` destination has at least one `CLOSE` action reachable from inside it (close button, and/or rely on `overlayBackgroundInteraction` if it's already set to `CLOSE_ON_CLICK_OUTSIDE` in the file — remember this property is read-only from script, so don't assume it without checking).
- [ ] **One flow starting point per flow**, pointing at that flow's actual first screen, named after the flow.

## 5. Verification script

Read-only audit script. Scope traversal to the smallest known ancestor per the `figma-use` gotcha — pass a specific page or top-level frame ID rather than scanning `figma.root`.

```js
// Run with figma.currentPage already set to the target page (once, per figma-use rules).
const page = figma.currentPage
const topFrames = page.children.filter(n => n.type === 'FRAME')

// (a) top-level frames with no inbound navigation
const allNodes = page.findAllWithCriteria({
  types: ['FRAME', 'INSTANCE', 'COMPONENT', 'GROUP', 'TEXT', 'RECTANGLE'],
})
const referencedIds = new Set()
const collectDestinations = (reactions) => {
  for (const r of reactions || []) {
    for (const a of r.actions || []) {
      if (a.type === 'NODE' && a.destinationId) referencedIds.add(a.destinationId)
      if (a.type === 'CONDITIONAL') {
        for (const block of a.conditionalBlocks || []) collectDestinations([{ actions: block.actions }])
      }
    }
  }
}
for (const n of allNodes) {
  if ('reactions' in n) collectDestinations(n.reactions)
}
for (const fp of page.flowStartingPoints) referencedIds.add(fp.nodeId)

const framesWithNoInbound = topFrames
  .filter(f => !referencedIds.has(f.id))
  .map(f => ({ id: f.id, name: f.name }))

// (b) interactive-looking instances with zero reactions
const NAME_PATTERN = /button|link|tab|input|checkbox|toggle|switch|radio|chip|menu|dropdown|select/i
const instances = page.findAllWithCriteria({ types: ['INSTANCE'] })
const uninstrumented = instances
  .filter(n => NAME_PATTERN.test(n.name) && (!n.reactions || n.reactions.length === 0))
  .map(n => ({ id: n.id, name: n.name }))

// (c) reactions whose destinationId no longer exists
const allIds = new Set(allNodes.map(n => n.id))
const brokenRefs = []
const checkBroken = (nodeId, nodeName, reactions) => {
  for (const r of reactions || []) {
    for (const a of r.actions || []) {
      if (a.type === 'NODE' && a.destinationId && !allIds.has(a.destinationId)) {
        brokenRefs.push({ nodeId, nodeName, destinationId: a.destinationId })
      }
      if (a.type === 'CONDITIONAL') {
        for (const block of a.conditionalBlocks || []) {
          checkBroken(nodeId, nodeName, [{ actions: block.actions }])
        }
      }
    }
  }
}
for (const n of allNodes) {
  if ('reactions' in n) checkBroken(n.id, n.name, n.reactions)
}

return {
  framesWithNoInbound,
  uninstrumented,
  brokenRefs,
}
```

Notes: `allIds` only covers the criteria-filtered `allNodes` types above — extend the `types` list if the file uses other interactive node shapes (e.g. vectors as tap targets), or note the scope limit in the report. `(a)` treats `flowStartingPoints` entries as valid inbound references, since a flow's first screen is legitimately never targeted by a `NODE` action. `(b)` is a name-based heuristic (per `gotchas.md`'s guidance that name-only lookups are the right tool absent a type/criteria signal) — treat it as a checklist prompt, not ground truth.

## 6. Common failure modes and fixes

- **Reactions silently don't stick.** `node.reactions = [...]` throws or is ignored under `"documentAccess": "dynamic-page"` (the mode `use_figma` runs in) — always use `await node.setReactionsAsync([...])`.
- **`setReactionsAsync` wipes out reactions the node already had.** It replaces the whole list. Read `node.reactions`, deep-clone it (`JSON.parse(JSON.stringify(...))`), append/edit, then write the full array back — don't pass just the new reaction.
- **`CHANGE_TO` fails or does nothing.** The destination must be a `COMPONENT` that is a variant inside the *same* `COMPONENT_SET` as the source node. `CHANGE_TO` pointed at an unrelated frame, or at a component in a different set, is not a supported combination per the `Navigation` union's intended use — verify both nodes share a `COMPONENT_SET` parent before wiring.
- **`NAVIGATE`/`SWAP`/`OVERLAY`/`SCROLL_TO` destination not found at runtime.** `destinationId` must resolve to a node that exists on the *same page* the trigger node lives on — Figma prototyping does not navigate across pages. Re-verify IDs with the [verification script](#5-verification-script)'s `(c)` check after any restructuring, since node IDs can change on detach/reparent operations (see the `figma-use` `detachInstance()` gotcha).
- **Component-level reaction gets silently overridden per-instance.** If someone later calls `setReactionsAsync` directly on an *instance* of a component whose variants already carry `CHANGE_TO` reactions, the instance-level call can add a second, redundant, or conflicting reaction rather than relying on inheritance. Don't call `setReactionsAsync` on instances for behaviour that's already defined on the master component's variants — only call it on instances for screen-specific navigation (principle 2).
- **Overlay position/background looks wrong and the script "did nothing."** `overlayPositionType`, `overlayBackground`, and `overlayBackgroundInteraction` are `readonly` in this API — there is no setter. These must be configured by hand in Figma's Prototype panel; a script cannot change them. Only `overlayRelativePosition` on the triggering action (for `MANUAL` position type) is writable.
- **`numberOfFixedChildren` pins the wrong layer.** It fixes children by index position (`children[0..N-1]`), not by name or flag. If a sticky header isn't sticking, check `frame.children` order first — `insertChild(0, header)` before setting the count.
- **`SCROLL_TO` doesn't move the viewport.** The destination frame needs `overflowDirection` set to something other than `'NONE'` for scrolling to be possible at all in presentation mode — set `overflowDirection` before wiring `SCROLL_TO` reactions into that frame.
- **Multiple `flowStartingPoints` entries pointing at the same node, or a flow with none.** Reassign the whole array (it's not append-only) — read existing entries, dedupe on `nodeId`, then write back, per the [flowStartingPoints snippet](#flowstartingpoints--one-per-user-flow).
- **Transition object shape mismatch.** `SimpleTransition` (`DISSOLVE`/`SMART_ANIMATE`/`SCROLL_ANIMATE`) requires `easing` + `duration`; `DirectionalTransition` (`MOVE_IN`/`MOVE_OUT`/`PUSH`/`SLIDE_IN`/`SLIDE_OUT`) additionally requires `direction` and `matchLayers` — mixing fields from one shape into the other throws a validation error. Stick to `SimpleTransition` (or `null`) per the [transitions guidance](#transitions-instant-vs-dissolvesmart-animate) above and this doesn't come up.
