# Browser Process Model

## Table of Contents
- [Introduction](#introduction)
- [Multi-Process Architecture](#multi-process-architecture)
- [Browser Process](#browser-process)
- [Renderer Process](#renderer-process)
- [GPU Process](#gpu-process)
- [Plugin and Extension Processes](#plugin-and-extension-processes)
- [Process Communication](#process-communication)
- [Site Isolation](#site-isolation)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Modern browsers use a multi-process architecture where different components run in separate processes. This design improves security, stability, and performance by isolating components and preventing failures from affecting the entire browser.

Key concepts:
- **Browser Process**: Main process managing UI and coordinating other processes
- **Renderer Process**: Renders web pages, one per site (with Site Isolation)
- **GPU Process**: Handles graphics operations for hardware acceleration
- **Plugin Process**: Runs browser plugins in isolation
- **Process Communication**: IPC (Inter-Process Communication) between processes

## Multi-Process Architecture

### Architecture Overview

Understanding Chrome's process model:

```javascript
// Browser process architecture
const browserArchitecture = {
  // Main coordinator
  browserProcess: {
    responsibilities: [
      'UI rendering (tabs, address bar, bookmarks)',
      'Network requests',
      'File system access',
      'Cookie/session management',
      'Process management',
      'Security policies'
    ],
    privileges: 'full system access'
  },

  // One per site (with Site Isolation)
  rendererProcess: {
    responsibilities: [
      'HTML/CSS parsing',
      'JavaScript execution',
      'Layout calculation',
      'Paint operations',
      'Event handling'
    ],
    privileges: 'sandboxed (limited access)',
    count: 'multiple (one per site)'
  },

  // Shared GPU process
  gpuProcess: {
    responsibilities: [
      'WebGL rendering',
      'Canvas acceleration',
      'Video decoding',
      'CSS animations',
      'Compositing'
    ],
    privileges: 'access to GPU',
    count: 'one'
  },

  // Optional processes
  pluginProcess: {
    responsibilities: ['Run plugins (Flash, PDF viewer)'],
    privileges: 'varies by plugin',
    count: 'one per plugin instance'
  },

  // Extension isolation
  extensionProcess: {
    responsibilities: ['Run browser extensions'],
    privileges: 'based on permissions',
    count: 'shared or dedicated based on extension'
  },

  // Background services
  utilityProcess: {
    responsibilities: [
      'Audio processing',
      'Data decoding',
      'Network service',
      'Storage service'
    ],
    privileges: 'sandboxed',
    count: 'multiple'
  }
};
```

### Process Creation

How browsers create processes:

```javascript
// Simplified process creation flow
class ProcessManager {
  constructor() {
    this.processes = new Map();
    this.nextProcessId = 1;
  }

  // Create new renderer process
  createRendererProcess(siteUrl) {
    const processId = this.nextProcessId++;
    
    // Check if process already exists for this site
    const site = this.getSiteFromUrl(siteUrl);
    const existingProcess = this.findProcessForSite(site);

    if (existingProcess && this.canReuseProcess(existingProcess)) {
      console.log(`Reusing process ${existingProcess.id} for ${site}`);
      return existingProcess;
    }

    // Create new process
    const process = {
      id: processId,
      type: 'renderer',
      site: site,
      pid: this.spawnProcess('renderer'),
      createdAt: Date.now(),
      memory: 0,
      cpu: 0
    };

    this.processes.set(processId, process);
    console.log(`Created renderer process ${processId} for ${site}`);

    return process;
  }

  getSiteFromUrl(url) {
    // Extract site from URL (scheme + eTLD+1)
    const parsed = new URL(url);
    return `${parsed.protocol}//${parsed.hostname}`;
  }

  findProcessForSite(site) {
    for (const process of this.processes.values()) {
      if (process.site === site && process.type === 'renderer') {
        return process;
      }
    }
    return null;
  }

  canReuseProcess(process) {
    // Check if process can handle another page
    const maxPagesPerProcess = 10;
    const pageCount = this.getPageCountForProcess(process.id);
    
    return pageCount < maxPagesPerProcess;
  }

  spawnProcess(type) {
    // Simulate process spawning
    // In reality, this calls OS APIs to create new process
    return Math.floor(Math.random() * 10000);
  }

  getPageCountForProcess(processId) {
    // Count pages in this process
    return 3; // Simplified
  }

  // Terminate process
  terminateProcess(processId) {
    const process = this.processes.get(processId);
    
    if (process) {
      console.log(`Terminating process ${processId} (PID: ${process.pid})`);
      this.processes.delete(processId);
      
      // OS call to kill process
      // process.kill(process.pid);
    }
  }

  // Get all processes
  getAllProcesses() {
    return Array.from(this.processes.values());
  }
}
```

### Process Limits

Managing process count:

```javascript
// Process limit management
class ProcessLimitManager {
  constructor() {
    this.maxRendererProcesses = this.calculateMaxProcesses();
    this.activeProcesses = [];
  }

  calculateMaxProcesses() {
    // Base calculation on system resources
    const totalRAM = this.getSystemRAM(); // GB
    const cpuCores = this.getCPUCores();

    // Heuristic: One process per GB of RAM or CPU core
    const ramBasedLimit = Math.floor(totalRAM / 2);
    const cpuBasedLimit = cpuCores * 4;

    // Take minimum, with absolute limits
    const calculated = Math.min(ramBasedLimit, cpuBasedLimit);
    const min = 2;
    const max = 20;

    return Math.max(min, Math.min(max, calculated));
  }

  getSystemRAM() {
    // In browser, approximate based on navigator.deviceMemory
    return navigator.deviceMemory || 4;
  }

  getCPUCores() {
    return navigator.hardwareConcurrency || 2;
  }

  canCreateProcess() {
    return this.activeProcesses.length < this.maxRendererProcesses;
  }

  requestProcess(site) {
    if (this.canCreateProcess()) {
      // Create new process
      return this.createProcess(site);
    } else {
      // Reuse existing process
      return this.findReusableProcess(site);
    }
  }

  createProcess(site) {
    const process = {
      id: Date.now(),
      site: site,
      pages: 1,
      memory: 50 * 1024 * 1024 // 50MB initial
    };

    this.activeProcesses.push(process);
    return process;
  }

  findReusableProcess(site) {
    // Find process with lowest page count for same site
    const siteProcesses = this.activeProcesses
      .filter(p => p.site === site)
      .sort((a, b) => a.pages - b.pages);

    if (siteProcesses.length > 0) {
      const process = siteProcesses[0];
      process.pages++;
      return process;
    }

    // No same-site process, find any process with space
    const available = this.activeProcesses
      .filter(p => p.pages < 10)
      .sort((a, b) => a.pages - b.pages);

    if (available.length > 0) {
      available[0].pages++;
      return available[0];
    }

    // Force create if absolutely necessary
    return this.createProcess(site);
  }
}
```

## Browser Process

### Responsibilities

The main coordinator process:

```javascript
// Browser process responsibilities
class BrowserProcess {
  constructor() {
    this.ui = new UIManager();
    this.network = new NetworkService();
    this.storage = new StorageService();
    this.processManager = new ProcessManager();
  }

  // UI Management
  handleUserInput(event) {
    switch (event.type) {
      case 'addressBarInput':
        this.handleNavigation(event.url);
        break;
      
      case 'newTab':
        this.createNewTab();
        break;
      
      case 'closeTab':
        this.closeTab(event.tabId);
        break;
      
      case 'bookmark':
        this.storage.addBookmark(event.url);
        break;
    }
  }

  // Navigation
  handleNavigation(url) {
    // 1. Check if URL is valid
    if (!this.isValidUrl(url)) {
      this.showError('Invalid URL');
      return;
    }

    // 2. Check security policies
    if (this.isMaliciousUrl(url)) {
      this.showSecurityWarning(url);
      return;
    }

    // 3. Request renderer process
    const site = this.getSiteFromUrl(url);
    const rendererProcess = this.processManager.createRendererProcess(url);

    // 4. Initiate navigation
    this.sendMessageToRenderer(rendererProcess.id, {
      type: 'navigate',
      url: url
    });

    // 5. Make network request
    this.network.fetch(url).then(response => {
      // 6. Send data to renderer
      this.sendMessageToRenderer(rendererProcess.id, {
        type: 'data',
        html: response.body
      });
    });
  }

  // Network requests (all go through browser process)
  makeNetworkRequest(rendererProcessId, url, options) {
    // Validate request from renderer
    if (!this.canMakeRequest(rendererProcessId, url)) {
      return Promise.reject('Unauthorized request');
    }

    // Make request on behalf of renderer
    return this.network.fetch(url, options);
  }

  // File system access
  readFile(rendererProcessId, filePath) {
    // Renderers can't access file system directly
    // Browser process mediates
    if (!this.hasFilePermission(rendererProcessId, filePath)) {
      throw new Error('No permission');
    }

    return this.storage.readFile(filePath);
  }

  // Cookie management
  getCookies(rendererProcessId, url) {
    return this.storage.getCookiesForUrl(url);
  }

  setCookie(rendererProcessId, url, cookie) {
    // Validate cookie
    if (this.isValidCookie(cookie)) {
      this.storage.setCookieForUrl(url, cookie);
    }
  }

  isValidUrl(url) {
    try {
      new URL(url);
      return true;
    } catch {
      return false;
    }
  }

  isMaliciousUrl(url) {
    // Check against Safe Browsing API
    return false; // Simplified
  }

  canMakeRequest(processId, url) {
    // Check CORS, CSP, etc.
    return true; // Simplified
  }

  sendMessageToRenderer(processId, message) {
    // IPC to renderer process
    console.log(`Browser → Renderer ${processId}:`, message);
  }
}
```

## Renderer Process

### Sandboxing

How renderer processes are sandboxed:

```javascript
// Renderer process sandbox
class RendererProcessSandbox {
  constructor() {
    this.restrictions = {
      // File system access
      fileSystem: 'denied',
      
      // Network access
      network: 'mediated', // Through browser process
      
      // System calls
      systemCalls: 'restricted',
      
      // Hardware access
      hardware: 'denied',
      
      // Cross-origin access
      crossOrigin: 'restricted'
    };
  }

  // Attempting restricted operation
  attemptFileRead(path) {
    if (this.restrictions.fileSystem === 'denied') {
      // Can't access directly, must ask browser process
      return this.requestFromBrowserProcess('readFile', { path });
    }
  }

  attemptNetworkRequest(url) {
    if (this.restrictions.network === 'mediated') {
      // All network goes through browser process
      return this.requestFromBrowserProcess('fetch', { url });
    }
  }

  requestFromBrowserProcess(action, params) {
    return new Promise((resolve, reject) => {
      // Send IPC message to browser process
      const messageId = Math.random();
      
      this.sendToBrowserProcess({
        id: messageId,
        action: action,
        params: params
      });

      // Wait for response
      this.waitForResponse(messageId).then(resolve).catch(reject);
    });
  }

  sendToBrowserProcess(message) {
    // IPC mechanism
    console.log('Renderer → Browser:', message);
  }

  waitForResponse(messageId) {
    // Wait for browser process response
    return new Promise(resolve => {
      setTimeout(() => {
        resolve({ success: true, data: 'mock data' });
      }, 10);
    });
  }
}
```

### Renderer Process Lifecycle

Managing renderer lifetime:

```javascript
// Renderer lifecycle management
class RendererLifecycle {
  constructor() {
    this.state = 'idle';
    this.pages = [];
    this.resources = {
      memory: 0,
      cpu: 0
    };
  }

  // Create page in renderer
  createPage(url) {
    const page = {
      id: Date.now(),
      url: url,
      domTree: null,
      renderTree: null,
      state: 'loading'
    };

    this.pages.push(page);
    this.state = 'active';

    // Start loading
    this.loadPage(page);

    return page;
  }

  async loadPage(page) {
    // 1. Request HTML from browser process
    const html = await this.requestResource(page.url);

    // 2. Parse HTML
    page.domTree = this.parseHTML(html);

    // 3. Request stylesheets
    const cssUrls = this.extractStylesheets(page.domTree);
    const stylesheets = await Promise.all(
      cssUrls.map(url => this.requestResource(url))
    );

    // 4. Parse CSS
    page.cssom = this.parseCSS(stylesheets.join('\n'));

    // 5. Build render tree
    page.renderTree = this.buildRenderTree(page.domTree, page.cssom);

    // 6. Layout
    this.layout(page.renderTree);

    // 7. Paint
    this.paint(page.renderTree);

    // 8. Mark as loaded
    page.state = 'loaded';
  }

  // Renderer crash handling
  crash() {
    console.log('Renderer process crashed!');
    
    // Send crash report to browser process
    this.sendToBrowserProcess({
      type: 'crash',
      pages: this.pages.map(p => p.url),
      reason: 'out of memory' // or other reason
    });

    // Browser process will:
    // 1. Show sad tab
    // 2. Terminate this process
    // 3. Create new process if user reloads
  }

  // Memory management
  monitorMemory() {
    const memoryUsage = this.calculateMemoryUsage();

    if (memoryUsage > 500 * 1024 * 1024) { // 500MB
      console.warn('High memory usage, attempting cleanup');
      this.garbageCollect();
    }

    if (memoryUsage > 1024 * 1024 * 1024) { // 1GB
      console.error('Memory limit exceeded');
      this.crash();
    }
  }

  calculateMemoryUsage() {
    // Sum of all resources
    return this.resources.memory;
  }

  garbageCollect() {
    // Trigger garbage collection
    // Release unused resources
    console.log('Running garbage collection');
  }

  requestResource(url) {
    return Promise.resolve('<html></html>');
  }

  parseHTML(html) {
    return {}; // Simplified
  }

  parseCSS(css) {
    return {}; // Simplified
  }

  extractStylesheets(dom) {
    return [];
  }

  buildRenderTree(dom, cssom) {
    return {};
  }

  layout(renderTree) {
    // Layout calculation
  }

  paint(renderTree) {
    // Paint operations
  }

  sendToBrowserProcess(message) {
    console.log('Renderer → Browser:', message);
  }
}
```

## GPU Process

### GPU Acceleration

How the GPU process works:

```javascript
// GPU process responsibilities
class GPUProcess {
  constructor() {
    this.context = this.initializeGPUContext();
    this.commandQueue = [];
    this.textures = new Map();
  }

  initializeGPUContext() {
    // Initialize WebGL context
    const canvas = new OffscreenCanvas(1, 1);
    return canvas.getContext('webgl2');
  }

  // Receive compositing commands from renderer
  receiveCompositingCommands(commands) {
    this.commandQueue.push(...commands);
    this.processCommands();
  }

  processCommands() {
    requestAnimationFrame(() => {
      for (const command of this.commandQueue) {
        this.executeCommand(command);
      }
      this.commandQueue = [];
    });
  }

  executeCommand(command) {
    switch (command.type) {
      case 'uploadTexture':
        this.uploadTexture(command.id, command.image);
        break;

      case 'drawQuad':
        this.drawTexturedQuad(
          command.textureId,
          command.transform,
          command.opacity
        );
        break;

      case 'applyFilter':
        this.applyShader(command.textureId, command.shader);
        break;

      case 'composite':
        this.compositeLayers(command.layers);
        break;
    }
  }

  // Upload texture to GPU
  uploadTexture(id, imageData) {
    const texture = this.context.createTexture();
    this.context.bindTexture(this.context.TEXTURE_2D, texture);
    
    this.context.texImage2D(
      this.context.TEXTURE_2D,
      0,
      this.context.RGBA,
      this.context.RGBA,
      this.context.UNSIGNED_BYTE,
      imageData
    );

    this.textures.set(id, texture);
  }

  // Draw textured quad with transform
  drawTexturedQuad(textureId, transform, opacity) {
    const texture = this.textures.get(textureId);
    
    if (!texture) {
      console.error('Texture not found:', textureId);
      return;
    }

    // Set up shader
    this.setupShader();

    // Apply transform
    this.applyTransform(transform);

    // Set opacity
    this.context.uniform1f(this.opacityLocation, opacity);

    // Bind texture
    this.context.bindTexture(this.context.TEXTURE_2D, texture);

    // Draw quad
    this.context.drawArrays(this.context.TRIANGLE_STRIP, 0, 4);
  }

  // Composite multiple layers
  compositeLayers(layers) {
    // Clear framebuffer
    this.context.clear(
      this.context.COLOR_BUFFER_BIT | this.context.DEPTH_BUFFER_BIT
    );

    // Draw each layer in order
    for (const layer of layers) {
      this.drawTexturedQuad(
        layer.textureId,
        layer.transform,
        layer.opacity
      );
    }

    // Present to screen
    this.present();
  }

  setupShader() {
    // Simplified shader setup
  }

  applyTransform(transform) {
    // Apply CSS transform matrix
  }

  present() {
    // Swap buffers, present to screen
  }

  // Video decoding
  decodeVideo(videoData) {
    // Hardware-accelerated video decoding
    console.log('Decoding video on GPU');
    return new Promise(resolve => {
      setTimeout(() => {
        resolve({ frames: [] });
      }, 10);
    });
  }
}
```

## Plugin and Extension Processes

### Plugin Isolation

Running plugins safely:

```javascript
// Plugin process isolation
class PluginProcess {
  constructor(pluginType) {
    this.pluginType = pluginType; // 'flash', 'pdf', etc.
    this.sandbox = this.createSandbox();
    this.state = 'idle';
  }

  createSandbox() {
    return {
      // Restricted capabilities
      fileSystem: 'limited',
      network: 'mediated',
      display: 'granted',
      
      // Memory limit
      maxMemory: 256 * 1024 * 1024 // 256MB
    };
  }

  loadPlugin(url, params) {
    console.log(`Loading ${this.pluginType} plugin:`, url);

    // Load plugin binary
    this.loadPluginBinary(url).then(binary => {
      // Initialize plugin in sandbox
      this.initializePlugin(binary, params);
      this.state = 'loaded';
    }).catch(error => {
      console.error('Plugin load failed:', error);
      this.notifyFailure();
    });
  }

  loadPluginBinary(url) {
    // Request plugin file through browser process
    return new Promise(resolve => {
      setTimeout(() => {
        resolve(new ArrayBuffer(1024));
      }, 100);
    });
  }

  initializePlugin(binary, params) {
    // Initialize plugin with parameters
    console.log('Plugin initialized');
  }

  // Plugin crash doesn't affect browser
  crash() {
    console.log('Plugin crashed');
    
    // Show error UI
    this.showPluginCrashError();
    
    // Isolate crash to this process only
    // Browser and other tabs continue working
  }

  showPluginCrashError() {
    // Send message to browser process
    this.sendToBrowserProcess({
      type: 'pluginCrash',
      pluginType: this.pluginType
    });
  }

  sendToBrowserProcess(message) {
    console.log('Plugin → Browser:', message);
  }

  notifyFailure() {
    this.sendToBrowserProcess({
      type: 'pluginLoadFailed',
      pluginType: this.pluginType
    });
  }
}

// Extension process
class ExtensionProcess {
  constructor(extensionId) {
    this.extensionId = extensionId;
    this.permissions = [];
    this.contentScripts = [];
  }

  initialize(manifest) {
    // Load extension manifest
    this.permissions = manifest.permissions || [];
    this.contentScripts = manifest.content_scripts || [];

    // Verify permissions
    this.verifyPermissions();
  }

  verifyPermissions() {
    // Check each requested permission
    for (const permission of this.permissions) {
      if (!this.isValidPermission(permission)) {
        throw new Error(`Invalid permission: ${permission}`);
      }
    }
  }

  isValidPermission(permission) {
    const validPermissions = [
      'tabs',
      'storage',
      'webRequest',
      'cookies',
      'history'
    ];

    return validPermissions.includes(permission);
  }

  // Execute content script
  injectContentScript(tabId) {
    // Request browser process to inject script
    this.sendToBrowserProcess({
      type: 'injectScript',
      tabId: tabId,
      scripts: this.contentScripts
    });
  }

  sendToBrowserProcess(message) {
    console.log('Extension → Browser:', message);
  }
}
```

## Process Communication

### IPC Mechanisms

Inter-process communication:

```javascript
// IPC (Inter-Process Communication)
class IPCChannel {
  constructor(sourceProcess, targetProcess) {
    this.source = sourceProcess;
    this.target = targetProcess;
    this.messageQueue = [];
    this.pendingResponses = new Map();
  }

  // Send message
  send(message) {
    const envelope = {
      id: Math.random(),
      from: this.source,
      to: this.target,
      timestamp: Date.now(),
      payload: message
    };

    // Serialize message
    const serialized = this.serialize(envelope);

    // Send through OS IPC mechanism
    this.sendToOS(serialized);
  }

  // Send with response
  sendSync(message) {
    return new Promise((resolve, reject) => {
      const messageId = Math.random();

      // Store pending response handler
      this.pendingResponses.set(messageId, { resolve, reject });

      // Send message
      this.send({
        ...message,
        id: messageId,
        expectsResponse: true
      });

      // Timeout after 5 seconds
      setTimeout(() => {
        if (this.pendingResponses.has(messageId)) {
          this.pendingResponses.delete(messageId);
          reject(new Error('IPC timeout'));
        }
      }, 5000);
    });
  }

  // Receive message
  receive(serialized) {
    const envelope = this.deserialize(serialized);

    // Check if this is a response
    if (envelope.payload.responseId) {
      this.handleResponse(envelope);
    } else {
      this.handleMessage(envelope);
    }
  }

  handleMessage(envelope) {
    const message = envelope.payload;

    // Route to handler
    this.routeMessage(message);

    // Send response if expected
    if (message.expectsResponse) {
      this.sendResponse(message.id, { success: true });
    }
  }

  handleResponse(envelope) {
    const response = envelope.payload;
    const pending = this.pendingResponses.get(response.responseId);

    if (pending) {
      pending.resolve(response.data);
      this.pendingResponses.delete(response.responseId);
    }
  }

  sendResponse(messageId, data) {
    this.send({
      responseId: messageId,
      data: data
    });
  }

  serialize(envelope) {
    // Serialize for IPC
    // In reality, uses Mojo IPC or similar
    return JSON.stringify(envelope);
  }

  deserialize(serialized) {
    return JSON.parse(serialized);
  }

  sendToOS(serialized) {
    // OS-specific IPC mechanism
    // Chrome uses Mojo IPC
    console.log('IPC:', serialized);
  }

  routeMessage(message) {
    // Route to appropriate handler
    console.log('Routing message:', message);
  }
}

// Example: Renderer to Browser communication
class RendererToBrowserIPC {
  constructor(rendererProcess) {
    this.renderer = rendererProcess;
    this.channel = new IPCChannel('renderer', 'browser');
  }

  // Request navigation
  async navigate(url) {
    return this.channel.sendSync({
      type: 'navigate',
      url: url
    });
  }

  // Request resource
  async fetch(url) {
    return this.channel.sendSync({
      type: 'fetch',
      url: url
    });
  }

  // Report crash
  reportCrash(reason) {
    this.channel.send({
      type: 'crash',
      reason: reason,
      fatal: true
    });
  }

  // Request cookie
  async getCookie(url) {
    return this.channel.sendSync({
      type: 'getCookie',
      url: url
    });
  }
}
```

## Site Isolation

### Per-Site Process Model

Isolating websites for security:

```javascript
// Site Isolation implementation
class SiteIsolationManager {
  constructor() {
    this.siteToProcess = new Map();
    this.processToSites = new Map();
  }

  // Get or create process for site
  getProcessForSite(url) {
    const site = this.extractSite(url);

    // Check if site already has a process
    if (this.siteToProcess.has(site)) {
      return this.siteToProcess.get(site);
    }

    // Create new process for site
    const process = this.createProcess(site);
    this.siteToProcess.set(site, process);

    // Track sites per process
    if (!this.processToSites.has(process.id)) {
      this.processToSites.set(process.id, []);
    }
    this.processToSites.get(process.id).push(site);

    return process;
  }

  extractSite(url) {
    // Extract "site" from URL
    // Site = scheme + eTLD+1
    // Example: https://example.com → https://example.com
    //          https://sub.example.com → https://example.com

    const parsed = new URL(url);
    const eTLD1 = this.getEffectiveTLD(parsed.hostname);

    return `${parsed.protocol}//${eTLD1}`;
  }

  getEffectiveTLD(hostname) {
    // Get eTLD+1 (simplified)
    // In reality, uses Public Suffix List
    const parts = hostname.split('.');
    
    if (parts.length <= 2) {
      return hostname;
    }

    // Return last two parts (simplified)
    return parts.slice(-2).join('.');
  }

  createProcess(site) {
    return {
      id: Date.now(),
      site: site,
      pid: Math.floor(Math.random() * 10000)
    };
  }

  // Check if two URLs are same-site
  isSameSite(url1, url2) {
    const site1 = this.extractSite(url1);
    const site2 = this.extractSite(url2);

    return site1 === site2;
  }

  // Cross-origin iframe gets separate process
  handleCrossOriginIframe(parentUrl, iframeUrl) {
    const parentSite = this.extractSite(parentUrl);
    const iframeSite = this.extractSite(iframeUrl);

    if (parentSite !== iframeSite) {
      // Different sites: use separate process
      console.log('Cross-origin iframe: separate process');
      return this.getProcessForSite(iframeUrl);
    } else {
      // Same site: can share process
      console.log('Same-origin iframe: shared process');
      return this.getProcessForSite(parentUrl);
    }
  }

  // Security benefits
  getSecurityBenefits() {
    return {
      spectreMitigation: 'Prevents Spectre attacks by process isolation',
      memoryIsolation: 'One site cannot read memory of another site',
      crashIsolation: 'Site crash does not affect other sites',
      sandboxEscape: 'Exploit in one site does not compromise others'
    };
  }
}
```

## Common Misconceptions

### Misconception 1: "One tab = one process"

**Reality**: Multiple tabs can share a process (same site), or one tab can use multiple processes (cross-origin iframes).

```javascript
// Examples:
// Tab 1: https://example.com        → Process A
// Tab 2: https://example.com/page2  → Process A (same site)
// Tab 3: https://other.com          → Process B (different site)

// Single tab with multiple processes:
// Parent: https://example.com       → Process A
// <iframe src="https://ads.com">    → Process B (cross-origin)
// <iframe src="https://example.com/widget"> → Process A (same-origin)
```

### Misconception 2: "Browser process does all the work"

**Reality**: Browser process coordinates, but renderer processes do the heavy lifting (parsing, layout, painting).

### Misconception 3: "More processes = better performance"

**Reality**: Too many processes increase memory overhead and IPC costs. Browsers balance between security and performance.

## Performance Implications

### Memory Usage

Process model memory cost:

```javascript
// Memory overhead per process
const processMemory = {
  baseOverhead: 10 * 1024 * 1024,      // 10MB per process
  v8Isolate: 5 * 1024 * 1024,          // 5MB for V8
  renderingContext: 15 * 1024 * 1024,   // 15MB for rendering
  total: 30 * 1024 * 1024               // ~30MB per process

  // With 10 processes:
  // 300MB just for process overhead!
};

// Optimization: Process sharing
const optimization = {
  singleProcess: {
    tabs: 10,
    processes: 1,
    overhead: 30 * 1024 * 1024, // 30MB
    risk: 'One crash affects all tabs'
  },

  perTabProcess: {
    tabs: 10,
    processes: 10,
    overhead: 300 * 1024 * 1024, // 300MB
    risk: 'High memory usage'
  },

  perSiteProcess: {
    tabs: 10,
    uniqueSites: 5,
    processes: 5,
    overhead: 150 * 1024 * 1024, // 150MB
    risk: 'Balanced approach'
  }
};
```

### IPC Overhead

Communication cost:

```javascript
// IPC performance
class IPCPerformance {
  measureIPCLatency() {
    const iterations = 1000;
    const start = performance.now();

    for (let i = 0; i < iterations; i++) {
      // Synchronous IPC (blocking)
      this.sendSync({ data: 'test' });
    }

    const end = performance.now();
    const avgLatency = (end - start) / iterations;

    console.log(`Average IPC latency: ${avgLatency.toFixed(2)}ms`);
    
    // Typical values:
    // Same machine IPC: 0.1-1ms
    // Impact: Adds overhead to every cross-process operation
  }

  sendSync(message) {
    // Simulate IPC round-trip
    return new Promise(resolve => {
      setTimeout(resolve, 0.5); // 0.5ms latency
    });
  }
}
```

## Interview Questions

### Question 1: Why do browsers use multiple processes?

**Answer**: Multi-process architecture provides:

1. **Security**: Sandboxed renderers can't access system resources directly
2. **Stability**: Renderer crash doesn't bring down entire browser
3. **Performance**: Parallel processing of different sites
4. **Isolation**: One site can't read memory from another (Spectre mitigation)

**Trade-off**: Higher memory usage due to process overhead.

### Question 2: What's the difference between browser and renderer processes?

**Answer**:

**Browser Process**:
- Main coordinator
- Handles UI, network, file system, cookies
- Full system privileges
- One per browser instance

**Renderer Process**:
- Renders web pages
- Sandboxed (limited privileges)
- One per site (with Site Isolation)
- Multiple per browser

**Communication**: Renderers request resources through browser process via IPC.

### Question 3: What is Site Isolation?

**Answer**: Site Isolation ensures different sites run in separate processes:

```javascript
// Example:
// https://example.com → Process A
// https://attacker.com → Process B

// Benefits:
// 1. Spectre mitigation: Can't read other site's memory
// 2. Security: Compromise of one site doesn't affect others
// 3. Stability: One site crash doesn't affect others
```

**How**: Browser determines "site" from scheme + eTLD+1 (effective top-level domain + 1 label).

### Question 4: What is the role of the GPU process?

**Answer**: GPU process handles graphics operations:

- WebGL rendering
- Canvas acceleration
- Video decoding
- CSS animations/transforms
- Layer compositing

**Why separate**: GPU operations need hardware access, so isolated for security. Single shared process serves all tabs.

### Question 5: How do processes communicate?

**Answer**: Via IPC (Inter-Process Communication):

```javascript
// Renderer wants to make network request
renderer.sendToBrowser({
  type: 'fetch',
  url: 'https://api.example.com/data'
});

// Browser makes request, sends response
browser.sendToRenderer({
  type: 'fetchResponse',
  data: { ... }
});
```

**Mechanism**: Chrome uses Mojo IPC for fast, type-safe communication.

**Cost**: IPC adds latency (0.1-1ms per round-trip).

### Question 6: What happens when a renderer process crashes?

**Answer**:

1. Browser process detects crash
2. Shows "Sad Tab" in affected tab(s)
3. Other tabs continue working (isolation benefit)
4. User can reload crashed tab
5. Browser creates new renderer process for reload

**Without multi-process**: Single crash would close entire browser.

### Question 7: How many processes does a browser create?

**Answer**: Depends on several factors:

```javascript
// Factors:
// - Number of unique sites open
// - Available system memory
// - CPU cores
// - Browser policy

// Typical limits:
// - Minimum: 2-4 processes
// - Maximum: 10-20 processes
// - Default: One per site (with sharing)

// Memory-constrained devices:
// Fewer processes, more sharing
```

### Question 8: What's the memory overhead of multi-process architecture?

**Answer**: Each process costs ~30-50MB overhead:

```javascript
// Breakdown per process:
// - Base process: ~10MB
// - V8 isolate: ~5-10MB
// - Rendering context: ~15-20MB
// - IPC buffers: ~5MB

// With 10 processes:
// Overhead: 300-500MB
```

**Optimization**: Browsers limit process count and share processes where safe to balance security and memory.

## Key Takeaways

1. **Multi-process architecture improves security, stability, and performance**
2. **Browser process coordinates, renderer processes do the work**
3. **Renderers are sandboxed - can't access system resources directly**
4. **Site Isolation: different sites in different processes (Spectre mitigation)**
5. **GPU process shared across all tabs for hardware acceleration**
6. **IPC enables cross-process communication but adds latency**
7. **Process count balanced between security and memory usage**
8. **Renderer crash isolated to affected tabs only**

## Resources

### Official Documentation
- [Chromium Multi-Process Architecture](https://www.chromium.org/developers/design-documents/multi-process-architecture/)
- [Site Isolation](https://www.chromium.org/Home/chromium-security/site-isolation/)
- [Chromium Security Architecture](https://www.chromium.org/Home/chromium-security/)

### Articles & Tutorials
- "Inside look at modern web browser" by Mariko Kosaka (Google)
- "Understanding Site Isolation" by Charlie Reis
- "Browser Process Model" by The Chromium Projects

### Videos
- Chrome University: Multi-Process Architecture
- BlinkOn: Site Isolation Deep Dive
- Google I/O: How Chrome Works

### Tools
- Chrome Task Manager (Shift+Esc)
- about://process-internals
- Chrome DevTools Performance Monitor

### Books
- "The Tangled Web" by Michal Zalewski
- "Web Browser Engineering" by Pavel Panchekha & Chris Harrelson
