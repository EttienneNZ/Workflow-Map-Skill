# workflow-map

A Claude Code [skill](https://docs.claude.com/en/docs/claude-code/skills) that generates an **interactive, single-page HTML workflow map** for a codebase. Click a flow on the left, watch the path light up across the canvas, hover edges and step numbers for payload-level detail, and read a numbered trace in the right-hand panel.

The output is two files — `workflows.json` (data) and `workflows.html` (renderer) — that you can drop into any documentation site or static host.

## What it produces

- **Nodes** for every file/module/external actor, grouped by package with a fixed column layout so the diagram stays stable across flows.
- **Flows** as ordered sequences of edges. Each step records its `from`, `to`, a short action label, the payload that is passed, and the file responsible.
- **Interactive renderer** with synchronized hover highlighting — hovering an edge, its step number, or the matching row in the right panel highlights the path, both endpoint boxes, and the row simultaneously.



## Installation

### From the `.skill` bundle (recommended for teammates)

```bash
unzip workflow-map.skill -d ~/.claude/skills/
```

Restart Claude Code. The skill auto-discovers and appears in your available skills list.

### From this repo

```bash
git clone <repo-url> team-skills
ln -s "$(pwd)/team-skills/workflow-map" ~/.claude/skills/workflow-map
```

Updates pulled with `git pull`. No restart needed for content edits, but restart Claude Code if SKILL.md frontmatter changes.

### Manual

Copy the directory to `~/.claude/skills/workflow-map/` and ensure the structure matches the [Repository layout](#repository-layout) below.

## Usage

Once installed, the skill triggers automatically when you ask Claude to map or visualize workflows. Phrasings that work:

- *"Map the workflows in this codebase"*
- *"Create an interactive flow diagram for this service"*
- *"Visualize how the packages talk to each other"*
- *"Build a clickable architecture map"*
- *"Document the request flow from entry to response"*

You can also invoke it explicitly: ask Claude to use the `workflow-map` skill.

### What Claude will do

1. Confirm scope — which flows to cover, where to write output, what granularity (file-level vs. function-level).
2. Explore the codebase, tracing actual call chains from each entry point through routers, handlers, service clients, and back out to external actors.
3. Write `workflows.json` with groups, nodes (fixed column coordinates), and flows.
4. Copy the renderer template into `workflows.html`.
5. Run the validator to confirm every flow step references an existing node.
6. Tell you how to view the result locally.

### Viewing the output

Browsers block `fetch` on `file://`, so serve over HTTP:

```bash
cd docs   # or wherever Claude wrote the files
python3 -m http.server 8000
```

Open `http://localhost:8000/workflows.html`.

## How the renderer works

- **Sidebar**: flows grouped by `category`. Categories appear in the order they first show up in `flows[]` — no hardcoded list, so any project-specific category name works.
- **Canvas**: SVG with fixed-position nodes in package columns left-to-right, edges drawn as cubic bezier curves with arrow markers and numbered badges.
- **Hover state**: mouseover an edge, its step badge, or the right-panel step row → all three plus both endpoint nodes recolor in unison. Arrowhead, step number, and node outlines all switch color together.
- **Z-order**: hovering a node lifts it above edges so its label stays readable even when paths cross the box.
- **Multi-step pairs**: when several steps run between the same two nodes (e.g. a poll loop), badges spread along the curve so numbers don't stack on a single midpoint.

## Output schema

See [`references/json-schema.md`](references/json-schema.md) for the full reference. Quick view:

```json
{
  "meta":   { "title": "...", "canvasWidth": 1500, "canvasHeight": 1360 },
  "groups": [ { "id": "handler", "label": "Handlers", "color": "#22c55e" } ],
  "nodes":  [ { "id": "h_connect", "label": "connect", "group": "handler",
                "x": 810, "y": 40, "file": "src/handlers/connect/index.ts" } ],
  "flows":  [ { "id": "connect", "name": "OAuth Connect", "category": "Slash Command",
                "steps": [
                  { "from": "user", "to": "slack", "label": "type /connect",
                    "payload": "slash command", "file": "" }
                ] } ]
}
```

## Extending an existing map

You can hand-edit `workflows.json` after generation — the renderer rereads everything on load. No HTML changes are needed for:

- Adding a flow → append to `flows[]` with an ordered `steps[]` array.
- Adding a node → append to `nodes[]` with an unused `(x, y)` slot.
- Adding a group → append to `groups[]` with an id/label/color.

Run `python3 scripts/validate.py path/to/workflows.json` after edits to catch broken references before you serve the page.

## Repository layout

```
workflow-map/
├── SKILL.md                  # Skill instructions (Claude reads this)
├── README.md                 # This file
├── assets/
│   └── template.html         # Renderer — copied verbatim into output dir
├── scripts/
│   └── validate.py           # JSON validator (checks refs, duplicates)
├── references/
│   └── json-schema.md        # Field-level schema reference
└── evals/
    └── evals.json            # Test prompts for the skill-creator workflow
```

## Validator

```
python3 scripts/validate.py path/to/workflows.json
```

Checks:

- JSON parses.
- Every `node.group` references an existing group.
- Every `step.from` and `step.to` references an existing node.
- No duplicate ids in `nodes`, `groups`, or `flows`.
- Required fields (`x`, `y`, `label`, etc.) are present.

Exits 0 on success, 1 on any failure with a per-issue diagnostic.

## Development

### Updating the renderer

Edit `assets/template.html`. The template is generic — no project-specific strings. To test changes against a real `workflows.json`, copy the updated template into that project's `docs/` directory and refresh the served page.

### Updating the instructions

Edit `SKILL.md`. The YAML frontmatter `description` field controls when Claude triggers the skill, so be specific about contexts. Body content is loaded only when the skill activates.

### Repackaging

If you maintain a `.skill` bundle for distribution, repackage with the skill-creator's packaging script:

```bash
cd path/to/skill-creator
python3 -m scripts.package_skill ~/.claude/skills/workflow-map
```

The resulting `workflow-map.skill` is a standard zip archive ready to share.

### Adding test cases

Append to `evals/evals.json`. Each eval is a realistic prompt + an `expected_output` describing what success looks like. The skill-creator can run these in parallel against your skill to measure quality across iterations.

## When not to use this skill

- You need a static block diagram → use a tool like Mermaid or D2.
- You need a sequence diagram for one flow → Mermaid `sequenceDiagram` is a better fit.
- You need prose-only architecture documentation → use the `document-architecture` skill.
- The codebase has no orchestrated flows — just utility functions with no entry points.

## License

Internal team use. Adjust to your organisation's policy before publishing externally.
