# Implement an Accessible Accordion

## The Idea

**In plain English:** An accordion is a list of sections on a webpage where each section has a clickable header — click it and the content below expands or collapses. "Accessible" means it also works for people using keyboards or screen readers (software that reads the screen aloud for visually impaired users), so everyone can use it, not just mouse users.

**Real-world analogy:** Think of a physical accordion folder — the kind with labeled tabs you use to organize documents. You pull open one tab's pocket to see the papers inside, and you can close it again by pressing it shut. If you're following strict rules (the "accessible" part), the folder also has Braille labels so someone who can't see can still find and open the right pocket.

- The labeled tab = the clickable button header (tells you what's inside)
- The pocket that opens/closes = the panel (the content that shows or hides)
- The Braille label = the ARIA attributes (extra info that screen readers announce aloud)

---

## ARIA Accordion Pattern

```text
role="region" (panel) + aria-labelledby pointing to its button
button with aria-expanded="true/false" + aria-controls pointing to panel id
```

Keyboard interaction:

- **Enter/Space**: toggle the focused panel
- **Tab**: move focus through interactive elements (buttons)
- Arrow keys (optional): navigate between headers

---

## Implementation

```tsx
interface AccordionItem {
  id: string;
  title: string;
  content: ReactNode;
}

interface AccordionProps {
  items: AccordionItem[];
  allowMultiple?: boolean; // can multiple panels be open simultaneously
  defaultOpenIds?: string[];
}

function Accordion({ items, allowMultiple = false, defaultOpenIds = [] }: AccordionProps) {
  const [openIds, setOpenIds] = useState<Set<string>>(new Set(defaultOpenIds));

  function toggle(id: string) {
    setOpenIds(prev => {
      const next = new Set(prev);
      if (next.has(id)) {
        next.delete(id);
      } else {
        if (!allowMultiple) next.clear(); // close others if single-expand
        next.add(id);
      }
      return next;
    });
  }

  return (
    <div className="accordion">
      {items.map(item => {
        const isOpen = openIds.has(item.id);
        const buttonId = `accordion-btn-${item.id}`;
        const panelId = `accordion-panel-${item.id}`;

        return (
          <div key={item.id} className="accordion-item">
            {/* Header button */}
            <h3 className="accordion-header">
              <button
                id={buttonId}
                aria-expanded={isOpen}
                aria-controls={panelId}
                className={`accordion-trigger ${isOpen ? 'open' : ''}`}
                onClick={() => toggle(item.id)}
              >
                {item.title}
                <span aria-hidden="true" className="accordion-icon">
                  {isOpen ? '▲' : '▼'}
                </span>
              </button>
            </h3>

            {/* Panel */}
            <div
              id={panelId}
              role="region"
              aria-labelledby={buttonId}
              hidden={!isOpen}
              className="accordion-panel"
            >
              <div className="accordion-panel-inner">
                {item.content}
              </div>
            </div>
          </div>
        );
      })}
    </div>
  );
}
```

---

## Smooth Animation (CSS + `hidden` vs CSS-only)

The `hidden` attribute removes content from the DOM and accessibility tree — correct but no animation. For animated open/close:

```tsx
// Use max-height animation instead of hidden
<div
  id={panelId}
  role="region"
  aria-labelledby={buttonId}
  className={`accordion-panel ${isOpen ? 'open' : ''}`}
  aria-hidden={!isOpen}
  inert={!isOpen || undefined}  // modern: inert prevents interaction while "hidden"
>
```

```css
.accordion-panel {
  max-height: 0;
  overflow: hidden;
  transition: max-height 0.3s ease-out;
}

.accordion-panel.open {
  max-height: 500px; /* estimate — or use JS to measure actual height */
  transition: max-height 0.3s ease-in;
}

@media (prefers-reduced-motion: reduce) {
  .accordion-panel {
    transition: none;
  }
}
```

For precise height animation (no arbitrary max-height):

```tsx
function AnimatedPanel({ isOpen, children }: { isOpen: boolean; children: ReactNode }) {
  const ref = useRef<HTMLDivElement>(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    if (ref.current) {
      setHeight(isOpen ? ref.current.scrollHeight : 0);
    }
  }, [isOpen]);

  return (
    <div style={{ height, overflow: 'hidden', transition: 'height 0.3s ease' }}>
      <div ref={ref}>{children}</div>
    </div>
  );
}
```

---

## Optional: Arrow Key Navigation Between Headers

```tsx
function Accordion({ items, ... }) {
  const buttonRefs = useRef<(HTMLButtonElement | null)[]>([]);

  function handleKeyDown(e: React.KeyboardEvent, index: number) {
    if (e.key === 'ArrowDown') {
      e.preventDefault();
      buttonRefs.current[(index + 1) % items.length]?.focus();
    } else if (e.key === 'ArrowUp') {
      e.preventDefault();
      buttonRefs.current[(index - 1 + items.length) % items.length]?.focus();
    } else if (e.key === 'Home') {
      e.preventDefault();
      buttonRefs.current[0]?.focus();
    } else if (e.key === 'End') {
      e.preventDefault();
      buttonRefs.current[items.length - 1]?.focus();
    }
  }

  // In render:
  <button
    ref={el => { buttonRefs.current[index] = el; }}
    onKeyDown={e => handleKeyDown(e, index)}
    ...
  >
}
```

Note: Arrow key navigation between headers is **optional** per WAI-ARIA — Tab navigation between buttons is required.

---

## Usage

```tsx
const items = [
  {
    id: 'item-1',
    title: 'What is React?',
    content: <p>A JavaScript library for building user interfaces.</p>,
  },
  {
    id: 'item-2',
    title: 'What is Angular?',
    content: <p>A TypeScript-based web application framework.</p>,
  },
];

<Accordion items={items} allowMultiple={false} defaultOpenIds={['item-1']} />
```

---

## Common Interview Questions

**Q: Why use `<h3>` (or heading) inside the accordion item?**
Accordion headers should be headings to maintain document outline and allow screen reader heading navigation (H key in NVDA/JAWS jumps to next heading). The heading level depends on context — `<h2>` or `<h3>` is common.

**Q: What's the difference between `aria-hidden` and `hidden`?**
`hidden` removes from DOM, accessibility tree, and makes it non-focusable. `aria-hidden="true"` only hides from the accessibility tree (screen readers); visually and in the DOM it still exists. For accordions, `hidden` is cleaner. For animated panels, `aria-hidden` + `inert` gives animation freedom.

**Q: Why is the icon (`▼`) marked `aria-hidden="true"`?**
Screen readers would otherwise read "accordion title down-pointing triangle" — confusing and redundant. The `aria-expanded` attribute already communicates open/close state. Decorative icons should always be hidden from screen readers.
