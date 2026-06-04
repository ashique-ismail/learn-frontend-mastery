# Critical Rendering Path

## Table of Contents
- [Introduction](#introduction)
- [DOM Construction](#dom-construction)
- [CSSOM Construction](#cssom-construction)
- [Render Tree Construction](#render-tree-construction)
- [Layout (Reflow)](#layout-reflow)
- [Paint](#paint)
- [Composite](#composite)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

The Critical Rendering Path (CRP) is the sequence of steps the browser takes to convert HTML, CSS, and JavaScript into pixels on the screen. Understanding this process is crucial for optimizing web performance and achieving fast page loads.

Key stages:
- **DOM Construction**: Parsing HTML into Document Object Model
- **CSSOM Construction**: Parsing CSS into CSS Object Model  
- **Render Tree**: Combining DOM and CSSOM into renderable tree
- **Layout**: Calculating position and size of elements
- **Paint**: Drawing pixels for visual properties
- **Composite**: Combining layers into final image

## DOM Construction

### HTML Parsing Process

How browsers convert HTML to DOM:

```javascript
// Simplified HTML parsing process
class HTMLParser {
  constructor(html) {
    this.html = html;
    this.position = 0;
    this.tokens = [];
  }

  parse() {
    // Step 1: Tokenization
    this.tokenize();
    
    // Step 2: Tree construction
    return this.buildDOMTree();
  }

  tokenize() {
    // Convert HTML string to tokens
    // <div class="container">Hello</div>
    // → [
    //     { type: 'StartTag', name: 'div', attrs: { class: 'container' } },
    //     { type: 'Character', data: 'Hello' },
    //     { type: 'EndTag', name: 'div' }
    //   ]

    const tokenRegex = /<\/?([a-zA-Z0-9]+)([^>]*)>|([^<]+)/g;
    let match;

    while ((match = tokenRegex.exec(this.html)) !== null) {
      if (match[1]) {
        // Tag token
        const isClosing = this.html[match.index + 1] === '/';
        const attrs = this.parseAttributes(match[2]);

        this.tokens.push({
          type: isClosing ? 'EndTag' : 'StartTag',
          name: match[1],
          attrs: attrs
        });
      } else if (match[3]) {
        // Text token
        this.tokens.push({
          type: 'Character',
          data: match[3].trim()
        });
      }
    }
  }

  parseAttributes(attrString) {
    const attrs = {};
    const attrRegex = /(\w+)="([^"]*)"/g;
    let match;

    while ((match = attrRegex.exec(attrString)) !== null) {
      attrs[match[1]] = match[2];
    }

    return attrs;
  }

  buildDOMTree() {
    // Build tree from tokens
    const root = { type: 'document', children: [] };
    const stack = [root];

    for (const token of this.tokens) {
      const current = stack[stack.length - 1];

      if (token.type === 'StartTag') {
        const element = {
          type: 'element',
          tagName: token.name,
          attributes: token.attrs,
          children: []
        };

        current.children.push(element);
        stack.push(element);
      } else if (token.type === 'EndTag') {
        stack.pop();
      } else if (token.type === 'Character') {
        current.children.push({
          type: 'text',
          content: token.data
        });
      }
    }

    return root;
  }
}

// Example usage
const html = `
  <div class="container">
    <h1 id="title">Welcome</h1>
    <p>Hello <strong>World</strong></p>
  </div>
`;

const parser = new HTMLParser(html);
const dom = parser.parse();
console.log(JSON.stringify(dom, null, 2));
```

### Incremental Parsing

Browsers parse HTML incrementally:

```javascript
// Browser receives HTML in chunks
class IncrementalHTMLParser {
  constructor() {
    this.buffer = '';
    this.dom = { type: 'document', children: [] };
    this.stack = [this.dom];
  }

  // Called as chunks arrive from network
  receiveChunk(chunk) {
    console.log('Received chunk:', chunk.length, 'bytes');
    this.buffer += chunk;

    // Parse complete tokens from buffer
    this.parseBuffer();
  }

  parseBuffer() {
    // Look for complete tags in buffer
    let tagMatch;
    const tagRegex = /<\/?([a-zA-Z0-9]+)([^>]*)>/;

    while ((tagMatch = tagRegex.exec(this.buffer)) !== null) {
      const fullMatch = tagMatch[0];
      const tagName = tagMatch[1];
      const isClosing = fullMatch[1] === '/';

      // Process tag
      if (isClosing) {
        this.handleEndTag(tagName);
      } else {
        this.handleStartTag(tagName, tagMatch[2]);
      }

      // Remove processed tag from buffer
      this.buffer = this.buffer.substring(
        tagMatch.index + fullMatch.length
      );
    }
  }

  handleStartTag(tagName, attrsString) {
    const element = {
      type: 'element',
      tagName: tagName,
      children: []
    };

    const current = this.stack[this.stack.length - 1];
    current.children.push(element);
    this.stack.push(element);

    // Trigger speculative parsing for links
    if (tagName === 'script' || tagName === 'link') {
      this.speculativelyLoadResource(element);
    }
  }

  handleEndTag(tagName) {
    this.stack.pop();
  }

  speculativelyLoadResource(element) {
    // Browser starts loading scripts/stylesheets
    // before fully parsing the HTML
    console.log('Speculative resource load:', element.tagName);
  }

  getDOM() {
    return this.dom;
  }
}

// Simulating incremental loading
const parser = new IncrementalHTMLParser();

// Chunk 1
parser.receiveChunk('<html><head>');
console.log('DOM after chunk 1:', parser.getDOM());

// Chunk 2
parser.receiveChunk('<title>Page</title></head>');
console.log('DOM after chunk 2:', parser.getDOM());

// Chunk 3
parser.receiveChunk('<body><div>Content</div></body></html>');
console.log('Final DOM:', parser.getDOM());
```

### Parser-Blocking Resources

Script tags block HTML parsing:

```html
<!DOCTYPE html>
<html>
<head>
  <!-- Non-blocking: async attribute -->
  <script src="analytics.js" async></script>

  <!-- Non-blocking: defer attribute -->
  <script src="app.js" defer></script>

  <!-- BLOCKING: Regular script (stops parser) -->
  <script src="critical.js"></script>
  
  <!-- HTML parsing blocked here until critical.js loads and executes -->
  
  <style>
    /* BLOCKING: External stylesheet (stops rendering) */
  </style>
</head>
<body>
  <h1>Hello World</h1>

  <!-- BLOCKING: Inline script (stops parser) -->
  <script>
    console.log('Inline script blocks parsing');
  </script>

  <!-- Parser resumes here -->
  <p>Content after script</p>

  <!-- Non-blocking: Script at end of body -->
  <script src="non-critical.js"></script>
</body>
</html>
```

```javascript
// Visualizing parser blocking
class ParserBlockingDemo {
  simulateParsingWithBlocking() {
    const timeline = [];

    timeline.push({
      time: 0,
      event: 'Start parsing HTML'
    });

    timeline.push({
      time: 10,
      event: 'Encounter <script src="app.js">'
    });

    timeline.push({
      time: 10,
      event: 'PAUSE HTML parsing'
    });

    timeline.push({
      time: 10,
      event: 'Download app.js (100ms)'
    });

    timeline.push({
      time: 110,
      event: 'Execute app.js (50ms)'
    });

    timeline.push({
      time: 160,
      event: 'RESUME HTML parsing'
    });

    timeline.push({
      time: 200,
      event: 'Parsing complete'
    });

    return timeline;
  }

  simulateParsingWithAsync() {
    const timeline = [];

    timeline.push({
      time: 0,
      event: 'Start parsing HTML'
    });

    timeline.push({
      time: 10,
      event: 'Encounter <script src="app.js" async>'
    });

    timeline.push({
      time: 10,
      event: 'Start downloading app.js (in parallel)'
    });

    timeline.push({
      time: 10,
      event: 'Continue HTML parsing'
    });

    timeline.push({
      time: 100,
      event: 'Parsing complete'
    });

    timeline.push({
      time: 110,
      event: 'app.js downloaded'
    });

    timeline.push({
      time: 110,
      event: 'PAUSE parsing to execute app.js'
    });

    timeline.push({
      time: 160,
      event: 'app.js execution complete'
    });

    return timeline;
  }
}
```

## CSSOM Construction

### CSS Parsing

Converting CSS to CSSOM:

```javascript
// Simplified CSS parser
class CSSParser {
  parse(css) {
    // Parse CSS into CSSOM
    const rules = [];
    const ruleRegex = /([^{]+)\{([^}]+)\}/g;
    let match;

    while ((match = ruleRegex.exec(css)) !== null) {
      const selector = match[1].trim();
      const declarations = this.parseDeclarations(match[2]);

      rules.push({
        selector: selector,
        specificity: this.calculateSpecificity(selector),
        declarations: declarations
      });
    }

    return { rules };
  }

  parseDeclarations(declarationsString) {
    const declarations = {};
    const declRegex = /([^:]+):([^;]+)/g;
    let match;

    while ((match = declRegex.exec(declarationsString)) !== null) {
      const property = match[1].trim();
      const value = match[2].trim();
      declarations[property] = value;
    }

    return declarations;
  }

  calculateSpecificity(selector) {
    // Simplified specificity calculation
    // Format: [inline, ids, classes/attrs, elements]
    let ids = (selector.match(/#/g) || []).length;
    let classes = (selector.match(/\./g) || []).length;
    let elements = (selector.match(/[a-z]/g) || []).length;

    return [0, ids, classes, elements];
  }
}

// Example
const css = `
  body { margin: 0; padding: 0; }
  .container { width: 100%; max-width: 1200px; }
  #header { background: blue; color: white; }
  .nav li a { text-decoration: none; color: inherit; }
`;

const parser = new CSSParser();
const cssom = parser.parse(css);
console.log(JSON.stringify(cssom, null, 2));
```

### CSSOM Structure

The CSSOM tree structure:

```javascript
// CSSOM representation
const cssom = {
  type: 'stylesheet',
  rules: [
    {
      selector: 'body',
      specificity: [0, 0, 0, 1],
      declarations: {
        margin: '0',
        padding: '0',
        'font-family': 'Arial, sans-serif'
      }
    },
    {
      selector: '.container',
      specificity: [0, 0, 1, 0],
      declarations: {
        width: '100%',
        'max-width': '1200px',
        margin: '0 auto'
      }
    },
    {
      selector: '.container .header',
      specificity: [0, 0, 2, 0],
      declarations: {
        background: '#333',
        color: 'white',
        padding: '20px'
      }
    }
  ]
};

// Computing final styles for an element
class StyleComputer {
  computeStyles(element, cssom) {
    const matchingRules = this.findMatchingRules(element, cssom);
    
    // Sort by specificity
    matchingRules.sort((a, b) => 
      this.compareSpecificity(a.specificity, b.specificity)
    );

    // Cascade: later rules override earlier ones
    const computedStyles = {};
    for (const rule of matchingRules) {
      Object.assign(computedStyles, rule.declarations);
    }

    // Inherit from parent
    const inheritedStyles = this.getInheritedStyles(element.parent);
    const finalStyles = { ...inheritedStyles, ...computedStyles };

    return finalStyles;
  }

  findMatchingRules(element, cssom) {
    return cssom.rules.filter(rule => 
      this.matchesSelector(element, rule.selector)
    );
  }

  matchesSelector(element, selector) {
    // Simplified selector matching
    if (selector.startsWith('#')) {
      return element.id === selector.substring(1);
    }
    if (selector.startsWith('.')) {
      return element.classList.includes(selector.substring(1));
    }
    return element.tagName === selector;
  }

  compareSpecificity(a, b) {
    for (let i = 0; i < 4; i++) {
      if (a[i] !== b[i]) {
        return a[i] - b[i];
      }
    }
    return 0;
  }

  getInheritedStyles(parent) {
    // Certain properties inherit from parent
    const inheritableProps = [
      'color',
      'font-family',
      'font-size',
      'line-height'
    ];

    const inherited = {};
    if (parent) {
      for (const prop of inheritableProps) {
        if (parent.computedStyles[prop]) {
          inherited[prop] = parent.computedStyles[prop];
        }
      }
    }

    return inherited;
  }
}
```

### Render-Blocking CSS

CSS blocks rendering:

```html
<!DOCTYPE html>
<html>
<head>
  <!-- BLOCKING: External stylesheet -->
  <!-- Rendering blocked until this loads -->
  <link rel="stylesheet" href="styles.css">

  <!-- BLOCKING: Critical inline styles -->
  <style>
    .above-fold { /* Critical styles for visible content */ }
  </style>

  <!-- Non-blocking: Media query doesn't apply initially -->
  <link rel="stylesheet" href="print.css" media="print">

  <!-- Non-blocking: Preload with low priority -->
  <link rel="preload" href="deferred.css" as="style" 
        onload="this.onload=null;this.rel='stylesheet'">
</head>
<body>
  <!-- Content here waits for styles.css to load -->
  <div class="above-fold">Visible content</div>
</body>
</html>
```

```javascript
// Critical CSS extraction strategy
class CriticalCSSExtractor {
  extractCriticalCSS(html, css, viewportHeight = 1080) {
    // 1. Parse HTML
    const dom = this.parseHTML(html);

    // 2. Find above-the-fold elements
    const aboveFoldElements = this.findAboveFold(dom, viewportHeight);

    // 3. Extract only CSS rules for these elements
    const cssom = this.parseCSS(css);
    const criticalRules = this.filterCriticalRules(
      cssom,
      aboveFoldElements
    );

    // 4. Generate critical CSS
    return this.generateCSS(criticalRules);
  }

  findAboveFold(dom, viewportHeight) {
    // Find elements visible in initial viewport
    const aboveFold = [];
    
    function traverse(node, currentY = 0) {
      if (currentY > viewportHeight) return;

      if (node.type === 'element') {
        aboveFold.push(node);
        
        for (const child of node.children || []) {
          traverse(child, currentY + (node.height || 0));
        }
      }
    }

    traverse(dom);
    return aboveFold;
  }

  filterCriticalRules(cssom, elements) {
    return cssom.rules.filter(rule => {
      return elements.some(el => 
        this.matchesSelector(el, rule.selector)
      );
    });
  }

  generateCSS(rules) {
    return rules.map(rule => {
      const declarations = Object.entries(rule.declarations)
        .map(([prop, value]) => `  ${prop}: ${value};`)
        .join('\n');

      return `${rule.selector} {\n${declarations}\n}`;
    }).join('\n\n');
  }
}
```

## Render Tree Construction

### Combining DOM and CSSOM

Creating the render tree:

```javascript
// Render tree builder
class RenderTreeBuilder {
  build(dom, cssom) {
    // Start from root
    return this.buildRenderNode(dom, cssom);
  }

  buildRenderNode(domNode, cssom, parent = null) {
    // Skip text nodes in this simplified version
    if (domNode.type === 'text') {
      return {
        type: 'text',
        content: domNode.content,
        computedStyles: parent?.computedStyles || {}
      };
    }

    // Skip document node
    if (domNode.type === 'document') {
      return {
        type: 'root',
        children: domNode.children.map(child =>
          this.buildRenderNode(child, cssom, null)
        )
      };
    }

    // Compute styles for this element
    const computedStyles = this.computeStyles(domNode, cssom);

    // Skip if display: none
    if (computedStyles.display === 'none') {
      return null;
    }

    // Build render node
    const renderNode = {
      type: 'render-object',
      tagName: domNode.tagName,
      computedStyles: computedStyles,
      children: []
    };

    // Process children
    if (domNode.children) {
      for (const child of domNode.children) {
        const childRenderNode = this.buildRenderNode(
          child,
          cssom,
          renderNode
        );

        if (childRenderNode) {
          renderNode.children.push(childRenderNode);
        }
      }
    }

    return renderNode;
  }

  computeStyles(domNode, cssom) {
    const styleComputer = new StyleComputer();
    return styleComputer.computeStyles(domNode, cssom);
  }
}

// Example
const dom = {
  type: 'document',
  children: [
    {
      type: 'element',
      tagName: 'body',
      children: [
        {
          type: 'element',
          tagName: 'div',
          classList: ['container'],
          children: [
            {
              type: 'element',
              tagName: 'h1',
              children: [
                { type: 'text', content: 'Hello' }
              ]
            },
            {
              type: 'element',
              tagName: 'p',
              computedStyles: { display: 'none' }, // Will be skipped
              children: [
                { type: 'text', content: 'Hidden' }
              ]
            }
          ]
        }
      ]
    }
  ]
};

const cssom = {
  rules: [
    {
      selector: 'body',
      declarations: { margin: '0', padding: '0' }
    },
    {
      selector: '.container',
      declarations: { width: '100%' }
    }
  ]
};

const builder = new RenderTreeBuilder();
const renderTree = builder.build(dom, cssom);
console.log(JSON.stringify(renderTree, null, 2));
```

### Render Objects

Different types of render objects:

```javascript
// Render object types
class RenderObject {
  constructor(type, domNode, styles) {
    this.type = type;
    this.domNode = domNode;
    this.styles = styles;
    this.children = [];
    this.parent = null;
  }
}

class RenderBlock extends RenderObject {
  constructor(domNode, styles) {
    super('block', domNode, styles);
    this.x = 0;
    this.y = 0;
    this.width = 0;
    this.height = 0;
  }
}

class RenderInline extends RenderObject {
  constructor(domNode, styles) {
    super('inline', domNode, styles);
  }
}

class RenderText extends RenderObject {
  constructor(text, styles) {
    super('text', null, styles);
    this.text = text;
  }
}

class RenderImage extends RenderObject {
  constructor(domNode, styles) {
    super('image', domNode, styles);
    this.src = domNode.getAttribute('src');
    this.intrinsicWidth = 0;
    this.intrinsicHeight = 0;
  }

  async loadImage() {
    // Simulate loading image to get dimensions
    return new Promise(resolve => {
      const img = new Image();
      img.onload = () => {
        this.intrinsicWidth = img.width;
        this.intrinsicHeight = img.height;
        resolve();
      };
      img.src = this.src;
    });
  }
}

// Creating render objects based on DOM nodes
class RenderObjectFactory {
  create(domNode, styles) {
    const display = styles.display || 'block';

    switch (domNode.tagName) {
      case 'img':
        return new RenderImage(domNode, styles);
      
      default:
        if (display === 'inline' || display === 'inline-block') {
          return new RenderInline(domNode, styles);
        }
        return new RenderBlock(domNode, styles);
    }
  }
}
```

## Layout (Reflow)

### Layout Calculation

Computing positions and sizes:

```javascript
// Layout engine
class LayoutEngine {
  layout(renderTree, viewportWidth, viewportHeight) {
    // Start layout from root
    this.layoutNode(
      renderTree,
      0,       // x
      0,       // y
      viewportWidth,
      viewportHeight
    );
  }

  layoutNode(node, x, y, availableWidth, availableHeight) {
    if (!node) return { width: 0, height: 0 };

    const styles = node.computedStyles || {};

    // Calculate dimensions
    const width = this.calculateWidth(node, availableWidth, styles);
    const height = this.calculateHeight(node, availableHeight, styles);

    // Set position
    node.x = x;
    node.y = y;
    node.width = width;
    node.height = height;

    // Layout children
    if (node.children && node.children.length > 0) {
      this.layoutChildren(node, styles);
    }

    return { width, height };
  }

  calculateWidth(node, availableWidth, styles) {
    // Width calculation priority:
    // 1. Explicit width
    // 2. Max-width constraint
    // 3. Min-width constraint
    // 4. Available width

    let width = availableWidth;

    if (styles.width) {
      width = this.parseLength(styles.width, availableWidth);
    }

    if (styles['max-width']) {
      const maxWidth = this.parseLength(styles['max-width'], availableWidth);
      width = Math.min(width, maxWidth);
    }

    if (styles['min-width']) {
      const minWidth = this.parseLength(styles['min-width'], availableWidth);
      width = Math.max(width, minWidth);
    }

    // Subtract padding and border
    const padding = this.calculatePadding(styles);
    const border = this.calculateBorder(styles);
    width -= (padding.left + padding.right + border.left + border.right);

    return width;
  }

  calculateHeight(node, availableHeight, styles) {
    if (styles.height) {
      return this.parseLength(styles.height, availableHeight);
    }

    // Auto height: sum of children
    let height = 0;
    for (const child of node.children || []) {
      height += child.height || 0;
    }

    // Add padding and border
    const padding = this.calculatePadding(styles);
    const border = this.calculateBorder(styles);
    height += (padding.top + padding.bottom + border.top + border.bottom);

    return height;
  }

  layoutChildren(parent, styles) {
    const display = styles.display || 'block';

    if (display === 'flex') {
      this.layoutFlex(parent, styles);
    } else if (display === 'grid') {
      this.layoutGrid(parent, styles);
    } else {
      this.layoutBlock(parent, styles);
    }
  }

  layoutBlock(parent, styles) {
    let currentY = parent.y;

    for (const child of parent.children) {
      this.layoutNode(
        child,
        parent.x,
        currentY,
        parent.width,
        parent.height
      );

      currentY += child.height;
    }
  }

  layoutFlex(parent, styles) {
    const flexDirection = styles['flex-direction'] || 'row';
    const justifyContent = styles['justify-content'] || 'flex-start';

    if (flexDirection === 'row') {
      this.layoutFlexRow(parent, justifyContent);
    } else {
      this.layoutFlexColumn(parent, justifyContent);
    }
  }

  layoutFlexRow(parent, justifyContent) {
    // Calculate total width of children
    let totalWidth = 0;
    for (const child of parent.children) {
      totalWidth += child.width || 0;
    }

    // Calculate starting position based on justify-content
    let currentX = parent.x;
    const remainingSpace = parent.width - totalWidth;

    if (justifyContent === 'center') {
      currentX += remainingSpace / 2;
    } else if (justifyContent === 'flex-end') {
      currentX += remainingSpace;
    } else if (justifyContent === 'space-between') {
      const gap = remainingSpace / (parent.children.length - 1);
      
      for (let i = 0; i < parent.children.length; i++) {
        const child = parent.children[i];
        this.layoutNode(child, currentX, parent.y, child.width, parent.height);
        currentX += child.width + gap;
      }
      return;
    }

    // Layout children
    for (const child of parent.children) {
      this.layoutNode(child, currentX, parent.y, child.width, parent.height);
      currentX += child.width;
    }
  }

  layoutFlexColumn(parent, justifyContent) {
    // Similar to layoutFlexRow but vertically
    let currentY = parent.y;

    for (const child of parent.children) {
      this.layoutNode(child, parent.x, currentY, parent.width, child.height);
      currentY += child.height;
    }
  }

  parseLength(value, relativeTo) {
    if (value.endsWith('%')) {
      const percentage = parseFloat(value) / 100;
      return relativeTo * percentage;
    }
    if (value.endsWith('px')) {
      return parseFloat(value);
    }
    return parseFloat(value);
  }

  calculatePadding(styles) {
    return {
      top: this.parseLength(styles['padding-top'] || '0', 0),
      right: this.parseLength(styles['padding-right'] || '0', 0),
      bottom: this.parseLength(styles['padding-bottom'] || '0', 0),
      left: this.parseLength(styles['padding-left'] || '0', 0)
    };
  }

  calculateBorder(styles) {
    return {
      top: this.parseLength(styles['border-top-width'] || '0', 0),
      right: this.parseLength(styles['border-right-width'] || '0', 0),
      bottom: this.parseLength(styles['border-bottom-width'] || '0', 0),
      left: this.parseLength(styles['border-left-width'] || '0', 0)
    };
  }
}
```

### Layout Optimization

Minimizing layout thrashing:

```javascript
// BAD: Layout thrashing (multiple reflows)
function badLayout() {
  const elements = document.querySelectorAll('.item');

  elements.forEach(el => {
    // Read (forces layout)
    const width = el.offsetWidth;
    
    // Write (invalidates layout)
    el.style.width = width + 10 + 'px';
    
    // Repeat: read → write → read → write
    // Each read forces a layout recalculation!
  });
}

// GOOD: Batch reads and writes
function goodLayout() {
  const elements = document.querySelectorAll('.item');
  
  // Batch all reads
  const widths = Array.from(elements).map(el => el.offsetWidth);
  
  // Batch all writes
  elements.forEach((el, i) => {
    el.style.width = widths[i] + 10 + 'px';
  });
  
  // Only ONE layout recalculation!
}

// Using requestAnimationFrame
class LayoutScheduler {
  constructor() {
    this.reads = [];
    this.writes = [];
    this.scheduled = false;
  }

  scheduleRead(callback) {
    this.reads.push(callback);
    this.scheduleUpdate();
  }

  scheduleWrite(callback) {
    this.writes.push(callback);
    this.scheduleUpdate();
  }

  scheduleUpdate() {
    if (this.scheduled) return;
    
    this.scheduled = true;
    requestAnimationFrame(() => this.flush());
  }

  flush() {
    // Execute all reads first
    this.reads.forEach(read => read());
    this.reads = [];

    // Then all writes
    this.writes.forEach(write => write());
    this.writes = [];

    this.scheduled = false;
  }
}

// Usage
const scheduler = new LayoutScheduler();

// Schedule reads
scheduler.scheduleRead(() => {
  const width = element.offsetWidth;
  console.log('Width:', width);
});

// Schedule writes
scheduler.scheduleWrite(() => {
  element.style.width = '100px';
});

// All reads execute, then all writes in next frame
```

## Paint

### Paint Process

Converting render objects to pixels:

```javascript
// Paint engine
class PaintEngine {
  constructor(context) {
    this.ctx = context;
    this.paintList = [];
  }

  paint(renderTree) {
    // Generate paint commands
    this.generatePaintCommands(renderTree);

    // Execute paint commands
    this.executePaintCommands();
  }

  generatePaintCommands(node, clipRect = null) {
    if (!node) return;

    const styles = node.computedStyles || {};

    // Background
    if (styles.background || styles['background-color']) {
      this.paintList.push({
        type: 'fillRect',
        x: node.x,
        y: node.y,
        width: node.width,
        height: node.height,
        color: styles['background-color'] || styles.background
      });
    }

    // Border
    if (styles.border || styles['border-width']) {
      this.paintList.push({
        type: 'strokeRect',
        x: node.x,
        y: node.y,
        width: node.width,
        height: node.height,
        color: styles['border-color'] || 'black',
        width: styles['border-width'] || '1px'
      });
    }

    // Text
    if (node.type === 'text') {
      this.paintList.push({
        type: 'fillText',
        text: node.content,
        x: node.x,
        y: node.y,
        color: styles.color || 'black',
        font: styles['font-family'] || 'Arial',
        size: styles['font-size'] || '16px'
      });
    }

    // Box shadow
    if (styles['box-shadow']) {
      this.paintList.push({
        type: 'shadow',
        x: node.x,
        y: node.y,
        width: node.width,
        height: node.height,
        shadow: styles['box-shadow']
      });
    }

    // Paint children
    for (const child of node.children || []) {
      this.generatePaintCommands(child, clipRect);
    }
  }

  executePaintCommands() {
    for (const command of this.paintList) {
      switch (command.type) {
        case 'fillRect':
          this.ctx.fillStyle = command.color;
          this.ctx.fillRect(command.x, command.y, command.width, command.height);
          break;

        case 'strokeRect':
          this.ctx.strokeStyle = command.color;
          this.ctx.lineWidth = parseFloat(command.width);
          this.ctx.strokeRect(command.x, command.y, command.width, command.height);
          break;

        case 'fillText':
          this.ctx.fillStyle = command.color;
          this.ctx.font = `${command.size} ${command.font}`;
          this.ctx.fillText(command.text, command.x, command.y);
          break;

        case 'shadow':
          this.ctx.shadowColor = 'rgba(0, 0, 0, 0.5)';
          this.ctx.shadowBlur = 10;
          this.ctx.shadowOffsetX = 2;
          this.ctx.shadowOffsetY = 2;
          break;
      }
    }
  }
}
```

### Paint Layers

Elements painted on separate layers:

```javascript
// Layer creation criteria
class LayerManager {
  shouldCreateLayer(element) {
    const styles = getComputedStyle(element);

    // Create layer if:
    return (
      // Has 3D transform
      styles.transform && styles.transform.includes('3d') ||
      
      // Has will-change
      styles.willChange !== 'auto' ||
      
      // Has opacity < 1
      parseFloat(styles.opacity) < 1 ||
      
      // Has CSS filter
      styles.filter && styles.filter !== 'none' ||
      
      // Has overflow and scrollable
      (styles.overflow !== 'visible' && element.scrollHeight > element.clientHeight) ||
      
      // Fixed or sticky position
      styles.position === 'fixed' || styles.position === 'sticky' ||
      
      // Is video or canvas
      element.tagName === 'VIDEO' || element.tagName === 'CANVAS'
    );
  }

  createLayers(renderTree) {
    const layers = [];

    function traverse(node, parentLayer) {
      if (!node) return;

      let layer = parentLayer;

      if (this.shouldCreateLayer(node.domNode)) {
        layer = {
          id: layers.length,
          node: node,
          children: [],
          parent: parentLayer
        };
        layers.push(layer);

        if (parentLayer) {
          parentLayer.children.push(layer);
        }
      }

      for (const child of node.children || []) {
        traverse(child, layer);
      }
    }

    const rootLayer = {
      id: 0,
      node: renderTree,
      children: [],
      parent: null
    };
    layers.push(rootLayer);

    traverse(renderTree, rootLayer);

    return layers;
  }
}
```

## Composite

### Compositor Thread

Combining layers into final image:

```javascript
// Simplified compositor
class Compositor {
  constructor() {
    this.layers = [];
    this.canvas = document.createElement('canvas');
    this.ctx = this.canvas.getContext('2d');
  }

  composite(layers, viewportWidth, viewportHeight) {
    // Set canvas size
    this.canvas.width = viewportWidth;
    this.canvas.height = viewportHeight;

    // Clear canvas
    this.ctx.clearRect(0, 0, viewportWidth, viewportHeight);

    // Composite layers in order (back to front)
    for (const layer of layers) {
      this.compositeLayer(layer);
    }

    return this.canvas;
  }

  compositeLayer(layer) {
    const node = layer.node;
    const styles = node.computedStyles || {};

    // Apply transform
    if (styles.transform) {
      this.applyTransform(styles.transform);
    }

    // Apply opacity
    if (styles.opacity) {
      this.ctx.globalAlpha = parseFloat(styles.opacity);
    }

    // Draw layer content
    this.drawLayer(layer);

    // Composite child layers
    for (const childLayer of layer.children) {
      this.compositeLayer(childLayer);
    }

    // Reset transform and opacity
    this.ctx.setTransform(1, 0, 0, 1, 0, 0);
    this.ctx.globalAlpha = 1.0;
  }

  applyTransform(transformString) {
    // Parse and apply CSS transform
    // Simplified: only handles translate and scale

    const translateMatch = /translate\(([^,]+),\s*([^)]+)\)/.exec(transformString);
    if (translateMatch) {
      const x = parseFloat(translateMatch[1]);
      const y = parseFloat(translateMatch[2]);
      this.ctx.translate(x, y);
    }

    const scaleMatch = /scale\(([^)]+)\)/.exec(transformString);
    if (scaleMatch) {
      const scale = parseFloat(scaleMatch[1]);
      this.ctx.scale(scale, scale);
    }
  }

  drawLayer(layer) {
    // Draw the layer's visual content
    const node = layer.node;
    
    // Use cached texture if available
    if (layer.texture) {
      this.ctx.drawImage(layer.texture, node.x, node.y);
    } else {
      // Paint layer and cache texture
      layer.texture = this.paintToTexture(node);
      this.ctx.drawImage(layer.texture, node.x, node.y);
    }
  }

  paintToTexture(node) {
    // Create off-screen canvas for this layer
    const layerCanvas = document.createElement('canvas');
    layerCanvas.width = node.width;
    layerCanvas.height = node.height;
    
    const layerCtx = layerCanvas.getContext('2d');
    
    // Paint node to layer canvas
    const painter = new PaintEngine(layerCtx);
    painter.paint(node);
    
    return layerCanvas;
  }
}
```

### Hardware Acceleration

GPU-accelerated compositing:

```css
/* Trigger GPU acceleration */
.accelerated {
  /* 3D transform creates a layer */
  transform: translateZ(0);
  
  /* Or use will-change */
  will-change: transform;
  
  /* Animated with transform (GPU-accelerated) */
  animation: slide 1s ease-in-out;
}

@keyframes slide {
  from { transform: translateX(0); }
  to { transform: translateX(100px); }
}

/* Avoid GPU acceleration pitfalls */
.too-many-layers {
  /* DON'T promote everything to layers */
  /* Limited GPU memory! */
  transform: translateZ(0); /* Unnecessary */
}
```

```javascript
// Checking layer creation
class LayerInspector {
  inspectLayers() {
    // Chrome DevTools: Layers panel
    // Shows: which elements have layers,
    // layer memory usage, paint count

    const elements = document.querySelectorAll('*');
    const layerElements = [];

    elements.forEach(el => {
      const styles = getComputedStyle(el);
      
      if (this.hasLayer(styles)) {
        layerElements.push({
          element: el,
          reason: this.getLayerReason(styles)
        });
      }
    });

    return layerElements;
  }

  hasLayer(styles) {
    return (
      styles.transform !== 'none' ||
      styles.willChange !== 'auto' ||
      parseFloat(styles.opacity) < 1 ||
      styles.filter !== 'none'
    );
  }

  getLayerReason(styles) {
    const reasons = [];
    
    if (styles.transform !== 'none') reasons.push('transform');
    if (styles.willChange !== 'auto') reasons.push('will-change');
    if (parseFloat(styles.opacity) < 1) reasons.push('opacity');
    if (styles.filter !== 'none') reasons.push('filter');
    
    return reasons.join(', ');
  }
}
```

## Common Misconceptions

### Misconception 1: "JavaScript execution blocks rendering"

**Reality**: JavaScript blocks DOM parsing, but CSS can block rendering independently.

```html
<!-- JavaScript blocks DOM parsing -->
<script src="heavy.js"></script>
<!-- HTML after this waits for heavy.js -->

<!-- CSS blocks rendering (not parsing) -->
<link rel="stylesheet" href="styles.css">
<!-- HTML parses, but doesn't render until styles.css loads -->
```

### Misconception 2: "display: none elements don't affect performance"

**Reality**: They're in DOM and CSSOM, just not in render tree.

```javascript
// display: none elements:
// ✓ In DOM (memory cost)
// ✓ In CSSOM (style calculation cost)
// ✗ Not in render tree (no layout/paint cost)

const hiddenDiv = document.querySelector('.hidden'); // display: none
console.log(hiddenDiv); // Element exists in DOM
console.log(getComputedStyle(hiddenDiv).display); // "none"
// But: no layout, no paint, no composite

// vs visibility: hidden:
// ✓ In DOM
// ✓ In CSSOM
// ✓ In render tree
// ✓ Layout calculated
// ✗ Not painted

// vs opacity: 0:
// ✓ Everything (full cost)
```

### Misconception 3: "Inline styles are faster than external CSS"

**Reality**: Inline styles block HTML parsing and aren't cacheable.

```html
<!-- BAD: Inline styles -->
<div style="width: 100px; height: 100px; background: red;">
  <!-- Increases HTML size, not cacheable -->
</div>

<!-- GOOD: External stylesheet -->
<link rel="stylesheet" href="styles.css">
<!-- Cacheable, parallel download, doesn't block parsing -->
```

## Performance Implications

### Critical Rendering Path Optimization

Strategies to optimize CRP:

```html
<!DOCTYPE html>
<html>
<head>
  <!-- 1. Critical CSS inline -->
  <style>
    /* Above-the-fold styles */
    .hero { /* ... */ }
    .nav { /* ... */ }
  </style>

  <!-- 2. Preload key resources -->
  <link rel="preload" href="font.woff2" as="font" crossorigin>
  <link rel="preload" href="hero.jpg" as="image">

  <!-- 3. Defer non-critical CSS -->
  <link rel="preload" href="non-critical.css" as="style"
        onload="this.onload=null;this.rel='stylesheet'">

  <!-- 4. Async scripts -->
  <script src="analytics.js" async></script>

  <!-- 5. Defer scripts -->
  <script src="app.js" defer></script>
</head>
<body>
  <!-- Content here renders faster -->
</body>
</html>
```

### Measuring CRP Metrics

Tracking CRP performance:

```javascript
class CRPMetrics {
  measure() {
    // Use Performance API
    const perfData = performance.getEntriesByType('navigation')[0];

    const metrics = {
      // DOM Construction
      domContentLoaded: perfData.domContentLoadedEventEnd - perfData.domContentLoadedEventStart,
      domComplete: perfData.domComplete - perfData.domInteractive,

      // CSSOM Construction
      cssLoadTime: this.measureCSSLoadTime(),

      // First Paint
      firstPaint: this.getFirstPaint(),

      // First Contentful Paint
      firstContentfulPaint: this.getFirstContentfulPaint(),

      // Largest Contentful Paint
      largestContentfulPaint: this.getLargestContentfulPaint()
    };

    return metrics;
  }

  measureCSSLoadTime() {
    const cssResources = performance.getEntriesByType('resource')
      .filter(entry => entry.initiatorType === 'link' || entry.name.endsWith('.css'));

    if (cssResources.length === 0) return 0;

    // Find slowest CSS resource
    return Math.max(...cssResources.map(r => r.responseEnd - r.startTime));
  }

  getFirstPaint() {
    const fpEntry = performance.getEntriesByName('first-paint')[0];
    return fpEntry ? fpEntry.startTime : 0;
  }

  getFirstContentfulPaint() {
    const fcpEntry = performance.getEntriesByName('first-contentful-paint')[0];
    return fcpEntry ? fcpEntry.startTime : 0;
  }

  getLargestContentfulPaint() {
    return new Promise(resolve => {
      const observer = new PerformanceObserver(list => {
        const entries = list.getEntries();
        const lastEntry = entries[entries.length - 1];
        resolve(lastEntry.startTime);
      });

      observer.observe({ entryTypes: ['largest-contentful-paint'] });

      // Stop observing after load
      window.addEventListener('load', () => {
        setTimeout(() => observer.disconnect(), 5000);
      });
    });
  }
}

// Usage
const metrics = new CRPMetrics();
metrics.measure().then(data => {
  console.log('CRP Metrics:', data);
  
  // Send to analytics
  sendToAnalytics('CRP_Metrics', data);
});
```

## Interview Questions

### Question 1: What is the Critical Rendering Path?

**Answer**: The Critical Rendering Path is the sequence of steps the browser takes to render a page:

1. **DOM Construction**: Parse HTML → DOM tree
2. **CSSOM Construction**: Parse CSS → CSSOM tree
3. **Render Tree**: Combine DOM + CSSOM → Render tree
4. **Layout**: Calculate positions and sizes
5. **Paint**: Draw pixels
6. **Composite**: Combine layers

Optimizing CRP means minimizing these steps for faster initial render.

### Question 2: How does CSS block rendering?

**Answer**: CSS is "render-blocking" - the browser waits for CSSOM construction before rendering:

1. HTML parsing continues
2. CSSOM must be built before render tree
3. No render tree = no layout/paint
4. Page stays blank until CSS loads

Optimization: inline critical CSS, defer non-critical CSS, use media queries to make CSS non-blocking.

### Question 3: What's the difference between reflow and repaint?

**Answer**:

**Reflow (Layout)**:
- Recalculates positions/sizes
- Triggered by: DOM changes, style changes affecting geometry
- Expensive: affects entire layout tree
- Examples: changing width, height, position

**Repaint (Paint)**:
- Redraws pixels without layout change
- Triggered by: visual changes not affecting geometry
- Less expensive: only affected elements
- Examples: changing color, background, visibility

**Neither**: Composite-only changes (transform, opacity) are cheapest.

### Question 4: How do browser layers work?

**Answer**: Browsers create separate layers for certain elements:

**Layer triggers**:
- 3D transforms
- will-change property
- opacity < 1
- filters
- fixed/sticky position
- overflow scroll

**Benefits**:
- Independent compositing
- GPU acceleration
- Smooth animations

**Cost**:
- Memory per layer
- Compositing overhead
- Don't over-promote to layers

### Question 5: What makes a script "parser-blocking"?

**Answer**: Regular `<script>` tags block HTML parsing:

```html
<!-- Parser-blocking -->
<script src="app.js"></script>

<!-- Non-blocking: async -->
<script src="app.js" async></script>

<!-- Non-blocking: defer -->
<script src="app.js" defer></script>
```

**Blocking**: Parse stops, wait for download + execution
**Async**: Download in parallel, execute when ready (blocks during execution)
**Defer**: Download in parallel, execute after parsing complete

### Question 6: How do you optimize the Critical Rendering Path?

**Answer**: Key strategies:

1. **Minimize critical resources**: Inline critical CSS, defer non-critical
2. **Minimize bytes**: Minify, compress, remove unused code
3. **Minimize roundtrips**: Preload, preconnect, CDN
4. **Optimize load order**: Critical resources first
5. **Eliminate parser-blocking**: Use async/defer scripts
6. **Lazy load**: Below-fold images, non-critical content

Goal: Fast First Contentful Paint and Time to Interactive.

### Question 7: What is the render tree and how is it built?

**Answer**: The render tree combines DOM and CSSOM:

1. Start at DOM root
2. For each visible node:
   - Match CSS rules
   - Compute final styles
   - Create render object
3. Skip invisible nodes (display: none)
4. Result: tree of visual elements with styles

Render tree != DOM tree:
- No `<head>` elements
- No display: none elements
- Pseudo-elements (::before, ::after) included

### Question 8: Explain the compositing stage

**Answer**: Compositing combines painted layers:

1. **Layer creation**: Elements promoted to layers
2. **Painting**: Each layer painted to texture
3. **Compositing**: Layers combined by compositor thread
4. **GPU acceleration**: Transform/opacity on GPU

**Benefits**:
- Smooth animations (no repaint)
- Parallel processing
- Reduced main thread work

**Use**: transform and opacity for animations, not layout properties.

## Key Takeaways

1. **CRP has 6 stages**: DOM → CSSOM → Render Tree → Layout → Paint → Composite
2. **CSS blocks rendering** but not HTML parsing
3. **Scripts block parsing** unless async/defer
4. **Render tree != DOM tree** - only visible elements
5. **Layout is expensive** - affects geometry calculation
6. **Layers enable GPU acceleration** but cost memory
7. **Optimize critical resources** for fast First Paint
8. **Use async/defer scripts** to prevent parser blocking

## Resources

### Official Documentation
- [Chrome's Critical Rendering Path](https://web.dev/critical-rendering-path/)
- [MDN: Critical Rendering Path](https://developer.mozilla.org/en-US/docs/Web/Performance/Critical_rendering_path)
- [Web.dev: Rendering Performance](https://web.dev/rendering-performance/)

### Articles & Tutorials
- "Understanding the Critical Rendering Path" by Ilya Grigorik
- "Render-tree Construction, Layout, and Paint" by Google
- "CSS and JavaScript animation performance" by Paul Lewis

### Videos
- Critical Rendering Path Course (Udacity)
- Chrome's Rendering Pipeline
- Building Fast Websites (Google I/O)

### Tools
- Chrome DevTools Performance Tab
- Lighthouse
- WebPageTest
- Chrome DevTools Layers Panel

### Books
- "High Performance Browser Networking" by Ilya Grigorik
- "Web Performance in Action" by Jeremy Wagner
