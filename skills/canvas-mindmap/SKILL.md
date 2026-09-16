---
name: canvas-mindmap
description: Create or refine concise Obsidian Canvas mind maps from Markdown notes in this repository. Use for topic-centered association maps, knowledge overviews, or requests to create or clean up a `.canvas` mind map. Do not use for detailed process flowcharts or standalone web visualizations.
---

# Canvas Mind Map

Create a clean topic-centered mind map that helps the reader recall adjacent knowledge quickly. Markdown remains the source of truth; the Canvas is a lightweight navigation layer.

## Before editing

- Read the relevant Markdown notes and inspect an existing Canvas before changing it.
- Preserve user-adjusted positions and unaffected branches when refining an existing map.
- Do not modify Markdown bodies unless the user also asks for content changes.

## Output convention

- Save every Canvas directly under `routemap/`; do not create subdirectories.
- Name the file after its core topic, for example `routemap/数据库选型.canvas`.
- Use full vault-relative WikiLinks for note nodes, for example `[[knowledge/database/mysql|MySQL]]`.

## Map shape

- Put one core topic at the center.
- Prefer 3–5 primary branches and only a few useful leaf concepts per branch.
- Use short keywords or phrases. Keep detailed explanations in Markdown notes.
- Use a balanced radial or left-to-right tree layout with generous spacing.
- Give every branch one consistent Canvas color; keep the center visually distinct.
- Use plain branch lines with `"toEnd": "none"`.
- Avoid group boxes, long text cards, legends, edge labels, and cross-branch edges unless the user explicitly needs them.
- When a source note is sparse, keep the map sparse too. Add only high-level prompts that are supported by the available context.

## Minimal Canvas structure

```json
{
  "nodes": [
    {
      "id": "topic",
      "type": "text",
      "text": "# [[knowledge/topic|核心主题]]",
      "x": 0,
      "y": 0,
      "width": 300,
      "height": 100,
      "color": "5"
    }
  ],
  "edges": []
}
```

Use unique stable node and edge IDs. Connect each leaf through its primary branch rather than directly to the center.

## Validation

- Parse the completed file as JSON.
- Verify that node and edge IDs are unique.
- Verify every edge references existing nodes.
- Verify every WikiLink target exists in the vault.
- Check that node rectangles do not overlap and that tree edges do not unnecessarily cross.
