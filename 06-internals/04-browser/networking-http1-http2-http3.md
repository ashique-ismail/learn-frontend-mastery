# HTTP Evolution: HTTP/1.1, HTTP/2, and HTTP/3

## Table of Contents
- [Introduction](#introduction)
- [HTTP/1.1: The Foundation](#http11-the-foundation)
- [HTTP/2: Multiplexing Revolution](#http2-multiplexing-revolution)
- [HTTP/3: QUIC Protocol](#http3-quic-protocol)
- [Performance Comparisons](#performance-comparisons)
- [Migration Strategies](#migration-strategies)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Browser Differences](#browser-differences)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

The evolution of HTTP protocols represents one of the most significant advances in web performance. Understanding the differences between HTTP/1.1, HTTP/2, and HTTP/3 is crucial for building high-performance web applications. Each protocol version addresses specific limitations of its predecessor while introducing new optimization opportunities.

## HTTP/1.1: The Foundation

### Core Characteristics

HTTP/1.1, standardized in 1997 and updated in 2014, has been the backbone of the web for decades:

```javascript
// HTTP/1.1 Request Structure
const http1Request = `
GET /api/users HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: application/json
Connection: keep-alive
Accept-Encoding: gzip, deflate
`;

// HTTP/1.1 Response Structure
const http1Response = `
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 1234
Connection: keep-alive
Cache-Control: max-age=3600

{"users": [...]}
`;
```

### Head-of-Line Blocking

The most significant limitation of HTTP/1.1 is head-of-line (HOL) blocking:

```javascript
// HTTP/1.1 Sequential Request Processing
class HTTP1Connection {
  constructor() {
    this.requestQueue = [];
    this.isProcessing = false;
  }

  async sendRequest(request) {
    // Add to queue
    this.requestQueue.push(request);
    
    // Process sequentially
    if (!this.isProcessing) {
      await this.processQueue();
    }
  }

  async processQueue() {
    this.isProcessing = true;
    
    while (this.requestQueue.length > 0) {
      const request = this.requestQueue.shift();
      
      // Must wait for complete response before next request
      await this.waitForCompleteResponse(request);
      
      // Blocking occurs here - next request waits
      console.log('Request completed, processing next...');
    }
    
    this.isProcessing = false;
  }

  async waitForCompleteResponse(request) {
    // Simulate network delay
    return new Promise(resolve => {
      setTimeout(resolve, Math.random() * 2000);
    });
  }
}

// Usage demonstrating HOL blocking
const conn = new HTTP1Connection();
console.log('Sending 5 requests...');
const start = Date.now();

Promise.all([
  conn.sendRequest({ url: '/api/users' }),
  conn.sendRequest({ url: '/api/posts' }),
  conn.sendRequest({ url: '/api/comments' }),
  conn.sendRequest({ url: '/api/images' }),
  conn.sendRequest({ url: '/api/metadata' })
]).then(() => {
  console.log(`Total time: ${Date.now() - start}ms`);
  // All requests processed sequentially on single connection
});
```

### Connection Limits

Browsers limit concurrent connections per domain:

```javascript
// HTTP/1.1 Connection Management
class HTTP1ConnectionPool {
  constructor(maxConnectionsPerHost = 6) {
    this.maxConnections = maxConnectionsPerHost;
    this.connections = new Map();
    this.waitingRequests = [];
  }

  async request(url) {
    const host = new URL(url).host;
    
    if (!this.connections.has(host)) {
      this.connections.set(host, []);
    }
    
    const hostConnections = this.connections.get(host);
    
    if (hostConnections.length < this.maxConnections) {
      // Create new connection
      const connection = this.createConnection(host);
      hostConnections.push(connection);
      return connection.send(url);
    } else {
      // Wait for available connection
      return this.queueRequest(url);
    }
  }

  createConnection(host) {
    return {
      host,
      inUse: false,
      async send(url) {
        this.inUse = true;
        // Send request
        await new Promise(resolve => setTimeout(resolve, 1000));
        this.inUse = false;
        return { status: 200, data: {} };
      }
    };
  }

  queueRequest(url) {
    return new Promise((resolve) => {
      this.waitingRequests.push({ url, resolve });
    });
  }
}
```

### HTTP/1.1 Optimization Techniques

Common workarounds for HTTP/1.1 limitations:

```javascript
// Domain Sharding
const SHARDS = [
  'static1.example.com',
  'static2.example.com',
  'static3.example.com',
  'static4.example.com'
];

function getShardedUrl(resource, index) {
  const shard = SHARDS[index % SHARDS.length];
  return `https://${shard}/${resource}`;
}

// Distribute resources across shards
const resources = [
  'image1.jpg', 'image2.jpg', 'image3.jpg',
  'style1.css', 'style2.css', 'script1.js'
];

resources.forEach((resource, index) => {
  const url = getShardedUrl(resource, index);
  console.log(`Loading: ${url}`);
});

// Resource Concatenation
class HTTP1Optimizer {
  concatenateCSS(files) {
    // Combine multiple CSS files into one
    return files.map(file => `@import url('${file}');`).join('\n');
  }

  concatenateJS(files) {
    // Bundle multiple JS files
    return files.map(file => `// File: ${file}\n${file.content}`).join('\n\n');
  }

  spriteImages(images) {
    // Combine images into sprite sheet
    return {
      spriteUrl: '/sprites/combined.png',
      coordinates: images.map((img, i) => ({
        name: img.name,
        x: (i % 10) * 32,
        y: Math.floor(i / 10) * 32,
        width: 32,
        height: 32
      }))
    };
  }
}
```

## HTTP/2: Multiplexing Revolution

### Binary Framing Layer

HTTP/2 introduces a binary protocol with frames and streams:

```javascript
// HTTP/2 Frame Structure (conceptual)
class HTTP2Frame {
  constructor(type, flags, streamId, payload) {
    this.length = payload.length; // 24 bits
    this.type = type;              // 8 bits (DATA, HEADERS, etc.)
    this.flags = flags;            // 8 bits
    this.streamId = streamId;      // 31 bits
    this.payload = payload;
  }

  serialize() {
    const buffer = Buffer.alloc(9 + this.payload.length);
    
    // Frame header (9 bytes)
    buffer.writeUInt24BE(this.length, 0);
    buffer.writeUInt8(this.type, 3);
    buffer.writeUInt8(this.flags, 4);
    buffer.writeUInt32BE(this.streamId, 5);
    
    // Payload
    this.payload.copy(buffer, 9);
    
    return buffer;
  }
}

// Frame Types
const FrameType = {
  DATA: 0x0,
  HEADERS: 0x1,
  PRIORITY: 0x2,
  RST_STREAM: 0x3,
  SETTINGS: 0x4,
  PUSH_PROMISE: 0x5,
  PING: 0x6,
  GOAWAY: 0x7,
  WINDOW_UPDATE: 0x8,
  CONTINUATION: 0x9
};
```

### Stream Multiplexing

Multiple requests over single connection:

```javascript
// HTTP/2 Stream Multiplexing
class HTTP2Connection {
  constructor() {
    this.streams = new Map();
    this.nextStreamId = 1;
    this.connection = null; // Single TCP connection
  }

  createStream() {
    const streamId = this.nextStreamId;
    this.nextStreamId += 2; // Client uses odd numbers
    
    const stream = new HTTP2Stream(streamId, this);
    this.streams.set(streamId, stream);
    
    return stream;
  }

  async sendMultipleRequests(requests) {
    // All requests sent concurrently over single connection
    const promises = requests.map(req => {
      const stream = this.createStream();
      return stream.send(req);
    });
    
    return Promise.all(promises);
  }

  receiveFrame(frame) {
    const stream = this.streams.get(frame.streamId);
    if (stream) {
      stream.handleFrame(frame);
    }
  }
}

class HTTP2Stream {
  constructor(id, connection) {
    this.id = id;
    this.connection = connection;
    this.state = 'idle';
    this.headers = {};
    this.data = [];
    this.promise = null;
  }

  async send(request) {
    this.state = 'open';
    
    // Send HEADERS frame
    const headersFrame = new HTTP2Frame(
      FrameType.HEADERS,
      0x04, // END_HEADERS flag
      this.id,
      this.encodeHeaders(request.headers)
    );
    
    this.connection.sendFrame(headersFrame);
    
    // Send DATA frame if body exists
    if (request.body) {
      const dataFrame = new HTTP2Frame(
        FrameType.DATA,
        0x01, // END_STREAM flag
        this.id,
        Buffer.from(request.body)
      );
      
      this.connection.sendFrame(dataFrame);
    }
    
    // Return promise that resolves when response complete
    return new Promise((resolve) => {
      this.promise = resolve;
    });
  }

  handleFrame(frame) {
    switch (frame.type) {
      case FrameType.HEADERS:
        this.headers = this.decodeHeaders(frame.payload);
        break;
        
      case FrameType.DATA:
        this.data.push(frame.payload);
        
        // Check for END_STREAM flag
        if (frame.flags & 0x01) {
          this.state = 'closed';
          this.promise({
            headers: this.headers,
            body: Buffer.concat(this.data)
          });
        }
        break;
    }
  }

  encodeHeaders(headers) {
    // HPACK compression (simplified)
    return Buffer.from(JSON.stringify(headers));
  }

  decodeHeaders(buffer) {
    // HPACK decompression (simplified)
    return JSON.parse(buffer.toString());
  }
}

// Usage Example
async function demonstrateMultiplexing() {
  const conn = new HTTP2Connection();
  
  console.log('Sending 5 concurrent requests on single connection...');
  const start = Date.now();
  
  const responses = await conn.sendMultipleRequests([
    { url: '/api/users', headers: { ':path': '/api/users' } },
    { url: '/api/posts', headers: { ':path': '/api/posts' } },
    { url: '/api/comments', headers: { ':path': '/api/comments' } },
    { url: '/api/images', headers: { ':path': '/api/images' } },
    { url: '/api/metadata', headers: { ':path': '/api/metadata' } }
  ]);
  
  console.log(`Total time: ${Date.now() - start}ms`);
  console.log('All requests completed concurrently!');
}
```

### HPACK Header Compression

HTTP/2 uses HPACK to compress headers:

```javascript
// HPACK Compression Simulation
class HPACKCompressor {
  constructor() {
    // Static table (predefined common headers)
    this.staticTable = [
      [':authority', ''],
      [':method', 'GET'],
      [':method', 'POST'],
      [':path', '/'],
      [':scheme', 'https'],
      ['accept', '*/*'],
      ['accept-encoding', 'gzip, deflate'],
      ['cache-control', 'no-cache'],
      ['content-type', 'application/json'],
      // ... 61 entries total
    ];
    
    // Dynamic table (learned from connection)
    this.dynamicTable = [];
    this.dynamicTableSize = 0;
    this.maxDynamicTableSize = 4096;
  }

  compress(headers) {
    const encoded = [];
    
    for (const [name, value] of Object.entries(headers)) {
      // Check static table
      const staticIndex = this.findInStaticTable(name, value);
      if (staticIndex !== -1) {
        // Indexed representation (1 byte if index < 127)
        encoded.push({ type: 'indexed', index: staticIndex });
        continue;
      }
      
      // Check dynamic table
      const dynamicIndex = this.findInDynamicTable(name, value);
      if (dynamicIndex !== -1) {
        encoded.push({ 
          type: 'indexed', 
          index: this.staticTable.length + dynamicIndex 
        });
        continue;
      }
      
      // Literal with incremental indexing
      encoded.push({ 
        type: 'literal-incremental',
        name,
        value
      });
      
      // Add to dynamic table
      this.addToDynamicTable(name, value);
    }
    
    return encoded;
  }

  findInStaticTable(name, value) {
    return this.staticTable.findIndex(
      ([n, v]) => n === name && (v === value || v === '')
    );
  }

  findInDynamicTable(name, value) {
    return this.dynamicTable.findIndex(
      ([n, v]) => n === name && v === value
    );
  }

  addToDynamicTable(name, value) {
    const size = name.length + value.length + 32; // 32 byte overhead
    
    // Evict entries if necessary
    while (this.dynamicTableSize + size > this.maxDynamicTableSize) {
      const removed = this.dynamicTable.pop();
      if (removed) {
        this.dynamicTableSize -= removed[0].length + removed[1].length + 32;
      }
    }
    
    this.dynamicTable.unshift([name, value]);
    this.dynamicTableSize += size;
  }

  calculateSavings(headers) {
    const uncompressed = JSON.stringify(headers).length;
    const compressed = this.compress(headers);
    const compressedSize = compressed.reduce((sum, item) => {
      if (item.type === 'indexed') return sum + 1;
      return sum + item.name.length + item.value.length + 2;
    }, 0);
    
    const savings = ((uncompressed - compressedSize) / uncompressed * 100).toFixed(1);
    
    return {
      uncompressed,
      compressed: compressedSize,
      savings: `${savings}%`
    };
  }
}

// Example
const compressor = new HPACKCompressor();
const headers = {
  ':authority': 'example.com',
  ':method': 'GET',
  ':path': '/api/users',
  ':scheme': 'https',
  'accept': '*/*',
  'accept-encoding': 'gzip, deflate',
  'user-agent': 'Mozilla/5.0'
};

const result = compressor.calculateSavings(headers);
console.log('HPACK Compression:', result);
// Typical savings: 50-70% for headers
```

### Server Push

Server can proactively send resources:

```javascript
// HTTP/2 Server Push (Node.js example)
const http2 = require('http2');
const fs = require('fs');

const server = http2.createSecureServer({
  key: fs.readFileSync('key.pem'),
  cert: fs.readFileSync('cert.pem')
});

server.on('stream', (stream, headers) => {
  const path = headers[':path'];
  
  if (path === '/index.html') {
    // Push critical resources before sending HTML
    pushResource(stream, '/styles/critical.css');
    pushResource(stream, '/scripts/app.js');
    pushResource(stream, '/images/logo.png');
    
    // Then send HTML
    stream.respond({
      ':status': 200,
      'content-type': 'text/html'
    });
    stream.end(fs.readFileSync('index.html'));
  }
});

function pushResource(stream, path) {
  const pushStream = stream.pushStream({ ':path': path }, (err, push) => {
    if (err) {
      console.error('Push error:', err);
      return;
    }
    
    const contentType = getContentType(path);
    push.respond({
      ':status': 200,
      'content-type': contentType
    });
    
    const content = fs.readFileSync(`.${path}`);
    push.end(content);
    
    console.log(`Pushed: ${path}`);
  });
}

// Client-side: Detecting pushed resources
if ('performance' in window && 'getEntriesByType' in performance) {
  const entries = performance.getEntriesByType('navigation');
  entries.forEach(entry => {
    if (entry.nextHopProtocol === 'h2') {
      console.log('HTTP/2 detected');
      
      // Check for pushed resources
      const resources = performance.getEntriesByType('resource');
      resources.forEach(resource => {
        // Pushed resources have no initiatorType
        if (!resource.initiatorType) {
          console.log('Pushed resource:', resource.name);
        }
      });
    }
  });
}
```

### Stream Prioritization

Prioritize important resources:

```javascript
// HTTP/2 Stream Priority
class HTTP2StreamPriority {
  constructor() {
    this.dependencies = new Map();
  }

  setPriority(streamId, priority) {
    // Priority: { exclusive, dependsOn, weight }
    this.dependencies.set(streamId, {
      exclusive: priority.exclusive || false,
      dependsOn: priority.dependsOn || 0,
      weight: priority.weight || 16 // 1-256
    });
  }

  calculateScheduling() {
    // Build dependency tree
    const tree = this.buildDependencyTree();
    
    // Calculate bandwidth allocation
    return this.allocateBandwidth(tree);
  }

  buildDependencyTree() {
    const tree = { id: 0, children: [], weight: 0 };
    
    for (const [streamId, priority] of this.dependencies) {
      const parent = this.findNode(tree, priority.dependsOn);
      if (parent) {
        if (priority.exclusive) {
          // Make existing children depend on this stream
          const children = parent.children;
          parent.children = [{
            id: streamId,
            weight: priority.weight,
            children: children
          }];
        } else {
          parent.children.push({
            id: streamId,
            weight: priority.weight,
            children: []
          });
        }
      }
    }
    
    return tree;
  }

  findNode(node, id) {
    if (node.id === id) return node;
    for (const child of node.children) {
      const found = this.findNode(child, id);
      if (found) return found;
    }
    return null;
  }

  allocateBandwidth(node, totalBandwidth = 100) {
    const allocation = new Map();
    
    if (node.children.length === 0) {
      return allocation;
    }
    
    const totalWeight = node.children.reduce((sum, child) => sum + child.weight, 0);
    
    for (const child of node.children) {
      const childBandwidth = (child.weight / totalWeight) * totalBandwidth;
      allocation.set(child.id, childBandwidth);
      
      // Recursively allocate to children
      const childAllocations = this.allocateBandwidth(child, childBandwidth);
      for (const [id, bw] of childAllocations) {
        allocation.set(id, bw);
      }
    }
    
    return allocation;
  }
}

// Example usage
const priority = new HTTP2StreamPriority();

// HTML document (highest priority)
priority.setPriority(1, { dependsOn: 0, weight: 256 });

// Critical CSS (depends on HTML)
priority.setPriority(3, { dependsOn: 1, weight: 220, exclusive: false });

// JavaScript (depends on HTML, lower priority)
priority.setPriority(5, { dependsOn: 1, weight: 128 });

// Images (depends on HTML, lowest priority)
priority.setPriority(7, { dependsOn: 1, weight: 32 });

const allocations = priority.calculateScheduling();
console.log('Bandwidth allocation:', allocations);
```

## HTTP/3: QUIC Protocol

### QUIC Transport Layer

HTTP/3 uses QUIC over UDP instead of TCP:

```javascript
// QUIC Connection Characteristics
class QUICConnection {
  constructor() {
    this.connectionId = this.generateConnectionId();
    this.streams = new Map();
    this.congestionControl = new QUICCongestionControl();
    this.encryption = 'TLS 1.3'; // Built-in encryption
  }

  generateConnectionId() {
    // Connection IDs allow connection migration
    return crypto.randomBytes(8).toString('hex');
  }

  // Key advantage: 0-RTT connection establishment
  async establish0RTT(serverAddress, cachedConfig) {
    if (cachedConfig) {
      // Can send application data immediately
      console.log('0-RTT: Sending data in first packet');
      return {
        rtt: 0,
        dataInFirstPacket: true
      };
    } else {
      // 1-RTT for first connection
      console.log('1-RTT: Initial handshake required');
      return {
        rtt: 1,
        dataInFirstPacket: false
      };
    }
  }

  // Connection migration (WiFi to cellular)
  migrateConnection(newPath) {
    console.log(`Migrating connection ${this.connectionId} to new path`);
    
    // QUIC can continue same connection with new IP/port
    this.updatePath(newPath);
    
    // No need to re-establish connection or re-negotiate TLS
    console.log('Connection migrated seamlessly');
  }

  updatePath(newPath) {
    this.currentPath = newPath;
    // Connection ID remains the same
  }
}

// QUIC Packet Structure
class QUICPacket {
  constructor(type, connectionId, packetNumber, payload) {
    this.headerForm = 'long'; // or 'short'
    this.type = type; // Initial, 0-RTT, Handshake, Retry, 1-RTT
    this.connectionId = connectionId;
    this.packetNumber = packetNumber;
    this.payload = payload; // Encrypted
  }

  serialize() {
    // QUIC header + encrypted payload
    return {
      connectionId: this.connectionId,
      packetNumber: this.packetNumber,
      encrypted: this.encryptPayload()
    };
  }

  encryptPayload() {
    // TLS 1.3 encryption (all packets encrypted)
    return `encrypted:${this.payload}`;
  }
}
```

### Solving Head-of-Line Blocking

QUIC eliminates TCP's HOL blocking:

```javascript
// TCP vs QUIC: Head-of-Line Blocking
class TCPStream {
  constructor() {
    this.receiveBuffer = [];
    this.nextExpectedSeq = 0;
  }

  receivePacket(packet) {
    if (packet.seq === this.nextExpectedSeq) {
      // In-order packet: deliver immediately
      this.deliverToApp(packet.data);
      this.nextExpectedSeq++;
      
      // Check if we can deliver buffered packets
      this.deliverBufferedPackets();
    } else {
      // Out-of-order: must buffer and wait
      console.log(`TCP: Buffering packet ${packet.seq}, waiting for ${this.nextExpectedSeq}`);
      this.receiveBuffer.push(packet);
      
      // ALL STREAMS BLOCKED until missing packet arrives
      console.log('TCP: All multiplexed streams blocked!');
    }
  }

  deliverBufferedPackets() {
    this.receiveBuffer.sort((a, b) => a.seq - b.seq);
    
    while (this.receiveBuffer.length > 0 && 
           this.receiveBuffer[0].seq === this.nextExpectedSeq) {
      const packet = this.receiveBuffer.shift();
      this.deliverToApp(packet.data);
      this.nextExpectedSeq++;
    }
  }

  deliverToApp(data) {
    console.log('TCP: Delivered data to application');
  }
}

class QUICStream {
  constructor(streamId) {
    this.streamId = streamId;
    this.receiveBuffer = [];
    this.nextExpectedOffset = 0;
  }

  receivePacket(packet) {
    if (packet.offset === this.nextExpectedOffset) {
      // In-order packet for THIS stream
      this.deliverToApp(packet.data);
      this.nextExpectedOffset += packet.data.length;
      this.deliverBufferedPackets();
    } else {
      // Out-of-order for THIS stream only
      console.log(`QUIC Stream ${this.streamId}: Buffering packet`);
      this.receiveBuffer.push(packet);
      
      // Other streams NOT blocked
      console.log('QUIC: Other streams continue unaffected!');
    }
  }

  deliverBufferedPackets() {
    this.receiveBuffer.sort((a, b) => a.offset - b.offset);
    
    while (this.receiveBuffer.length > 0 && 
           this.receiveBuffer[0].offset === this.nextExpectedOffset) {
      const packet = this.receiveBuffer.shift();
      this.deliverToApp(packet.data);
      this.nextExpectedOffset += packet.data.length;
    }
  }

  deliverToApp(data) {
    console.log(`QUIC Stream ${this.streamId}: Delivered data to application`);
  }
}

// Comparison
function demonstrateHOLBlocking() {
  console.log('=== TCP (HTTP/2) ===');
  const tcp = new TCPStream();
  tcp.receivePacket({ seq: 0, data: 'packet0' }); // OK
  tcp.receivePacket({ seq: 2, data: 'packet2' }); // Buffered
  tcp.receivePacket({ seq: 3, data: 'packet3' }); // Buffered
  // Packet 1 lost - ALL streams blocked!
  console.log('Waiting for packet 1...\n');
  
  console.log('=== QUIC (HTTP/3) ===');
  const stream1 = new QUICStream(1);
  const stream2 = new QUICStream(2);
  
  stream1.receivePacket({ offset: 0, data: 'stream1-packet0' }); // OK
  stream1.receivePacket({ offset: 20, data: 'stream1-packet2' }); // Buffered
  // Stream 1 waiting for packet 1
  
  stream2.receivePacket({ offset: 0, data: 'stream2-packet0' }); // OK
  stream2.receivePacket({ offset: 10, data: 'stream2-packet1' }); // OK
  // Stream 2 continues unaffected!
}
```

### Connection Migration

QUIC supports seamless network changes:

```javascript
// Connection Migration Example
class QUICConnectionMigration {
  constructor() {
    this.connectionId = 'persistent-connection-id';
    this.currentPath = null;
    this.pathValidation = null;
  }

  async handleNetworkChange(newNetwork) {
    console.log(`Network changed: ${this.currentPath?.type} -> ${newNetwork.type}`);
    
    // Start path validation
    const validationToken = this.generateValidationToken();
    
    await this.sendPathChallenge(newNetwork, validationToken);
    
    const isValid = await this.waitForPathResponse(validationToken);
    
    if (isValid) {
      // Migrate to new path
      this.currentPath = newNetwork;
      console.log('Connection migrated successfully');
      
      // Continue existing streams without interruption
      console.log('All streams continue on new path');
    } else {
      console.log('Path validation failed, keeping current path');
    }
  }

  generateValidationToken() {
    return crypto.randomBytes(8);
  }

  async sendPathChallenge(network, token) {
    // Send PATH_CHALLENGE frame on new path
    console.log('Sending PATH_CHALLENGE on new path');
    return new Promise(resolve => setTimeout(resolve, 50));
  }

  async waitForPathResponse(token) {
    // Wait for PATH_RESPONSE frame
    console.log('Received PATH_RESPONSE, path validated');
    return true;
  }

  // Practical scenario
  async demonstrateScenario() {
    // User is on WiFi
    this.currentPath = { type: 'WiFi', ip: '192.168.1.100' };
    console.log('Streaming video on WiFi');
    
    // User walks out of WiFi range, switches to cellular
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    await this.handleNetworkChange({
      type: 'Cellular',
      ip: '10.20.30.40'
    });
    
    console.log('Video continues without buffering or reconnection!');
  }
}

// Usage
const migration = new QUICConnectionMigration();
migration.demonstrateScenario();
```

### Improved Congestion Control

QUIC has more flexible congestion control:

```javascript
// QUIC Congestion Control
class QUICCongestionControl {
  constructor() {
    this.algorithm = 'Cubic'; // or 'BBR', 'NewReno'
    this.cwnd = 10; // Congestion window (packets)
    this.ssthresh = 64; // Slow start threshold
    this.rtt = 100; // Round-trip time (ms)
    this.lossDetection = new QUICLossDetection();
  }

  onPacketSent(packet) {
    packet.sentTime = Date.now();
    this.lossDetection.trackPacket(packet);
  }

  onAckReceived(ackedPackets) {
    const now = Date.now();
    
    ackedPackets.forEach(packet => {
      // Update RTT
      const sampleRTT = now - packet.sentTime;
      this.updateRTT(sampleRTT);
      
      // Increase congestion window
      if (this.cwnd < this.ssthresh) {
        // Slow start: exponential growth
        this.cwnd += 1;
      } else {
        // Congestion avoidance: linear growth
        this.cwnd += 1 / this.cwnd;
      }
      
      this.lossDetection.onAck(packet);
    });
  }

  onPacketLoss(lostPackets) {
    // Multiplicative decrease
    this.ssthresh = Math.max(this.cwnd / 2, 2);
    this.cwnd = this.ssthresh;
    
    console.log(`Packet loss detected. Reduced cwnd to ${this.cwnd}`);
    
    // Retransmit lost packets
    lostPackets.forEach(packet => {
      this.retransmit(packet);
    });
  }

  updateRTT(sample) {
    // Exponential moving average
    this.rtt = 0.875 * this.rtt + 0.125 * sample;
  }

  retransmit(packet) {
    console.log(`Retransmitting packet ${packet.number}`);
    // QUIC uses new packet number for retransmission
    packet.retransmissionNumber = this.getNextPacketNumber();
  }

  getNextPacketNumber() {
    return Math.random() * 1000000 | 0;
  }
}

class QUICLossDetection {
  constructor() {
    this.sentPackets = new Map();
    this.lossTimeout = null;
  }

  trackPacket(packet) {
    this.sentPackets.set(packet.number, packet);
    
    // Set loss detection timer
    this.setLossTimer();
  }

  onAck(packet) {
    this.sentPackets.delete(packet.number);
  }

  setLossTimer() {
    clearTimeout(this.lossTimeout);
    
    this.lossTimeout = setTimeout(() => {
      this.detectLost();
    }, this.calculateTimeout());
  }

  calculateTimeout() {
    // Typically RTT + 4 * RTT_variance
    return 100; // Simplified
  }

  detectLost() {
    const now = Date.now();
    const lostPackets = [];
    
    for (const [number, packet] of this.sentPackets) {
      if (now - packet.sentTime > this.calculateTimeout()) {
        lostPackets.push(packet);
      }
    }
    
    return lostPackets;
  }
}
```

## Performance Comparisons

### Waterfall Comparison

```javascript
// Performance Comparison Simulation
class ProtocolPerformanceTest {
  constructor(protocol) {
    this.protocol = protocol;
    this.resources = [
      { name: 'HTML', size: 50, priority: 1 },
      { name: 'CSS', size: 100, priority: 2 },
      { name: 'JS', size: 200, priority: 3 },
      { name: 'Image1', size: 150, priority: 4 },
      { name: 'Image2', size: 150, priority: 4 },
      { name: 'Image3', size: 150, priority: 4 },
      { name: 'Font1', size: 80, priority: 5 },
      { name: 'Font2', size: 80, priority: 5 },
    ];
    this.rtt = 100; // ms
    this.bandwidth = 5; // Mbps
  }

  async simulateHTTP1() {
    const connectionsPerDomain = 6;
    let totalTime = this.rtt; // Initial connection
    
    // Process in chunks of 6 concurrent connections
    for (let i = 0; i < this.resources.length; i += connectionsPerDomain) {
      const batch = this.resources.slice(i, i + connectionsPerDomain);
      const maxTime = Math.max(...batch.map(r => this.downloadTime(r)));
      totalTime += this.rtt + maxTime; // RTT + download
    }
    
    return {
      protocol: 'HTTP/1.1',
      totalTime,
      connections: Math.ceil(this.resources.length / connectionsPerDomain)
    };
  }

  async simulateHTTP2() {
    let totalTime = this.rtt; // Initial connection
    
    // All resources over single connection
    // Account for multiplexing but shared bandwidth
    const totalSize = this.resources.reduce((sum, r) => sum + r.size, 0);
    const downloadTime = this.downloadTime({ size: totalSize });
    
    totalTime += downloadTime;
    
    return {
      protocol: 'HTTP/2',
      totalTime,
      connections: 1,
      multiplexing: true
    };
  }

  async simulateHTTP3() {
    // 0-RTT if cached
    let totalTime = 0; // 0-RTT
    
    // All resources over single connection
    const totalSize = this.resources.reduce((sum, r) => sum + r.size, 0);
    const downloadTime = this.downloadTime({ size: totalSize });
    
    // Benefit from improved loss recovery
    const lossReduction = 0.9; // 10% improvement
    totalTime += downloadTime * lossReduction;
    
    return {
      protocol: 'HTTP/3',
      totalTime,
      connections: 1,
      multiplexing: true,
      zeroRTT: true
    };
  }

  downloadTime(resource) {
    // Convert KB to Mb, then divide by bandwidth
    return (resource.size * 8 / 1024) / this.bandwidth * 1000;
  }

  async runComparison() {
    const results = await Promise.all([
      this.simulateHTTP1(),
      this.simulateHTTP2(),
      this.simulateHTTP3()
    ]);
    
    console.log('Performance Comparison:');
    console.log('======================');
    results.forEach(result => {
      console.log(`${result.protocol}: ${result.totalTime.toFixed(0)}ms`);
    });
    
    const baseline = results[0].totalTime;
    results.forEach((result, i) => {
      if (i > 0) {
        const improvement = ((baseline - result.totalTime) / baseline * 100).toFixed(1);
        console.log(`${result.protocol} improvement: ${improvement}%`);
      }
    });
    
    return results;
  }
}

// Run comparison
const test = new ProtocolPerformanceTest();
test.runComparison();
```

### Real-World Performance Metrics

```javascript
// Measuring HTTP Version Performance
class HTTPVersionDetector {
  static async detectAndMeasure() {
    const resources = performance.getEntriesByType('resource');
    const protocols = {};
    
    resources.forEach(resource => {
      const protocol = resource.nextHopProtocol;
      
      if (!protocols[protocol]) {
        protocols[protocol] = {
          count: 0,
          totalDuration: 0,
          resources: []
        };
      }
      
      protocols[protocol].count++;
      protocols[protocol].totalDuration += resource.duration;
      protocols[protocol].resources.push({
        name: resource.name,
        duration: resource.duration,
        size: resource.transferSize
      });
    });
    
    // Calculate averages
    Object.keys(protocols).forEach(protocol => {
      const data = protocols[protocol];
      data.avgDuration = data.totalDuration / data.count;
      data.avgSize = data.resources.reduce((sum, r) => sum + r.size, 0) / data.count;
    });
    
    return protocols;
  }

  static async measureConnectionSetup() {
    const navigation = performance.getEntriesByType('navigation')[0];
    
    return {
      dns: navigation.domainLookupEnd - navigation.domainLookupStart,
      tcp: navigation.connectEnd - navigation.connectStart,
      tls: navigation.secureConnectionStart > 0 
        ? navigation.connectEnd - navigation.secureConnectionStart 
        : 0,
      total: navigation.connectEnd - navigation.domainLookupStart
    };
  }

  static generateReport() {
    const protocols = this.detectAndMeasure();
    const setup = this.measureConnectionSetup();
    
    console.log('HTTP Protocol Analysis');
    console.log('=====================');
    
    Object.entries(protocols).forEach(([protocol, data]) => {
      console.log(`\n${protocol}:`);
      console.log(`  Resources: ${data.count}`);
      console.log(`  Avg Duration: ${data.avgDuration.toFixed(2)}ms`);
      console.log(`  Avg Size: ${(data.avgSize / 1024).toFixed(2)}KB`);
    });
    
    console.log('\nConnection Setup:');
    console.log(`  DNS: ${setup.dns.toFixed(2)}ms`);
    console.log(`  TCP: ${setup.tcp.toFixed(2)}ms`);
    console.log(`  TLS: ${setup.tls.toFixed(2)}ms`);
    console.log(`  Total: ${setup.total.toFixed(2)}ms`);
  }
}

// Usage in browser console
// HTTPVersionDetector.generateReport();
```

## Migration Strategies

### HTTP/1.1 to HTTP/2 Migration

```javascript
// HTTP/2 Migration Checklist
class HTTP2Migration {
  static analyze() {
    const optimizations = {
      toRemove: [],
      toKeep: [],
      toAdd: []
    };

    // Check for HTTP/1.1 optimizations to remove
    if (this.hasDomainSharding()) {
      optimizations.toRemove.push({
        name: 'Domain Sharding',
        reason: 'HTTP/2 multiplexing makes this counterproductive',
        action: 'Consolidate all static assets to single domain'
      });
    }

    if (this.hasResourceConcatenation()) {
      optimizations.toRemove.push({
        name: 'Resource Concatenation',
        reason: 'Prevents granular caching, unnecessary with multiplexing',
        action: 'Split into individual files for better cache efficiency'
      });
    }

    if (this.hasImageSprites()) {
      optimizations.toRemove.push({
        name: 'Image Sprites',
        reason: 'Individual images load better with HTTP/2',
        action: 'Use individual images or consider SVG sprites for icons'
      });
    }

    if (this.hasInlinedResources()) {
      optimizations.toKeep.push({
        name: 'Critical CSS Inlining',
        reason: 'Still beneficial for above-the-fold content',
        action: 'Keep for critical path, use server push for rest'
      });
    }

    // New optimizations for HTTP/2
    optimizations.toAdd.push({
      name: 'Server Push',
      reason: 'Eliminate round trips for critical resources',
      action: 'Configure server to push CSS, JS, and critical images'
    });

    optimizations.toAdd.push({
      name: 'Resource Prioritization',
      reason: 'Control load order of multiplexed resources',
      action: 'Implement priority hints and dependency chains'
    });

    return optimizations;
  }

  static hasDomainSharding() {
    const hosts = new Set();
    performance.getEntriesByType('resource').forEach(r => {
      hosts.add(new URL(r.name).host);
    });
    return hosts.size > 2; // Main domain + CDN is OK
  }

  static hasResourceConcatenation() {
    const resources = performance.getEntriesByType('resource');
    const largeFiles = resources.filter(r => 
      (r.name.includes('.js') || r.name.includes('.css')) && 
      r.transferSize > 500000 // > 500KB
    );
    return largeFiles.length > 0;
  }

  static hasImageSprites() {
    const images = performance.getEntriesByType('resource').filter(r => 
      r.name.match(/\.(png|jpg|gif)$/)
    );
    return images.some(img => 
      img.name.includes('sprite') && img.transferSize > 100000
    );
  }

  static hasInlinedResources() {
    const doc = document.documentElement.outerHTML;
    return doc.includes('<style>') && doc.includes('<script>');
  }
}

// Server configuration example
const http2ServerConfig = {
  // Enable HTTP/2
  http2: true,
  
  // Configure server push
  push: {
    enabled: true,
    resources: [
      { path: '/styles/critical.css', as: 'style' },
      { path: '/scripts/app.js', as: 'script' },
      { path: '/fonts/main.woff2', as: 'font', crossorigin: true }
    ]
  },
  
  // Set up resource hints
  headers: {
    'Link': [
      '</styles/critical.css>; rel=preload; as=style',
      '</scripts/app.js>; rel=preload; as=script'
    ]
  }
};
```

### HTTP/2 to HTTP/3 Migration

```javascript
// HTTP/3 Migration Strategy
class HTTP3Migration {
  static checkReadiness() {
    const checks = {
      server: this.checkServerSupport(),
      cdn: this.checkCDNSupport(),
      client: this.checkClientSupport(),
      certificate: this.checkCertificate()
    };

    const ready = Object.values(checks).every(check => check.ready);
    
    return {
      ready,
      checks,
      recommendations: this.getRecommendations(checks)
    };
  }

  static checkServerSupport() {
    // Check if server supports HTTP/3
    return {
      ready: false, // Depends on server software
      details: {
        nginx: 'Requires nginx 1.25.0+ with --with-http_v3_module',
        apache: 'Requires Apache 2.4.48+ with mod_http2',
        node: 'Limited support, use Cloudflare Workers or similar'
      }
    };
  }

  static checkCDNSupport() {
    return {
      ready: true,
      providers: {
        cloudflare: 'Full HTTP/3 support',
        fastly: 'Full HTTP/3 support',
        akamai: 'Full HTTP/3 support',
        cloudfront: 'Partial support'
      }
    };
  }

  static checkClientSupport() {
    // Check browser support
    const supported = 'performance' in window && 
                     performance.getEntriesByType('navigation').some(
                       entry => entry.nextHopProtocol === 'h3'
                     );
    
    return {
      ready: true,
      support: {
        chrome: '90+',
        firefox: '88+',
        safari: '14+',
        edge: '90+'
      },
      current: supported ? 'Supported' : 'Not detected'
    };
  }

  static checkCertificate() {
    return {
      ready: true,
      requirements: 'Standard TLS 1.3 certificate (same as HTTP/2)'
    };
  }

  static getRecommendations(checks) {
    const recommendations = [];

    if (!checks.server.ready) {
      recommendations.push({
        priority: 'high',
        item: 'Use CDN with HTTP/3 support',
        reason: 'Easier than upgrading origin server',
        action: 'Configure Cloudflare, Fastly, or similar'
      });
    }

    recommendations.push({
      priority: 'medium',
      item: 'Implement Alt-Svc header',
      reason: 'Advertise HTTP/3 availability to clients',
      action: 'Add Alt-Svc: h3=":443"; ma=86400 header'
    });

    recommendations.push({
      priority: 'medium',
      item: 'Enable 0-RTT',
      reason: 'Maximize performance benefits',
      action: 'Configure server with 0-RTT support'
    });

    recommendations.push({
      priority: 'low',
      item: 'Monitor fallback behavior',
      reason: 'Ensure graceful degradation',
      action: 'Track connection success rates and fallback to HTTP/2'
    });

    return recommendations;
  }
}

// Implementation example
class HTTP3Implementation {
  static setupAltSvc() {
    // Server-side header
    return {
      'Alt-Svc': 'h3=":443"; ma=86400, h2=":443"; ma=86400'
    };
  }

  static async enableOnCDN() {
    // Example: Cloudflare API
    const config = {
      settings: {
        http3: 'on',
        zero_rtt: 'on',
        websockets: 'on'
      }
    };
    
    return config;
  }

  static monitorPerformance() {
    const observer = new PerformanceObserver((list) => {
      list.getEntries().forEach((entry) => {
        if (entry.nextHopProtocol === 'h3') {
          console.log('HTTP/3 connection established');
          
          // Track metrics
          const metrics = {
            protocol: 'h3',
            duration: entry.duration,
            transferSize: entry.transferSize,
            encodedBodySize: entry.encodedBodySize
          };
          
          // Send to analytics
          this.sendMetrics(metrics);
        }
      });
    });
    
    observer.observe({ entryTypes: ['navigation', 'resource'] });
  }

  static sendMetrics(metrics) {
    // Send to analytics service
    console.log('HTTP/3 metrics:', metrics);
  }
}
```

## Common Misconceptions

### Misconception 1: "HTTP/2 is always faster than HTTP/1.1"

**Reality:** HTTP/2 can be slower for single requests or on high-latency connections without proper optimization.

```javascript
// When HTTP/1.1 might be faster
const scenarios = {
  singleRequest: {
    http1: 'RTT + download',
    http2: 'RTT + download + protocol overhead',
    winner: 'HTTP/1.1 (marginally)'
  },
  
  highLatency: {
    http1: 'Multiple connections can parallelize RTT',
    http2: 'Single connection serializes handshake',
    winner: 'Depends on implementation'
  },
  
  smallFiles: {
    http1: 'Lower per-request overhead',
    http2: 'Frame overhead can be significant',
    winner: 'HTTP/1.1 for tiny files'
  }
};
```

### Misconception 2: "Server push always improves performance"

**Reality:** Overly aggressive server push can waste bandwidth and slow down initial render.

```javascript
// Server push pitfalls
const pushProblems = {
  cachingIssues: 'Pushing cached resources wastes bandwidth',
  prioritization: 'Pushed resources compete with HTML parsing',
  timing: 'Push too early = wasted bandwidth, too late = no benefit',
  
  bestPractices: {
    rule1: 'Only push truly critical resources',
    rule2: 'Check if resource is cached (Cookie-based tracking)',
    rule3: 'Use Link rel=preload as alternative',
    rule4: 'Monitor push reset stream (client already has resource)'
  }
};
```

### Misconception 3: "HTTP/3 eliminates all latency issues"

**Reality:** HTTP/3 improves certain scenarios but doesn't eliminate latency.

```javascript
const http3Limitations = {
  initialConnection: '0-RTT requires prior connection',
  udpBlocking: 'Some networks block UDP',
  cpuOverhead: 'QUIC processing more CPU intensive',
  maturity: 'Fewer optimization tools available'
};
```

### Misconception 4: "Resource concatenation is always bad with HTTP/2"

**Reality:** Extreme granularity can hurt due to overhead and cache efficiency.

```javascript
// Finding the balance
const resourceStrategy = {
  avoid: {
    oneGiantBundle: 'Poor cache granularity',
    tinyIndividualFiles: 'Too much overhead'
  },
  
  optimal: {
    approach: 'Logical chunking by functionality',
    example: [
      'vendor.js (rarely changes)',
      'app.js (main application)',
      'feature1.js (lazy loaded)',
      'feature2.js (lazy loaded)'
    ]
  }
};
```

## Performance Implications

### Bandwidth Efficiency

```javascript
// Bandwidth comparison
const bandwidthAnalysis = {
  headers: {
    http1: '500-800 bytes per request (uncompressed)',
    http2: '50-100 bytes per request (HPACK)',
    savings: '85-90%'
  },
  
  connections: {
    http1: '6 TCP connections = 6x handshake overhead',
    http2: '1 TCP connection = 1x handshake overhead',
    savings: '83%'
  },
  
  protocol: {
    http1: 'Text-based, verbose',
    http2: 'Binary, compact',
    savings: '10-20%'
  }
};
```

### Mobile Network Performance

```javascript
// Mobile considerations
class MobileOptimization {
  static analyzeProtocolBenefit(network) {
    const benefits = {
      '3G': {
        http2: 'High - RTT reduction critical (200-300ms RTT)',
        http3: 'Very High - Loss recovery crucial (5-10% loss)'
      },
      '4G': {
        http2: 'Medium - Lower RTT (50-100ms RTT)',
        http3: 'High - Connection migration valuable'
      },
      '5G': {
        http2: 'Low - RTT already low (20-30ms RTT)',
        http3: 'Medium - Mainly for connection migration'
      }
    };
    
    return benefits[network];
  }

  static optimizeForMobile() {
    return {
      enableHTTP3: true,
      enable0RTT: true,
      connectionMigration: true,
      adaptiveBitrate: true,
      resourceHints: [
        'dns-prefetch',
        'preconnect',
        'prefetch for predictable navigation'
      ]
    };
  }
}
```

## Browser Differences

### Protocol Support Matrix

```javascript
const browserSupport = {
  chrome: {
    http2: 'v43+ (2015)',
    http3: 'v90+ (2021)',
    serverPush: 'v43-105 (removed in 2022)',
    notes: 'First with HTTP/3 stable support'
  },
  
  firefox: {
    http2: 'v36+ (2015)',
    http3: 'v88+ (2021)',
    serverPush: 'v36+',
    notes: 'Can disable HTTP/3 via about:config'
  },
  
  safari: {
    http2: 'v9+ (2015)',
    http3: 'v14+ (2020)',
    serverPush: 'v9+',
    notes: 'Early HTTP/3 adopter'
  },
  
  edge: {
    http2: 'v14+ (2016)',
    http3: 'v90+ (2021)',
    serverPush: 'v14-105 (removed)',
    notes: 'Follows Chromium after v79'
  }
};

// Feature detection
function detectHTTP3Support() {
  return new Promise((resolve) => {
    fetch('https://example.com', { method: 'HEAD' })
      .then(response => {
        const entry = performance.getEntriesByType('resource')
          .find(e => e.name.includes('example.com'));
        
        resolve({
          supported: entry?.nextHopProtocol === 'h3',
          protocol: entry?.nextHopProtocol
        });
      });
  });
}
```

### Implementation Differences

```javascript
// Browser-specific behaviors
const browserQuirks = {
  chromium: {
    connectionPooling: 'Aggressive connection reuse',
    priority: 'Uses request priority hints',
    http3Fallback: 'Quick fallback to HTTP/2 on failures'
  },
  
  firefox: {
    connectionPooling: 'More conservative',
    priority: 'Custom prioritization logic',
    http3Fallback: 'Can be slower to fallback'
  },
  
  safari: {
    connectionPooling: 'Most conservative',
    priority: 'Limited priority hint support',
    http3Fallback: 'Reliable fallback mechanism'
  }
};
```

## Interview Questions

### Question 1: Explain head-of-line blocking in HTTP/1.1, HTTP/2, and HTTP/3

**Answer:** HTTP/1.1 suffers from application-layer HOL blocking where requests must complete sequentially on each connection. HTTP/2 eliminates application-layer blocking through multiplexing but still suffers from TCP-layer HOL blocking - if a TCP packet is lost, all streams wait for retransmission. HTTP/3 uses QUIC over UDP, eliminating TCP-layer HOL blocking by handling packet loss per-stream rather than per-connection. Only the affected stream needs to wait for retransmission while others continue.

### Question 2: Why did Chrome remove HTTP/2 Server Push support?

**Answer:** Chrome removed Server Push (2022) because: 1) Usage was very low (< 1% of HTTP/2 connections), 2) It was often misused, pushing resources already in cache, 3) It competed with HTML parsing for bandwidth, 4) Resource hints (preload, prefetch) provided better control, and 5) Early Hints (103 status) offered a superior alternative. Server Push's complexity didn't justify its limited real-world benefits.

### Question 3: How does HPACK compression work in HTTP/2?

**Answer:** HPACK uses static and dynamic tables to compress headers. The static table contains 61 predefined common header name-value pairs. The dynamic table learns from connection traffic. Headers are encoded as: indexed (reference to table), literal with indexing (add to dynamic table), or literal without indexing. This typically achieves 85-90% header compression. The dynamic table has a size limit and uses FIFO eviction. Both client and server maintain synchronized tables per connection.

### Question 4: What is 0-RTT in HTTP/3 and what are its security implications?

**Answer:** 0-RTT (Zero Round Trip Time) allows clients to send application data in the first packet when reconnecting to a previously-visited server, using cached session information. Security implications: 0-RTT data is not forward-secret and is vulnerable to replay attacks - a malicious middleman could capture and replay the initial packets. Therefore, only idempotent operations (GET requests, not POSTs) should be sent in 0-RTT data. Servers must implement replay protection mechanisms.

### Question 5: How does connection migration work in QUIC/HTTP/3?

**Answer:** QUIC uses connection IDs instead of the traditional 4-tuple (source IP/port, dest IP/port). When a client's network changes (WiFi to cellular), QUIC can continue the same connection by: 1) Sending packets from the new network path with the same connection ID, 2) Performing path validation using PATH_CHALLENGE/PATH_RESPONSE frames, 3) Migrating streams to the validated path. This happens transparently to the application, eliminating connection re-establishment and TLS re-negotiation.

### Question 6: Explain HTTP/2 stream prioritization and its limitations

**Answer:** HTTP/2 uses a dependency tree where streams can depend on other streams with weights (1-256) determining bandwidth allocation. A stream can exclusively depend on another, making existing children depend on it. Limitations: 1) Many servers don't implement prioritization correctly, 2) Client implementations vary widely, 3) Bandwidth allocation is complex to calculate, 4) No standard for priority hints from HTML. This led to development of Extensible Priorities (RFC 9218) providing simpler urgency/incremental model.

### Question 7: What are the tradeoffs between resource bundling and granular files in HTTP/2?

**Answer:** Bundling pros: Fewer files means less per-file overhead, better compression ratios within bundles, simpler cache warming. Cons: Larger initial download, poor cache granularity (changing one function invalidates entire bundle), delays initial parse/execution. Granular files pros: Better cache efficiency, faster incremental updates, parallel processing. Cons: More protocol overhead, potential over-fetching, complex dependency management. Optimal strategy: Logical chunking by change frequency and dependency relationships, typically 3-10 bundles rather than hundreds of files or one giant bundle.

### Question 8: How do you measure and compare HTTP protocol performance in production?

**Answer:** Key approaches: 1) Performance API: Check `nextHopProtocol` on navigation/resource entries to identify protocol version, 2) Connection metrics: Measure connection time, DNS, TCP, TLS handshake durations, 3) Resource timing: Compare duration, transferSize, encodedBodySize across protocols, 4) Real User Monitoring: Track Core Web Vitals (LCP, FID, CLS) segmented by protocol, 5) Server logs: Analyze protocol distribution and error rates, 6) A/B testing: Compare user cohorts with different protocol configurations, 7) Synthetic monitoring: Controlled tests across different network conditions and locations.

## Key Takeaways

1. **HTTP/1.1 Limitations**: Head-of-line blocking and connection limits (6 per domain) drove workarounds like domain sharding, concatenation, and sprites that are counterproductive with modern protocols

2. **HTTP/2 Multiplexing**: Single connection handles multiple concurrent streams using binary framing, eliminating application-layer HOL blocking but still subject to TCP-layer blocking

3. **HPACK Compression**: HTTP/2's header compression typically achieves 85-90% reduction using static/dynamic tables, critical for mobile performance with high header-to-payload ratios

4. **Server Push Evolution**: Initially promising but largely deprecated due to cache inefficiency and better alternatives (resource hints, Early Hints 103 status code)

5. **HTTP/3 Transport**: QUIC over UDP eliminates TCP HOL blocking by handling loss recovery per-stream, enabling true independent stream processing

6. **0-RTT Benefits**: HTTP/3's zero round-trip connection establishment eliminates handshake latency on repeat visits, crucial for mobile where RTT is 100-300ms

7. **Connection Migration**: QUIC's connection IDs enable seamless network changes (WiFi to cellular) without re-establishing connections or TLS handshakes

8. **Migration Strategy**: Moving from HTTP/1.1 to HTTP/2 requires removing anti-patterns (sharding, excessive concatenation), while HTTP/3 adoption is primarily infrastructure (CDN/server support)

9. **Browser Support**: HTTP/2 is universal (2015+), HTTP/3 is widely supported (2021+) but with implementation variations; feature detection is essential

10. **Performance Context**: Protocol benefits vary by network conditions - HTTP/3 shows greatest improvements on high-latency, lossy mobile networks (3G/4G) compared to low-latency, reliable connections

## Resources

### Official Specifications
- **HTTP/2 RFC 7540**: https://tools.ietf.org/html/rfc7540
- **HPACK RFC 7541**: https://tools.ietf.org/html/rfc7541
- **HTTP/3 RFC 9114**: https://www.rfc-editor.org/rfc/rfc9114.html
- **QUIC RFC 9000**: https://www.rfc-editor.org/rfc/rfc9000.html

### Implementation Guides
- **Cloudflare HTTP/3 Guide**: https://blog.cloudflare.com/http3-the-past-present-and-future/
- **Akamai HTTP/2 Best Practices**: https://http2.akamai.com/
- **Google Web Fundamentals HTTP/2**: https://developers.google.com/web/fundamentals/performance/http2

### Tools
- **HTTP/3 Check**: https://http3check.net/
- **WebPageTest**: Protocol comparison testing
- **Chrome DevTools**: Network protocol column
- **curl**: HTTP/2 and HTTP/3 support with --http2 and --http3 flags

### Articles
- **HTTP/2 is the future of the Web**: https://http2.github.io/
- **HTTP/3: the past, present, and future**: Cloudflare blog series
- **A Comprehensive Guide to HTTP/3 and QUIC**: Smashing Magazine
- **Why HTTP/3 is eating the world**: FastlyO blog
