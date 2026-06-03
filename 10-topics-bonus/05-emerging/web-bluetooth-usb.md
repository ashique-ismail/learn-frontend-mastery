# Web Bluetooth and Web USB

## Introduction

Web Bluetooth and Web USB APIs enable web applications to communicate directly with Bluetooth Low Energy devices and USB peripherals. These APIs bring hardware integration capabilities to the web, enabling applications for IoT devices, health monitors, gaming peripherals, and more.

## Web Bluetooth API

### Basic Bluetooth Connection

```javascript
async function connectToBluetoothDevice() {
    try {
        // Request device
        const device = await navigator.bluetooth.requestDevice({
            filters: [{
                services: ['heart_rate']
            }],
            optionalServices: ['battery_service']
        });
        
        console.log('Device:', device.name);
        
        // Connect to GATT server
        const server = await device.gatt.connect();
        console.log('Connected to GATT server');
        
        return server;
    } catch (error) {
        console.error('Bluetooth error:', error);
    }
}
```

### Reading Characteristics

```javascript
async function readHeartRate(server) {
    try {
        // Get service
        const service = await server.getPrimaryService('heart_rate');
        
        // Get characteristic
        const characteristic = await service.getCharacteristic('heart_rate_measurement');
        
        // Read value
        const value = await characteristic.readValue();
        const heartRate = value.getUint8(1);
        
        console.log('Heart rate:', heartRate);
        
        // Start notifications
        await characteristic.startNotifications();
        
        characteristic.addEventListener('characteristicvaluechanged', (event) => {
            const value = event.target.value;
            const heartRate = value.getUint8(1);
            console.log('Heart rate update:', heartRate);
        });
    } catch (error) {
        console.error('Error reading heart rate:', error);
    }
}
```

### Writing to Device

```javascript
async function writeToDevice(server, data) {
    try {
        const service = await server.getPrimaryService('custom_service');
        const characteristic = await service.getCharacteristic('custom_characteristic');
        
        // Convert string to array buffer
        const encoder = new TextEncoder();
        const dataBuffer = encoder.encode(data);
        
        await characteristic.writeValue(dataBuffer);
        console.log('Data written successfully');
    } catch (error) {
        console.error('Write error:', error);
    }
}
```

### Complete Bluetooth Device Manager

```javascript
class BluetoothDeviceManager {
    constructor() {
        this.device = null;
        this.server = null;
        this.characteristics = new Map();
    }
    
    async connect(filters, optionalServices = []) {
        try {
            this.device = await navigator.bluetooth.requestDevice({
                filters,
                optionalServices
            });
            
            this.device.addEventListener('gattserverdisconnected', () => {
                console.log('Device disconnected');
                this.handleDisconnection();
            });
            
            this.server = await this.device.gatt.connect();
            console.log('Connected to:', this.device.name);
            
            return true;
        } catch (error) {
            console.error('Connection failed:', error);
            return false;
        }
    }
    
    async getCharacteristic(serviceName, characteristicName) {
        const key = `${serviceName}-${characteristicName}`;
        
        if (this.characteristics.has(key)) {
            return this.characteristics.get(key);
        }
        
        const service = await this.server.getPrimaryService(serviceName);
        const characteristic = await service.getCharacteristic(characteristicName);
        
        this.characteristics.set(key, characteristic);
        return characteristic;
    }
    
    async read(serviceName, characteristicName) {
        const characteristic = await this.getCharacteristic(serviceName, characteristicName);
        const value = await characteristic.readValue();
        return value;
    }
    
    async write(serviceName, characteristicName, data) {
        const characteristic = await this.getCharacteristic(serviceName, characteristicName);
        await characteristic.writeValue(data);
    }
    
    async subscribe(serviceName, characteristicName, callback) {
        const characteristic = await this.getCharacteristic(serviceName, characteristicName);
        
        await characteristic.startNotifications();
        
        characteristic.addEventListener('characteristicvaluechanged', (event) => {
            callback(event.target.value);
        });
    }
    
    async disconnect() {
        if (this.server && this.server.connected) {
            await this.server.disconnect();
        }
    }
    
    handleDisconnection() {
        this.server = null;
        this.characteristics.clear();
        // Attempt reconnection if needed
    }
}

// Usage
const btManager = new BluetoothDeviceManager();

await btManager.connect([
    { services: ['heart_rate'] }
], ['battery_service']);

await btManager.subscribe('heart_rate', 'heart_rate_measurement', (value) => {
    console.log('Heart rate:', value.getUint8(1));
});
```

## Web USB API

### Requesting USB Device

```javascript
async function connectUSBDevice() {
    try {
        const device = await navigator.usb.requestDevice({
            filters: [{
                vendorId: 0x2341, // Arduino vendor ID
                productId: 0x8037
            }]
        });
        
        console.log('USB Device:', device.productName);
        
        await device.open();
        
        // Select configuration
        await device.selectConfiguration(1);
        
        // Claim interface
        await device.claimInterface(0);
        
        return device;
    } catch (error) {
        console.error('USB error:', error);
    }
}
```

### Reading from USB Device

```javascript
async function readFromUSB(device) {
    try {
        // Control transfer
        const result = await device.controlTransferIn({
            requestType: 'vendor',
            recipient: 'device',
            request: 0x01,
            value: 0x00,
            index: 0x00
        }, 64);
        
        if (result.status === 'ok') {
            const data = new Uint8Array(result.data.buffer);
            console.log('Received:', data);
        }
    } catch (error) {
        console.error('Read error:', error);
    }
}
```

### Writing to USB Device

```javascript
async function writeToUSB(device, data) {
    try {
        // Convert data to Uint8Array if needed
        const dataArray = new Uint8Array(data);
        
        const result = await device.controlTransferOut({
            requestType: 'vendor',
            recipient: 'device',
            request: 0x02,
            value: 0x00,
            index: 0x00
        }, dataArray);
        
        if (result.status === 'ok') {
            console.log('Data sent successfully');
        }
    } catch (error) {
        console.error('Write error:', error);
    }
}
```

### Complete USB Device Manager

```javascript
class USBDeviceManager {
    constructor() {
        this.device = null;
        this.setupEventListeners();
    }
    
    setupEventListeners() {
        navigator.usb.addEventListener('connect', (event) => {
            console.log('USB device connected:', event.device);
        });
        
        navigator.usb.addEventListener('disconnect', (event) => {
            console.log('USB device disconnected:', event.device);
            if (this.device === event.device) {
                this.device = null;
            }
        });
    }
    
    async connect(filters) {
        try {
            this.device = await navigator.usb.requestDevice({ filters });
            
            await this.device.open();
            await this.device.selectConfiguration(1);
            await this.device.claimInterface(0);
            
            console.log('Connected to:', this.device.productName);
            return true;
        } catch (error) {
            console.error('Connection failed:', error);
            return false;
        }
    }
    
    async read(length = 64) {
        if (!this.device) {
            throw new Error('Device not connected');
        }
        
        const result = await this.device.transferIn(1, length);
        
        if (result.status === 'ok') {
            return new Uint8Array(result.data.buffer);
        }
        
        throw new Error('Read failed');
    }
    
    async write(data) {
        if (!this.device) {
            throw new Error('Device not connected');
        }
        
        const result = await this.device.transferOut(1, data);
        
        if (result.status !== 'ok') {
            throw new Error('Write failed');
        }
    }
    
    async disconnect() {
        if (this.device) {
            await this.device.close();
            this.device = null;
        }
    }
}

// Usage
const usbManager = new USBDeviceManager();

await usbManager.connect([
    { vendorId: 0x2341 } // Arduino
]);

const data = await usbManager.read();
console.log('Read:', data);

await usbManager.write(new Uint8Array([0x01, 0x02, 0x03]));
```

## Practical Examples

### LED Controller (Bluetooth)

```javascript
class LEDController {
    constructor() {
        this.btManager = new BluetoothDeviceManager();
    }
    
    async connect() {
        await this.btManager.connect([
            { services: ['led_service'] }
        ]);
    }
    
    async setColor(r, g, b) {
        const color = new Uint8Array([r, g, b]);
        await this.btManager.write('led_service', 'color_characteristic', color);
    }
    
    async setBrightness(brightness) {
        const value = new Uint8Array([brightness]);
        await this.btManager.write('led_service', 'brightness_characteristic', value);
    }
}

// Usage
const led = new LEDController();
await led.connect();
await led.setColor(255, 0, 0); // Red
await led.setBrightness(128);   // 50% brightness
```

### Keyboard Input (USB)

```javascript
class USBKeyboard {
    constructor() {
        this.usbManager = new USBDeviceManager();
        this.onKeyPress = null;
    }
    
    async connect() {
        await this.usbManager.connect([
            { vendorId: 0x046d } // Logitech
        ]);
        
        this.startReading();
    }
    
    async startReading() {
        while (this.usbManager.device) {
            try {
                const data = await this.usbManager.read();
                this.processKeyData(data);
            } catch (error) {
                if (error.message !== 'Device not connected') {
                    console.error('Read error:', error);
                }
                break;
            }
        }
    }
    
    processKeyData(data) {
        // Parse keyboard data
        const keyCode = data[2];
        
        if (keyCode !== 0 && this.onKeyPress) {
            this.onKeyPress(keyCode);
        }
    }
}

// Usage
const keyboard = new USBKeyboard();
keyboard.onKeyPress = (keyCode) => {
    console.log('Key pressed:', keyCode);
};
await keyboard.connect();
```

## Browser Support

| Feature | Chrome | Firefox | Safari | Edge |
|---------|--------|---------|--------|------|
| Web Bluetooth | 56+ | ❌ | ❌ | 79+ |
| Web USB | 61+ | ❌ | ❌ | 79+ |

## Security Considerations

1. **User Permission Required** - Always requires user gesture
2. **HTTPS Only** - Must be served over HTTPS
3. **No Auto-Connect** - Cannot connect without user action
4. **Limited Access** - Can only access allowed services
5. **Privacy** - No device enumeration without permission

## Best Practices

1. Handle connection failures gracefully
2. Implement reconnection logic
3. Show clear device selection UI
4. Provide fallbacks for unsupported browsers
5. Test with actual hardware
6. Handle device disconnections
7. Document device requirements
8. Consider battery life impact

## Common Issues

1. **Device Not Found** - Check filters
2. **Permission Denied** - User must grant permission
3. **HTTPS Required** - Deploy over HTTPS
4. **Connection Failures** - Implement retry logic
5. **Browser Support** - Limited to Chromium browsers

## Key Takeaways

- Web Bluetooth enables BLE device communication
- Web USB enables USB device communication
- Both require user permission and HTTPS
- Limited browser support (mainly Chrome)
- Great for IoT and hardware projects
- Security-focused design

## Resources

- [Web Bluetooth API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API)
- [Web USB API](https://developer.mozilla.org/en-US/docs/Web/API/USB)
- [Web Bluetooth Samples](https://googlechrome.github.io/samples/web-bluetooth/)
- [WebUSB Explainer](https://wicg.github.io/webusb/)
