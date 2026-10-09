# Figma Plugin API gotchas

Traps found on real projects, each of which cost a failed run, a rollback or a bug the user
had to report. Read before writing `use_figma` scripts; `figma-use` covers the general
mechanics, this file only what bit us anyway. Prototype-specific limits (reactions,
overlays, scroll) are in `prototyping.md` → "Known limits".

When a project hits a new one, add it to the project's `.ux-prototype/LESSONS.md`; if it
is general, propose adding it here.

## Scripts and transactions

- **A throwing script rolls back everything it did.** Split work into a read-only pass that
  collects IDs and checks preconditions, then a mutating pass that only uses known-good IDs.
  Guard every `getNodeByIdAsync` result before using it.
- **`get_metadata` without a node ID lists only one page.** To see all pages, list them with
  a script (`figma.root.children.map(p => ({ id: p.id, name: p.name }))`).

## Sizing and layout

- **`resize()` on an instance (or a cell inside one) resets its sizing to `FIXED`.** Set
  `layoutSizingHorizontal` / `layoutSizingVertical` back to `HUG`/`FILL` right after.
- **An inherited `maxWidth` can pin a `FILL` node** — it still reads `FILL` but stops
  growing. Clear `maxWidth` (set to `null`) before relying on `FILL`.
- Run the sizing read-back snippet in `build-rules.md` §5 after any layout edit.

## Instances, components and slots

- **A parent component can't drive the text of a nested instance** through its own text
  property. Expose the property on the nested component and surface it, or set the
  override on each instance.
- **Instance children can't be `remove()`d.** Hide them (`visible = false`) or use a
  boolean property.
- **Editing a main component's slot doesn't reach instances that already hold their own
  slot content.** After changing slot defaults, sweep the instances and update them.
- **Nodes moved out of a SLOT get new IDs.** Re-read IDs after the move; don't reuse the old
  ones in the same or later scripts.
- **Cloning a screen freezes reactions inside it as overrides** (e.g. sidebar navigation).
  Call `resetOverrides()` on the cloned navigation instance so it inherits from the main
  component again.

## Vectors and tokens

- **`vectorPaths` doesn't accept SVG arc commands (`A`/`a`).** Convert arcs to cubic Béziers,
  or use an existing icon component instead of drawing.
- **Don't tokenise frame dimensions.** A builder once created `spacing/432 = 1728` to bind a
  screen width. Screen and overlay frame sizes are presentation settings, not tokens.
