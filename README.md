# Figma — Testing & Basic Drawing

A reference project for working with the Figma API and MCP tooling to automate testing and create drawings programmatically.

## Prerequisites

- [Figma account](https://figma.com) with a valid Personal Access Token
- [Node.js](https://nodejs.org) 18+ (if running scripts locally)
- Warp with the Figma MCP server configured

## Setup

1. **Clone the repo**
   ```bash
   git clone <repo-url>
   cd figma
   ```

2. **Set your Figma token**
   ```bash
   export FIGMA_TOKEN=your_personal_access_token
   ```
   You can generate a token in Figma under **Settings → Account → Personal access tokens**.

## Basic Figma Drawing

Figma files are organized as a tree of nodes. The most common node types used when drawing:

| Node Type     | Description                              |
|---------------|------------------------------------------|
| `FRAME`       | Container — equivalent to an artboard    |
| `RECTANGLE`   | Basic shape                              |
| `ELLIPSE`     | Circle or oval                           |
| `TEXT`        | Text layer                               |
| `GROUP`       | Non-layout grouping of nodes             |
| `COMPONENT`   | Reusable design component                |

### Drawing via the Figma Plugin API (in-browser)

```js
// Create a rectangle
const rect = figma.createRectangle();
rect.x = 100;
rect.y = 100;
rect.resize(200, 100);
rect.fills = [{ type: 'SOLID', color: { r: 0.2, g: 0.5, b: 1 } }];
figma.currentPage.appendChild(rect);
```

```js
// Create a text node
const text = figma.createText();
await figma.loadFontAsync({ family: 'Inter', style: 'Regular' });
text.characters = 'Hello, Figma!';
text.x = 100;
text.y = 220;
figma.currentPage.appendChild(text);
```

### Drawing via the REST API

The Figma REST API is read-only for node content; use the Plugin API or the Figma MCP `use_figma` tool to write to a canvas.

```bash
# Fetch a file's node tree
curl -H "X-Figma-Token: $FIGMA_TOKEN" \
  "https://api.figma.com/v1/files/{FILE_KEY}"

# Export a specific node as PNG
curl -H "X-Figma-Token: $FIGMA_TOKEN" \
  "https://api.figma.com/v1/images/{FILE_KEY}?ids={NODE_ID}&format=png"
```

## Testing Figma Designs

### Visual regression with Playwright

Use Playwright to export frames and compare them against baselines.

```bash
npx playwright test
```

Example test skeleton:

```ts
import { test, expect } from '@playwright/test';
import fetch from 'node-fetch';

test('hero frame matches snapshot', async () => {
  const res = await fetch(
    `https://api.figma.com/v1/images/${process.env.FILE_KEY}?ids=${process.env.NODE_ID}&format=png`,
    { headers: { 'X-Figma-Token': process.env.FIGMA_TOKEN! } }
  );
  const { images } = await res.json() as any;
  const imageUrl = Object.values(images)[0] as string;

  const img = await fetch(imageUrl);
  const buffer = Buffer.from(await img.arrayBuffer());

  expect(buffer).toMatchSnapshot('hero-frame.png');
});
```

### Checking design tokens / structure

```ts
import { test, expect } from '@playwright/test';
import fetch from 'node-fetch';

test('primary button uses correct fill color', async () => {
  const res = await fetch(
    `https://api.figma.com/v1/files/${process.env.FILE_KEY}/nodes?ids=${process.env.BUTTON_NODE_ID}`,
    { headers: { 'X-Figma-Token': process.env.FIGMA_TOKEN! } }
  );
  const data = await res.json() as any;
  const node = data.nodes[process.env.BUTTON_NODE_ID!].document;
  const fill = node.fills[0].color;

  // Expect brand blue: rgb(0.2, 0.5, 1)
  expect(fill.r).toBeCloseTo(0.2, 1);
  expect(fill.g).toBeCloseTo(0.5, 1);
  expect(fill.b).toBeCloseTo(1.0, 1);
});
```

## Useful Resources

- [Figma REST API docs](https://www.figma.com/developers/api)
- [Figma Plugin API docs](https://www.figma.com/plugin-docs/)
- [Playwright docs](https://playwright.dev)
- [Figma MCP server](https://github.com/figma/figma-developer-mcp)
