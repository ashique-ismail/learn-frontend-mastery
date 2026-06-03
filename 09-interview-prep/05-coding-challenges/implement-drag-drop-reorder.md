# Implement Drag-and-Drop List Reorder

## HTML5 Drag-and-Drop API

```tsx
function DraggableList<T extends { id: string | number }>({
  items,
  onReorder,
  renderItem,
}: {
  items: T[];
  onReorder: (items: T[]) => void;
  renderItem: (item: T) => ReactNode;
}) {
  const dragIndexRef = useRef<number | null>(null);
  const [dragOverIndex, setDragOverIndex] = useState<number | null>(null);

  function handleDragStart(e: React.DragEvent, index: number) {
    dragIndexRef.current = index;
    e.dataTransfer.effectAllowed = 'move';
    e.dataTransfer.setData('text/plain', String(index)); // required for Firefox
  }

  function handleDragOver(e: React.DragEvent, index: number) {
    e.preventDefault(); // required to allow drop
    e.dataTransfer.dropEffect = 'move';
    setDragOverIndex(index);
  }

  function handleDrop(e: React.DragEvent, dropIndex: number) {
    e.preventDefault();
    const dragIndex = dragIndexRef.current;
    if (dragIndex === null || dragIndex === dropIndex) return;

    const newItems = [...items];
    const [removed] = newItems.splice(dragIndex, 1);
    newItems.splice(dropIndex, 0, removed);
    onReorder(newItems);

    dragIndexRef.current = null;
    setDragOverIndex(null);
  }

  function handleDragEnd() {
    dragIndexRef.current = null;
    setDragOverIndex(null);
  }

  return (
    <ul>
      {items.map((item, index) => (
        <li
          key={item.id}
          draggable
          onDragStart={(e) => handleDragStart(e, index)}
          onDragOver={(e) => handleDragOver(e, index)}
          onDrop={(e) => handleDrop(e, index)}
          onDragEnd={handleDragEnd}
          style={{
            opacity: dragIndexRef.current === index ? 0.5 : 1,
            borderTop: dragOverIndex === index ? '2px solid blue' : 'none',
          }}
        >
          <span className="drag-handle" aria-hidden="true">⠿</span>
          {renderItem(item)}
        </li>
      ))}
    </ul>
  );
}
```

---

## With Touch Support (Mobile)

HTML5 drag-and-drop doesn't work on touch devices. Implement via touch events:

```tsx
function useTouchDrag(items: any[], onReorder: (items: any[]) => void) {
  const dragStartIndex = useRef<number | null>(null);
  const currentY = useRef(0);

  function handleTouchStart(e: TouchEvent, index: number) {
    dragStartIndex.current = index;
    currentY.current = e.touches[0].clientY;
  }

  function handleTouchMove(e: TouchEvent) {
    e.preventDefault(); // prevent scroll
    currentY.current = e.touches[0].clientY;
    // Visually move the dragged element...
  }

  function handleTouchEnd(e: TouchEvent) {
    // Determine drop target by querying element at touch point
    const dropEl = document.elementFromPoint(
      e.changedTouches[0].clientX,
      e.changedTouches[0].clientY
    );
    const dropIndex = getIndexFromElement(dropEl);

    if (dragStartIndex.current !== null && dropIndex !== null) {
      const newItems = [...items];
      const [removed] = newItems.splice(dragStartIndex.current, 1);
      newItems.splice(dropIndex, 0, removed);
      onReorder(newItems);
    }
    dragStartIndex.current = null;
  }

  return { handleTouchStart, handleTouchMove, handleTouchEnd };
}
```

---

## Accessible Keyboard Reordering

Screen reader users can't drag — provide arrow key reordering:

```tsx
function AccessibleDraggableList({ items, onReorder }) {
  const [selectedIndex, setSelectedIndex] = useState<number | null>(null);

  function handleKeyDown(e: React.KeyboardEvent, index: number) {
    if (e.key === ' ' || e.key === 'Enter') {
      // Toggle selection
      setSelectedIndex(prev => prev === index ? null : index);
    }

    if (selectedIndex !== null && e.key === 'ArrowDown') {
      e.preventDefault();
      const newItems = [...items];
      const newIndex = Math.min(index + 1, items.length - 1);
      [newItems[index], newItems[newIndex]] = [newItems[newIndex], newItems[index]];
      onReorder(newItems);
      setSelectedIndex(newIndex);
    }

    if (selectedIndex !== null && e.key === 'ArrowUp') {
      e.preventDefault();
      const newItems = [...items];
      const newIndex = Math.max(index - 1, 0);
      [newItems[index], newItems[newIndex]] = [newItems[newIndex], newItems[index]];
      onReorder(newItems);
      setSelectedIndex(newIndex);
    }

    if (e.key === 'Escape') setSelectedIndex(null);
  }

  return (
    <ul role="listbox" aria-label="Reorderable list">
      {items.map((item, index) => (
        <li
          key={item.id}
          role="option"
          aria-selected={selectedIndex === index}
          tabIndex={0}
          onKeyDown={(e) => handleKeyDown(e, index)}
          aria-label={`${item.name}, position ${index + 1} of ${items.length}. Press Space to select, then arrow keys to move.`}
        >
          {item.name}
        </li>
      ))}
    </ul>
  );
}
```

---

## Using @dnd-kit (Production Library)

```tsx
import { DndContext, closestCenter, useSensor, PointerSensor } from '@dnd-kit/core';
import { SortableContext, verticalListSortingStrategy, arrayMove, useSortable } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';

function SortableItem({ id, children }) {
  const { attributes, listeners, setNodeRef, transform, transition, isDragging } = useSortable({ id });

  return (
    <li
      ref={setNodeRef}
      style={{ transform: CSS.Transform.toString(transform), transition, opacity: isDragging ? 0.5 : 1 }}
      {...attributes}
      {...listeners}
    >
      {children}
    </li>
  );
}

function SortableList({ items, onReorder }) {
  const sensor = useSensor(PointerSensor);

  return (
    <DndContext
      sensors={[sensor]}
      collisionDetection={closestCenter}
      onDragEnd={({ active, over }) => {
        if (over && active.id !== over.id) {
          const oldIndex = items.findIndex(i => i.id === active.id);
          const newIndex = items.findIndex(i => i.id === over.id);
          onReorder(arrayMove(items, oldIndex, newIndex));
        }
      }}
    >
      <SortableContext items={items.map(i => i.id)} strategy={verticalListSortingStrategy}>
        <ul>
          {items.map(item => (
            <SortableItem key={item.id} id={item.id}>{item.name}</SortableItem>
          ))}
        </ul>
      </SortableContext>
    </DndContext>
  );
}
```

---

## Common Interview Questions

**Q: Why doesn't HTML5 DnD work on mobile?**
The HTML5 Drag-and-Drop API was designed for desktop (mouse events). Touch events (`touchstart`, `touchmove`, `touchend`) are separate and require separate implementation or a library.

**Q: What does `e.preventDefault()` do in `dragover`?**
By default, elements are not valid drop targets — the browser cancels drop. Calling `e.preventDefault()` on `dragover` signals that this element accepts drops, enabling the `drop` event to fire.

**Q: How do you indicate the drop position visually?**
Add a visual indicator (border, placeholder element) at the drop target index. Track the `dragOverIndex` in state and apply styling to the corresponding list item.
