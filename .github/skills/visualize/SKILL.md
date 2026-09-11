---
name: visualize
description: Create and debug Excalidraw drawings in .excalidraw files, including valid element schemas, composition, and VS Code rendering troubleshooting.
---

# Drawing Effective Excalidraw Files

Use this skill when the user asks to draw, sketch, diagram, visualize, or edit an
`.excalidraw` file. Prefer a simple, readable composition with a small number of
well-formed elements over a large amount of decorative detail.

## File Structure

Keep the standard Excalidraw document envelope:

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://marketplace.visualstudio.com/items?itemName=pomdtr.excalidraw-editor",
  "elements": [],
  "appState": {
    "gridSize": 20,
    "gridStep": 5,
    "gridModeEnabled": false,
    "viewBackgroundColor": "#ffffff"
  },
  "files": {}
}
```

## Element Validity

Every element must include the fields expected by the Excalidraw schema. At
minimum, generate these shared fields:

```json
{
  "id": "unique-id",
  "type": "rectangle",
  "x": 100,
  "y": 100,
  "width": 120,
  "height": 80,
  "angle": 0,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "transparent",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 1,
  "opacity": 100,
  "groupIds": [],
  "frameId": null,
  "roundness": null,
  "boundElements": null,
  "updated": 1700000000000,
  "link": null,
  "locked": false,
  "isDeleted": false,
  "seed": 100001,
  "version": 1,
  "versionNonce": 100002,
  "index": "a0"
}
```

Use unique `id`, `seed`, `versionNonce`, and `index` values for each element.
`opacity` uses the 0–100 range, not 0–1. Use `index` values such as `a0`,
`a1`, and `a2` to preserve z-order.

Supported basic element types include `rectangle`, `ellipse`, `diamond`,
`line`, `arrow`, `freedraw`, `text`, `image`, and `frame`. Do not emit
`triangle`; represent a triangle as a closed `line` with points.

For line-like elements, include the line-specific fields:

```json
{
  "points": [[0, 0], [40, -30], [80, 0]],
  "lastCommittedPoint": null,
  "startBinding": null,
  "endBinding": null,
  "startArrowhead": null,
  "endArrowhead": null
}
```

For text elements, include `text`, `originalText`, `fontSize`, `fontFamily`,
`textAlign`, `verticalAlign`, `containerId`, `lineHeight`, and `autoResize`.
Use `fontFamily: 1` for the normal hand-drawn font.

## Composition Guidelines

- Establish a clear canvas area and keep related elements near each other.
- Put background elements first so they render behind the subject.
- Use consistent stroke widths and a restrained color palette.
- Use closed line paths for custom shapes and set `backgroundColor` when the
  shape should be filled.
- Give labels enough width and height to avoid clipping.
- Keep the drawing within a predictable area, such as roughly `x: 60–600` and
  `y: 60–450`, unless the user requests a different scale.

## Validation and VS Code Troubleshooting

After writing a drawing, validate that it is strict JSON and that every element
has the required shared fields. If possible, load the file in an Excalidraw
renderer or preview to verify that the element count is non-zero and the canvas
shows the intended content.

If VS Code shows a blank canvas after the file was updated, close the existing
Excalidraw editor tab and reopen the file. The editor can retain stale
in-memory state and overwrite a corrected file with its previous empty state.

## Learnings

- Excalidraw may silently discard elements with incomplete schemas. A file can
  be valid JSON yet render as an empty canvas if fields such as `angle`,
  `frameId`, `index`, or text metadata are missing.
- `type: "triangle"` is not a portable Excalidraw element type. Use a closed
  `line` path instead.
- A browser preview is useful for separating invalid element data from a stale
  VS Code editor tab: if the preview renders but VS Code is blank, reopen the
  editor tab before rewriting the file.