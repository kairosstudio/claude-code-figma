---
name: figma-designer
description: Design and modify Figma files using the Figma Plugin API via browser automation. Use when the user wants to create shapes, edit layouts, modify text, manage components, or extract design info from an open Figma file.
allowed-tools: Read, Grep, Glob, navigate_page, evaluate_script, take_screenshot, take_snapshot
---

# Figma Designer Skill

Control Figma programmatically through the browser using the Figma Plugin API.

## Setup Flow

1. Use `navigate_page` to go to [Figma](https://www.figma.com/). Prompt the user to log in if needed and open a design file.
2. Verify API access:
   ```javascript
   () => typeof figma !== 'undefined' ? 'ready' : 'not available'
   ```
3. If `figma` is not defined, see [Troubleshooting](#troubleshooting).
4. Once confirmed, execute the user's requests via `evaluate_script`.

## Rules of Engagement

- Always explain in plain English what you are about to do before running code. Assume the user cannot read code.
- Use `take_screenshot` or `take_snapshot` to verify your work visually after making changes.
- Do not use the REST API or manually interact with the Figma UI — only use `evaluate_script` with the Plugin API.
- When building complex designs, work incrementally: create one element, verify, then continue.

---

## Figma Plugin API Reference

All code runs via `evaluate_script`. **Top-level `await` is NOT supported** — always wrap async code:

```javascript
(async () => {
  // your code here
})()
```

### Creating Nodes

```javascript
// Frame (container)
const frame = figma.createFrame();
frame.name = 'My Frame';
frame.resize(800, 600);
frame.x = 0; frame.y = 0;

// Rectangle
const rect = figma.createRectangle();
rect.resize(200, 100);
rect.cornerRadius = 8;

// Ellipse
const ellipse = figma.createEllipse();
ellipse.resize(100, 100);

// Line
const line = figma.createLine();
line.resize(400, 0);

// Text (MUST load font first!)
const text = figma.createText();
await figma.loadFontAsync({ family: 'Inter', style: 'Regular' });
text.characters = 'Hello World';

// Component
const comp = figma.createComponent();
comp.name = 'Button';
comp.resize(120, 40);

// Component instance
const instance = comp.createInstance();
instance.x = 200;
```

### Fills (Colors & Backgrounds)

Colors use `{r, g, b}` with values **0–1** (not 0–255). Divide by 255.

```javascript
// Solid fill
node.fills = [{ type: 'SOLID', color: { r: 0.2, g: 0.4, b: 1 } }];

// Solid fill with opacity
node.fills = [{ type: 'SOLID', color: { r: 1, g: 0, b: 0 }, opacity: 0.5 }];

// No fill (transparent)
node.fills = [];

// Linear gradient
node.fills = [{
  type: 'GRADIENT_LINEAR',
  gradientStops: [
    { color: { r: 1, g: 0, b: 0, a: 1 }, position: 0 },
    { color: { r: 0, g: 0, b: 1, a: 1 }, position: 1 }
  ],
  gradientTransform: [[1, 0, 0], [0, 1, 0]]
}];
```

### Strokes

```javascript
node.strokes = [{ type: 'SOLID', color: { r: 0, g: 0, b: 0 } }];
node.strokeWeight = 2;
node.strokeAlign = 'INSIDE'; // 'INSIDE' | 'OUTSIDE' | 'CENTER'
node.dashPattern = [10, 5]; // dashed line
```

**GOTCHA:** Stroke paints do NOT accept an `a` (alpha) property on the color. Use `node.opacity` instead for transparency.
**GOTCHA:** `strokesTop`, `strokesBottom` etc. are NOT valid properties. To create a single-side border, use a 1px rectangle as a divider.

### Effects (Shadows, Blurs)

```javascript
// Drop shadow
node.effects = [{
  type: 'DROP_SHADOW',
  color: { r: 0, g: 0, b: 0, a: 0.25 },
  offset: { x: 0, y: 4 },
  radius: 8,
  spread: 0,
  visible: true
}];

// Inner shadow
node.effects = [{
  type: 'INNER_SHADOW',
  color: { r: 0, g: 0, b: 0, a: 0.1 },
  offset: { x: 0, y: 2 },
  radius: 4,
  spread: 0,
  visible: true
}];

// Layer blur
node.effects = [{
  type: 'LAYER_BLUR',
  radius: 10,
  visible: true
}];
```

### Text

**CRITICAL: You MUST call `figma.loadFontAsync()` before creating or modifying ANY text.** This applies every time — not just once.

```javascript
(async () => {
  const text = figma.createText();

  // Load font BEFORE setting characters or text properties
  await figma.loadFontAsync({ family: 'Inter', style: 'Regular' });
  text.characters = 'Hello';
  text.fontSize = 24;
  text.fontName = { family: 'Inter', style: 'Regular' };

  // Text alignment
  text.textAlignHorizontal = 'CENTER'; // 'LEFT' | 'CENTER' | 'RIGHT' | 'JUSTIFIED'
  text.textAlignVertical = 'CENTER';   // 'TOP' | 'CENTER' | 'BOTTOM'

  // Line height
  text.lineHeight = { value: 28, unit: 'PIXELS' };
  // or: { value: 150, unit: 'PERCENT' }
  // or: { unit: 'AUTO' }

  // Letter spacing
  text.letterSpacing = { value: 1.5, unit: 'PIXELS' };

  // Auto-resize behavior
  text.textAutoResize = 'WIDTH_AND_HEIGHT'; // 'NONE' | 'HEIGHT' | 'WIDTH_AND_HEIGHT' | 'TRUNCATE'

  // Text color (fills, does NOT require font loading)
  text.fills = [{ type: 'SOLID', color: { r: 0.2, g: 0.2, b: 0.2 } }];

  // Bold text — load the bold style
  await figma.loadFontAsync({ family: 'Inter', style: 'Bold' });
  text.fontName = { family: 'Inter', style: 'Bold' };

  // Mixed styles on ranges
  await figma.loadFontAsync({ family: 'Inter', style: 'Bold' });
  text.setRangeFontSize(0, 5, 32);
  text.setRangeFontName(0, 5, { family: 'Inter', style: 'Bold' });
  text.setRangeFills(0, 5, [{ type: 'SOLID', color: { r: 1, g: 0, b: 0 } }]);
})()
```

**Font loading for existing text with multiple fonts:**
```javascript
(async () => {
  const text = figma.currentPage.findOne(n => n.type === 'TEXT' && n.name === 'MyText');
  // Load ALL fonts used in the text node
  await Promise.all(
    text.getRangeAllFontNames(0, text.characters.length)
      .map(figma.loadFontAsync)
  );
  text.characters = 'Updated text';
})()
```

### Positioning & Sizing

```javascript
node.x = 100;        // horizontal position
node.y = 200;        // vertical position
node.resize(300, 150); // width, height
node.rotation = 45;  // degrees
node.opacity = 0.8;  // 0–1
node.visible = true;
node.locked = false;
node.name = 'Layer Name';
```

### Auto-Layout (Frames)

```javascript
const frame = figma.createFrame();
frame.layoutMode = 'VERTICAL';    // 'NONE' | 'HORIZONTAL' | 'VERTICAL'
frame.primaryAxisAlignItems = 'CENTER';   // 'MIN' | 'CENTER' | 'MAX' | 'SPACE_BETWEEN'
frame.counterAxisAlignItems = 'CENTER';   // 'MIN' | 'CENTER' | 'MAX' | 'BASELINE'
frame.itemSpacing = 16;
frame.paddingTop = 20;
frame.paddingBottom = 20;
frame.paddingLeft = 20;
frame.paddingRight = 20;
frame.clipsContent = true;

// Sizing behavior
frame.layoutSizingHorizontal = 'FIXED'; // 'FIXED' | 'HUG' | 'FILL'
frame.layoutSizingVertical = 'HUG';
```

**Use `layoutMode = 'NONE'` for absolute positioning** of children within a frame.

### Hierarchy & Selection

```javascript
// Add child to frame
frame.appendChild(rect);

// Current page
const page = figma.currentPage;

// Selection
const selected = figma.currentPage.selection;
figma.currentPage.selection = [node1, node2];

// Find nodes
const allText = page.findAll(n => n.type === 'TEXT');
const named = page.findOne(n => n.name === 'Header');
const children = frame.children; // direct children

// Remove node
node.remove();

// Viewport
figma.viewport.scrollAndZoomIntoView([frame]);
```

### Groups & Components

```javascript
// Group nodes
const group = figma.group([rect1, rect2], figma.currentPage);
group.name = 'My Group';

// Component with instances
const comp = figma.createComponent();
comp.name = 'Card';
comp.resize(300, 200);
comp.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];

// Create instance and override text
const inst = comp.createInstance();
const label = inst.findOne(n => n.name === 'Label' && n.type === 'TEXT');
if (label) {
  await figma.loadFontAsync(label.fontName);
  label.characters = 'New Label';
}
```

**Component instance rules:**
- You CANNOT override child positions (x, y) in instances
- You CAN override: text content (`.characters`), fills, visibility, opacity

### Images

```javascript
(async () => {
  // From URL (fetch must be available)
  const resp = await fetch('http://example.com/image.png');
  const buf = await resp.arrayBuffer();
  const img = figma.createImage(new Uint8Array(buf));
  const rect = figma.createRectangle();
  rect.resize(400, 300);
  rect.fills = [{ type: 'IMAGE', imageHash: img.hash, scaleMode: 'FILL' }];
  // scaleMode: 'FILL' | 'FIT' | 'CROP' | 'TILE'
})()
```

### Notifications

```javascript
figma.notify('Done!');               // brief notification
figma.notify('Error', { error: true }); // error style
```

---

## Common Gotchas Quick Reference

| Mistake | Fix |
|---|---|
| Top-level `await` | Wrap in `(async () => { ... })()` |
| Text without loading font | Always `await figma.loadFontAsync()` before text ops |
| Colors as 0–255 | Divide by 255: `{r: 66/255, g: 133/255, b: 244/255}` |
| `stroke.color = {r,g,b,a}` | Strokes don't accept `a` — use `node.opacity` |
| `node.strokesTop = ...` | Not a real property — use a 1px rectangle divider |
| `layoutMode = 'NONE'` missing | Required for absolute positioning of children |
| Setting `child.x` on instance | Not allowed — bake positions into the component |
| `el.className` on SVG elements | Use `el.getAttribute('class') \|\| ''` |
| `node.width = 100` | Use `node.resize(100, node.height)` instead |
| Font not found | Check `text.hasMissingFont` before modifying |

---

## Troubleshooting

If `figma is not defined`:
1. Ensure the user has **edit permissions** on the file. If not, suggest creating a branch.
2. If still unavailable, instruct the user to **manually open any plugin** in Figma and close it, then try again. There's a known bug where the `figma` global isn't available until a plugin has been opened at least once.
3. Verify the user has a **design file** open (not just the Figma home/dashboard).

## Additional Documentation

Full Figma Plugin API reference: [developers.figma.com/docs/plugins/api/](https://developers.figma.com/docs/plugins/api/api-reference/)
