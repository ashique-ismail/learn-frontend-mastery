# File Upload and Download

## Overview

File uploads and downloads are common requirements in modern web applications. Angular's HttpClient provides comprehensive support for handling file operations, including multipart form uploads, progress tracking, streaming downloads, and handling various file types. Understanding FormData, Blob, File APIs, and HTTP progress events is essential for implementing robust file handling.

This guide covers uploading files with progress tracking, downloading files as Blobs, handling multiple file uploads, managing large files with streaming, implementing drag-and-drop uploads, and providing user feedback during long-running file operations.

## Core Concepts

### 1. Basic File Upload

Upload files using `FormData` to send multipart/form-data requests.

**Basic File Upload:**

```typescript
// file-upload.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpHeaders } from '@angular/common/http';
import { Observable } from 'rxjs';

interface UploadResponse {
  fileId: string;
  fileName: string;
  fileSize: number;
  url: string;
}

@Injectable({
  providedIn: 'root'
})
export class FileUploadService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  // Simple file upload
  uploadFile(file: File): Observable<UploadResponse> {
    const formData = new FormData();
    formData.append('file', file);

    return this.http.post<UploadResponse>(
      `${this.apiUrl}/upload`,
      formData
    );
  }

  // Upload with additional metadata
  uploadFileWithMetadata(file: File, metadata: any): Observable<UploadResponse> {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('title', metadata.title);
    formData.append('description', metadata.description);
    formData.append('category', metadata.category);

    return this.http.post<UploadResponse>(
      `${this.apiUrl}/upload`,
      formData
    );
  }

  // Upload with custom filename
  uploadFileWithName(file: File, customName: string): Observable<UploadResponse> {
    const formData = new FormData();
    formData.append('file', file, customName);

    return this.http.post<UploadResponse>(
      `${this.apiUrl}/upload`,
      formData
    );
  }

  // Multiple files upload
  uploadMultipleFiles(files: File[]): Observable<UploadResponse[]> {
    const formData = new FormData();
    
    files.forEach((file, index) => {
      formData.append(`files[${index}]`, file);
    });

    return this.http.post<UploadResponse[]>(
      `${this.apiUrl}/upload/multiple`,
      formData
    );
  }

  // Upload with JSON metadata
  uploadWithJsonMetadata(file: File, metadata: any): Observable<UploadResponse> {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('metadata', JSON.stringify(metadata));

    return this.http.post<UploadResponse>(
      `${this.apiUrl}/upload`,
      formData
    );
  }
}

// Component usage
@Component({
  selector: 'app-file-upload',
  template: `
    <input type="file" (change)="onFileSelected($event)" #fileInput>
    <button (click)="upload()" [disabled]="!selectedFile">Upload</button>
    
    <div *ngIf="uploading">Uploading...</div>
    <div *ngIf="uploadSuccess">Upload successful!</div>
  `
})
export class FileUploadComponent {
  private uploadService = inject(FileUploadService);
  
  selectedFile: File | null = null;
  uploading = false;
  uploadSuccess = false;

  onFileSelected(event: Event): void {
    const input = event.target as HTMLInputElement;
    if (input.files && input.files.length > 0) {
      this.selectedFile = input.files[0];
    }
  }

  upload(): void {
    if (!this.selectedFile) return;

    this.uploading = true;
    this.uploadService.uploadFile(this.selectedFile).subscribe({
      next: (response) => {
        console.log('Upload successful:', response);
        this.uploadSuccess = true;
        this.uploading = false;
      },
      error: (error) => {
        console.error('Upload failed:', error);
        this.uploading = false;
      }
    });
  }
}
```

### 2. Progress Tracking

Track upload/download progress using `reportProgress` and HTTP events.

**Progress Tracking:**

```typescript
// file-upload-progress.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpEvent, HttpEventType, HttpProgressEvent } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map, tap } from 'rxjs/operators';

interface UploadProgress {
  progress: number;
  loaded: number;
  total: number;
  state: 'pending' | 'uploading' | 'complete' | 'error';
}

@Injectable({
  providedIn: 'root'
})
export class FileUploadProgressService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  // Upload with raw progress events
  uploadWithEvents(file: File): Observable<HttpEvent<any>> {
    const formData = new FormData();
    formData.append('file', file);

    return this.http.post(`${this.apiUrl}/upload`, formData, {
      reportProgress: true,
      observe: 'events'
    });
  }

  // Upload with progress percentage
  uploadWithProgress(file: File): Observable<number> {
    const formData = new FormData();
    formData.append('file', file);

    return this.http.post(`${this.apiUrl}/upload`, formData, {
      reportProgress: true,
      observe: 'events'
    }).pipe(
      map(event => {
        if (event.type === HttpEventType.UploadProgress) {
          const progress = event.total
            ? Math.round((100 * event.loaded) / event.total)
            : 0;
          return progress;
        }
        return event.type === HttpEventType.Response ? 100 : 0;
      })
    );
  }

  // Upload with detailed progress info
  uploadWithDetailedProgress(file: File): Observable<UploadProgress> {
    const formData = new FormData();
    formData.append('file', file);

    return this.http.post(`${this.apiUrl}/upload`, formData, {
      reportProgress: true,
      observe: 'events'
    }).pipe(
      map(event => {
        switch (event.type) {
          case HttpEventType.Sent:
            return {
              progress: 0,
              loaded: 0,
              total: 0,
              state: 'pending' as const
            };

          case HttpEventType.UploadProgress:
            const progress = event.total
              ? Math.round((100 * event.loaded) / event.total)
              : 0;
            return {
              progress,
              loaded: event.loaded,
              total: event.total || 0,
              state: 'uploading' as const
            };

          case HttpEventType.Response:
            return {
              progress: 100,
              loaded: event.body?.fileSize || 0,
              total: event.body?.fileSize || 0,
              state: 'complete' as const
            };

          default:
            return {
              progress: 0,
              loaded: 0,
              total: 0,
              state: 'pending' as const
            };
        }
      })
    );
  }

  // Multiple files with individual progress
  uploadMultipleWithProgress(
    files: File[]
  ): Observable<{ file: File; progress: UploadProgress }[]> {
    const uploads = files.map(file => ({
      file,
      upload$: this.uploadWithDetailedProgress(file)
    }));

    return combineLatest(
      uploads.map(({ file, upload$ }) =>
        upload$.pipe(
          map(progress => ({ file, progress }))
        )
      )
    );
  }
}

// Component with progress bar
@Component({
  selector: 'app-upload-progress',
  template: `
    <input type="file" (change)="onFileSelected($event)">
    <button (click)="upload()" [disabled]="!selectedFile || uploading">
      Upload
    </button>

    <div *ngIf="uploading" class="progress-container">
      <div class="progress-bar" [style.width.%]="uploadProgress">
        {{ uploadProgress }}%
      </div>
      <p>{{ uploadedBytes | fileSize }} / {{ totalBytes | fileSize }}</p>
      <p>{{ uploadState }}</p>
    </div>

    <div *ngIf="uploadComplete">
      Upload complete! ✓
    </div>
  `,
  styles: [`
    .progress-container {
      margin: 20px 0;
    }
    .progress-bar {
      height: 30px;
      background: #4CAF50;
      transition: width 0.3s;
      display: flex;
      align-items: center;
      justify-content: center;
      color: white;
    }
  `]
})
export class UploadProgressComponent {
  private uploadService = inject(FileUploadProgressService);
  
  selectedFile: File | null = null;
  uploading = false;
  uploadComplete = false;
  uploadProgress = 0;
  uploadedBytes = 0;
  totalBytes = 0;
  uploadState = 'pending';

  onFileSelected(event: Event): void {
    const input = event.target as HTMLInputElement;
    if (input.files && input.files.length > 0) {
      this.selectedFile = input.files[0];
    }
  }

  upload(): void {
    if (!this.selectedFile) return;

    this.uploading = true;
    this.uploadComplete = false;

    this.uploadService.uploadWithDetailedProgress(this.selectedFile).subscribe({
      next: (progressInfo) => {
        this.uploadProgress = progressInfo.progress;
        this.uploadedBytes = progressInfo.loaded;
        this.totalBytes = progressInfo.total;
        this.uploadState = progressInfo.state;

        if (progressInfo.state === 'complete') {
          this.uploading = false;
          this.uploadComplete = true;
        }
      },
      error: (error) => {
        console.error('Upload failed:', error);
        this.uploading = false;
        this.uploadState = 'error';
      }
    });
  }
}
```

### 3. File Download

Download files as Blobs with various file types.

**File Download:**

```typescript
// file-download.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpResponse } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class FileDownloadService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';

  // Download file as Blob
  downloadFile(fileId: string): Observable<Blob> {
    return this.http.get(
      `${this.apiUrl}/files/${fileId}`,
      { responseType: 'blob' }
    );
  }

  // Download with filename from response headers
  downloadFileWithName(fileId: string): Observable<{ blob: Blob; filename: string }> {
    return this.http.get(
      `${this.apiUrl}/files/${fileId}`,
      {
        responseType: 'blob',
        observe: 'response'
      }
    ).pipe(
      map((response: HttpResponse<Blob>) => {
        const filename = this.getFilenameFromHeaders(response.headers);
        return {
          blob: response.body!,
          filename: filename || 'download'
        };
      })
    );
  }

  // Download specific file types
  downloadPdf(fileId: string): Observable<Blob> {
    return this.http.get(
      `${this.apiUrl}/files/${fileId}/pdf`,
      {
        responseType: 'blob',
        headers: { 'Accept': 'application/pdf' }
      }
    );
  }

  downloadImage(fileId: string): Observable<Blob> {
    return this.http.get(
      `${this.apiUrl}/files/${fileId}/image`,
      {
        responseType: 'blob',
        headers: { 'Accept': 'image/*' }
      }
    );
  }

  downloadZip(fileId: string): Observable<Blob> {
    return this.http.get(
      `${this.apiUrl}/files/${fileId}/zip`,
      {
        responseType: 'blob',
        headers: { 'Accept': 'application/zip' }
      }
    );
  }

  // Download and save file
  downloadAndSave(fileId: string, filename: string): void {
    this.downloadFile(fileId).subscribe({
      next: (blob) => {
        this.saveBlob(blob, filename);
      },
      error: (error) => {
        console.error('Download failed:', error);
      }
    });
  }

  // Helper: Save blob as file
  private saveBlob(blob: Blob, filename: string): void {
    const url = window.URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = filename;
    link.click();
    window.URL.revokeObjectURL(url);
  }

  // Helper: Extract filename from Content-Disposition header
  private getFilenameFromHeaders(headers: HttpHeaders): string | null {
    const contentDisposition = headers.get('Content-Disposition');
    if (!contentDisposition) return null;

    const matches = /filename[^;=\n]*=((['"]).*?\2|[^;\n]*)/.exec(contentDisposition);
    if (matches && matches[1]) {
      return matches[1].replace(/['"]/g, '');
    }
    return null;
  }

  // Download with progress
  downloadWithProgress(fileId: string): Observable<{
    progress: number;
    blob?: Blob;
    complete: boolean;
  }> {
    return this.http.get(
      `${this.apiUrl}/files/${fileId}`,
      {
        responseType: 'blob',
        observe: 'events',
        reportProgress: true
      }
    ).pipe(
      map(event => {
        if (event.type === HttpEventType.DownloadProgress) {
          const progress = event.total
            ? Math.round((100 * event.loaded) / event.total)
            : 0;
          return { progress, complete: false };
        }
        
        if (event.type === HttpEventType.Response) {
          return {
            progress: 100,
            blob: event.body!,
            complete: true
          };
        }
        
        return { progress: 0, complete: false };
      })
    );
  }
}

// Component usage
@Component({
  selector: 'app-file-download',
  template: `
    <button (click)="download()">Download File</button>
    
    <div *ngIf="downloading">
      Downloading... {{ downloadProgress }}%
    </div>
  `
})
export class FileDownloadComponent {
  private downloadService = inject(FileDownloadService);
  
  downloading = false;
  downloadProgress = 0;

  download(): void {
    this.downloading = true;

    this.downloadService.downloadWithProgress('file-123').subscribe({
      next: (state) => {
        this.downloadProgress = state.progress;
        
        if (state.complete && state.blob) {
          this.downloadService['saveBlob'](state.blob, 'document.pdf');
          this.downloading = false;
        }
      },
      error: (error) => {
        console.error('Download failed:', error);
        this.downloading = false;
      }
    });
  }
}
```

### 4. Drag and Drop Upload

Implement drag-and-drop file upload functionality.

**Drag and Drop:**

```typescript
// drag-drop-upload.directive.ts
import { Directive, EventEmitter, HostListener, Output } from '@angular/core';

@Directive({
  selector: '[appDragDropUpload]',
  standalone: true,
  host: {
    '[class.drag-over]': 'isDragOver'
  }
})
export class DragDropUploadDirective {
  @Output() filesDropped = new EventEmitter<File[]>();
  
  isDragOver = false;

  @HostListener('dragover', ['$event'])
  onDragOver(event: DragEvent): void {
    event.preventDefault();
    event.stopPropagation();
    this.isDragOver = true;
  }

  @HostListener('dragleave', ['$event'])
  onDragLeave(event: DragEvent): void {
    event.preventDefault();
    event.stopPropagation();
    this.isDragOver = false;
  }

  @HostListener('drop', ['$event'])
  onDrop(event: DragEvent): void {
    event.preventDefault();
    event.stopPropagation();
    this.isDragOver = false;

    const files = event.dataTransfer?.files;
    if (files && files.length > 0) {
      const fileArray = Array.from(files);
      this.filesDropped.emit(fileArray);
    }
  }
}

// drag-drop-upload.component.ts
@Component({
  selector: 'app-drag-drop-upload',
  standalone: true,
  imports: [CommonModule, DragDropUploadDirective],
  template: `
    <div 
      class="drop-zone"
      appDragDropUpload
      (filesDropped)="onFilesDropped($event)">
      
      <div class="drop-zone-content">
        <svg><!-- Upload icon --></svg>
        <p>Drag and drop files here</p>
        <p>or</p>
        <button (click)="fileInput.click()">Browse Files</button>
        <input 
          #fileInput
          type="file"
          multiple
          (change)="onFileSelected($event)"
          style="display: none">
      </div>

      <div *ngIf="files.length > 0" class="file-list">
        <h3>Selected Files:</h3>
        <div *ngFor="let file of files; let i = index" class="file-item">
          <span>{{ file.name }} ({{ file.size | fileSize }})</span>
          <button (click)="removeFile(i)">Remove</button>
        </div>
      </div>

      <button 
        *ngIf="files.length > 0"
        (click)="uploadAll()"
        [disabled]="uploading">
        Upload All Files
      </button>

      <div *ngIf="uploading" class="upload-progress">
        <div *ngFor="let progress of uploadProgress" class="progress-item">
          <span>{{ progress.filename }}</span>
          <div class="progress-bar">
            <div 
              class="progress-fill" 
              [style.width.%]="progress.progress">
            </div>
          </div>
          <span>{{ progress.progress }}%</span>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .drop-zone {
      border: 2px dashed #ccc;
      border-radius: 8px;
      padding: 40px;
      text-align: center;
      transition: all 0.3s;
    }

    .drop-zone.drag-over {
      border-color: #4CAF50;
      background-color: #f0f8ff;
    }

    .file-list {
      margin-top: 20px;
      text-align: left;
    }

    .file-item {
      display: flex;
      justify-content: space-between;
      padding: 10px;
      border: 1px solid #ddd;
      margin-bottom: 5px;
      border-radius: 4px;
    }

    .progress-item {
      margin-bottom: 10px;
    }

    .progress-bar {
      height: 20px;
      background: #f0f0f0;
      border-radius: 10px;
      overflow: hidden;
      margin: 5px 0;
    }

    .progress-fill {
      height: 100%;
      background: #4CAF50;
      transition: width 0.3s;
    }
  `]
})
export class DragDropUploadComponent {
  private uploadService = inject(FileUploadService);
  
  files: File[] = [];
  uploading = false;
  uploadProgress: { filename: string; progress: number }[] = [];

  onFilesDropped(files: File[]): void {
    this.files.push(...files);
  }

  onFileSelected(event: Event): void {
    const input = event.target as HTMLInputElement;
    if (input.files) {
      this.files.push(...Array.from(input.files));
    }
  }

  removeFile(index: number): void {
    this.files.splice(index, 1);
  }

  uploadAll(): void {
    if (this.files.length === 0) return;

    this.uploading = true;
    this.uploadProgress = this.files.map(file => ({
      filename: file.name,
      progress: 0
    }));

    this.files.forEach((file, index) => {
      this.uploadService.uploadWithProgress(file).subscribe({
        next: (progress) => {
          this.uploadProgress[index].progress = progress;
        },
        error: (error) => {
          console.error(`Upload failed for ${file.name}:`, error);
        },
        complete: () => {
          const allComplete = this.uploadProgress.every(p => p.progress === 100);
          if (allComplete) {
            this.uploading = false;
            this.files = [];
            this.uploadProgress = [];
          }
        }
      });
    });
  }
}
```

### 5. Chunked File Upload

Upload large files in chunks for better reliability and progress tracking.

**Chunked Upload:**

```typescript
// chunked-upload.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, from, concat, last } from 'rxjs';
import { mergeMap, map } from 'rxjs/operators';

interface ChunkUploadResponse {
  chunkIndex: number;
  totalChunks: number;
  uploaded: boolean;
}

interface CompleteUploadResponse {
  fileId: string;
  fileName: string;
  fileSize: number;
}

@Injectable({
  providedIn: 'root'
})
export class ChunkedUploadService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';
  private readonly CHUNK_SIZE = 1024 * 1024; // 1MB chunks

  uploadFileInChunks(
    file: File,
    onProgress?: (progress: number) => void
  ): Observable<CompleteUploadResponse> {
    const totalChunks = Math.ceil(file.size / this.CHUNK_SIZE);
    const chunks = this.createChunks(file, totalChunks);
    const uploadId = this.generateUploadId();

    let uploadedChunks = 0;

    return from(chunks).pipe(
      mergeMap((chunk, index) => {
        return this.uploadChunk(chunk, index, totalChunks, uploadId, file.name).pipe(
          map(response => {
            uploadedChunks++;
            const progress = (uploadedChunks / totalChunks) * 100;
            
            if (onProgress) {
              onProgress(Math.round(progress));
            }

            return response;
          })
        );
      }, 3), // Upload 3 chunks concurrently
      last(),
      mergeMap(() => this.completeUpload(uploadId, file.name))
    );
  }

  private createChunks(file: File, totalChunks: number): Blob[] {
    const chunks: Blob[] = [];
    
    for (let i = 0; i < totalChunks; i++) {
      const start = i * this.CHUNK_SIZE;
      const end = Math.min(start + this.CHUNK_SIZE, file.size);
      chunks.push(file.slice(start, end));
    }

    return chunks;
  }

  private uploadChunk(
    chunk: Blob,
    index: number,
    totalChunks: number,
    uploadId: string,
    filename: string
  ): Observable<ChunkUploadResponse> {
    const formData = new FormData();
    formData.append('chunk', chunk);
    formData.append('chunkIndex', index.toString());
    formData.append('totalChunks', totalChunks.toString());
    formData.append('uploadId', uploadId);
    formData.append('filename', filename);

    return this.http.post<ChunkUploadResponse>(
      `${this.apiUrl}/upload/chunk`,
      formData
    );
  }

  private completeUpload(
    uploadId: string,
    filename: string
  ): Observable<CompleteUploadResponse> {
    return this.http.post<CompleteUploadResponse>(
      `${this.apiUrl}/upload/complete`,
      { uploadId, filename }
    );
  }

  private generateUploadId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  // Resume interrupted upload
  resumeUpload(
    file: File,
    uploadId: string,
    uploadedChunks: number[]
  ): Observable<CompleteUploadResponse> {
    const totalChunks = Math.ceil(file.size / this.CHUNK_SIZE);
    const chunks = this.createChunks(file, totalChunks);
    
    // Filter out already uploaded chunks
    const remainingChunks = chunks.filter((_, index) => 
      !uploadedChunks.includes(index)
    );

    return from(remainingChunks).pipe(
      mergeMap((chunk, relativeIndex) => {
        // Calculate actual chunk index
        const actualIndex = chunks.indexOf(chunk);
        return this.uploadChunk(chunk, actualIndex, totalChunks, uploadId, file.name);
      }, 3),
      last(),
      mergeMap(() => this.completeUpload(uploadId, file.name))
    );
  }
}
```

### 6. Image Upload with Preview

Upload images with client-side preview and validation.

**Image Upload with Preview:**

```typescript
// image-upload.component.ts
import { Component, signal } from '@angular/core';
import { CommonModule } from '@angular/common';

interface ImagePreview {
  file: File;
  url: string;
  width: number;
  height: number;
}

@Component({
  selector: 'app-image-upload',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="image-upload-container">
      <input 
        type="file"
        accept="image/*"
        multiple
        (change)="onImagesSelected($event)"
        #fileInput
        style="display: none">

      <button (click)="fileInput.click()">Select Images</button>

      <div *ngIf="images().length > 0" class="image-grid">
        <div *ngFor="let image of images(); let i = index" class="image-item">
          <img [src]="image.url" [alt]="image.file.name">
          <div class="image-info">
            <p>{{ image.file.name }}</p>
            <p>{{ image.file.size | fileSize }}</p>
            <p>{{ image.width }} x {{ image.height }}</p>
          </div>
          <button (click)="removeImage(i)">Remove</button>
        </div>
      </div>

      <button 
        *ngIf="images().length > 0"
        (click)="uploadAll()"
        [disabled]="uploading()">
        Upload All Images
      </button>

      <div *ngIf="error()" class="error">
        {{ error() }}
      </div>
    </div>
  `,
  styles: [`
    .image-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
      gap: 20px;
      margin: 20px 0;
    }

    .image-item {
      position: relative;
      border: 1px solid #ddd;
      border-radius: 8px;
      padding: 10px;
    }

    .image-item img {
      width: 100%;
      height: 200px;
      object-fit: cover;
      border-radius: 4px;
    }

    .image-info {
      margin: 10px 0;
      font-size: 12px;
    }

    .error {
      color: red;
      margin: 10px 0;
    }
  `]
})
export class ImageUploadComponent {
  private uploadService = inject(FileUploadService);
  
  images = signal<ImagePreview[]>([]);
  uploading = signal(false);
  error = signal('');

  private readonly MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB
  private readonly ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];

  async onImagesSelected(event: Event): Promise<void> {
    const input = event.target as HTMLInputElement;
    if (!input.files) return;

    const files = Array.from(input.files);
    
    for (const file of files) {
      // Validate file
      const validation = this.validateImage(file);
      if (!validation.valid) {
        this.error.set(validation.error || 'Invalid file');
        continue;
      }

      // Create preview
      try {
        const preview = await this.createImagePreview(file);
        this.images.update(images => [...images, preview]);
      } catch (error) {
        this.error.set('Failed to load image preview');
      }
    }

    // Reset input
    input.value = '';
  }

  private validateImage(file: File): { valid: boolean; error?: string } {
    if (!this.ALLOWED_TYPES.includes(file.type)) {
      return {
        valid: false,
        error: `Invalid file type: ${file.type}. Allowed types: JPEG, PNG, GIF, WebP`
      };
    }

    if (file.size > this.MAX_FILE_SIZE) {
      return {
        valid: false,
        error: `File size exceeds maximum of 5MB`
      };
    }

    return { valid: true };
  }

  private createImagePreview(file: File): Promise<ImagePreview> {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      const img = new Image();

      reader.onload = (e) => {
        img.src = e.target?.result as string;
      };

      img.onload = () => {
        resolve({
          file,
          url: img.src,
          width: img.width,
          height: img.height
        });
      };

      img.onerror = reject;
      reader.onerror = reject;

      reader.readAsDataURL(file);
    });
  }

  removeImage(index: number): void {
    this.images.update(images => {
      const newImages = [...images];
      // Revoke object URL to free memory
      URL.revokeObjectURL(newImages[index].url);
      newImages.splice(index, 1);
      return newImages;
    });
  }

  uploadAll(): void {
    if (this.images().length === 0) return;

    this.uploading.set(true);
    this.error.set('');

    const files = this.images().map(img => img.file);
    
    this.uploadService.uploadMultipleFiles(files).subscribe({
      next: (responses) => {
        console.log('Upload successful:', responses);
        this.images.set([]);
        this.uploading.set(false);
      },
      error: (error) => {
        console.error('Upload failed:', error);
        this.error.set('Upload failed. Please try again.');
        this.uploading.set(false);
      }
    });
  }

  ngOnDestroy(): void {
    // Clean up object URLs
    this.images().forEach(img => URL.revokeObjectURL(img.url));
  }
}
```

### 7. File Type Validation

Validate file types and sizes before upload.

**File Validation:**

```typescript
// file-validator.service.ts
import { Injectable } from '@angular/core';

interface ValidationResult {
  valid: boolean;
  errors: string[];
}

interface ValidationRules {
  maxSize?: number;
  minSize?: number;
  allowedTypes?: string[];
  allowedExtensions?: string[];
  maxFiles?: number;
}

@Injectable({
  providedIn: 'root'
})
export class FileValidatorService {
  private readonly DEFAULT_MAX_SIZE = 10 * 1024 * 1024; // 10MB

  private readonly MIME_TYPE_MAP: Record<string, string[]> = {
    'image': ['image/jpeg', 'image/png', 'image/gif', 'image/webp', 'image/svg+xml'],
    'document': ['application/pdf', 'application/msword', 'application/vnd.openxmlformats-officedocument.wordprocessingml.document'],
    'spreadsheet': ['application/vnd.ms-excel', 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'],
    'video': ['video/mp4', 'video/webm', 'video/ogg'],
    'audio': ['audio/mpeg', 'audio/wav', 'audio/ogg']
  };

  validateFile(file: File, rules: ValidationRules = {}): ValidationResult {
    const errors: string[] = [];

    // Size validation
    if (rules.maxSize && file.size > rules.maxSize) {
      errors.push(`File size exceeds maximum of ${this.formatBytes(rules.maxSize)}`);
    }

    if (rules.minSize && file.size < rules.minSize) {
      errors.push(`File size is below minimum of ${this.formatBytes(rules.minSize)}`);
    }

    // Type validation
    if (rules.allowedTypes && !rules.allowedTypes.includes(file.type)) {
      errors.push(`File type ${file.type} is not allowed`);
    }

    // Extension validation
    if (rules.allowedExtensions) {
      const extension = this.getFileExtension(file.name);
      if (!rules.allowedExtensions.includes(extension)) {
        errors.push(`File extension .${extension} is not allowed`);
      }
    }

    return {
      valid: errors.length === 0,
      errors
    };
  }

  validateFiles(files: File[], rules: ValidationRules = {}): ValidationResult {
    const errors: string[] = [];

    // Max files validation
    if (rules.maxFiles && files.length > rules.maxFiles) {
      errors.push(`Maximum ${rules.maxFiles} files allowed`);
    }

    // Validate each file
    files.forEach((file, index) => {
      const result = this.validateFile(file, rules);
      if (!result.valid) {
        errors.push(`File ${index + 1} (${file.name}): ${result.errors.join(', ')}`);
      }
    });

    return {
      valid: errors.length === 0,
      errors
    };
  }

  // Preset validators
  validateImages(files: File[]): ValidationResult {
    return this.validateFiles(files, {
      allowedTypes: this.MIME_TYPE_MAP['image'],
      maxSize: 5 * 1024 * 1024, // 5MB
      maxFiles: 10
    });
  }

  validateDocuments(files: File[]): ValidationResult {
    return this.validateFiles(files, {
      allowedTypes: this.MIME_TYPE_MAP['document'],
      maxSize: 20 * 1024 * 1024, // 20MB
      maxFiles: 5
    });
  }

  validateVideos(files: File[]): ValidationResult {
    return this.validateFiles(files, {
      allowedTypes: this.MIME_TYPE_MAP['video'],
      maxSize: 100 * 1024 * 1024, // 100MB
      maxFiles: 3
    });
  }

  private getFileExtension(filename: string): string {
    return filename.split('.').pop()?.toLowerCase() || '';
  }

  private formatBytes(bytes: number): string {
    if (bytes === 0) return '0 Bytes';
    const k = 1024;
    const sizes = ['Bytes', 'KB', 'MB', 'GB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return Math.round(bytes / Math.pow(k, i) * 100) / 100 + ' ' + sizes[i];
  }

  // Get human-readable error message
  getErrorMessage(result: ValidationResult): string {
    if (result.valid) return '';
    return result.errors.join('; ');
  }
}
```

### 8. Resumable Upload

Implement resumable uploads for large files.

**Resumable Upload:**

```typescript
// resumable-upload.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, throwError, of } from 'rxjs';
import { catchError, retry, tap } from 'rxjs/operators';

interface UploadState {
  uploadId: string;
  filename: string;
  fileSize: number;
  uploadedChunks: number[];
  completed: boolean;
}

@Injectable({
  providedIn: 'root'
})
export class ResumableUploadService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com';
  private readonly STORAGE_KEY = 'resumable_uploads';

  uploadFile(file: File): Observable<any> {
    // Check for existing upload state
    const existingState = this.getUploadState(file.name);
    
    if (existingState && !existingState.completed) {
      console.log('Resuming upload:', existingState.uploadId);
      return this.resumeUpload(file, existingState);
    }

    console.log('Starting new upload');
    return this.startNewUpload(file);
  }

  private startNewUpload(file: File): Observable<any> {
    const uploadId = this.generateUploadId();
    
    // Save initial state
    this.saveUploadState({
      uploadId,
      filename: file.name,
      fileSize: file.size,
      uploadedChunks: [],
      completed: false
    });

    return this.uploadChunks(file, uploadId, []);
  }

  private resumeUpload(file: File, state: UploadState): Observable<any> {
    return this.uploadChunks(file, state.uploadId, state.uploadedChunks);
  }

  private uploadChunks(
    file: File,
    uploadId: string,
    uploadedChunks: number[]
  ): Observable<any> {
    const chunkSize = 1024 * 1024; // 1MB
    const totalChunks = Math.ceil(file.size / chunkSize);
    
    // Upload remaining chunks
    // Implementation similar to chunked upload
    // Update state after each successful chunk
    
    return new Observable(subscriber => {
      let currentChunk = uploadedChunks.length;
      
      const uploadNextChunk = () => {
        if (currentChunk >= totalChunks) {
          this.completeUpload(uploadId, file.name).subscribe({
            next: (response) => {
              this.markUploadComplete(file.name);
              subscriber.next(response);
              subscriber.complete();
            },
            error: (error) => subscriber.error(error)
          });
          return;
        }

        const start = currentChunk * chunkSize;
        const end = Math.min(start + chunkSize, file.size);
        const chunk = file.slice(start, end);

        this.uploadChunk(chunk, currentChunk, uploadId).pipe(
          retry(3),
          catchError(error => {
            // Save progress before failing
            this.updateUploadProgress(file.name, currentChunk);
            return throwError(() => error);
          })
        ).subscribe({
          next: () => {
            this.updateUploadProgress(file.name, currentChunk);
            currentChunk++;
            uploadNextChunk();
          },
          error: (error) => subscriber.error(error)
        });
      };

      uploadNextChunk();
    });
  }

  private uploadChunk(
    chunk: Blob,
    chunkIndex: number,
    uploadId: string
  ): Observable<any> {
    const formData = new FormData();
    formData.append('chunk', chunk);
    formData.append('chunkIndex', chunkIndex.toString());
    formData.append('uploadId', uploadId);

    return this.http.post(`${this.apiUrl}/upload/chunk`, formData);
  }

  private completeUpload(uploadId: string, filename: string): Observable<any> {
    return this.http.post(`${this.apiUrl}/upload/complete`, {
      uploadId,
      filename
    });
  }

  // State management
  private saveUploadState(state: UploadState): void {
    const states = this.getAllUploadStates();
    states[state.filename] = state;
    localStorage.setItem(this.STORAGE_KEY, JSON.stringify(states));
  }

  private getUploadState(filename: string): UploadState | null {
    const states = this.getAllUploadStates();
    return states[filename] || null;
  }

  private updateUploadProgress(filename: string, completedChunk: number): void {
    const state = this.getUploadState(filename);
    if (state) {
      state.uploadedChunks.push(completedChunk);
      this.saveUploadState(state);
    }
  }

  private markUploadComplete(filename: string): void {
    const state = this.getUploadState(filename);
    if (state) {
      state.completed = true;
      this.saveUploadState(state);
    }
  }

  private getAllUploadStates(): Record<string, UploadState> {
    const data = localStorage.getItem(this.STORAGE_KEY);
    return data ? JSON.parse(data) : {};
  }

  private generateUploadId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  // Clean up completed uploads
  clearCompletedUploads(): void {
    const states = this.getAllUploadStates();
    const activeStates: Record<string, UploadState> = {};
    
    Object.entries(states).forEach(([key, state]) => {
      if (!state.completed) {
        activeStates[key] = state;
      }
    });

    localStorage.setItem(this.STORAGE_KEY, JSON.stringify(activeStates));
  }
}
```

### 9. File Size Pipe

Create a pipe to display file sizes in human-readable format.

**File Size Pipe:**

```typescript
// file-size.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'fileSize',
  standalone: true
})
export class FileSizePipe implements PipeTransform {
  transform(bytes: number, decimals: number = 2): string {
    if (bytes === 0) return '0 Bytes';

    const k = 1024;
    const dm = decimals < 0 ? 0 : decimals;
    const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB'];

    const i = Math.floor(Math.log(bytes) / Math.log(k));

    return parseFloat((bytes / Math.pow(k, i)).toFixed(dm)) + ' ' + sizes[i];
  }
}

// Usage in template
// {{ file.size | fileSize }}
// {{ 1024 | fileSize:0 }} // "1 KB"
// {{ 1536 | fileSize:2 }} // "1.50 KB"
```

### 10. Download Manager

Create a download manager for handling multiple downloads.

**Download Manager:**

```typescript
// download-manager.service.ts
import { Injectable, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { forkJoin, Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

interface Download {
  id: string;
  filename: string;
  progress: number;
  status: 'pending' | 'downloading' | 'completed' | 'error' | 'cancelled';
  error?: string;
  cancel$: Subject<void>;
}

@Injectable({
  providedIn: 'root'
})
export class DownloadManagerService {
  private http = inject(HttpClient);
  private downloads = signal<Download[]>([]);

  getDownloads = this.downloads.asReadonly();

  startDownload(fileId: string, filename: string): void {
    const downloadId = this.generateId();
    const cancel$ = new Subject<void>();

    const download: Download = {
      id: downloadId,
      filename,
      progress: 0,
      status: 'pending',
      cancel$
    };

    this.downloads.update(downloads => [...downloads, download]);

    this.http.get(`/api/files/${fileId}`, {
      responseType: 'blob',
      observe: 'events',
      reportProgress: true
    }).pipe(
      takeUntil(cancel$)
    ).subscribe({
      next: (event) => {
        if (event.type === HttpEventType.DownloadProgress) {
          const progress = event.total
            ? Math.round((100 * event.loaded) / event.total)
            : 0;
          
          this.updateDownload(downloadId, {
            progress,
            status: 'downloading'
          });
        }
        
        if (event.type === HttpEventType.Response) {
          this.saveFile(event.body!, filename);
          this.updateDownload(downloadId, {
            progress: 100,
            status: 'completed'
          });
        }
      },
      error: (error) => {
        this.updateDownload(downloadId, {
          status: 'error',
          error: error.message
        });
      }
    });
  }

  cancelDownload(downloadId: string): void {
    const download = this.downloads().find(d => d.id === downloadId);
    if (download) {
      download.cancel$.next();
      download.cancel$.complete();
      this.updateDownload(downloadId, { status: 'cancelled' });
    }
  }

  clearCompleted(): void {
    this.downloads.update(downloads =>
      downloads.filter(d => d.status !== 'completed')
    );
  }

  private updateDownload(id: string, updates: Partial<Download>): void {
    this.downloads.update(downloads =>
      downloads.map(d => d.id === id ? { ...d, ...updates } : d)
    );
  }

  private saveFile(blob: Blob, filename: string): void {
    const url = window.URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = filename;
    link.click();
    window.URL.revokeObjectURL(url);
  }

  private generateId(): string {
    return `download-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

## Common Mistakes

### 1. Not Using FormData for File Uploads

```typescript
// WRONG: Trying to send file directly
this.http.post('/api/upload', file);

// CORRECT: Use FormData
const formData = new FormData();
formData.append('file', file);
this.http.post('/api/upload', formData);
```

### 2. Forgetting reportProgress Option

```typescript
// WRONG: No progress tracking
this.http.post('/api/upload', formData);

// CORRECT: Enable progress tracking
this.http.post('/api/upload', formData, {
  reportProgress: true,
  observe: 'events'
});
```

### 3. Not Setting responseType for Downloads

```typescript
// WRONG: Returns JSON instead of file
this.http.get('/api/file');

// CORRECT: Set responseType to blob
this.http.get('/api/file', { responseType: 'blob' });
```

### 4. Memory Leaks with Object URLs

```typescript
// WRONG: Not revoking object URL
const url = URL.createObjectURL(blob);

// CORRECT: Revoke after use
const url = URL.createObjectURL(blob);
// Use URL...
URL.revokeObjectURL(url);
```

### 5. Not Validating Files

```typescript
// WRONG: Upload without validation
uploadFile(file: File) {
  return this.http.post('/api/upload', formData);
}

// CORRECT: Validate before upload
uploadFile(file: File) {
  if (file.size > MAX_SIZE) {
    throw new Error('File too large');
  }
  if (!ALLOWED_TYPES.includes(file.type)) {
    throw new Error('Invalid file type');
  }
  return this.http.post('/api/upload', formData);
}
```

## Best Practices

### 1. Validate Files Client-Side

```typescript
// Check file size and type before upload
const validation = this.validateFile(file);
if (!validation.valid) {
  throw new Error(validation.error);
}
```

### 2. Show Progress for Large Files

```typescript
// Always show progress for files > 1MB
if (file.size > 1024 * 1024) {
  return this.uploadWithProgress(file);
}
```

### 3. Handle Upload Errors Gracefully

```typescript
this.upload(file).subscribe({
  error: (error) => {
    this.notificationService.error('Upload failed. Please try again.');
    this.logService.error(error);
  }
});
```

### 4. Implement Chunking for Large Files

```typescript
// Use chunked upload for files > 10MB
if (file.size > 10 * 1024 * 1024) {
  return this.uploadInChunks(file);
}
```

### 5. Clean Up Resources

```typescript
ngOnDestroy() {
  // Revoke object URLs
  this.previews.forEach(url => URL.revokeObjectURL(url));
  
  // Cancel pending uploads
  this.cancelSubject.next();
  this.cancelSubject.complete();
}
```

## Interview Questions

**Q: How do you upload files in Angular?**

A: Use `FormData` to create a multipart/form-data request: `const formData = new FormData(); formData.append('file', file);` then POST with HttpClient.

**Q: How do you track upload progress?**

A: Set `reportProgress: true` and `observe: 'events'` in request options, then listen for `HttpEventType.UploadProgress` events which contain `loaded` and `total` bytes.

**Q: How do you download files as Blobs?**

A: Set `responseType: 'blob'` in GET request options: `http.get(url, { responseType: 'blob' })`, then create an object URL and trigger download with an anchor element.

**Q: What is chunked uploading and when should you use it?**

A: Chunked uploading splits large files into smaller pieces and uploads them separately. Use it for files > 10MB to improve reliability, enable resume capability, and provide better progress tracking.

**Q: How do you handle file validation?**

A: Check file size (`file.size`), type (`file.type`), and extension before upload. Use a validation service to enforce rules like maximum size, allowed MIME types, and extension whitelist.

## Key Takeaways

1. Use `FormData` to create multipart/form-data requests for file uploads
2. Enable progress tracking with `reportProgress: true` and `observe: 'events'`
3. Download files as Blobs with `responseType: 'blob'` option
4. Validate files client-side (size, type, extension) before uploading
5. Implement chunked uploads for large files (> 10MB) for reliability
6. Show progress indicators for better user experience during long operations
7. Use drag-and-drop directives for intuitive file selection
8. Always revoke object URLs with `URL.revokeObjectURL()` to prevent memory leaks
9. Implement resumable uploads for large files to handle network interruptions
10. Create preview images for uploaded files using FileReader API

## Resources

- [MDN FormData API](https://developer.mozilla.org/en-US/docs/Web/API/FormData)
- [Angular HttpClient Guide](https://angular.io/guide/http)
- [MDN Blob API](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- [FileReader API](https://developer.mozilla.org/en-US/docs/Web/API/FileReader)
- [HTTP Progress Events](https://angular.io/guide/http-track-and-show-request-progress)
