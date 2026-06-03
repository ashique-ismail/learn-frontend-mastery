# WebRTC

## Introduction

WebRTC (Web Real-Time Communication) enables peer-to-peer audio, video, and data sharing between browsers without plugins or intermediaries. It's the foundation for video conferencing, file sharing, gaming, and real-time collaboration applications on the web.

## Getting Media Devices

```javascript
// Access camera and microphone
async function getMedia() {
    try {
        const stream = await navigator.mediaDevices.getUserMedia({
            video: {
                width: { ideal: 1280 },
                height: { ideal: 720 },
                facingMode: 'user'
            },
            audio: {
                echoCancellation: true,
                noiseSuppression: true
            }
        });
        
        const videoElement = document.getElementById('localVideo');
        videoElement.srcObject = stream;
        
        return stream;
    } catch (error) {
        console.error('Error accessing media devices:', error);
    }
}
```

## Peer Connection

```javascript
// Create peer connection
const configuration = {
    iceServers: [
        { urls: 'stun:stun.l.google.com:19302' },
        { urls: 'stun:stun1.l.google.com:19302' }
    ]
};

const peerConnection = new RTCPeerConnection(configuration);

// Add local stream
const localStream = await getMedia();
localStream.getTracks().forEach(track => {
    peerConnection.addTrack(track, localStream);
});

// Handle remote stream
peerConnection.ontrack = (event) => {
    const remoteVideo = document.getElementById('remoteVideo');
    remoteVideo.srcObject = event.streams[0];
};

// Handle ICE candidates
peerConnection.onicecandidate = (event) => {
    if (event.candidate) {
        // Send candidate to remote peer via signaling server
        sendToSignalingServer({
            type: 'ice-candidate',
            candidate: event.candidate
        });
    }
};
```

## Signaling

```javascript
// WebSocket signaling
const ws = new WebSocket('wss://signaling-server.com');

ws.onmessage = async (event) => {
    const message = JSON.parse(event.data);
    
    switch (message.type) {
        case 'offer':
            await peerConnection.setRemoteDescription(message.offer);
            const answer = await peerConnection.createAnswer();
            await peerConnection.setLocalDescription(answer);
            ws.send(JSON.stringify({ type: 'answer', answer }));
            break;
            
        case 'answer':
            await peerConnection.setRemoteDescription(message.answer);
            break;
            
        case 'ice-candidate':
            await peerConnection.addIceCandidate(message.candidate);
            break;
    }
};

// Create offer
async function createOffer() {
    const offer = await peerConnection.createOffer();
    await peerConnection.setLocalDescription(offer);
    ws.send(JSON.stringify({ type: 'offer', offer }));
}
```

## Data Channels

```javascript
// Create data channel
const dataChannel = peerConnection.createDataChannel('chat', {
    ordered: true
});

dataChannel.onopen = () => {
    console.log('Data channel opened');
    dataChannel.send('Hello!');
};

dataChannel.onmessage = (event) => {
    console.log('Received:', event.data);
};

dataChannel.onerror = (error) => {
    console.error('Data channel error:', error);
};

// On receiving end
peerConnection.ondatachannel = (event) => {
    const receiveChannel = event.channel;
    
    receiveChannel.onmessage = (e) => {
        console.log('Message:', e.data);
    };
};
```

## Screen Sharing

```javascript
async function shareScreen() {
    try {
        const screenStream = await navigator.mediaDevices.getDisplayMedia({
            video: {
                cursor: 'always'
            },
            audio: false
        });
        
        // Replace video track
        const videoTrack = screenStream.getVideoTracks()[0];
        const sender = peerConnection.getSenders().find(s => 
            s.track?.kind === 'video'
        );
        
        if (sender) {
            sender.replaceTrack(videoTrack);
        }
        
        // Handle screen share stop
        videoTrack.onended = () => {
            console.log('Screen sharing stopped');
        };
        
        return screenStream;
    } catch (error) {
        console.error('Error sharing screen:', error);
    }
}
```

## Video Conferencing App

```javascript
class VideoConference {
    constructor() {
        this.localStream = null;
        this.peers = new Map();
        this.signalingServer = new WebSocket('wss://signal.example.com');
        this.setupSignaling();
    }
    
    async init() {
        this.localStream = await this.getLocalStream();
        this.displayLocalVideo();
    }
    
    async getLocalStream() {
        return await navigator.mediaDevices.getUserMedia({
            video: true,
            audio: true
        });
    }
    
    displayLocalVideo() {
        const video = document.getElementById('localVideo');
        video.srcObject = this.localStream;
        video.muted = true;
    }
    
    setupSignaling() {
        this.signalingServer.onmessage = async (event) => {
            const message = JSON.parse(event.data);
            await this.handleSignaling(message);
        };
    }
    
    async createPeerConnection(peerId) {
        const pc = new RTCPeerConnection({
            iceServers: [
                { urls: 'stun:stun.l.google.com:19302' }
            ]
        });
        
        this.localStream.getTracks().forEach(track => {
            pc.addTrack(track, this.localStream);
        });
        
        pc.ontrack = (event) => {
            this.displayRemoteVideo(peerId, event.streams[0]);
        };
        
        pc.onicecandidate = (event) => {
            if (event.candidate) {
                this.sendToSignaling({
                    type: 'ice-candidate',
                    target: peerId,
                    candidate: event.candidate
                });
            }
        };
        
        this.peers.set(peerId, pc);
        return pc;
    }
    
    displayRemoteVideo(peerId, stream) {
        let video = document.getElementById(`remote-${peerId}`);
        
        if (!video) {
            video = document.createElement('video');
            video.id = `remote-${peerId}`;
            video.autoplay = true;
            document.getElementById('remoteVideos').appendChild(video);
        }
        
        video.srcObject = stream;
    }
    
    async handleSignaling(message) {
        const { type, from, offer, answer, candidate } = message;
        let pc = this.peers.get(from);
        
        if (!pc) {
            pc = await this.createPeerConnection(from);
        }
        
        switch (type) {
            case 'offer':
                await pc.setRemoteDescription(offer);
                const answer_sd = await pc.createAnswer();
                await pc.setLocalDescription(answer_sd);
                this.sendToSignaling({
                    type: 'answer',
                    target: from,
                    answer: answer_sd
                });
                break;
                
            case 'answer':
                await pc.setRemoteDescription(answer);
                break;
                
            case 'ice-candidate':
                await pc.addIceCandidate(candidate);
                break;
        }
    }
    
    sendToSignaling(message) {
        this.signalingServer.send(JSON.stringify(message));
    }
    
    async callPeer(peerId) {
        const pc = await this.createPeerConnection(peerId);
        const offer = await pc.createOffer();
        await pc.setLocalDescription(offer);
        
        this.sendToSignaling({
            type: 'offer',
            target: peerId,
            offer
        });
    }
}

// Usage
const conference = new VideoConference();
await conference.init();
```

## Recording

```javascript
// Record media stream
let mediaRecorder;
let recordedChunks = [];

function startRecording(stream) {
    recordedChunks = [];
    
    mediaRecorder = new MediaRecorder(stream, {
        mimeType: 'video/webm; codecs=vp9'
    });
    
    mediaRecorder.ondataavailable = (event) => {
        if (event.data.size > 0) {
            recordedChunks.push(event.data);
        }
    };
    
    mediaRecorder.onstop = () => {
        const blob = new Blob(recordedChunks, {
            type: 'video/webm'
        });
        
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = 'recording.webm';
        a.click();
    };
    
    mediaRecorder.start();
}

function stopRecording() {
    if (mediaRecorder && mediaRecorder.state !== 'inactive') {
        mediaRecorder.stop();
    }
}
```

## Browser Support

| Feature | Chrome | Firefox | Safari | Edge |
|---------|--------|---------|--------|------|
| getUserMedia | 53+ | 36+ | 11+ | 79+ |
| RTCPeerConnection | 56+ | 44+ | 11+ | 79+ |
| Data Channels | 56+ | 44+ | 11+ | 79+ |
| Screen Sharing | 72+ | 66+ | 13+ | 79+ |

## Common Issues

1. **NAT Traversal** - Use TURN servers
2. **Signaling** - Implement reliable signaling
3. **Network Changes** - Handle ICE restart
4. **Permissions** - Request media access properly
5. **Browser Compatibility** - Test across browsers

## Best Practices

1. Use STUN/TURN servers
2. Implement proper signaling
3. Handle connection failures
4. Monitor connection quality
5. Optimize bandwidth usage
6. Handle permissions gracefully
7. Test on various networks
8. Implement reconnection logic

## Key Takeaways

- WebRTC enables P2P communication
- Requires signaling server
- STUN/TURN servers help with NAT
- Data channels for non-media data
- Screen sharing is built-in
- Recording streams is straightforward
- Excellent browser support

## Resources

- [MDN: WebRTC API](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API)
- [WebRTC Samples](https://webrtc.github.io/samples/)
- [WebRTC for the Curious](https://webrtcforthecurious.com/)
- [Getting Started with WebRTC](https://web.dev/webrtc-basics/)
