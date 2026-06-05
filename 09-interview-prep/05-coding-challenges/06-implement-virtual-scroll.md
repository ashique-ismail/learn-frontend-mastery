# Implement Virtual Scrolling

## The Idea

**In plain English:** Virtual scrolling is a trick where a webpage only draws the list items you can currently see on screen, instead of drawing all 10,000 at once — it swaps items in and out as you scroll, keeping the page fast no matter how long the list is.

**Real-world analogy:** Imagine reading a very long paper scroll through a small window cut in a cardboard box. The full scroll exists, but you only ever see the few lines framed by the window — as you pull the scroll downward, new lines appear at the bottom and old ones disappear at the top.

- The cardboard box window = the visible viewport (the fixed-height container on screen)
- The lines currently visible through the window = the DOM elements actually rendered
- Pulling the scroll through the box = the user scrolling, which triggers swapping new items in and old items out

---

## Overview

Implement virtual scrolling from scratch to efficiently render large lists by only rendering visible items.

## Basic Implementation

```typescript
class VirtualScroll {
  private container: HTMLElement;
  private itemHeight: number;
  private visibleCount: number;
  private totalCount: number;
  private scrollTop: number = 0;
  private startIndex: number = 0;
  private renderCallback: (items: any[]) => void;

  constructor(config: {
    container: HTMLElement;
    itemHeight: number;
    totalCount: number;
    renderCallback: (items: any[]) => void;
  }) {
    this.container = config.container;
    this.itemHeight = config.itemHeight;
    this.totalCount = config.totalCount;
    this.renderCallback = config.renderCallback;
    this.visibleCount = Math.ceil(this.container.clientHeight / this.itemHeight) + 2;

    this.setupContainer();
    this.attachScrollListener();
    this.render();
  }

  private setupContainer() {
    this.container.style.overflow = 'auto';
    this.container.style.position = 'relative';
  }

  private attachScrollListener() {
    this.container.addEventListener('scroll', () => {
      this.scrollTop = this.container.scrollTop;
      this.updateView();
    });
  }

  private updateView() {
    const newStartIndex = Math.floor(this.scrollTop / this.itemHeight);
    
    if (newStartIndex !== this.startIndex) {
      this.startIndex = newStartIndex;
      this.render();
    }
  }

  private render() {
    const endIndex = Math.min(
      this.startIndex + this.visibleCount,
      this.totalCount
    );

    const items = [];
    for (let i = this.startIndex; i < endIndex; i++) {
      items.push({
        index: i,
        top: i * this.itemHeight
      });
    }

    // Set container height to enable scrolling
    const totalHeight = this.totalCount * this.itemHeight;
    this.container.style.height = `${totalHeight}px`;

    this.renderCallback(items);
  }
}

// Usage
const container = document.getElementById('list-container')!;
const virtualScroll = new VirtualScroll({
  container,
  itemHeight: 50,
  totalCount: 10000,
  renderCallback: (items) => {
    container.innerHTML = items.map(item => `
      <div style="position: absolute; top: ${item.top}px; height: 50px; width: 100%;">
        Item ${item.index}
      </div>
    `).join('');
  }
});
```

## Advanced Implementation with Dynamic Heights

```typescript
interface VirtualItem {
  index: number;
  height: number;
  top: number;
}

class DynamicVirtualScroll<T> {
  private container: HTMLElement;
  private items: T[];
  private itemHeights: Map<number, number>;
  private estimatedItemHeight: number;
  private visibleRange: { start: number; end: number };
  private renderCallback: (items: Array<{ item: T; index: number; top: number }>) => void;
  private scrollTop: number = 0;
  private containerHeight: number = 0;
  private totalHeight: number = 0;
  private measuredHeights: Map<number, number> = new Map();

  constructor(config: {
    container: HTMLElement;
    items: T[];
    estimatedItemHeight: number;
    renderCallback: (items: Array<{ item: T; index: number; top: number }>) => void;
  }) {
    this.container = config.container;
    this.items = config.items;
    this.estimatedItemHeight = config.estimatedItemHeight;
    this.renderCallback = config.renderCallback;
    this.itemHeights = new Map();
    this.visibleRange = { start: 0, end: 0 };
    this.containerHeight = this.container.clientHeight;

    this.initialize();
  }

  private initialize() {
    this.setupContainer();
    this.attachListeners();
    this.calculateVisibleRange();
    this.render();
  }

  private setupContainer() {
    this.container.style.overflow = 'auto';
    this.container.style.position = 'relative';
  }

  private attachListeners() {
    this.container.addEventListener('scroll', () => {
      this.scrollTop = this.container.scrollTop;
      this.handleScroll();
    });

    // Observe resize
    const resizeObserver = new ResizeObserver(() => {
      this.containerHeight = this.container.clientHeight;
      this.calculateVisibleRange();
      this.render();
    });

    resizeObserver.observe(this.container);
  }

  private handleScroll() {
    requestAnimationFrame(() => {
      this.calculateVisibleRange();
      this.render();
    });
  }

  private getItemHeight(index: number): number {
    return this.measuredHeights.get(index) || this.estimatedItemHeight;
  }

  private calculateTotalHeight(): number {
    let height = 0;
    for (let i = 0; i < this.items.length; i++) {
      height += this.getItemHeight(i);
    }
    return height;
  }

  private calculateVisibleRange() {
    let currentTop = 0;
    let startIndex = 0;
    let endIndex = 0;

    // Find start index
    for (let i = 0; i < this.items.length; i++) {
      const itemHeight = this.getItemHeight(i);
      
      if (currentTop + itemHeight > this.scrollTop) {
        startIndex = Math.max(0, i - 2); // Add buffer
        break;
      }
      
      currentTop += itemHeight;
    }

    // Find end index
    currentTop = this.getOffsetTop(startIndex);
    endIndex = startIndex;
    
    while (currentTop < this.scrollTop + this.containerHeight && endIndex < this.items.length) {
      currentTop += this.getItemHeight(endIndex);
      endIndex++;
    }

    endIndex = Math.min(endIndex + 2, this.items.length); // Add buffer

    this.visibleRange = { start: startIndex, end: endIndex };
  }

  private getOffsetTop(index: number): number {
    let top = 0;
    for (let i = 0; i < index; i++) {
      top += this.getItemHeight(i);
    }
    return top;
  }

  private render() {
    const visibleItems = [];
    
    for (let i = this.visibleRange.start; i < this.visibleRange.end; i++) {
      visibleItems.push({
        item: this.items[i],
        index: i,
        top: this.getOffsetTop(i)
      });
    }

    this.totalHeight = this.calculateTotalHeight();
    
    // Update container height
    const scrollContainer = this.container.querySelector('.scroll-container');
    if (scrollContainer) {
      (scrollContainer as HTMLElement).style.height = `${this.totalHeight}px`;
    }

    this.renderCallback(visibleItems);
    
    // Measure rendered items
    this.measureItems();
  }

  private measureItems() {
    // Measure all visible items
    for (let i = this.visibleRange.start; i < this.visibleRange.end; i++) {
      const element = this.container.querySelector(`[data-index="${i}"]`) as HTMLElement;
      
      if (element) {
        const height = element.getBoundingClientRect().height;
        if (height !== this.measuredHeights.get(i)) {
          this.measuredHeights.set(i, height);
          // Recalculate if height changed
          this.calculateVisibleRange();
        }
      }
    }
  }

  updateItems(newItems: T[]) {
    this.items = newItems;
    this.measuredHeights.clear();
    this.calculateVisibleRange();
    this.render();
  }

  scrollToIndex(index: number) {
    const top = this.getOffsetTop(index);
    this.container.scrollTop = top;
  }
}
```

## React Implementation

```typescript
import React, { useEffect, useRef, useState, useCallback } from 'react';

interface VirtualListProps<T> {
  items: T[];
  itemHeight: number;
  height: number;
  renderItem: (item: T, index: number) => React.ReactNode;
}

function VirtualList<T>({ items, itemHeight, height, renderItem }: VirtualListProps<T>) {
  const containerRef = useRef<HTMLDivElement>(null);
  const [scrollTop, setScrollTop] = useState(0);

  const handleScroll = useCallback((e: React.UIEvent<HTMLDivElement>) => {
    setScrollTop(e.currentTarget.scrollTop);
  }, []);

  const visibleCount = Math.ceil(height / itemHeight) + 2;
  const startIndex = Math.max(0, Math.floor(scrollTop / itemHeight) - 1);
  const endIndex = Math.min(items.length, startIndex + visibleCount);

  const visibleItems = items.slice(startIndex, endIndex);
  const offsetY = startIndex * itemHeight;
  const totalHeight = items.length * itemHeight;

  return (
    <div
      ref={containerRef}
      style={{
        height,
        overflow: 'auto',
        position: 'relative'
      }}
      onScroll={handleScroll}>
      
      <div style={{ height: totalHeight, position: 'relative' }}>
        <div style={{ transform: `translateY(${offsetY}px)` }}>
          {visibleItems.map((item, i) => (
            <div
              key={startIndex + i}
              style={{ height: itemHeight }}>
              {renderItem(item, startIndex + i)}
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}

// Usage
function App() {
  const items = Array.from({ length: 10000 }, (_, i) => ({
    id: i,
    name: `Item ${i}`
  }));

  return (
    <VirtualList
      items={items}
      itemHeight={50}
      height={600}
      renderItem={(item, index) => (
        <div style={{ padding: '10px', borderBottom: '1px solid #ddd' }}>
          {item.name}
        </div>
      )}
    />
  );
}
```

## Angular Implementation

```typescript
@Component({
  selector: 'app-virtual-list',
  template: `
    <div
      class="virtual-scroll-container"
      (scroll)="onScroll($event)"
      [style.height.px]="containerHeight">
      
      <div
        class="virtual-scroll-spacer"
        [style.height.px]="totalHeight">
        
        <div
          class="virtual-scroll-content"
          [style.transform]="'translateY(' + offsetY + 'px)'">
          
          <div
            *ngFor="let item of visibleItems; trackBy: trackByIndex"
            class="virtual-item"
            [style.height.px]="itemHeight">
            <ng-container *ngTemplateOutlet="itemTemplate; context: { $implicit: item }"></ng-container>
          </div>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .virtual-scroll-container {
      overflow: auto;
      position: relative;
    }

    .virtual-scroll-spacer {
      position: relative;
    }
  `]
})
export class VirtualListComponent implements OnInit {
  @Input() items: any[] = [];
  @Input() itemHeight = 50;
  @Input() containerHeight = 600;
  @ContentChild(TemplateRef) itemTemplate!: TemplateRef<any>;

  visibleItems: any[] = [];
  offsetY = 0;
  totalHeight = 0;
  
  private scrollTop = 0;
  private visibleCount = 0;

  ngOnInit() {
    this.visibleCount = Math.ceil(this.containerHeight / this.itemHeight) + 2;
    this.totalHeight = this.items.length * this.itemHeight;
    this.updateVisibleItems();
  }

  onScroll(event: Event) {
    this.scrollTop = (event.target as HTMLElement).scrollTop;
    requestAnimationFrame(() => {
      this.updateVisibleItems();
    });
  }

  private updateVisibleItems() {
    const startIndex = Math.max(0, Math.floor(this.scrollTop / this.itemHeight) - 1);
    const endIndex = Math.min(this.items.length, startIndex + this.visibleCount);

    this.visibleItems = this.items.slice(startIndex, endIndex);
    this.offsetY = startIndex * this.itemHeight;
  }

  trackByIndex(index: number, item: any): number {
    return index;
  }
}
```

## Key Takeaways

1. **Only Render Visible**: Render only items in viewport plus small buffer for smooth scrolling.

2. **Request Animation Frame**: Use RAF for smooth scroll performance and avoid layout thrashing.

3. **Fixed vs Dynamic Heights**: Fixed heights are simpler and faster, dynamic requires measurement.

4. **Viewport Calculation**: Calculate visible range based on scroll position and container height.

5. **Buffer Items**: Render few extra items above/below viewport to prevent blank spaces during scroll.

6. **Total Height**: Set spacer height to total list height to maintain proper scrollbar.

7. **Absolute Positioning**: Use absolute or transform positioning for precise item placement.

8. **Performance**: Virtual scrolling enables smooth 60fps scrolling with 100K+ items.

9. **Measurement**: For dynamic heights, measure rendered items and update calculations.

10. **Scroll To Index**: Implement scrollToIndex by calculating offset of target item.

## Interview Talking Points

- Explain virtual scrolling concept and benefits
- Describe viewport calculation algorithm
- Discuss fixed vs dynamic height trade-offs
- Explain buffer strategy for smooth scrolling
- Describe measurement approach for dynamic heights
- Discuss performance optimization techniques
- Explain scroll position calculation
- Describe how to handle item updates
- Discuss accessibility considerations
- Compare with CSS-based solutions like content-visibility
