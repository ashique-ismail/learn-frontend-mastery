# Future Web APIs and Standards

## The Idea

**In plain English:** Future Web APIs are new built-in powers that browsers are gaining, letting websites do things they previously could not — like reading files directly from your computer, talking to physical gadgets plugged into a USB port, or streaming live data at very high speed. An API (Application Programming Interface) is simply a set of instructions the browser provides so your code can ask it to do something specific.

**Real-world analogy:** Imagine a hotel adding brand-new services over the years — first there was just a phone in your room, then room service, then a gym, then a spa. Each new service is announced at the front desk (the browser), and you request it by calling down (your code calling the API).

- The hotel front desk = the browser
- Each new hotel service (gym, spa, room service) = a new Web API (File System Access, Web Serial, WebTransport, etc.)
- You calling the front desk to request a service = your JavaScript code calling the API to ask the browser to do something

---

## Introduction

The web platform continuously evolves with new APIs that bring native-like capabilities to web applications. This guide covers emerging and experimental APIs including File System Access, Web Serial, WebTransport, and other cutting-edge features shaping the future of web development.

## File System Access API

### Reading Files

```javascript
async function openFile() {
    try {
        const [fileHandle] = await window.showOpenFilePicker({
            types: [{
                description: 'Text Files',
                accept: { 'text/plain': ['.txt'] }
            }],
            multiple: false
        });
        
        const file = await fileHandle.getFile();
        const contents = await file.text();
        
        console.log('File contents:', contents);
        return contents;
    } catch (error) {
        console.error('Error opening file:', error);
    }
}
```

### Writing Files

```javascript
async function saveFile(contents) {
    try {
        const fileHandle = await window.showSaveFilePicker({
            suggestedName: 'document.txt',
            types: [{
                description: 'Text Files',
                accept: { 'text/plain': ['.txt'] }
            }]
        });
        
        const writable = await fileHandle.createWritable();
        await writable.write(contents);
        await writable.close();
        
        console.log('File saved successfully');
    } catch (error) {
        console.error('Error saving file:', error);
    }
}
```

### Directory Access

```javascript
async function openDirectory() {
    try {
        const dirHandle = await window.showDirectoryPicker();
        
        for await (const entry of dirHandle.values()) {
            if (entry.kind === 'file') {
                const file = await entry.getFile();
                console.log('File:', file.name);
            } else if (entry.kind === 'directory') {
                console.log('Directory:', entry.name);
            }
        }
    } catch (error) {
        console.error('Error accessing directory:', error);
    }
}
```

### File Editor Example

```javascript
class FileEditor {
    constructor() {
        this.fileHandle = null;
        this.modified = false;
    }
    
    async new() {
        this.fileHandle = null;
        this.modified = false;
        document.getElementById('editor').value = '';
    }
    
    async open() {
        const [handle] = await window.showOpenFilePicker({
            types: [{
                description: 'Text Files',
                accept: { 'text/*': ['.txt', '.js', '.html', '.css'] }
            }]
        });
        
        this.fileHandle = handle;
        const file = await handle.getFile();
        const contents = await file.text();
        
        document.getElementById('editor').value = contents;
        this.modified = false;
    }
    
    async save() {
        if (!this.fileHandle) {
            return this.saveAs();
        }
        
        const writable = await this.fileHandle.createWritable();
        await writable.write(document.getElementById('editor').value);
        await writable.close();
        
        this.modified = false;
    }
    
    async saveAs() {
        const handle = await window.showSaveFilePicker({
            types: [{
                description: 'Text Files',
                accept: { 'text/plain': ['.txt'] }
            }]
        });
        
        this.fileHandle = handle;
        await this.save();
    }
}

const editor = new FileEditor();
```

## Web Serial API

### Basic Serial Communication

```javascript
async function connectSerial() {
    try {
        const port = await navigator.serial.requestPort();
        
        await port.open({ baudRate: 9600 });
        
        console.log('Serial port opened');
        return port;
    } catch (error) {
        console.error('Serial connection error:', error);
    }
}

async function readSerial(port) {
    const reader = port.readable.getReader();
    
    try {
        while (true) {
            const { value, done } = await reader.read();
            if (done) break;
            
            // value is a Uint8Array
            console.log('Received:', new TextDecoder().decode(value));
        }
    } catch (error) {
        console.error('Read error:', error);
    } finally {
        reader.releaseLock();
    }
}

async function writeSerial(port, data) {
    const writer = port.writable.getWriter();
    const encoder = new TextEncoder();
    
    await writer.write(encoder.encode(data));
    
    writer.releaseLock();
}
```

### Arduino Communication

```javascript
class ArduinoController {
    constructor() {
        this.port = null;
        this.reader = null;
        this.writer = null;
    }
    
    async connect() {
        this.port = await navigator.serial.requestPort();
        await this.port.open({ baudRate: 9600 });
        
        this.reader = this.port.readable.getReader();
        this.writer = this.port.writable.getWriter();
        
        this.startReading();
    }
    
    async startReading() {
        try {
            while (true) {
                const { value, done } = await this.reader.read();
                if (done) break;
                
                const text = new TextDecoder().decode(value);
                this.handleData(text);
            }
        } catch (error) {
            console.error('Read error:', error);
        }
    }
    
    handleData(data) {
        console.log('Arduino:', data);
        // Process sensor data, etc.
    }
    
    async sendCommand(command) {
        const encoder = new TextEncoder();
        await this.writer.write(encoder.encode(command + '\n'));
    }
    
    async disconnect() {
        if (this.reader) {
            await this.reader.cancel();
            this.reader.releaseLock();
        }
        if (this.writer) {
            this.writer.releaseLock();
        }
        if (this.port) {
            await this.port.close();
        }
    }
}

// Usage
const arduino = new ArduinoController();
await arduino.connect();
await arduino.sendCommand('LED_ON');
```

## WebTransport API

### Establishing Connection

```javascript
async function connectWebTransport() {
    const url = 'https://example.com:4433/transport';
    
    const transport = new WebTransport(url);
    
    await transport.ready;
    console.log('WebTransport connected');
    
    return transport;
}

async function sendData(transport, data) {
    const writer = transport.datagrams.writable.getWriter();
    const encoder = new TextEncoder();
    
    await writer.write(encoder.encode(data));
    writer.releaseLock();
}

async function receiveData(transport) {
    const reader = transport.datagrams.readable.getReader();
    
    try {
        while (true) {
            const { value, done } = await reader.read();
            if (done) break;
            
            const data = new TextDecoder().decode(value);
            console.log('Received:', data);
        }
    } catch (error) {
        console.error('Receive error:', error);
    } finally {
        reader.releaseLock();
    }
}
```

## Web NFC API

```javascript
async function readNFC() {
    try {
        const ndef = new NDEFReader();
        await ndef.scan();
        
        ndef.addEventListener('reading', ({ message, serialNumber }) => {
            console.log('NFC tag serial:', serialNumber);
            
            for (const record of message.records) {
                console.log('Record type:', record.recordType);
                console.log('Data:', record.data);
            }
        });
    } catch (error) {
        console.error('NFC error:', error);
    }
}

async function writeNFC() {
    try {
        const ndef = new NDEFReader();
        
        await ndef.write({
            records: [{
                recordType: 'text',
                data: 'Hello NFC!'
            }]
        });
        
        console.log('NFC tag written');
    } catch (error) {
        console.error('Write error:', error);
    }
}
```

## WebCodecs API

```javascript
// Video encoding
async function encodeVideo(videoFrame) {
    const encoder = new VideoEncoder({
        output: (chunk, metadata) => {
            // Handle encoded chunk
            console.log('Encoded chunk:', chunk);
        },
        error: (error) => {
            console.error('Encoding error:', error);
        }
    });
    
    encoder.configure({
        codec: 'vp8',
        width: 1280,
        height: 720,
        bitrate: 2_000_000,
        framerate: 30
    });
    
    encoder.encode(videoFrame);
    await encoder.flush();
    encoder.close();
}

// Image decoding
async function decodeImage(imageData) {
    const decoder = new ImageDecoder({
        data: imageData,
        type: 'image/png'
    });
    
    const result = await decoder.decode();
    const frame = result.image;
    
    // Use the frame
    console.log('Decoded frame:', frame);
    
    frame.close();
}
```

## Screen Wake Lock API

```javascript
async function requestWakeLock() {
    try {
        const wakeLock = await navigator.wakeLock.request('screen');
        
        console.log('Wake lock active');
        
        wakeLock.addEventListener('release', () => {
            console.log('Wake lock released');
        });
        
        // Release after some time
        setTimeout(async () => {
            await wakeLock.release();
        }, 60000); // 1 minute
        
        return wakeLock;
    } catch (error) {
        console.error('Wake lock error:', error);
    }
}
```

## Web Share API (Level 2)

```javascript
async function shareWithFiles() {
    try {
        const files = [
            new File(['Hello'], 'hello.txt', { type: 'text/plain' })
        ];
        
        if (navigator.canShare && navigator.canShare({ files })) {
            await navigator.share({
                files,
                title: 'Shared File',
                text: 'Check out this file'
            });
        }
    } catch (error) {
        console.error('Share error:', error);
    }
}
```

## Eye Dropper API

```javascript
async function pickColor() {
    try {
        const eyeDropper = new EyeDropper();
        const result = await eyeDropper.open();
        
        console.log('Selected color:', result.sRGBHex);
        return result.sRGBHex;
    } catch (error) {
        console.error('Eye dropper error:', error);
    }
}
```

## Browser Support Matrix

| API | Chrome | Firefox | Safari | Status |
|-----|--------|---------|--------|--------|
| File System Access | 86+ | ❌ | ❌ | Stable |
| Web Serial | 89+ | ❌ | ❌ | Stable |
| WebTransport | 97+ | ❌ | ❌ | Stable |
| Web NFC | 89+ | ❌ | ❌ | Stable |
| WebCodecs | 94+ | ❌ | ❌ | Stable |
| Wake Lock | 84+ | 126+ | 16.4+ | Stable |
| Eye Dropper | 95+ | ❌ | ❌ | Stable |

## Security Considerations

1. **Permission Required** - All APIs require user permission
2. **HTTPS Only** - Most APIs require secure context
3. **User Gesture** - Many require user interaction
4. **Privacy** - Designed with privacy in mind
5. **Sandbox** - Limited access by design

## Best Practices

1. Check API availability before use
2. Handle permission denials gracefully
3. Provide clear user feedback
4. Test across browsers
5. Implement fallbacks
6. Document browser requirements
7. Consider progressive enhancement
8. Monitor adoption rates

## Future Outlook

### Upcoming APIs

- **WebGPU** - Advanced graphics
- **WebML** - Machine learning
- **Origin Trials** - Test new features
- **Project Fugu** - Closing capability gap

### Experimental Features

Enable in chrome://flags or use origin trials to test bleeding-edge features.

## Key Takeaways

- New APIs bring native capabilities to web
- Most require Chrome currently
- Security and privacy by design
- Progressive enhancement recommended
- Active development ongoing
- Test with origin trials
- Monitor browser support
- Plan for fallbacks

## Resources

- [Web.dev: New Capabilities](https://web.dev/new-capabilities/)
- [Project Fugu API Tracker](https://fugu-tracker.web.app/)
- [Chrome Platform Status](https://chromestatus.com/)
- [MDN: Experimental Features](https://developer.mozilla.org/en-US/docs/Web/API#experimental)
- [Origin Trials](https://developers.chrome.com/origintrials/)
- [Web Platform Status](https://www.webstatus.dev/)

## Conclusion

The web platform continues to evolve rapidly, bringing powerful new capabilities that enable developers to build increasingly sophisticated applications. While many of these APIs are currently Chrome-only, they represent the future direction of web development. By understanding and experimenting with these technologies now, developers can prepare for the next generation of web applications.
