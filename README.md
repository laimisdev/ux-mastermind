# UX Mastermind

A Claude Code skill that plans, builds and fully prototypes wireframe-level UX inside your Figma file (shadcn design system) through the Figma MCP — reusing existing components, atomic design, variables/tokens, auto layout, Mobbin + web UX research, and a persistent `.ux-prototype/` project memory so any session or teammate can continue where the last one stopped.

## Install

```bash
# into the current project (.claude/skills/)
npx skills add laimisdev/ux-mastermind -a claude-code

# or once, for all projects
npx skills add laimisdev/ux-mastermind -a claude-code -g
```

## Update

```bash
npx skills update
```

## Use

In Claude Code, type `/ux-mastermind` or just share a Figma link with a brief. The skill walks you through setup one step at a time (Figma MCP → Mobbin MCP → Figma file → briefs → live product → questions incl. presentation device and UI language → design-system inventory → plan) and then builds one flow at a time, pausing for a short review after each. Anything it can't know goes into `NEEDED-INFO.md` while it keeps building. When all flows are approved it offers a hand-off cleanup.

Requirements: Figma MCP connected; Mobbin MCP (optional, paid plan) — the skill adds it with `claude mcp add mobbin --scope user --transport http https://api.mobbin.com/mcp`.

The project memory lives in `.ux-prototype/` in your product repo — commit it so teammates can continue each other's work.
