# AI-Assisted UI: Streaming LLM Responses in the Browser

## Overview

Building AI-powered interfaces requires a fundamentally different mental model than traditional request/response UIs. LLMs produce tokens over time — sometimes 30–100 tokens per second — and blocking the UI until the entire response arrives destroys perceived performance. Streaming responses arrive as Server-Sent Events or chunked HTTP, and the browser must render partial state gracefully. This guide covers the full stack: from `ReadableStream` mechanics to React Server Components with AI, abort controls, optimistic rendering, and production patterns used with Claude and OpenAI APIs.

```
Traditional API call:
Request ─────────────────────────────────────► Response (3–30s wait)
                                                    │
                                               User stares at spinner

Streaming API call:
Request ──► token ─► token ─► token ─► token ─► token ─► [done]
              │         │         │         │
           Render    Render    Render    Render    (incremental UI)
```

## ReadableStream and the Fetch API

Modern streaming from LLM APIs uses chunked HTTP responses with `Transfer-Encoding: chunked` or Server-Sent Events. The `Response.body` is a `ReadableStream<Uint8Array>`.

### Basic Stream Consumption

```typescript
async function streamCompletion(prompt: string): Promise<void> {
  const response = await fetch('/api/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ prompt }),
  });

  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  if (!response.body) throw new Error('No response body');

  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      // value is Uint8Array — decode to string
      const chunk = decoder.decode(value, { stream: true });
      process(chunk); // append to UI
    }
  } finally {
    reader.releaseLock();
  }
}
```

### Using `for await` with AsyncIterable (Cleaner Pattern)

```typescript
// Convert ReadableStream to AsyncIterable
async function* streamToAsyncIterable(
  stream: ReadableStream<Uint8Array>
): AsyncGenerator<string> {
  const reader = stream.getReader();
  const decoder = new TextDecoder();

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      yield decoder.decode(value, { stream: true });
    }
  } finally {
    reader.releaseLock();
  }
}

// Usage — much cleaner with for await
async function streamToUI(prompt: string, onChunk: (text: string) => void) {
  const response = await fetch('/api/chat', {
    method: 'POST',
    body: JSON.stringify({ prompt }),
    headers: { 'Content-Type': 'application/json' },
  });

  for await (const chunk of streamToAsyncIterable(response.body!)) {
    // Parse SSE lines: "data: {...}\n\n"
    const lines = chunk.split('\n').filter((l) => l.startsWith('data: '));
    for (const line of lines) {
      const raw = line.slice(6);
      if (raw === '[DONE]') return;
      try {
        const json = JSON.parse(raw);
        const delta = json.choices?.[0]?.delta?.content ?? '';
        if (delta) onChunk(delta);
      } catch {
        // partial JSON — accumulate if needed
      }
    }
  }
}
```

### Node.js / Edge API Route (Server Side)

```typescript
// app/api/chat/route.ts (Next.js App Router)
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic();

export async function POST(req: Request) {
  const { messages } = await req.json();

  // Create a TransformStream to pipe Anthropic SDK stream to browser
  const encoder = new TextEncoder();
  const stream = new TransformStream<string, Uint8Array>();
  const writer = stream.writable.getWriter();

  // Kick off async work — don't await, let the stream flow
  (async () => {
    try {
      const sdkStream = await anthropic.messages.stream({
        model: 'claude-sonnet-4-5',
        max_tokens: 1024,
        messages,
      });

      for await (const event of sdkStream) {
        if (
          event.type === 'content_block_delta' &&
          event.delta.type === 'text_delta'
        ) {
          const data = `data: ${JSON.stringify({ text: event.delta.text })}\n\n`;
          await writer.write(encoder.encode(data));
        }
      }
      await writer.write(encoder.encode('data: [DONE]\n\n'));
    } finally {
      await writer.close();
    }
  })();

  return new Response(stream.readable, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      Connection: 'keep-alive',
    },
  });
}
```

## Abort Controls for LLM Requests

Users change their mind. Network conditions fluctuate. Always provide a way to stop streaming.

```typescript
// React hook with abort support
import { useState, useRef, useCallback } from 'react';

interface StreamState {
  text: string;
  isStreaming: boolean;
  error: string | null;
}

function useStreamingLLM() {
  const [state, setState] = useState<StreamState>({
    text: '',
    isStreaming: false,
    error: null,
  });
  const abortControllerRef = useRef<AbortController | null>(null);

  const startStream = useCallback(async (prompt: string) => {
    // Cancel any in-flight request
    abortControllerRef.current?.abort();
    abortControllerRef.current = new AbortController();

    setState({ text: '', isStreaming: true, error: null });

    try {
      const response = await fetch('/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ prompt }),
        signal: abortControllerRef.current.signal,
      });

      if (!response.body) throw new Error('No body');

      const reader = response.body.getReader();
      const decoder = new TextDecoder();

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const chunk = decoder.decode(value, { stream: true });
        // Parse SSE and append
        for (const line of chunk.split('\n')) {
          if (!line.startsWith('data: ')) continue;
          const raw = line.slice(6);
          if (raw === '[DONE]') break;
          try {
            const { text } = JSON.parse(raw);
            setState((prev) => ({ ...prev, text: prev.text + text }));
          } catch {}
        }
      }

      setState((prev) => ({ ...prev, isStreaming: false }));
    } catch (err) {
      if ((err as Error).name === 'AbortError') {
        setState((prev) => ({ ...prev, isStreaming: false }));
      } else {
        setState((prev) => ({
          ...prev,
          isStreaming: false,
          error: (err as Error).message,
        }));
      }
    }
  }, []);

  const stopStream = useCallback(() => {
    abortControllerRef.current?.abort();
  }, []);

  return { ...state, startStream, stopStream };
}
```

## Loading States for Streaming

Streaming needs richer loading states than a simple boolean. There are distinct phases:

```
IDLE ──► CONNECTING ──► STREAMING ──► DONE
                                 └──► ERROR
                   └──► ABORTED
```

```typescript
type StreamPhase = 'idle' | 'connecting' | 'streaming' | 'done' | 'error' | 'aborted';

interface StreamingState {
  phase: StreamPhase;
  text: string;
  tokenCount: number;
  startedAt: number | null;
  error: string | null;
}

// Component showing different UI per phase
function AIResponsePanel({ state }: { state: StreamingState }) {
  return (
    <div className="ai-response">
      {state.phase === 'connecting' && (
        <div className="skeleton-pulse">Thinking...</div>
      )}
      
      {(state.phase === 'streaming' || state.phase === 'done') && (
        <>
          <div className="response-text">
            {state.text}
            {state.phase === 'streaming' && (
              <span className="cursor-blink" aria-hidden="true">▋</span>
            )}
          </div>
          {state.phase === 'streaming' && (
            <div className="token-counter">{state.tokenCount} tokens</div>
          )}
        </>
      )}
      
      {state.phase === 'error' && (
        <div className="error-banner" role="alert">
          {state.error}
        </div>
      )}
    </div>
  );
}
```

### Blinking Cursor CSS

```css
.cursor-blink {
  display: inline-block;
  width: 2px;
  background: currentColor;
  animation: blink 1s step-end infinite;
  margin-left: 1px;
}

@keyframes blink {
  0%, 100% { opacity: 1; }
  50%       { opacity: 0; }
}
```

## Optimistic Rendering for AI

Optimistic UI assumes the request succeeds and shows results immediately, rolling back on error. For AI this means:

1. **Show user message immediately** before the server confirms receipt
2. **Show an "AI is typing" placeholder** before the first token arrives
3. **Append tokens as they stream** without waiting for the full response

```typescript
interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  status: 'sending' | 'sent' | 'streaming' | 'done' | 'error';
}

function useChatOptimistic() {
  const [messages, setMessages] = useState<Message[]>([]);

  const sendMessage = async (content: string) => {
    const userMsgId = crypto.randomUUID();
    const asstMsgId = crypto.randomUUID();

    // 1. Optimistically add user message
    setMessages((prev) => [
      ...prev,
      { id: userMsgId, role: 'user', content, status: 'sending' },
    ]);

    // 2. Add empty assistant placeholder immediately
    setMessages((prev) => [
      ...prev,
      { id: asstMsgId, role: 'assistant', content: '', status: 'streaming' },
    ]);

    try {
      const response = await fetch('/api/chat', {
        method: 'POST',
        body: JSON.stringify({ message: content }),
        headers: { 'Content-Type': 'application/json' },
      });

      // Mark user message as sent
      setMessages((prev) =>
        prev.map((m) => (m.id === userMsgId ? { ...m, status: 'sent' } : m))
      );

      const reader = response.body!.getReader();
      const decoder = new TextDecoder();

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const chunk = decoder.decode(value, { stream: true });
        // Parse and append delta
        const delta = parseSSEChunk(chunk);
        if (delta) {
          setMessages((prev) =>
            prev.map((m) =>
              m.id === asstMsgId
                ? { ...m, content: m.content + delta }
                : m
            )
          );
        }
      }

      // Mark done
      setMessages((prev) =>
        prev.map((m) => (m.id === asstMsgId ? { ...m, status: 'done' } : m))
      );
    } catch (err) {
      // Rollback — mark both messages as error
      setMessages((prev) =>
        prev.map((m) =>
          m.id === userMsgId || m.id === asstMsgId
            ? { ...m, status: 'error' }
            : m
        )
      );
    }
  };

  return { messages, sendMessage };
}
```

## React Server Components + AI (Next.js App Router)

React Server Components enable a different pattern: render AI output on the server and stream the HTML/RSC payload. The `ai` package from Vercel and `streamUI` from the AI SDK make this ergonomic.

```typescript
// app/chat/page.tsx — Server Component streaming AI response
import { streamText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { createStreamableUI } from 'ai/rsc';

// Server Action — runs on server, streams RSC back to client
async function generateResponse(prompt: string) {
  'use server';

  const ui = createStreamableUI();

  // Run async, don't block
  (async () => {
    const { textStream } = await streamText({
      model: anthropic('claude-sonnet-4-5'),
      prompt,
    });

    let accumulated = '';
    for await (const delta of textStream) {
      accumulated += delta;
      ui.update(<MarkdownRenderer content={accumulated} />);
    }
    ui.done();
  })();

  return ui.value;
}
```

### AI SDK useChat Hook (Client)

```typescript
'use client';
import { useChat } from 'ai/react';

export function ChatInterface() {
  const {
    messages,
    input,
    handleInputChange,
    handleSubmit,
    isLoading,
    stop,    // abort current stream
    reload,  // retry last message
    error,
  } = useChat({
    api: '/api/chat',
    onFinish: (message) => {
      console.log('Stream complete:', message.content.length, 'chars');
    },
    onError: (err) => {
      console.error('Stream error:', err);
    },
  });

  return (
    <div>
      <ul>
        {messages.map((m) => (
          <li key={m.id} data-role={m.role}>
            {m.content}
          </li>
        ))}
      </ul>

      {isLoading && (
        <button type="button" onClick={stop}>
          Stop generating
        </button>
      )}

      {error && (
        <button type="button" onClick={reload}>
          Retry
        </button>
      )}

      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} />
        <button type="submit" disabled={isLoading}>
          Send
        </button>
      </form>
    </div>
  );
}
```

## Streaming Architecture Patterns

```
Client                   Edge/Server              LLM API
  │                           │                      │
  │──── POST /api/chat ───────►│                      │
  │                           │──── stream req ──────►│
  │◄─── SSE headers ──────────│                      │
  │                           │◄──── token ───────────│
  │◄─── data: {token} ────────│                      │
  │                           │◄──── token ───────────│
  │◄─── data: {token} ────────│                      │
  │                           │◄──── [DONE] ──────────│
  │◄─── data: [DONE] ─────────│                      │
  │                           │                      │
```

### Buffering Partial JSON

LLM APIs often send JSON objects that span multiple chunks. A robust parser buffers:

```typescript
class SSEParser {
  private buffer = '';

  feed(chunk: string): string[] {
    this.buffer += chunk;
    const results: string[] = [];
    
    const lines = this.buffer.split('\n');
    // Keep the last (potentially incomplete) line in the buffer
    this.buffer = lines.pop() ?? '';

    for (const line of lines) {
      if (!line.startsWith('data: ')) continue;
      const data = line.slice(6).trim();
      if (data === '[DONE]') continue;
      try {
        const parsed = JSON.parse(data);
        const text =
          parsed.choices?.[0]?.delta?.content ??   // OpenAI
          parsed.delta?.text ??                     // Anthropic
          parsed.text ??                            // Custom
          '';
        if (text) results.push(text);
      } catch {
        // Malformed — skip
      }
    }

    return results;
  }

  flush(): string {
    const remaining = this.buffer;
    this.buffer = '';
    return remaining;
  }
}
```

## Rate Limiting and Backpressure

Don't slam the UI with thousands of tiny React re-renders. Batch token updates:

```typescript
function useThrottledStream(intervalMs = 50) {
  const [displayText, setDisplayText] = useState('');
  const bufferRef = useRef('');
  const timerRef = useRef<ReturnType<typeof setInterval> | null>(null);

  const appendToken = useCallback((token: string) => {
    bufferRef.current += token;

    if (!timerRef.current) {
      timerRef.current = setInterval(() => {
        if (bufferRef.current) {
          setDisplayText((prev) => prev + bufferRef.current);
          bufferRef.current = '';
        }
      }, intervalMs);
    }
  }, [intervalMs]);

  const flushAndStop = useCallback(() => {
    if (timerRef.current) {
      clearInterval(timerRef.current);
      timerRef.current = null;
    }
    if (bufferRef.current) {
      setDisplayText((prev) => prev + bufferRef.current);
      bufferRef.current = '';
    }
  }, []);

  return { displayText, appendToken, flushAndStop };
}
```

## Error Handling and Retry Logic

```typescript
async function streamWithRetry(
  prompt: string,
  onChunk: (text: string) => void,
  maxRetries = 3
): Promise<void> {
  let attempt = 0;

  while (attempt < maxRetries) {
    try {
      const response = await fetch('/api/chat', {
        method: 'POST',
        body: JSON.stringify({ prompt }),
        headers: { 'Content-Type': 'application/json' },
      });

      if (response.status === 429) {
        // Rate limited — exponential backoff
        const retryAfter = response.headers.get('Retry-After');
        const delay = retryAfter
          ? parseInt(retryAfter) * 1000
          : Math.pow(2, attempt) * 1000;
        await new Promise((r) => setTimeout(r, delay));
        attempt++;
        continue;
      }

      if (!response.ok) throw new Error(`HTTP ${response.status}`);

      // Success — stream
      for await (const chunk of streamToAsyncIterable(response.body!)) {
        onChunk(chunk);
      }
      return;
    } catch (err) {
      if (attempt === maxRetries - 1) throw err;
      attempt++;
      await new Promise((r) => setTimeout(r, Math.pow(2, attempt) * 500));
    }
  }
}
```

## Tool Use / Function Calling in the UI

Modern LLMs support tool calls — the model emits structured JSON instead of prose. Stream these as partial JSON then execute:

```typescript
interface ToolCall {
  id: string;
  name: string;
  input: Record<string, unknown>;
}

interface StreamingMessage {
  text: string;
  toolCalls: ToolCall[];
  isStreamingTool: boolean;
}

async function* streamWithTools(
  messages: Message[]
): AsyncGenerator<StreamingMessage> {
  const state: StreamingMessage = {
    text: '',
    toolCalls: [],
    isStreamingTool: false,
  };

  const sdkStream = anthropic.messages.stream({
    model: 'claude-opus-4-5',
    max_tokens: 4096,
    tools: [
      {
        name: 'search',
        description: 'Search the web',
        input_schema: {
          type: 'object' as const,
          properties: { query: { type: 'string' } },
          required: ['query'],
        },
      },
    ],
    messages,
  });

  for await (const event of sdkStream) {
    if (event.type === 'content_block_start') {
      if (event.content_block.type === 'tool_use') {
        state.isStreamingTool = true;
        state.toolCalls.push({
          id: event.content_block.id,
          name: event.content_block.name,
          input: {},
        });
      }
    } else if (event.type === 'content_block_delta') {
      if (event.delta.type === 'text_delta') {
        state.text += event.delta.text;
      } else if (event.delta.type === 'input_json_delta') {
        // Accumulate partial JSON for current tool call
        const last = state.toolCalls[state.toolCalls.length - 1];
        if (last) {
          // Would merge partial JSON strings here
        }
      }
    }
    yield { ...state };
  }
}
```

## Performance Considerations

### Virtualized Message Lists

Long conversations need virtualization to prevent thousands of DOM nodes:

```typescript
import { Virtualizer } from '@tanstack/react-virtual';

function VirtualizedChat({ messages }: { messages: Message[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = Virtualizer({
    count: messages.length,
    getScrollElement: () => parentRef.current,
    estimateSize: (i) => {
      // Estimate based on content length
      return Math.max(64, messages[i].content.length * 0.5);
    },
    overscan: 5,
  });

  // Auto-scroll to bottom during streaming
  useEffect(() => {
    const last = messages[messages.length - 1];
    if (last?.status === 'streaming') {
      virtualizer.scrollToIndex(messages.length - 1, { behavior: 'smooth' });
    }
  }, [messages.length]);

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map((item) => (
          <div
            key={item.key}
            style={{
              position: 'absolute',
              top: item.start,
              width: '100%',
            }}
          >
            <MessageRow message={messages[item.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

## Security Considerations

```typescript
// Never expose raw API keys client-side
// WRONG:
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY, dangerouslyAllowBrowser: true });

// RIGHT: Always proxy through your server
// client sends to /api/chat → server adds auth → forwards to LLM

// Sanitize AI output before rendering as HTML
import DOMPurify from 'dompurify';
import { marked } from 'marked';

function SafeMarkdown({ content }: { content: string }) {
  const html = DOMPurify.sanitize(marked(content));
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// Validate and limit prompt length
function validatePrompt(prompt: string): string {
  if (prompt.length > 4000) throw new Error('Prompt too long');
  // Strip potential injection attempts in system-level text
  return prompt.trim();
}
```

## Interview Questions

1. **How does `ReadableStream` differ from a Node.js `Readable` stream, and how do you bridge them?**
   Browser `ReadableStream` uses the WHATWG Streams API with `getReader()` and `read()`. Node.js streams are EventEmitter-based. In Node 18+, `Response.body` is a Web `ReadableStream` natively. Use `Readable.fromWeb()` or `pipeTo(new WritableStream(...))` to bridge.

2. **Why should LLM token updates be batched/throttled in React rather than calling setState on every token?**
   Each `setState` schedules a re-render. At 50 tokens/second with a large message list, you would trigger 50 renders/second. React's concurrent mode batches some updates, but explicit throttling (e.g., 60ms intervals) ensures smooth rendering without visual jank and reduces CPU pressure.

3. **What happens to an in-flight `fetch` stream when you call `abortController.abort()`?**
   The fetch is cancelled and the `ReadableStream` is closed. The `read()` Promise rejects with an `AbortError`. The server stops receiving data on its end of the connection (TCP FIN). The LLM API should also be cancelled server-side — many SDKs handle this via the `signal` prop.

4. **Explain the difference between optimistic UI and streaming UI in an AI chat application.**
   Optimistic UI assumes success before server confirmation (shows user message instantly). Streaming UI renders partial server responses as they arrive. They work in concert: optimistic for user messages, streaming for assistant responses.

5. **How do React Server Components change the AI streaming model compared to client-only `useChat`?**
   RSC allows streaming serialized React component trees from the server (not just text). The server can render rich components (charts, formatted code) as part of the LLM response, and the client receives rendered RSC payloads rather than raw text it must render itself.

6. **What are the security risks of rendering LLM output as HTML, and how do you mitigate them?**
   LLMs can be prompt-injected to produce malicious HTML/JS. Mitigation: always sanitize with DOMPurify before `dangerouslySetInnerHTML`, prefer markdown parsers with restricted HTML output, use Content Security Policy to block inline scripts, never render AI output in a privileged security context.

7. **How would you implement "stop generation" without leaving the server streaming forever?**
   Pass `AbortController.signal` to the `fetch` call. When the user clicks stop, call `abort()`. The server's readable stream pipe should respect the client disconnect — in Node.js, `req.signal.aborted` or listening on `req.on('close')`. Also call `stream.controller.abort()` in the Anthropic SDK to cancel the upstream request.

8. **What is backpressure in streaming, and why does it matter for LLM UIs?**
   Backpressure is when the consumer (UI) cannot process data as fast as the producer (LLM API) emits it. For LLM UIs, this manifests as queued renders and potential memory buildup. Solutions: use `ReadableStream`'s built-in backpressure via BYOB readers, or throttle state updates as shown with the interval buffer pattern.
