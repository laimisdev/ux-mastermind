# Orchestration

How the main session plans, delegates and keeps records. The aim is the same output quality at a fraction of the tokens: the main session holds the plan and the conversation, subagents hold the bulk.

## Contents
1. Why delegate
2. Roles and models
3. What may run in parallel
4. Briefing a subagent
5. Brief templates
6. Handling results and failures

## 1. Why delegate

Figma reads are large (metadata trees, variable dumps, screenshots) and Plugin API scripts take several attempts. If that happens in the main context, the session fills up before the second flow and the plan gets summarised away. A subagent absorbs the noise and returns twenty lines. Cheap models are good at well-specified, checkable work — so your job is to make the work well-specified and checkable.

## 2. Roles and models

Pass the model explicitly when spawning (`model: "haiku"` / `"sonnet"`). Record the model used in the task's "Agent model" column.

| Role | Model | Reads | Does | Returns |
|---|---|---|---|---|
| **Resource summariser** | haiku | one or a few documents/URLs | extracts goals, users, features, constraints, open questions | ≤ 40-line summary per resource |
| **Inventory agent** | haiku (sonnet if the file is large/messy and haiku's catalogue comes back thin) | Figma file via `get_metadata`, `search_design_system`, `get_variable_defs`, read-only `use_figma` scripts | catalogues components, variants, properties, keys/IDs, variables, styles, conventions | writes `DESIGN-SYSTEM.md`; returns gaps + conventions digest |
| **Researcher** | haiku; sonnet for novel or regulated domains | `references/ux-research.md`, `PROJECT.md` | Mobbin + web research for one feature | writes `research/<feature>.md`; returns digest + open questions |
| **Component builder** | sonnet | `build-rules.md`, `prototyping.md`, relevant rows of `DESIGN-SYSTEM.md` / `NEW-COMPONENTS.md`, research note | creates one missing component (set) with states, props, variables, main-component interactions | node IDs, variant list, props, interactions, tokens added |
| **Screen builder** | sonnet | same + the flow's block from `FLOWS.md` | builds a template or a batch of 1–4 closely related screens from instances | node IDs per screen, components used, anything missing |
| **Prototype wirer** | sonnet | `prototyping.md`, flow block with node IDs | wires navigation/overlays/back, sets flow starting point | wiring table rows, unresolved dead ends |
| **QA verifier** | haiku | `build-rules.md` §DoD, `prototyping.md` §verification, node IDs to check | runs verification scripts, resize screenshots, reports defects; does **not** fix | pass/fail list with node IDs |
| **Fixer** | sonnet | QA report | fixes listed defects only | what changed |

Keep for yourself: the plan, every user conversation, choosing between research-backed alternatives, component architecture when a new organism has non-obvious slots/variants, and all writes to `STATE.md`, `TASKS.md`, `FLOWS.md`, `PROJECT.md`, `NEW-COMPONENTS.md`, `LOG.md`.

Every subagent that calls `use_figma` must first load Figma's `figma-use` skill; component builders also load `figma-generate-library`. Say so in the brief — a cold subagent doesn't know.

## 3. What may run in parallel

Read-only and off-Figma work parallelises freely: all researchers for a flow at once, summarisers, and the inventory alongside them.

Writes to one Figma file need care, because two agents editing the same page or component produce conflicts and half-applied scripts:

- One writer per Figma page (or per Section, if the brief names the section and its canvas area) at a time.
- Anything shared — a new component, a new variable, a template — is built **serially and before** the agents that depend on it start. Finish the atomic level below before fanning out the level above.
- Good fan-out: after templates and organisms for a flow exist, split its screens across 2–3 screen builders, each given its own section and x/y origin. Or build two independent flows on two pages simultaneously.
- Launch independent agents in a single message so they actually run concurrently. Run in the background when you have other useful work (e.g. talking to the user); otherwise wait.
- QA runs after the builders for that area have all returned.

Typical flow schedule: researchers (parallel) → you refine the screen list → component builders (serial, bottom-up) → template builder → screen builders (parallel by section) → wirer → QA → fixer if needed → you record and checkpoint with the user.

## 4. Briefing a subagent

The subagent starts with nothing: no conversation, no project knowledge, no idea this skill exists. A good brief is self-contained and small:

- **Goal and why** in two sentences, including the product context that affects judgement.
- **Exact targets**: Figma file URL + key, page/section name, node IDs of components to use, canvas origin, frame width 1512.
- **What to read**: absolute paths to the skill's reference files and the specific `.ux-prototype/` files (or paste just the relevant rows — cheaper than making it read a long inventory).
- **Boundaries**: which pages it may touch, that it must not detach/restyle/delete existing components, that it must not edit the orchestrator-owned state files.
- **Definition of done**: the checks it must run on its own work before returning.
- **Return format**: compact and structured so you can paste it into the records; cap the length. Ask for failures and doubts explicitly — a silent workaround is worse than an honest "couldn't".

Give one agent one coherent job. A brief that needs more than ~40 lines usually hides two jobs.

## 5. Brief templates

Fill the `<…>` parts; trim anything irrelevant.

**Researcher**
```
You are researching UX best practice for ONE feature before it is prototyped.
Feature: <feature> — in the context of: <2-line product/user summary>.
Follow <skill path>/references/ux-research.md exactly (procedure, budget, note template).
Project context: read <cwd>/.ux-prototype/PROJECT.md.
Write the note to <cwd>/.ux-prototype/research/<slug>.md.
Return: the path, a 5-bullet digest, and open questions for the user. Nothing else.
```

**Inventory agent**
```
Catalogue the design system in Figma file <url> (key <key>) so later agents can pick
existing components without re-reading the file. Read-only: do not modify the file.
Load the figma-use skill before any use_figma call. Use get_metadata / search_design_system /
get_variable_defs and small read-only use_figma scripts that return compact JSON; page through
the file rather than dumping it in one call.
Fill in <cwd>/.ux-prototype/DESIGN-SYSTEM.md (template already there): conventions, every
component set with atomic level, node ID/key, variants & properties, "use it for / don't use it
for", variable collections & modes, text/effect styles.
Planned flows (to judge gaps): <list>.
Return ≤ 25 lines: counts, naming conventions, notable gaps, anything odd (detached copies,
duplicates, unpublished libraries).
```

**Component builder**
```
Create ONE new component in Figma file <url> (key <key>) because nothing existing fits.
Component: <name, atomic level> — purpose: <what it does in the flow>.
Build it from these existing components (instances): <name → node ID/key list>.
States/variants required: <list>. Properties: <text/boolean/instance-swap list>.
Place it on page "<page>", section "<section>", near x=<x>, y=<y>.
Read first: <skill path>/references/build-rules.md (§2–5) and references/prototyping.md (§1, §3).
Load the figma-use and figma-generate-library skills before use_figma.
Use only existing variables/styles: <relevant tokens>. If a value is truly missing, add a
variable following the collection's naming and modes and report it.
Wire state interactions on the main component variants. Fill in the component description.
Self-check: run the unbound-values and no-auto-layout snippets on the component; screenshot it.
Do not edit any .ux-prototype files.
Return: component set node ID + key, variants, properties, interactions wired, tokens added,
doubts. ≤ 30 lines.
```

**Screen builder**
```
Build these screens for flow "<flow>" in Figma file <url> (key <key>), page "<page>",
section "<section>", first frame at x=<x>, y=<y>, 1512 wide, 120px gaps, left-to-right in order:
<# | screen | purpose | states | template | organisms/molecules to use (name → node ID)>.
Context: <2 lines about product/users>. Research digest: <5 bullets or note path>.
Read first: <skill path>/references/build-rules.md. Load the figma-use skill before use_figma.
Everything is instances of existing components with realistic content; if something needed
does not exist, STOP building that part and report it rather than drawing raw layers.
Work in small scripts that return created node IDs. Screenshot each finished screen and fix
what looks broken. Run the definition-of-done checks.
Do not wire screen-to-screen navigation (a later agent does) unless told otherwise. Do not
edit any .ux-prototype files.
Return: table of screen → node ID → components used; missing pieces; doubts. ≤ 30 lines.
```

**Prototype wirer**
```
Wire the prototype for flow "<flow>" in Figma file <url> (key <key>), page "<page>".
Screens and node IDs: <table>. Intended wiring: <From › element | trigger | action | To>.
Read first: <skill path>/references/prototyping.md. Load the figma-use skill before use_figma.
Component state behaviour belongs on main components — if an instance lacks inherited
behaviour, fix the main component (<NEW-COMPONENTS rows>) rather than wiring the instance.
Set one flow starting point named "<flow>". Run the verification snippet and resolve or list
every dead end and unreachable screen.
Return: completed wiring rows, verification JSON summary, unresolved items. ≤ 30 lines.
```

**QA verifier**
```
Verify, do not fix. Figma file <url> (key <key>), nodes: <IDs>.
Read <skill path>/references/build-rules.md (§8 definition of done + verification snippets) and
references/prototyping.md (§ verification). Load the figma-use skill before use_figma.
Run the snippets (read-only), take screenshots at current width, and resize-check at 1280 and
1728 (restore 1512 afterwards).
Return a defect list: node ID | rule broken | evidence. Say "PASS" per screen when clean. ≤ 40 lines.
```

## 6. Handling results and failures

- Record immediately: paste returned node IDs into `FLOWS.md` / `NEW-COMPONENTS.md`, advance `TASKS.md`. A crash after this point loses nothing.
- Trust but spot-check: take one screenshot of a builder's output yourself before fanning out more work that depends on it.
- If a subagent reports a missing component, that becomes a new task ahead of the screen in `TASKS.md`; don't let the screen builder improvise.
- If a cheap model fails a clear brief once, retry once on the next model up with the failure attached; note it in the task. If it fails again, the brief or the plan is wrong — rethink it yourself.
- If the Figma or Mobbin MCP becomes unavailable mid-session, mark affected tasks `blocked` with the reason, tell the user what they need to do, and continue with work that doesn't depend on it.
