# WebSockets and Server-Sent Events

## The Idea

**In plain English:** WebSockets and Server-Sent Events (SSE) are two ways for a website and a server to stay in an ongoing conversation — instead of the website having to ask the server a question every time it wants new information, the server can push updates whenever something changes. WebSockets let both sides talk at the same time (like a phone call), while SSE only lets the server talk to the browser (like a radio broadcast).

**Real-world analogy:** Imagine a sports scoreboard app during a live game. The stadium's scoreboard operator (the server) updates the score every time a point is scored, and fans watching on their phones (the browsers) see the new score instantly without refreshing the page.

- The stadium scoreboard operator = the server sending live updates
- The phone app receiving the score = the browser (client) listening for changes
- The live broadcast signal (one-way, operator to fans) = Server-Sent Events
- A two-way radio where fans can also talk back to the operator = WebSockets

---

## Overview

HTTP's request-response model is inherently one-directional: the client initiates, the server responds. Real-time applications need either full-duplex communication (chat, collaborative editing, live games) or efficient server-to-client streaming (live feeds, notifications, dashboards). **WebSockets** provide a persistent, full-duplex TCP channel. **Server-Sent Events (SSE)** provide a lightweight, unidirectional server-to-client stream over standard HTTP. Understanding both—and choosing between them—is a common discussion point in senior frontend and full-stack interviews.

## WebSockets

### Protocol Overview

WebSockets start with an HTTP upgrade handshake:

```http
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After the handshake, the TCP connection stays open and both sides can send framed messages at any time—with low overhead (2-14 byte frame header vs. full HTTP headers).

### WebSocket API

```javascript
// Connect
const ws = new WebSocket('wss://api.example.com/chat'); // wss:// for TLS, ws:// for plain

// Event handlers
ws.onopen = (event) => {
  console.log('Connected:', ws.url);
  ws.send(JSON.stringify({ type: 'auth', token: authToken }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Message:', data);
};

ws.onerror = (event) => {
  // event does not contain detailed error info for security reasons
  console.error('WebSocket error');
};

ws.onclose = (event) => {
  console.log('Closed:', event.code, event.reason, event.wasClean);
  // event.code: 1000=Normal, 1001=Going Away, 1006=Abnormal (no close frame)
  // event.wasClean: true if proper close handshake occurred
};
```

### Sending Data

```javascript
// String (text frame)
ws.send('Hello, server');

// JSON (stringify first)
ws.send(JSON.stringify({ type: 'message', text: 'Hi' }));

// Binary (binary frame)
ws.send(new Uint8Array([1, 2, 3]).buffer); // ArrayBuffer
ws.send(blob);                             // Blob
ws.binaryType = 'arraybuffer';             // 'blob' (default) or 'arraybuffer'
```

### Connection State

```javascript
ws.readyState; // 0=CONNECTING, 1=OPEN, 2=CLOSING, 3=CLOSED

// Guard before sending
function safeSend(ws, data) {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(data);
  }
}
```

### Buffering and Backpressure

```javascript
// ws.bufferedAmount: bytes queued but not yet sent
// Use this to avoid flooding the send buffer
function sendWithBackpressure(ws, data) {
  if (ws.bufferedAmount > 16 * 1024) {
    // 16KB threshold — skip or queue
    console.warn('WebSocket send buffer full, dropping frame');
    return;
  }
  ws.send(data);
}
```

### Reconnection Strategy

The WebSocket API has no built-in reconnect. A production client must implement it:

```javascript
class ReconnectingWebSocket {
  #url;
  #ws = null;
  #retries = 0;
  #maxRetries = 10;
  #handlers = { message: [], open: [], close: [] };
  #closed = false;

  constructor(url) {
    this.#url = url;
    this.#connect();
  }

  #connect() {
    if (this.#closed) return;
    this.#ws = new WebSocket(this.#url);

    this.#ws.onopen = (e) => {
      this.#retries = 0;
      this.#handlers.open.forEach(h => h(e));
    };

    this.#ws.onmessage = (e) => {
      this.#handlers.message.forEach(h => h(e));
    };

    this.#ws.onclose = (e) => {
      this.#handlers.close.forEach(h => h(e));
      if (!this.#closed && this.#retries < this.#maxRetries) {
        const delay = Math.min(1000 * 2 ** this.#retries, 30000); // exponential backoff
        setTimeout(() => this.#connect(), delay);
        this.#retries++;
      }
    };
  }

  send(data) { safeSend(this.#ws, data); }
  on(event, handler) { this.#handlers[event]?.push(handler); }
  close() {
    this.#closed = true;
    this.#ws?.close(1000, 'Client closing');
  }
}
```

### Heartbeat (Ping/Pong)

The WebSocket protocol defines ping/pong frames, but browsers do not expose them via the API. Application-level heartbeats are common:

```javascript
class HeartbeatWS {
  #ws;
  #timer;
  #interval = 30000;

  constructor(url) {
    this.#ws = new WebSocket(url);
    this.#ws.onopen = () => this.#startHeartbeat();
    this.#ws.onmessage = (e) => {
      if (JSON.parse(e.data).type === 'pong') return; // ignore pong
      this.#handleMessage(e);
    };
    this.#ws.onclose = () => clearInterval(this.#timer);
  }

  #startHeartbeat() {
    this.#timer = setInterval(() => {
      if (this.#ws.readyState === WebSocket.OPEN) {
        this.#ws.send(JSON.stringify({ type: 'ping' }));
      }
    }, this.#interval);
  }
}
```

### WebSocket with Subprotocols

```javascript
// Negotiate protocol during handshake
const ws = new WebSocket('wss://api.example.com', ['v2.chat', 'v1.chat']);
ws.onopen = () => console.log('Protocol agreed:', ws.protocol); // 'v2.chat'
```

## Server-Sent Events (SSE)

### Protocol Overview

SSE uses a standard HTTP connection with `Content-Type: text/event-stream`. The server sends a stream of newline-delimited text events. The connection is persistent and the browser automatically reconnects.

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

id: 1
event: message
data: {"text": "Hello"}

id: 2
data: {"text": "Simple message, no event type"}

: this is a comment, ignored by client

retry: 3000

```

### EventSource API

```javascript
// Connect to an SSE endpoint
const es = new EventSource('/api/events', {
  withCredentials: true, // send cookies (cross-origin)
});

// Default event type 'message' (when no event: field)
es.onmessage = (e) => {
  const data = JSON.parse(e.data);
  console.log(e.lastEventId, data);
};

// Named events
es.addEventListener('user:joined', (e) => {
  const user = JSON.parse(e.data);
  console.log('Joined:', user.name);
});

es.addEventListener('heartbeat', (e) => {
  // keep-alive signal
});

es.onerror = (e) => {
  if (e.readyState === EventSource.CLOSED) {
    console.log('Connection closed');
  }
};

// Close the connection
es.close();
```

### Server-Side Implementation (Node.js)

```javascript
app.get('/api/events', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'X-Accel-Buffering': 'no', // disable nginx buffering
  });

  // Helper to send an event
  const send = (event, data, id) => {
    if (id) res.write(`id: ${id}\n`);
    if (event !== 'message') res.write(`event: ${event}\n`);
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };

  // Heartbeat to keep proxy connections alive
  const heartbeat = setInterval(() => res.write(': heartbeat\n\n'), 15000);

  // Subscribe to data source
  const unsubscribe = dataStream.subscribe((item) => {
    send('update', item, item.id);
  });

  req.on('close', () => {
    clearInterval(heartbeat);
    unsubscribe();
    res.end();
  });
});
```

### Automatic Reconnection and Last-Event-ID

SSE reconnects automatically. The browser sends the `Last-Event-ID` header with the last received event ID, allowing the server to resume from where it left off:

```javascript
// Server reads Last-Event-ID to avoid resending old events
app.get('/api/events', (req, res) => {
  const lastId = req.headers['last-event-id'];
  const cursor = lastId ? parseInt(lastId) : 0;

  // Replay missed events
  const missed = eventStore.getSince(cursor);
  missed.forEach(event => send(event.type, event.data, event.id));

  // Subscribe to new events
  // ...
});
```

### SSE with Fetch (Manual)

`EventSource` cannot set custom headers. For authenticated SSE, use `fetch` with a `ReadableStream`:

```javascript
async function connectSSE(url, token, onEvent) {
  const res = await fetch(url, {
    headers: { Authorization: `Bearer ${token}` },
    signal: abortController.signal,
  });

  const reader = res.body.pipeThrough(new TextDecoderStream()).getReader();
  let buffer = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += value;
    const events = buffer.split('\n\n');
    buffer = events.pop(); // incomplete last event

    for (const block of events) {
      const event = parseSSEBlock(block);
      if (event) onEvent(event);
    }
  }
}

function parseSSEBlock(block) {
  const lines = block.split('\n');
  const obj = {};
  for (const line of lines) {
    if (line.startsWith('data:')) obj.data = line.slice(5).trim();
    if (line.startsWith('event:')) obj.event = line.slice(6).trim();
    if (line.startsWith('id:')) obj.id = line.slice(3).trim();
  }
  return obj.data ? obj : null;
}
```

## Comparison Table

| Feature | WebSocket | Server-Sent Events | Long Polling |
|---------|-----------|--------------------|--------------|
| Direction | Full-duplex | Server → Client | Server → Client |
| Protocol | ws:// / wss:// | HTTP | HTTP |
| Binary support | Yes | No (text only) | Via encoding |
| Auto-reconnect | No (manual) | Yes (built-in) | Manual |
| Custom headers | Yes | No (use fetch workaround) | Yes |
| HTTP/2 multiplexing | No (separate connection) | Yes | Yes |
| Proxy/firewall | Sometimes blocked | Always works | Always works |
| Browser support | All | All (not IE natively) | All |
| Use case | Chat, gaming, collaboration | Notifications, live feeds | Simple push |

## Best Practices

### 1. Use SSE When Communication is One-Way

```javascript
// Live dashboard, notification feed, progress updates
// Don't introduce WebSocket complexity when you only push from server
const es = new EventSource('/api/live-metrics');
```

### 2. Always Implement Reconnection for WebSockets

```javascript
// Production code must handle disconnections
// Use exponential backoff with jitter to avoid thundering herd
const delay = Math.min(1000 * 2 ** retries + Math.random() * 1000, 30000);
```

### 3. Namespace WebSocket Message Types

```javascript
// Structured messages are easier to dispatch
const MSG = {
  AUTH: 'auth',
  SUBSCRIBE: 'subscribe',
  UPDATE: 'update',
  ERROR: 'error',
};

ws.send(JSON.stringify({ type: MSG.SUBSCRIBE, channel: 'prices' }));

ws.onmessage = (e) => {
  const { type, payload } = JSON.parse(e.data);
  handlers[type]?.(payload);
};
```

### 4. Close Connections When the Component Unmounts

```javascript
// React example
useEffect(() => {
  const ws = new WebSocket('wss://api.example.com');
  // ...
  return () => ws.close(1000, 'Component unmounting');
}, []);
```

## Interview Questions

### Q1: When would you choose WebSockets over Server-Sent Events?

**Answer:** Choose WebSockets when you need full-duplex communication—the client sends data frequently (chat messages, game input, collaborative edits). Choose SSE when communication is primarily or exclusively server-to-client (live feeds, notifications, progress updates). SSE is simpler, works over standard HTTP/2 (multiplexed, no extra connection), auto-reconnects, and passes through all proxies. WebSockets require a protocol upgrade and may be blocked by some corporate firewalls.

### Q2: Why does the WebSocket API not expose ping/pong frame control?

**Answer:** The WebSocket protocol defines ping/pong control frames for keep-alive, but the browser handles them at the protocol level transparently. The JavaScript API intentionally does not expose them—responding to pings is the browser's responsibility. Applications that need application-level heartbeats implement their own via regular data frames (e.g., sending `{"type":"ping"}` on a timer).

### Q3: What is the Last-Event-ID mechanism in SSE and why does it matter?

**Answer:** When the SSE connection drops and reconnects, the browser automatically sends the `Last-Event-ID` header containing the ID of the last event it successfully received. The server can read this header and resume streaming from that point, replaying any missed events. This makes SSE naturally resilient to brief disconnections with no message loss, without any client-side logic.

### Q4: EventSource cannot send custom headers. How do you authenticate an SSE connection?

**Answer:** Use a query parameter (acceptable for short-lived tokens), set a cookie before connecting, or implement the SSE protocol manually using `fetch` with a `ReadableStream`. The fetch approach allows full control over request headers, including `Authorization`, while still streaming the text/event-stream format.

### Q5: How do you handle backpressure in WebSockets?

**Answer:** Check `ws.bufferedAmount` before sending. If it exceeds a threshold (e.g., 16KB), either skip the frame, queue it with a delay, or signal the application to slow down data production. Without this check, rapid sends can overflow the browser's send buffer and eventually cause the connection to drop.

## Common Pitfalls

### 1. Not Handling WebSocket Close Codes

```javascript
ws.onclose = (e) => {
  if (e.code === 1000) {
    // Normal closure — do not reconnect
  } else {
    // Abnormal — reconnect with backoff
  }
};
// Code 1006 means connection was lost without a proper close frame
```

### 2. Sending Before the Connection is Open

```javascript
const ws = new WebSocket('wss://...');
ws.send('hello'); // Error: WebSocket is not open

// Wait for onopen
ws.onopen = () => ws.send('hello');
```

### 3. SSE Not Working Behind Nginx

```javascript
// Nginx buffers responses by default, breaking SSE
// Must set in server config or via header:
// proxy_buffering off;
// X-Accel-Buffering: no
res.setHeader('X-Accel-Buffering', 'no');
```

### 4. Forgetting to Clean Up Event Listeners on Reconnect

```javascript
// If you add onmessage inside a reconnect function,
// each reconnect adds another handler — use assignment not addEventListener
// ws.onmessage = handler; (replaces)
// vs.
// ws.addEventListener('message', handler); (accumulates)
```

## Resources

- [MDN: WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [MDN: Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [MDN: EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)
- [RFC 6455: The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)
- [HTML Living Standard: Server-Sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [web.dev: Stream Updates with SSE](https://web.dev/eventsource-basics/)
