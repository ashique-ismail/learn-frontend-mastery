# File Uploads

## The Idea

**In plain English:** File uploads let a webpage receive actual files from your computer — like photos, documents, or videos — and send them to a server for storage or processing. A "server" is just another computer on the internet that holds your data.

**Real-world analogy:** Think of dropping off a physical package at a post office counter. You hand the clerk your parcel, they check it meets the size and content rules, and then it gets stored in the back warehouse.

- The parcel you hand over = the file selected by the user
- The counter's size and content rules = client-side validation (file size limit, allowed types)
- The clerk's stamp of approval = the `accept` attribute filtering what files are allowed
- The back warehouse = the server that stores the uploaded file

---

## Learning Objectives

- Master the file input type and its attributes
- Implement single and multiple file uploads
- Build drag-and-drop file upload interfaces
- Validate files before upload (size, type, dimensions)
- Handle file preview and upload progress
- Understand security considerations for file uploads

---

## The File Input Element

### Basic File Input

```html
<form method="POST" enctype="multipart/form-data">
  <label for="file">Choose file:</label>
  <input type="file" id="file" name="file">
  <button type="submit">Upload</button>
</form>
```

**Critical:** Form must have `enctype="multipart/form-data"` for file uploads!

---

### File Input Attributes

#### `accept` - Allowed File Types

**MIME types:**
```html
<!-- Images only -->
<input type="file" accept="image/png, image/jpeg, image/gif">

<!-- PDFs only -->
<input type="file" accept="application/pdf">

<!-- Videos only -->
<input type="file" accept="video/mp4, video/quicktime">

<!-- Audio only -->
<input type="file" accept="audio/mpeg, audio/wav">
```

**File extensions:**
```html
<!-- Specific extensions -->
<input type="file" accept=".pdf, .doc, .docx">

<!-- Multiple image formats -->
<input type="file" accept=".jpg, .jpeg, .png, .gif, .webp">
```

**Wildcard patterns:**
```html
<!-- Any image -->
<input type="file" accept="image/*">

<!-- Any video -->
<input type="file" accept="video/*">

<!-- Any audio -->
<input type="file" accept="audio/*">
```

**Combined:**
```html
<input type="file" accept="image/*, .pdf">
<!-- Accepts any image OR PDF files -->
```

**Note:** `accept` is just a **hint** for file picker. Always validate server-side!

---

#### `multiple` - Multiple File Selection

```html
<label for="photos">Select photos:</label>
<input type="file" id="photos" name="photos" accept="image/*" multiple>
```

**Behavior:**
- Users can select multiple files (Ctrl+Click, Shift+Click, or "Select All")
- All selected files available in JavaScript as `files` array
- All files submitted with the same field name

---

#### `capture` - Mobile Camera/Microphone

```html
<!-- Rear camera -->
<input type="file" accept="image/*" capture="environment">

<!-- Front camera (selfie) -->
<input type="file" accept="image/*" capture="user">

<!-- Video recording -->
<input type="file" accept="video/*" capture="environment">

<!-- Audio recording -->
<input type="file" accept="audio/*" capture>
```

**Desktop:** Opens file picker (ignores `capture`)

**Mobile:** Opens camera/microphone app directly

---

## Accessing Files with JavaScript

### The FileList and File Objects

```html
<input type="file" id="file-input" multiple>

<script>
const input = document.getElementById('file-input');

input.addEventListener('change', (e) => {
  const files = e.target.files; // FileList object
  
  console.log('Number of files:', files.length);
  
  // Iterate through files
  for (let i = 0; i < files.length; i++) {
    const file = files[i]; // File object
    
    console.log('Name:', file.name);
    console.log('Size:', file.size, 'bytes');
    console.log('Type:', file.type);
    console.log('Last modified:', new Date(file.lastModified));
  }
  
  // Convert to array for easier manipulation
  const fileArray = Array.from(files);
  fileArray.forEach(file => {
    console.log(file.name);
  });
});
</script>
```

---

### File Properties

```javascript
const file = input.files[0];

file.name          // "photo.jpg"
file.size          // 1234567 (bytes)
file.type          // "image/jpeg"
file.lastModified  // 1621234567890 (timestamp)
```

---

## Single File Upload

### Basic Implementation

```html
<form id="upload-form">
  <label for="document">Upload document:</label>
  <input type="file" id="document" name="document" accept=".pdf, .doc, .docx" required>
  
  <button type="submit">Upload</button>
</form>

<div id="status"></div>

<script>
const form = document.getElementById('upload-form');
const status = document.getElementById('status');

form.addEventListener('submit', async (e) => {
  e.preventDefault();
  
  const fileInput = document.getElementById('document');
  const file = fileInput.files[0];
  
  if (!file) {
    status.textContent = 'Please select a file';
    return;
  }
  
  // Create FormData
  const formData = new FormData();
  formData.append('document', file);
  
  try {
    status.textContent = 'Uploading...';
    
    const response = await fetch('/api/upload', {
      method: 'POST',
      body: formData
    });
    
    if (!response.ok) {
      throw new Error('Upload failed');
    }
    
    const result = await response.json();
    status.textContent = `Upload successful! File ID: ${result.id}`;
    
  } catch (error) {
    status.textContent = `Error: ${error.message}`;
  }
});
</script>
```

---

### With File Preview

```html
<input type="file" id="avatar" accept="image/*">
<img id="preview" style="max-width: 200px; display: none;">

<script>
const input = document.getElementById('avatar');
const preview = document.getElementById('preview');

input.addEventListener('change', (e) => {
  const file = e.target.files[0];
  
  if (file) {
    const reader = new FileReader();
    
    reader.onload = (e) => {
      preview.src = e.target.result;
      preview.style.display = 'block';
    };
    
    reader.readAsDataURL(file);
  }
});
</script>
```

---

### Custom Styled File Input

```html
<input type="file" id="file-upload" name="file" hidden>
<label for="file-upload" class="custom-file-upload">
  <svg class="icon" viewBox="0 0 24 24">
    <path d="M9 16h6v-6h4l-7-7-7 7h4zm-4 2h14v2H5z"/>
  </svg>
  Choose File
</label>
<span id="file-name">No file chosen</span>

<style>
.custom-file-upload {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 10px 20px;
  background: #007bff;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  font-weight: 600;
  transition: background 0.2s;
}

.custom-file-upload:hover {
  background: #0056b3;
}

.icon {
  width: 20px;
  height: 20px;
  fill: currentColor;
}

#file-name {
  margin-left: 12px;
  color: #666;
  font-size: 14px;
}
</style>

<script>
const input = document.getElementById('file-upload');
const fileName = document.getElementById('file-name');

input.addEventListener('change', (e) => {
  const file = e.target.files[0];
  fileName.textContent = file ? file.name : 'No file chosen';
});
</script>
```

---

## Multiple File Upload

### Basic Implementation

```html
<form id="multi-upload-form">
  <label for="photos">Select photos:</label>
  <input type="file" id="photos" name="photos" accept="image/*" multiple required>
  
  <div id="file-list"></div>
  
  <button type="submit">Upload All</button>
</form>

<div id="status"></div>

<script>
const form = document.getElementById('multi-upload-form');
const input = document.getElementById('photos');
const fileList = document.getElementById('file-list');
const status = document.getElementById('status');

// Show selected files
input.addEventListener('change', (e) => {
  const files = Array.from(e.target.files);
  
  fileList.innerHTML = `
    <h3>Selected files (${files.length}):</h3>
    <ul>
      ${files.map(file => `
        <li>
          ${file.name} - ${formatBytes(file.size)}
        </li>
      `).join('')}
    </ul>
  `;
});

// Upload files
form.addEventListener('submit', async (e) => {
  e.preventDefault();
  
  const files = Array.from(input.files);
  
  if (files.length === 0) {
    status.textContent = 'Please select at least one file';
    return;
  }
  
  const formData = new FormData();
  files.forEach((file, index) => {
    formData.append('photos', file);
  });
  
  try {
    status.textContent = `Uploading ${files.length} files...`;
    
    const response = await fetch('/api/upload-multiple', {
      method: 'POST',
      body: formData
    });
    
    const result = await response.json();
    status.textContent = `Successfully uploaded ${result.count} files!`;
    
  } catch (error) {
    status.textContent = `Error: ${error.message}`;
  }
});

function formatBytes(bytes) {
  if (bytes === 0) return '0 Bytes';
  const k = 1024;
  const sizes = ['Bytes', 'KB', 'MB', 'GB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return Math.round(bytes / Math.pow(k, i) * 100) / 100 + ' ' + sizes[i];
}
</script>
```

---

### With Thumbnails

```html
<input type="file" id="photos" accept="image/*" multiple>
<div id="thumbnails"></div>

<style>
#thumbnails {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 12px;
  margin-top: 20px;
}

.thumbnail {
  position: relative;
  aspect-ratio: 1;
  border: 2px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
}

.thumbnail img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.thumbnail .remove {
  position: absolute;
  top: 4px;
  right: 4px;
  background: rgba(255, 0, 0, 0.8);
  color: white;
  border: none;
  border-radius: 50%;
  width: 24px;
  height: 24px;
  cursor: pointer;
  font-size: 16px;
  line-height: 1;
}
</style>

<script>
const input = document.getElementById('photos');
const thumbnails = document.getElementById('thumbnails');
let selectedFiles = [];

input.addEventListener('change', (e) => {
  const files = Array.from(e.target.files);
  selectedFiles = files;
  displayThumbnails(files);
});

function displayThumbnails(files) {
  thumbnails.innerHTML = '';
  
  files.forEach((file, index) => {
    const reader = new FileReader();
    
    reader.onload = (e) => {
      const div = document.createElement('div');
      div.className = 'thumbnail';
      div.innerHTML = `
        <img src="${e.target.result}" alt="${file.name}">
        <button class="remove" onclick="removeFile(${index})">&times;</button>
      `;
      thumbnails.appendChild(div);
    };
    
    reader.readAsDataURL(file);
  });
}

function removeFile(index) {
  selectedFiles.splice(index, 1);
  
  // Update file input (requires DataTransfer)
  const dt = new DataTransfer();
  selectedFiles.forEach(file => dt.items.add(file));
  input.files = dt.files;
  
  displayThumbnails(selectedFiles);
}
</script>
```

---

## Drag-and-Drop File Upload

### Basic Drag-and-Drop

```html
<div id="drop-zone">
  <p>Drag files here or <label for="file-input" style="color: blue; cursor: pointer;">browse</label></p>
  <input type="file" id="file-input" multiple hidden>
</div>

<div id="file-list"></div>

<style>
#drop-zone {
  border: 2px dashed #ccc;
  border-radius: 8px;
  padding: 40px;
  text-align: center;
  background: #f9f9f9;
  transition: all 0.3s;
}

#drop-zone.dragover {
  border-color: #007bff;
  background: #e7f3ff;
}

#file-list {
  margin-top: 20px;
}

.file-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  margin-bottom: 8px;
}

.file-item button {
  background: #dc3545;
  color: white;
  border: none;
  padding: 6px 12px;
  border-radius: 4px;
  cursor: pointer;
}
</style>

<script>
const dropZone = document.getElementById('drop-zone');
const fileInput = document.getElementById('file-input');
const fileList = document.getElementById('file-list');
let files = [];

// Prevent default drag behaviors
['dragenter', 'dragover', 'dragleave', 'drop'].forEach(eventName => {
  dropZone.addEventListener(eventName, preventDefaults, false);
  document.body.addEventListener(eventName, preventDefaults, false);
});

function preventDefaults(e) {
  e.preventDefault();
  e.stopPropagation();
}

// Highlight drop zone when dragging over
['dragenter', 'dragover'].forEach(eventName => {
  dropZone.addEventListener(eventName, () => {
    dropZone.classList.add('dragover');
  });
});

['dragleave', 'drop'].forEach(eventName => {
  dropZone.addEventListener(eventName, () => {
    dropZone.classList.remove('dragover');
  });
});

// Handle dropped files
dropZone.addEventListener('drop', (e) => {
  const droppedFiles = e.dataTransfer.files;
  handleFiles(droppedFiles);
});

// Handle file input
fileInput.addEventListener('change', (e) => {
  handleFiles(e.target.files);
});

function handleFiles(newFiles) {
  files = [...files, ...Array.from(newFiles)];
  displayFiles();
}

function displayFiles() {
  fileList.innerHTML = files.map((file, index) => `
    <div class="file-item">
      <span>${file.name} - ${formatBytes(file.size)}</span>
      <button onclick="removeFile(${index})">Remove</button>
    </div>
  `).join('');
}

function removeFile(index) {
  files.splice(index, 1);
  displayFiles();
}

function formatBytes(bytes) {
  if (bytes === 0) return '0 Bytes';
  const k = 1024;
  const sizes = ['Bytes', 'KB', 'MB', 'GB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return Math.round(bytes / Math.pow(k, i) * 100) / 100 + ' ' + sizes[i];
}
</script>
```

---

### Advanced Drag-and-Drop with Preview

```html
<div id="upload-area">
  <div id="drop-zone">
    <svg class="upload-icon" viewBox="0 0 24 24">
      <path d="M9 16h6v-6h4l-7-7-7 7h4zm-4 2h14v2H5z"/>
    </svg>
    <h3>Drag & Drop Files Here</h3>
    <p>or</p>
    <label for="file-input" class="browse-btn">Browse Files</label>
    <input type="file" id="file-input" accept="image/*" multiple hidden>
    <p class="hint">Supports: JPG, PNG, GIF (max 5MB each)</p>
  </div>
  
  <div id="preview-area"></div>
  
  <button id="upload-btn" disabled>Upload Files</button>
</div>

<style>
#upload-area {
  max-width: 800px;
  margin: 0 auto;
}

#drop-zone {
  border: 3px dashed #ccc;
  border-radius: 12px;
  padding: 60px 20px;
  text-align: center;
  background: #fafafa;
  transition: all 0.3s;
}

#drop-zone.dragover {
  border-color: #007bff;
  background: #e7f3ff;
  transform: scale(1.02);
}

.upload-icon {
  width: 64px;
  height: 64px;
  fill: #999;
  margin-bottom: 16px;
}

.browse-btn {
  display: inline-block;
  padding: 12px 24px;
  background: #007bff;
  color: white;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
  transition: background 0.2s;
}

.browse-btn:hover {
  background: #0056b3;
}

.hint {
  margin-top: 16px;
  font-size: 14px;
  color: #666;
}

#preview-area {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 16px;
  margin-top: 24px;
}

.preview-card {
  position: relative;
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
  background: white;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.preview-card img {
  width: 100%;
  height: 200px;
  object-fit: cover;
}

.preview-info {
  padding: 12px;
}

.preview-name {
  font-weight: 600;
  font-size: 14px;
  margin-bottom: 4px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.preview-size {
  font-size: 12px;
  color: #666;
}

.remove-btn {
  position: absolute;
  top: 8px;
  right: 8px;
  background: rgba(220, 53, 69, 0.9);
  color: white;
  border: none;
  border-radius: 50%;
  width: 32px;
  height: 32px;
  cursor: pointer;
  font-size: 20px;
  line-height: 1;
  transition: background 0.2s;
}

.remove-btn:hover {
  background: rgb(220, 53, 69);
}

#upload-btn {
  width: 100%;
  margin-top: 24px;
  padding: 16px;
  background: #28a745;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

#upload-btn:hover:not(:disabled) {
  background: #218838;
}

#upload-btn:disabled {
  background: #ccc;
  cursor: not-allowed;
}
</style>

<script>
const dropZone = document.getElementById('drop-zone');
const fileInput = document.getElementById('file-input');
const previewArea = document.getElementById('preview-area');
const uploadBtn = document.getElementById('upload-btn');
let selectedFiles = [];

// Drag-and-drop handlers
['dragenter', 'dragover', 'dragleave', 'drop'].forEach(eventName => {
  dropZone.addEventListener(eventName, preventDefaults, false);
});

function preventDefaults(e) {
  e.preventDefault();
  e.stopPropagation();
}

['dragenter', 'dragover'].forEach(eventName => {
  dropZone.addEventListener(eventName, () => dropZone.classList.add('dragover'));
});

['dragleave', 'drop'].forEach(eventName => {
  dropZone.addEventListener(eventName, () => dropZone.classList.remove('dragover'));
});

dropZone.addEventListener('drop', (e) => {
  const files = Array.from(e.dataTransfer.files).filter(file => 
    file.type.startsWith('image/')
  );
  addFiles(files);
});

fileInput.addEventListener('change', (e) => {
  addFiles(Array.from(e.target.files));
});

function addFiles(files) {
  files.forEach(file => {
    // Validate
    if (file.size > 5 * 1024 * 1024) {
      alert(`${file.name} is too large (max 5MB)`);
      return;
    }
    
    if (selectedFiles.some(f => f.name === file.name && f.size === file.size)) {
      alert(`${file.name} is already added`);
      return;
    }
    
    selectedFiles.push(file);
  });
  
  displayPreviews();
  uploadBtn.disabled = selectedFiles.length === 0;
}

function displayPreviews() {
  previewArea.innerHTML = '';
  
  selectedFiles.forEach((file, index) => {
    const reader = new FileReader();
    
    reader.onload = (e) => {
      const card = document.createElement('div');
      card.className = 'preview-card';
      card.innerHTML = `
        <img src="${e.target.result}" alt="${file.name}">
        <button class="remove-btn" onclick="removeFile(${index})">&times;</button>
        <div class="preview-info">
          <div class="preview-name" title="${file.name}">${file.name}</div>
          <div class="preview-size">${formatBytes(file.size)}</div>
        </div>
      `;
      previewArea.appendChild(card);
    };
    
    reader.readAsDataURL(file);
  });
}

function removeFile(index) {
  selectedFiles.splice(index, 1);
  displayPreviews();
  uploadBtn.disabled = selectedFiles.length === 0;
}

uploadBtn.addEventListener('click', async () => {
  const formData = new FormData();
  selectedFiles.forEach(file => formData.append('photos', file));
  
  try {
    uploadBtn.textContent = 'Uploading...';
    uploadBtn.disabled = true;
    
    const response = await fetch('/api/upload', {
      method: 'POST',
      body: formData
    });
    
    const result = await response.json();
    alert(`Successfully uploaded ${result.count} files!`);
    
    selectedFiles = [];
    displayPreviews();
    
  } catch (error) {
    alert(`Upload failed: ${error.message}`);
  } finally {
    uploadBtn.textContent = 'Upload Files';
    uploadBtn.disabled = selectedFiles.length === 0;
  }
});

function formatBytes(bytes) {
  if (bytes === 0) return '0 Bytes';
  const k = 1024;
  const sizes = ['Bytes', 'KB', 'MB', 'GB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return Math.round(bytes / Math.pow(k, i) * 100) / 100 + ' ' + sizes[i];
}
</script>
```

---

## File Validation

### Client-Side Validation

**Size validation:**
```javascript
const MAX_SIZE = 5 * 1024 * 1024; // 5MB

fileInput.addEventListener('change', (e) => {
  const file = e.target.files[0];
  
  if (file.size > MAX_SIZE) {
    alert(`File too large! Maximum size is ${formatBytes(MAX_SIZE)}`);
    fileInput.value = ''; // Clear selection
    return;
  }
});
```

**Type validation:**
```javascript
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/gif'];

fileInput.addEventListener('change', (e) => {
  const file = e.target.files[0];
  
  if (!ALLOWED_TYPES.includes(file.type)) {
    alert('Invalid file type! Please upload JPEG, PNG, or GIF.');
    fileInput.value = '';
    return;
  }
});
```

**Image dimension validation:**
```javascript
fileInput.addEventListener('change', (e) => {
  const file = e.target.files[0];
  
  if (!file.type.startsWith('image/')) return;
  
  const img = new Image();
  img.onload = () => {
    const maxWidth = 4000;
    const maxHeight = 4000;
    
    if (img.width > maxWidth || img.height > maxHeight) {
      alert(`Image too large! Maximum dimensions: ${maxWidth}x${maxHeight}`);
      fileInput.value = '';
      return;
    }
    
    console.log(`Valid image: ${img.width}x${img.height}`);
  };
  
  img.src = URL.createObjectURL(file);
});
```

**Combined validation:**
```javascript
function validateFile(file) {
  const errors = [];
  
  // Size
  const maxSize = 5 * 1024 * 1024;
  if (file.size > maxSize) {
    errors.push(`File too large (max ${formatBytes(maxSize)})`);
  }
  
  // Type
  const allowedTypes = ['image/jpeg', 'image/png', 'image/gif'];
  if (!allowedTypes.includes(file.type)) {
    errors.push('Invalid file type (JPEG, PNG, or GIF only)');
  }
  
  // Name
  if (file.name.length > 255) {
    errors.push('Filename too long (max 255 characters)');
  }
  
  return errors;
}

fileInput.addEventListener('change', (e) => {
  const file = e.target.files[0];
  const errors = validateFile(file);
  
  if (errors.length > 0) {
    alert('Validation errors:\n' + errors.join('\n'));
    fileInput.value = '';
  }
});
```

---

## Upload Progress

### XMLHttpRequest with Progress

```html
<form id="upload-form">
  <input type="file" id="file-input" required>
  <button type="submit">Upload</button>
</form>

<div id="progress-container" style="display: none;">
  <progress id="progress-bar" value="0" max="100"></progress>
  <span id="progress-text">0%</span>
</div>

<div id="status"></div>

<style>
#progress-container {
  margin-top: 20px;
}

#progress-bar {
  width: 100%;
  height: 30px;
}

#progress-text {
  display: block;
  margin-top: 8px;
  font-weight: 600;
}
</style>

<script>
const form = document.getElementById('upload-form');
const fileInput = document.getElementById('file-input');
const progressContainer = document.getElementById('progress-container');
const progressBar = document.getElementById('progress-bar');
const progressText = document.getElementById('progress-text');
const status = document.getElementById('status');

form.addEventListener('submit', (e) => {
  e.preventDefault();
  
  const file = fileInput.files[0];
  if (!file) return;
  
  const formData = new FormData();
  formData.append('file', file);
  
  const xhr = new XMLHttpRequest();
  
  // Track upload progress
  xhr.upload.addEventListener('progress', (e) => {
    if (e.lengthComputable) {
      const percentComplete = (e.loaded / e.total) * 100;
      progressBar.value = percentComplete;
      progressText.textContent = `${percentComplete.toFixed(0)}% (${formatBytes(e.loaded)} / ${formatBytes(e.total)})`;
    }
  });
  
  // Upload started
  xhr.upload.addEventListener('loadstart', () => {
    progressContainer.style.display = 'block';
    status.textContent = 'Uploading...';
  });
  
  // Upload completed
  xhr.addEventListener('load', () => {
    if (xhr.status === 200) {
      status.textContent = 'Upload successful!';
      setTimeout(() => {
        progressContainer.style.display = 'none';
        progressBar.value = 0;
      }, 2000);
    } else {
      status.textContent = `Upload failed: ${xhr.statusText}`;
    }
  });
  
  // Upload failed
  xhr.addEventListener('error', () => {
    status.textContent = 'Upload error!';
    progressContainer.style.display = 'none';
  });
  
  // Upload aborted
  xhr.addEventListener('abort', () => {
    status.textContent = 'Upload canceled';
    progressContainer.style.display = 'none';
  });
  
  xhr.open('POST', '/api/upload');
  xhr.send(formData);
});

function formatBytes(bytes) {
  if (bytes === 0) return '0 Bytes';
  const k = 1024;
  const sizes = ['Bytes', 'KB', 'MB', 'GB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return Math.round(bytes / Math.pow(k, i) * 100) / 100 + ' ' + sizes[i];
}
</script>
```

---

## Security Considerations

### Client-Side Security

**1. Validate file types:**
```javascript
// ❌ Don't trust file extension
if (file.name.endsWith('.jpg')) { ... }

// ✅ Check MIME type (still not foolproof)
if (file.type === 'image/jpeg') { ... }

// ✅ Best: Validate on server by reading file headers
```

**2. Limit file size:**
```javascript
const MAX_SIZE = 10 * 1024 * 1024; // 10MB
if (file.size > MAX_SIZE) {
  alert('File too large');
  return;
}
```

**3. Sanitize filenames:**
```javascript
function sanitizeFilename(filename) {
  return filename
    .replace(/[^a-zA-Z0-9._-]/g, '_') // Replace special chars
    .substring(0, 255); // Limit length
}
```

**4. Preview safely:**
```javascript
// ✅ Use blob URL (revoked after use)
const url = URL.createObjectURL(file);
img.src = url;
img.onload = () => URL.revokeObjectURL(url);

// ✅ Or use data URL
const reader = new FileReader();
reader.onload = (e) => img.src = e.target.result;
reader.readAsDataURL(file);
```

---

### Server-Side Security (Guidelines)

**1. Validate MIME type by reading file headers:**
```javascript
// Example: Check for JPEG magic number
const buffer = await file.arrayBuffer();
const bytes = new Uint8Array(buffer);

// JPEG starts with FF D8 FF
if (bytes[0] === 0xFF && bytes[1] === 0xD8 && bytes[2] === 0xFF) {
  console.log('Valid JPEG');
}
```

**2. Scan for malware** - Use antivirus API

**3. Store files outside web root** - Prevent direct execution

**4. Generate unique filenames** - Prevent overwrites and directory traversal

**5. Set proper Content-Type headers** - When serving files

**6. Implement rate limiting** - Prevent abuse

---

## Common Mistakes

❌ **Missing `enctype`**
```html
<!-- Wrong: Files won't upload -->
<form method="POST">
  <input type="file" name="file">
</form>

<!-- Correct -->
<form method="POST" enctype="multipart/form-data">
  <input type="file" name="file">
</form>
```

❌ **Not validating file size**
```javascript
// Wrong: Upload huge file, waste bandwidth
await fetch('/upload', { method: 'POST', body: formData });

// Correct: Validate first
if (file.size > MAX_SIZE) {
  alert('File too large');
  return;
}
```

❌ **Trusting file extensions**
```javascript
// Wrong: Easy to bypass
if (file.name.endsWith('.jpg')) { ... }

// Better: Check MIME type
if (file.type === 'image/jpeg') { ... }

// Best: Validate on server
```

❌ **Not revoking blob URLs**
```javascript
// Wrong: Memory leak
const url = URL.createObjectURL(file);
img.src = url;

// Correct: Revoke when done
img.onload = () => URL.revokeObjectURL(url);
```

❌ **Blocking UI during upload**
```javascript
// Wrong: Freezes UI
const data = await file.arrayBuffer(); // Blocking
await fetch('/upload', { body: data });

// Correct: Show progress
const xhr = new XMLHttpRequest();
xhr.upload.onprogress = (e) => updateProgress(e);
```

---

## Best Practices

✅ **Always validate files** - Size, type, dimensions (client AND server)

✅ **Provide clear feedback** - Show selected files, upload progress

✅ **Support both click and drag-and-drop** - Better UX

✅ **Show previews for images** - Confirm correct file selected

✅ **Allow file removal** - Let users change their mind

✅ **Handle errors gracefully** - Network errors, invalid files, server errors

✅ **Use appropriate `accept` attribute** - Helps users select correct files

✅ **Limit file size** - Prevent huge uploads

✅ **Use FormData for uploads** - Simplest approach

✅ **Compress images client-side** - Faster uploads (use canvas or libraries)

✅ **Implement retry logic** - Handle network failures

✅ **Test on mobile** - Touch interfaces, camera integration

---

## Resources

- [MDN: input type="file"](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/file)
- [MDN: File API](https://developer.mozilla.org/en-US/docs/Web/API/File_API)
- [MDN: FileReader](https://developer.mozilla.org/en-US/docs/Web/API/FileReader)
- [MDN: Drag and Drop API](https://developer.mozilla.org/en-US/docs/Web/API/HTML_Drag_and_Drop_API)
- [web.dev: File upload best practices](https://web.dev/file-upload-best-practices/)
- [OWASP: File Upload Security](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
