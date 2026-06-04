# Drag and drop API

## Overview

The HTML Drag and Drop API allows elements to be dragged and dropped using mouse or touch. It provides events and data transfer mechanisms for building drag-and-drop interfaces.

## Making elements draggable

### Basic draggable element

```html
<div draggable="true" id="drag-item">
  Drag me
</div>
```

### Draggable attribute values

- `true` - Element is draggable
- `false` - Element is not draggable
- Not set - Default behavior (text/links draggable, others not)

## Drag events

### On draggable element

1. **dragstart** - User starts dragging
2. **drag** - Element is being dragged (fires continuously)
3. **dragend** - Drag operation ends

### On drop target

1. **dragenter** - Dragged element enters target
2. **dragover** - Dragged element is over target (fires continuously)
3. **dragleave** - Dragged element leaves target
4. **drop** - Element is dropped on target

## Basic example

```html
<div id="drag-item" draggable="true">
  Drag me
</div>

<div id="drop-zone">
  Drop here
</div>

<style>
  #drag-item {
    padding: 20px;
    background: lightblue;
    cursor: move;
    width: 150px;
    text-align: center;
  }
  
  #drop-zone {
    padding: 50px;
    background: lightgray;
    margin-top: 20px;
    border: 2px dashed #666;
    min-height: 100px;
  }
  
  #drop-zone.drag-over {
    background: lightyellow;
    border-color: orange;
  }
</style>

<script>
const dragItem = document.getElementById('drag-item');
const dropZone = document.getElementById('drop-zone');

// Drag events
dragItem.addEventListener('dragstart', (e) => {
  e.dataTransfer.effectAllowed = 'move';
  e.dataTransfer.setData('text/html', e.target.innerHTML);
  e.target.style.opacity = '0.5';
});

dragItem.addEventListener('dragend', (e) => {
  e.target.style.opacity = '1';
});

// Drop zone events
dropZone.addEventListener('dragover', (e) => {
  e.preventDefault(); // Allow drop
  e.dataTransfer.dropEffect = 'move';
  dropZone.classList.add('drag-over');
});

dropZone.addEventListener('dragleave', () => {
  dropZone.classList.remove('drag-over');
});

dropZone.addEventListener('drop', (e) => {
  e.preventDefault();
  dropZone.classList.remove('drag-over');
  
  const data = e.dataTransfer.getData('text/html');
  dropZone.innerHTML = data;
});
</script>
```

## DataTransfer object

### Setting data

```javascript
dragItem.addEventListener('dragstart', (e) => {
  // Set data with type and value
  e.dataTransfer.setData('text/plain', 'Hello World');
  e.dataTransfer.setData('text/html', '<strong>Hello</strong>');
  e.dataTransfer.setData('application/json', JSON.stringify({ id: 1, name: 'Item' }));
});
```

### Getting data

```javascript
dropZone.addEventListener('drop', (e) => {
  e.preventDefault();
  
  const text = e.dataTransfer.getData('text/plain');
  const html = e.dataTransfer.getData('text/html');
  const json = JSON.parse(e.dataTransfer.getData('application/json'));
  
  console.log(text, html, json);
});
```

### Effect allowed and drop effect

```javascript
// On drag start
e.dataTransfer.effectAllowed = 'move';  // 'copy', 'link', 'move', 'copyMove', 'all', 'none'

// On drag over
e.dataTransfer.dropEffect = 'move';  // 'copy', 'move', 'link', 'none'
```

## Drag image

### Custom drag image

```javascript
dragItem.addEventListener('dragstart', (e) => {
  const dragImage = document.createElement('div');
  dragImage.textContent = 'Dragging...';
  dragImage.style.position = 'absolute';
  dragImage.style.top = '-1000px';
  document.body.appendChild(dragImage);
  
  e.dataTransfer.setDragImage(dragImage, 0, 0);
  
  setTimeout(() => {
    document.body.removeChild(dragImage);
  }, 0);
});
```

### Using existing element

```javascript
const img = document.getElementById('custom-cursor');
e.dataTransfer.setDragImage(img, 25, 25);
```

## Complete examples

### Sortable list

```html
<ul id="sortable-list">
  <li draggable="true">Item 1</li>
  <li draggable="true">Item 2</li>
  <li draggable="true">Item 3</li>
  <li draggable="true">Item 4</li>
  <li draggable="true">Item 5</li>
</ul>

<style>
  #sortable-list {
    list-style: none;
    padding: 0;
    max-width: 300px;
  }
  
  #sortable-list li {
    padding: 12px;
    margin: 8px 0;
    background: #f0f0f0;
    border: 2px solid #ddd;
    cursor: move;
    user-select: none;
  }
  
  #sortable-list li.dragging {
    opacity: 0.5;
  }
  
  #sortable-list li.drag-over {
    border-color: blue;
    border-style: dashed;
  }
</style>

<script>
const list = document.getElementById('sortable-list');
const items = list.querySelectorAll('li');

let draggedItem = null;

items.forEach(item => {
  item.addEventListener('dragstart', (e) => {
    draggedItem = item;
    item.classList.add('dragging');
    e.dataTransfer.effectAllowed = 'move';
  });
  
  item.addEventListener('dragend', () => {
    item.classList.remove('dragging');
    draggedItem = null;
  });
  
  item.addEventListener('dragover', (e) => {
    e.preventDefault();
    item.classList.add('drag-over');
  });
  
  item.addEventListener('dragleave', () => {
    item.classList.remove('drag-over');
  });
  
  item.addEventListener('drop', (e) => {
    e.preventDefault();
    item.classList.remove('drag-over');
    
    if (draggedItem !== item) {
      const allItems = [...list.querySelectorAll('li')];
      const draggedIndex = allItems.indexOf(draggedItem);
      const targetIndex = allItems.indexOf(item);
      
      if (draggedIndex < targetIndex) {
        item.parentNode.insertBefore(draggedItem, item.nextSibling);
      } else {
        item.parentNode.insertBefore(draggedItem, item);
      }
    }
  });
});
</script>
```

### Kanban board

```html
<div class="kanban">
  <div class="column" data-status="todo">
    <h3>To Do</h3>
    <div class="cards">
      <div class="card" draggable="true" data-id="1">
        <h4>Task 1</h4>
        <p>Description</p>
      </div>
      <div class="card" draggable="true" data-id="2">
        <h4>Task 2</h4>
        <p>Description</p>
      </div>
    </div>
  </div>
  
  <div class="column" data-status="in-progress">
    <h3>In Progress</h3>
    <div class="cards"></div>
  </div>
  
  <div class="column" data-status="done">
    <h3>Done</h3>
    <div class="cards"></div>
  </div>
</div>

<style>
  .kanban {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 20px;
    padding: 20px;
  }
  
  .column {
    background: #f5f5f5;
    border-radius: 8px;
    padding: 16px;
  }
  
  .column h3 {
    margin: 0 0 16px 0;
  }
  
  .cards {
    min-height: 200px;
  }
  
  .cards.drag-over {
    background: rgba(0, 123, 255, 0.1);
    border: 2px dashed #007bff;
    border-radius: 4px;
  }
  
  .card {
    background: white;
    border: 1px solid #ddd;
    border-radius: 4px;
    padding: 12px;
    margin-bottom: 8px;
    cursor: move;
  }
  
  .card.dragging {
    opacity: 0.5;
  }
  
  .card h4 {
    margin: 0 0 8px 0;
  }
  
  .card p {
    margin: 0;
    color: #666;
    font-size: 14px;
  }
</style>

<script>
const cards = document.querySelectorAll('.card');
const columns = document.querySelectorAll('.cards');

let draggedCard = null;

// Card events
cards.forEach(card => {
  card.addEventListener('dragstart', (e) => {
    draggedCard = card;
    card.classList.add('dragging');
    e.dataTransfer.effectAllowed = 'move';
    e.dataTransfer.setData('text/html', card.innerHTML);
  });
  
  card.addEventListener('dragend', () => {
    card.classList.remove('dragging');
  });
});

// Column events
columns.forEach(column => {
  column.addEventListener('dragover', (e) => {
    e.preventDefault();
    column.classList.add('drag-over');
    e.dataTransfer.dropEffect = 'move';
  });
  
  column.addEventListener('dragleave', (e) => {
    if (e.target === column) {
      column.classList.remove('drag-over');
    }
  });
  
  column.addEventListener('drop', (e) => {
    e.preventDefault();
    column.classList.remove('drag-over');
    
    if (draggedCard) {
      column.appendChild(draggedCard);
      
      // Get new status
      const newStatus = column.closest('.column').dataset.status;
      console.log(`Card ${draggedCard.dataset.id} moved to ${newStatus}`);
      
      // Update backend
      updateCardStatus(draggedCard.dataset.id, newStatus);
    }
  });
});

function updateCardStatus(cardId, status) {
  console.log('Updating card:', cardId, 'to status:', status);
  // API call here
}
</script>
```

### File upload drop zone

```html
<div id="file-drop-zone">
  <p>Drag and drop files here</p>
  <p class="hint">or click to select</p>
  <input type="file" id="file-input" multiple hidden>
</div>

<div id="file-list"></div>

<style>
  #file-drop-zone {
    border: 3px dashed #ccc;
    border-radius: 8px;
    padding: 40px;
    text-align: center;
    cursor: pointer;
    transition: all 0.3s;
  }
  
  #file-drop-zone:hover {
    border-color: #999;
    background: #f9f9f9;
  }
  
  #file-drop-zone.drag-over {
    border-color: #007bff;
    background: #e7f3ff;
  }
  
  #file-drop-zone p {
    margin: 8px 0;
    color: #666;
  }
  
  #file-drop-zone .hint {
    font-size: 14px;
    color: #999;
  }
  
  #file-list {
    margin-top: 20px;
  }
  
  .file-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 12px;
    background: #f0f0f0;
    border-radius: 4px;
    margin-bottom: 8px;
  }
</style>

<script>
const dropZone = document.getElementById('file-drop-zone');
const fileInput = document.getElementById('file-input');
const fileList = document.getElementById('file-list');

// Prevent default drag behaviors
['dragenter', 'dragover', 'dragleave', 'drop'].forEach(eventName => {
  dropZone.addEventListener(eventName, preventDefaults, false);
  document.body.addEventListener(eventName, preventDefaults, false);
});

function preventDefaults(e) {
  e.preventDefault();
  e.stopPropagation();
}

// Highlight drop zone
['dragenter', 'dragover'].forEach(eventName => {
  dropZone.addEventListener(eventName, () => {
    dropZone.classList.add('drag-over');
  });
});

['dragleave', 'drop'].forEach(eventName => {
  dropZone.addEventListener(eventName, () => {
    dropZone.classList.remove('drag-over');
  });
});

// Handle dropped files
dropZone.addEventListener('drop', (e) => {
  const files = e.dataTransfer.files;
  handleFiles(files);
});

// Handle click to select
dropZone.addEventListener('click', () => {
  fileInput.click();
});

fileInput.addEventListener('change', (e) => {
  handleFiles(e.target.files);
});

function handleFiles(files) {
  [...files].forEach(file => {
    displayFile(file);
    uploadFile(file);
  });
}

function displayFile(file) {
  const item = document.createElement('div');
  item.className = 'file-item';
  item.innerHTML = `
    <span>${file.name} (${formatFileSize(file.size)})</span>
    <span class="status">Uploading...</span>
  `;
  fileList.appendChild(item);
}

function uploadFile(file) {
  // Simulate upload
  console.log('Uploading:', file.name);
  
  // Real implementation:
  // const formData = new FormData();
  // formData.append('file', file);
  // fetch('/upload', { method: 'POST', body: formData });
}

function formatFileSize(bytes) {
  if (bytes < 1024) return bytes + ' B';
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB';
  return (bytes / (1024 * 1024)).toFixed(1) + ' MB';
}
</script>
```

### Trello-style card reordering

```html
<div class="board">
  <div class="list">
    <h3>List 1</h3>
    <div class="cards">
      <div class="card" draggable="true">Card 1</div>
      <div class="card" draggable="true">Card 2</div>
      <div class="card" draggable="true">Card 3</div>
    </div>
  </div>
  
  <div class="list">
    <h3>List 2</h3>
    <div class="cards">
      <div class="card" draggable="true">Card 4</div>
      <div class="card" draggable="true">Card 5</div>
    </div>
  </div>
</div>

<script>
const board = document.querySelector('.board');
let draggedCard = null;

// Enable dropping on cards containers
document.querySelectorAll('.cards').forEach(container => {
  container.addEventListener('dragover', (e) => {
    e.preventDefault();
    
    const afterElement = getDragAfterElement(container, e.clientY);
    if (afterElement == null) {
      container.appendChild(draggedCard);
    } else {
      container.insertBefore(draggedCard, afterElement);
    }
  });
});

// Card drag events
document.querySelectorAll('.card').forEach(card => {
  card.addEventListener('dragstart', () => {
    draggedCard = card;
    card.classList.add('dragging');
  });
  
  card.addEventListener('dragend', () => {
    card.classList.remove('dragging');
  });
});

function getDragAfterElement(container, y) {
  const draggableElements = [...container.querySelectorAll('.card:not(.dragging)')];
  
  return draggableElements.reduce((closest, child) => {
    const box = child.getBoundingClientRect();
    const offset = y - box.top - box.height / 2;
    
    if (offset < 0 && offset > closest.offset) {
      return { offset: offset, element: child };
    } else {
      return closest;
    }
  }, { offset: Number.NEGATIVE_INFINITY }).element;
}
</script>
```

## Accessibility considerations

Drag and drop is not accessible via keyboard by default.

### Keyboard alternative

```html
<div class="item" draggable="true" tabindex="0">
  Item
  <button class="move-up">↑</button>
  <button class="move-down">↓</button>
</div>

<script>
// Keyboard navigation
document.querySelectorAll('.move-up').forEach(btn => {
  btn.addEventListener('click', (e) => {
    e.stopPropagation();
    const item = btn.closest('.item');
    const prev = item.previousElementSibling;
    if (prev) {
      item.parentNode.insertBefore(item, prev);
    }
  });
});

document.querySelectorAll('.move-down').forEach(btn => {
  btn.addEventListener('click', (e) => {
    e.stopPropagation();
    const item = btn.closest('.item');
    const next = item.nextElementSibling;
    if (next) {
      item.parentNode.insertBefore(next, item);
    }
  });
});
</script>
```

## Mobile touch support

Native drag and drop doesn't work well on mobile. Consider:

1. Touch libraries (Sortable.js, Dragula)
2. Touch events (touchstart, touchmove, touchend)
3. Pointer events

## Browser support

- All modern browsers support drag and drop
- Mobile support limited (requires polyfill or touch events)

## Best practices

1. Always call `preventDefault()` on dragover
2. Provide visual feedback during drag
3. Provide keyboard alternatives for accessibility
4. Test on mobile devices
5. Use appropriate cursor styles
6. Clean up event listeners
7. Handle edge cases (drag outside window)
8. Consider touch events for mobile
9. Use libraries for complex scenarios
10. Validate dropped data

## Key takeaways

- Use `draggable="true"` to make elements draggable
- Handle dragstart, dragover, and drop events
- Use `dataTransfer` to pass data
- Call `preventDefault()` on dragover to allow drop
- Provide visual feedback during drag operations
- Not keyboard accessible by default
- Limited mobile support
- Great for sortable lists, kanban boards, file uploads
- Consider libraries for complex implementations
- Always provide fallback interactions
