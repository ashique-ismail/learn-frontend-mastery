# Implement Tab Component with Keyboard Navigation

## ARIA Tabs Pattern

```
role="tablist"          ← container
  role="tab"            ← each tab button (aria-selected, aria-controls)
role="tabpanel"         ← each content panel (aria-labelledby)
```

Keyboard interaction:
- **Arrow Left/Right**: Navigate between tabs (roving tabindex)
- **Home**: First tab
- **End**: Last tab
- **Enter/Space**: Activate focused tab (if manual activation)
- **Tab**: Move focus into the active panel

---

## Implementation (Automatic Activation)

```tsx
interface Tab {
  id: string;
  label: string;
  content: ReactNode;
}

function Tabs({ tabs, defaultTab }: { tabs: Tab[]; defaultTab?: string }) {
  const [activeTab, setActiveTab] = useState(defaultTab ?? tabs[0]?.id);
  const tabRefs = useRef<(HTMLButtonElement | null)[]>([]);

  function handleKeyDown(e: React.KeyboardEvent, currentIndex: number) {
    let nextIndex: number | null = null;

    switch (e.key) {
      case 'ArrowRight':
        nextIndex = (currentIndex + 1) % tabs.length;
        break;
      case 'ArrowLeft':
        nextIndex = (currentIndex - 1 + tabs.length) % tabs.length;
        break;
      case 'Home':
        nextIndex = 0;
        break;
      case 'End':
        nextIndex = tabs.length - 1;
        break;
      default:
        return;
    }

    if (nextIndex !== null) {
      e.preventDefault();
      setActiveTab(tabs[nextIndex].id);      // auto-activate on arrow key
      tabRefs.current[nextIndex]?.focus();   // move focus
    }
  }

  return (
    <div>
      {/* Tab list */}
      <div role="tablist" aria-label="Content tabs">
        {tabs.map((tab, index) => (
          <button
            key={tab.id}
            ref={el => { tabRefs.current[index] = el; }}
            role="tab"
            id={`tab-${tab.id}`}
            aria-selected={activeTab === tab.id}
            aria-controls={`panel-${tab.id}`}
            tabIndex={activeTab === tab.id ? 0 : -1}  // roving tabindex
            onClick={() => setActiveTab(tab.id)}
            onKeyDown={(e) => handleKeyDown(e, index)}
            className={activeTab === tab.id ? 'tab active' : 'tab'}
          >
            {tab.label}
          </button>
        ))}
      </div>

      {/* Tab panels */}
      {tabs.map(tab => (
        <div
          key={tab.id}
          role="tabpanel"
          id={`panel-${tab.id}`}
          aria-labelledby={`tab-${tab.id}`}
          hidden={activeTab !== tab.id}
          tabIndex={0}  // makes panel focusable (Tab from tablist enters panel)
        >
          {tab.content}
        </div>
      ))}
    </div>
  );
}
```

---

## Roving `tabIndex` Explained

Instead of all tabs being tabbable (which means many Tab keypresses), only the active tab has `tabIndex={0}`. Others have `tabIndex={-1}`:

```
[Tab1: tabIndex=0] [Tab2: tabIndex=-1] [Tab3: tabIndex=-1]
     ↑ focused

User presses →:
[Tab1: tabIndex=-1] [Tab2: tabIndex=0] [Tab3: tabIndex=-1]
                         ↑ focused and activated
```

This way, Tab from outside moves focus to the active tab (not all tabs). Arrow keys move between tabs.

---

## Manual Activation (ARIA pattern variant)

Some UIs activate only on Enter/Space, not on arrow keys (useful when panels are slow to load):

```tsx
function handleKeyDown(e: React.KeyboardEvent, currentIndex: number) {
  const [focusedIndex, setFocusedIndex] = useState(0);

  switch (e.key) {
    case 'ArrowRight':
      const next = (currentIndex + 1) % tabs.length;
      setFocusedIndex(next);
      tabRefs.current[next]?.focus();
      // Don't activate — just move focus
      break;
    case 'Enter':
    case ' ':
      e.preventDefault();
      setActiveTab(tabs[currentIndex].id);  // activate on Enter/Space
      break;
  }
}
```

---

## Lazy Panel Rendering

For performance, only render the active panel (or mount all but hide with CSS):

```tsx
// Option 1: Unmount inactive panels (re-mounts on tab switch)
{tabs.map(tab =>
  activeTab === tab.id && (
    <div key={tab.id} role="tabpanel" tabIndex={0}>
      {tab.content}
    </div>
  )
)}

// Option 2: Keep mounted, hide with CSS (preserves state)
{tabs.map(tab => (
  <div
    key={tab.id}
    role="tabpanel"
    hidden={activeTab !== tab.id}  // hidden removes from accessibility tree too
    tabIndex={0}
  >
    {tab.content}
  </div>
))}
```

---

## Common Interview Questions

**Q: Why use roving tabindex instead of letting all tabs be tabbable?**
If you have 10 tabs, tabbing through the page would require 10 Tab presses to get past the tab list. With roving tabindex, only one Tab press exits the tab list. Arrow keys navigate between tabs — a common and expected keyboard pattern (matching native OS tab controls).

**Q: What does `hidden` do vs `display: none` vs `visibility: hidden`?**
`hidden` attribute (HTML boolean) adds `display: none` AND removes the element from the accessibility tree. Screen readers won't read hidden panels, which is correct — only the active panel should be read.

**Q: Should you use `<button>` or `<a>` for tabs?**
`<button>` — tabs switch panels within the same page (no navigation, no URL change). Use `<a>` only if each tab has its own URL (URL-based tabs, which are just styled navigation links).
