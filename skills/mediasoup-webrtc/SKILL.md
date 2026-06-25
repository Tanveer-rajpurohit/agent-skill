---
name: mediasoup-webrtc
description: >
  Build WebRTC video/audio conferencing with MediaSoup SFU. Trigger for:
  "mediasoup", "WebRTC", "SFU", "video call", "audio/video streaming",
  "transport", "producer", "consumer", "room management", "real-time
  media", "screen share", "500 concurrent streams", "media server",
  "webrtc signaling", "DTLS", "ICE", "RTP", "codec". Based on
  Tanveer's production CoNestify SFU achieving <200ms latency at 500+
  concurrent streams.
---

# MediaSoup / WebRTC Skill

Based on Tanveer's production guide + CoNestify experience:
custom WebRTC SFU, 500+ concurrent streams, sub-200ms latency,
60% faster than P2P mesh by eliminating O(n²) bandwidth scaling.

---

## Mental Model — Why SFU

```
P2P Mesh (bad at scale):
  A ←→ B ←→ C      Each peer uploads to n-1 others
  ↕       ↕        Bandwidth: O(n²)
  D ←→ E            At 10 users: 90 connections

SFU (MediaSoup):
  A → SFU → B       Each peer uploads ONCE to server
  B → SFU → A,C,D   Server fans out
  C → SFU → ...     Bandwidth: O(n)
                     At 10 users: 10 uploads + 90 downloads
                     BUT server handles the fan-out
```

MediaSoup = C++ media handling + Node.js control plane.
You handle signaling (Socket.IO). MediaSoup handles media.

---

## Core Concepts

```
Worker    → OS process, handles CPU-heavy media work
Router    → One per room, knows available codecs, routes media
Transport → WebRTC connection between peer and server
Producer  → Peer's outgoing stream (camera/mic → server)
Consumer  → Peer's incoming stream (server → viewer)
```

---

## Server Setup

### 1. Worker + Room Storage
```js
import mediasoup from 'mediasoup'

let worker
const rooms = new Map() // roomId → { router, peers }

const worker = await mediasoup.createWorker({
  logLevel: 'warn',
  rtcMinPort: 10000,
  rtcMaxPort: 10100,
})

worker.on('died', () => {
  console.error('MediaSoup worker died — restarting process')
  process.exit(1) // let your process manager restart
})
```

### 2. Create Room (Router)
```js
const MEDIA_CODECS = [
  { kind: 'audio', mimeType: 'audio/opus',  clockRate: 48000, channels: 2 },
  { kind: 'video', mimeType: 'video/VP8',   clockRate: 90000 },
  { kind: 'video', mimeType: 'video/H264',  clockRate: 90000,
    parameters: { 'packetization-mode': 1, 'profile-level-id': '42e01f' } },
]

async function getOrCreateRoom(roomId) {
  if (rooms.has(roomId)) return rooms.get(roomId)
  const router = await worker.createRouter({ mediaCodecs: MEDIA_CODECS })
  const room = { router, peers: new Map() }
  rooms.set(roomId, room)
  return room
}
```

### 3. Create WebRTC Transport
```js
async function createWebRtcTransport(router) {
  const transport = await router.createWebRtcTransport({
    listenIps: [
      { ip: '0.0.0.0', announcedIp: process.env.PUBLIC_IP } // set for production
    ],
    enableUdp: true,   // UDP preferred — lower latency
    enableTcp: true,   // TCP fallback for restrictive firewalls
    preferUdp: true,
    initialAvailableOutgoingBitrate: 1_000_000, // 1 Mbps start
  })

  // Important: close transport if connection fails
  transport.on('dtlsstatechange', (state) => {
    if (state === 'failed' || state === 'closed') transport.close()
  })

  return {
    transport,
    params: {
      id:             transport.id,
      iceParameters:  transport.iceParameters,
      iceCandidates:  transport.iceCandidates,
      dtlsParameters: transport.dtlsParameters,
    }
  }
}
```

### 4. Socket.IO Signaling Flow
```js
io.on('connection', (socket) => {
  let currentRoomId = null

  // Step 1: Join room → get RTP capabilities
  socket.on('joinRoom', async ({ roomId, username }, cb) => {
    const room = await getOrCreateRoom(roomId)
    currentRoomId = roomId
    socket.join(roomId)
    room.peers.set(socket.id, { transports: {}, producers: [], consumers: [], username })
    
    // Send router's codecs to client — needed to init Device
    cb(room.router.rtpCapabilities)

    // Tell new peer about existing producers
    room.peers.forEach((peer, peerId) => {
      if (peerId === socket.id) return
      peer.producers.forEach(p => {
        socket.emit('newProducer', { producerId: p.id, socketId: peerId, username: peer.username })
      })
    })

    // Tell room about new peer
    socket.to(roomId).emit('userJoined', { socketId: socket.id, username })
  })

  // Step 2: Create transport (one for send, one per producer for receive)
  socket.on('createTransport', async ({ forProducerId }, cb) => {
    const room = rooms.get(currentRoomId)
    const { transport, params } = await createWebRtcTransport(room.router)
    room.peers.get(socket.id).transports[transport.id] = { transport, forProducerId }
    cb(params)
  })

  // Step 3: Connect transport (exchange DTLS)
  socket.on('connectTransport', async ({ transportId, dtlsParameters }, cb) => {
    const peer = rooms.get(currentRoomId)?.peers.get(socket.id)
    const entry = peer?.transports[transportId]
    if (!entry) return cb(new Error(`Transport ${transportId} not found`))
    await entry.transport.connect({ dtlsParameters })
    cb()
  })

  // Step 4: Produce (start sending media)
  socket.on('produce', async ({ transportId, kind, rtpParameters }, cb) => {
    const peer = rooms.get(currentRoomId).peers.get(socket.id)
    const entry = peer.transports[transportId]
    if (!entry) return cb({ error: 'Transport not found' })

    const producer = await entry.transport.produce({ kind, rtpParameters })
    peer.producers.push(producer)

    producer.on('transportclose', () => producer.close())
    producer.on('trackended', () => producer.close())

    cb({ id: producer.id })

    // Tell all other peers about this new producer
    socket.to(currentRoomId).emit('newProducer', {
      producerId: producer.id, socketId: socket.id, username: peer.username
    })
  })

  // Step 5: Consume (start receiving someone's media)
  socket.on('consume', async ({ producerId, rtpCapabilities }, cb) => {
    const room = rooms.get(currentRoomId)
    if (!room.router.canConsume({ producerId, rtpCapabilities })) {
      return cb({ error: 'Cannot consume — incompatible RTP capabilities' })
    }

    // Find the transport created specifically for this producer
    const peer = room.peers.get(socket.id)
    const entry = Object.values(peer.transports).find(e => e.forProducerId === producerId)
    if (!entry) return cb({ error: `No transport for producer ${producerId}` })

    const consumer = await entry.transport.consume({
      producerId,
      rtpCapabilities,
      paused: false, // start playing immediately
    })
    peer.consumers.push(consumer)

    consumer.on('transportclose', () => consumer.close())
    consumer.on('producerclose', () => {
      consumer.close()
      socket.emit('consumerClosed', { consumerId: consumer.id })
    })

    cb({ id: consumer.id, kind: consumer.kind, rtpParameters: consumer.rtpParameters, producerId })
  })

  // Media toggle — just a signaling event, no MediaSoup call needed
  socket.on('toggleMedia', ({ kind, enabled }) => {
    socket.to(currentRoomId).emit('mediaToggled', { socketId: socket.id, kind, enabled })
  })

  // Cleanup on disconnect — critical for memory
  socket.on('disconnect', () => {
    const room = rooms.get(currentRoomId)
    if (!room) return
    const peer = room.peers.get(socket.id)
    if (!peer) return

    peer.producers.forEach(p => p.close())
    peer.consumers.forEach(c => c.close())
    Object.values(peer.transports).forEach(({ transport }) => transport.close())
    room.peers.delete(socket.id)

    socket.to(currentRoomId).emit('userDisconnected', { socketId: socket.id })

    // Clean up empty rooms
    if (room.peers.size === 0) {
      room.router.close()
      rooms.delete(currentRoomId)
    }
  })
})
```

---

## Client Setup (React)

### Device Initialization
```js
// After receiving rtpCapabilities from server:
const device = new mediasoupClient.Device()
await device.load({ routerRtpCapabilities: rtpCapabilities })
```

### Send Transport (produce)
```js
const transportInfo = await socketEmit('createTransport', { forProducerId: undefined })
const sendTransport = device.createSendTransport(transportInfo)

sendTransport.on('connect', ({ dtlsParameters }, callback) => {
  socketEmit('connectTransport', { transportId: sendTransport.id, dtlsParameters })
    .then(callback)
})

sendTransport.on('produce', async ({ kind, rtpParameters }, callback) => {
  const { id } = await socketEmit('produce', { transportId: sendTransport.id, kind, rtpParameters })
  callback({ id })
})

// Get user media and produce
const stream = await navigator.mediaDevices.getUserMedia({ audio: true, video: true })
const videoProducer = await sendTransport.produce({ track: stream.getVideoTracks()[0] })
const audioProducer = await sendTransport.produce({ track: stream.getAudioTracks()[0] })
```

### Receive Transport (consume)
```js
async function consumeProducer(producerId, socketId) {
  const transportInfo = await socketEmit('createTransport', { forProducerId: producerId })
  const recvTransport = device.createRecvTransport(transportInfo)

  recvTransport.on('connect', ({ dtlsParameters }, callback) => {
    socketEmit('connectTransport', { transportId: recvTransport.id, dtlsParameters })
      .then(callback)
  })

  const { id, kind, rtpParameters } = await socketEmit('consume', {
    producerId,
    rtpCapabilities: device.rtpCapabilities,
  })

  const consumer = await recvTransport.consume({ id, producerId, kind, rtpParameters })

  // Add track to MediaStream and play it
  const stream = new MediaStream([consumer.track])
  // attach to <video ref> element
  videoRef.current.srcObject = stream
}
```

---

## Production Checklist

- [ ] `announcedIp` set to server's public IP (not 0.0.0.0)
- [ ] UDP ports (rtcMinPort → rtcMaxPort) open in firewall
- [ ] Worker restart on `died` event (or use PM2 cluster)
- [ ] Empty room cleanup (close router, delete from Map)
- [ ] Producer/consumer cleanup on disconnect
- [ ] dtlsstatechange handler closes failed transports
- [ ] Multiple workers for multi-core (one per CPU core)
- [ ] TURN server for clients behind symmetric NAT

## Scale — When to Add Workers
```js
// One worker per CPU core for production
const numWorkers = os.cpus().length
const workers = []

for (let i = 0; i < numWorkers; i++) {
  const worker = await mediasoup.createWorker({ ... })
  workers.push(worker)
}

// Round-robin assign routers to workers
function getNextWorker() {
  const worker = workers[nextWorkerIdx % workers.length]
  nextWorkerIdx++
  return worker
}
```
