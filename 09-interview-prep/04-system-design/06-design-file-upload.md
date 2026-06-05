# Design Resumable/Chunked File Upload

## The Idea

**In plain English:** Chunked file upload is a way of sending a large file to a server by breaking it into many small pieces and sending each piece separately, so that if something goes wrong you only need to resend the missing pieces rather than starting over from scratch.

**Real-world analogy:** Imagine mailing a very long novel to a friend by tearing out each chapter, putting each chapter in its own envelope, and mailing them all separately. If one envelope gets lost in the mail, you only resend that one chapter — not the whole book.

- The novel = the large file you want to upload
- Each chapter in its own envelope = a chunk (a fixed-size slice of the file sent as one request)
- The post office tracking number on each envelope = the chunk index (a number that tells the server which piece this is and where it fits)

---

## Why Chunked Uploads?

For large files (videos, archives, datasets):
- Single request fails → user must restart from zero
- Browser/proxy timeouts kill long uploads
- No progress visibility for the user
- Server memory: loading 4 GB file into memory crashes the process

---

## Architecture Overview

```
Client                          Server
  ├── Split file into chunks      ├── Receive chunk + metadata
  ├── Track upload state          ├── Store chunk temporarily
  ├── Parallel chunk uploads      ├── Track received chunks
  ├── Retry failed chunks         ├── Reassemble when all arrived
  └── Show progress               └── Store final file
```

---

## Chunking Strategy

```ts
const CHUNK_SIZE = 5 * 1024 * 1024; // 5 MB per chunk

async function uploadFile(file: File) {
  const totalChunks = Math.ceil(file.size / CHUNK_SIZE);
  const uploadId = await initiateUpload(file.name, file.size, totalChunks);

  const chunks = Array.from({ length: totalChunks }, (_, i) => ({
    index: i,
    start: i * CHUNK_SIZE,
    end: Math.min((i + 1) * CHUNK_SIZE, file.size),
  }));

  return chunks;
}
```

---

## Parallel Uploads with Concurrency Control

```ts
async function uploadChunks(
  file: File,
  chunks: ChunkInfo[],
  uploadId: string,
  onProgress: (pct: number) => void
) {
  const CONCURRENCY = 3; // upload 3 chunks at a time
  const uploaded = new Set<number>();
  let uploadedBytes = 0;

  async function uploadChunk(chunk: ChunkInfo): Promise<void> {
    const blob = file.slice(chunk.start, chunk.end);
    const formData = new FormData();
    formData.append('uploadId', uploadId);
    formData.append('chunkIndex', String(chunk.index));
    formData.append('chunk', blob);

    await fetchWithRetry(`/api/upload/chunk`, {
      method: 'POST',
      body: formData,
    });

    uploaded.add(chunk.index);
    uploadedBytes += chunk.end - chunk.start;
    onProgress(uploadedBytes / file.size);
  }

  // Process chunks CONCURRENCY at a time
  for (let i = 0; i < chunks.length; i += CONCURRENCY) {
    const batch = chunks.slice(i, i + CONCURRENCY);
    await Promise.all(batch.map(uploadChunk));
  }
}
```

---

## Retry with Exponential Backoff

```ts
async function fetchWithRetry(
  url: string,
  options: RequestInit,
  maxRetries = 3
): Promise<Response> {
  let lastError: Error;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);
      if (response.ok) return response;
      throw new Error(`HTTP ${response.status}`);
    } catch (error) {
      lastError = error as Error;
      if (attempt < maxRetries - 1) {
        const delay = Math.pow(2, attempt) * 1000 + Math.random() * 1000; // jitter
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }

  throw lastError!;
}
```

---

## Resumability

Store upload state in localStorage so users can resume after page reload:

```ts
interface UploadState {
  uploadId: string;
  fileName: string;
  fileSize: number;
  fileLastModified: number;
  uploadedChunks: number[];
}

function saveUploadState(state: UploadState) {
  localStorage.setItem(`upload-${state.fileName}`, JSON.stringify(state));
}

async function resumeOrStart(file: File) {
  const savedState = getSavedUploadState(file);

  if (savedState && savedState.fileLastModified === file.lastModified) {
    // Resume: only upload missing chunks
    const missingChunks = getAllChunks(file).filter(
      c => !savedState.uploadedChunks.includes(c.index)
    );
    await uploadChunks(file, missingChunks, savedState.uploadId, onProgress);
  } else {
    // Start fresh
    const uploadId = await initiateUpload(file);
    await uploadChunks(file, getAllChunks(file), uploadId, onProgress);
  }
}
```

---

## React Upload Component

```tsx
function FileUpload() {
  const [progress, setProgress] = useState(0);
  const [status, setStatus] = useState<'idle' | 'uploading' | 'done' | 'error'>('idle');
  const abortRef = useRef<AbortController | null>(null);

  async function handleFileSelect(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;

    abortRef.current = new AbortController();
    setStatus('uploading');

    try {
      await resumeOrStart(file, {
        onProgress: setProgress,
        signal: abortRef.current.signal,
      });
      setStatus('done');
    } catch (err) {
      if (err.name === 'AbortError') setStatus('idle');
      else setStatus('error');
    }
  }

  return (
    <div>
      <input type="file" onChange={handleFileSelect} disabled={status === 'uploading'} />
      {status === 'uploading' && (
        <>
          <progress value={progress} max={1} />
          <button onClick={() => abortRef.current?.abort()}>Cancel</button>
        </>
      )}
    </div>
  );
}
```

---

## Direct-to-S3 Multipart Upload

For production at scale, skip your server for the file bytes:

```ts
// 1. Backend generates presigned URLs for each part
const { uploadId, presignedUrls } = await api.initMultipartUpload({
  filename: file.name,
  parts: totalChunks,
});

// 2. Client uploads directly to S3 (no server in the middle)
const etags = await Promise.all(
  presignedUrls.map(async ({ url, partNumber }) => {
    const chunk = file.slice((partNumber - 1) * CHUNK_SIZE, partNumber * CHUNK_SIZE);
    const response = await fetch(url, { method: 'PUT', body: chunk });
    return { partNumber, etag: response.headers.get('ETag') };
  })
);

// 3. Tell backend to complete the multipart upload
await api.completeMultipartUpload({ uploadId, parts: etags });
```

---

## Interview Discussion Points

**Q: What chunk size is optimal?**
5 MB is a common default. Smaller chunks = more HTTP overhead + more parallelism. Larger chunks = fewer requests + higher memory per request. Match S3's minimum (5 MB for multipart except last chunk).

**Q: How do you show upload speed (MB/s)?**
Track bytes uploaded over time: `speed = (bytesUploadedSince1sAgo) / 1`. Use a sliding window for smooth display.

**Q: How do you prevent duplicate chunk uploads?**
Server tracks received chunks by (uploadId, chunkIndex). If a chunk is re-sent (retry), the server can detect it's already received and return success without storing again.
