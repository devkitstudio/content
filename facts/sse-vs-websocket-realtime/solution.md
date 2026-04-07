# SSE vs WebSocket Comparison

## Server-Sent Events (SSE): One-Way Push

### What is SSE?
- HTTP-based, one-way communication (server → client)
- Built on HTTP EventSource API
- Uses long-polling under the hood
- Automatic reconnection
- Works over standard HTTPS

```javascript
// Server-side: Node.js Express
app.get('/api/price-updates', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  // Send initial connection message
  res.write('data: {"status":"connected"}\n\n');

  // Store this client's response stream
  const clientId = req.user.id;
  clients.set(clientId, res);

  // Send ping every 30 seconds to keep connection alive
  const pingInterval = setInterval(() => {
    res.write(': ping\n\n');
  }, 30000);

  // Handle client disconnect
  req.on('close', () => {
    clearInterval(pingInterval);
    clients.delete(clientId);
  });
});

// Broadcast price updates to all connected clients
function broadcastPriceUpdate(priceData) {
  clients.forEach((clientRes) => {
    clientRes.write(`data: ${JSON.stringify(priceData)}\n\n`);
  });
}

// Update price
app.post('/api/prices/:id', async (req, res) => {
  const price = req.body.price;
  await Price.update(req.params.id, price);

  // Broadcast to all SSE clients
  broadcastPriceUpdate({ id: req.params.id, price });

  res.json({ success: true });
});
```

### Client-side: SSE
```javascript
// Browser JavaScript
const eventSource = new EventSource('/api/price-updates');

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  updateUI(data);
};

eventSource.onerror = () => {
  console.error('SSE connection lost');
  // Automatic reconnection happens here
};

// Close connection when done
// eventSource.close();
```

### SSE Pros
- Simple to implement
- Works with existing HTTP infrastructure
- Automatic reconnection
- Lower latency than polling
- Firewall-friendly (uses HTTP)

### SSE Cons
- One-way only (server → client)
- No binary data (text-only)
- Limited concurrency (browser limits)
- Still uses polling (not true push)

---

## WebSocket: Two-Way Communication

### What is WebSocket?
- TCP-based, bidirectional
- Persistent connection
- Low latency (<100ms)
- Works over WSS (WebSocket Secure)
- True event-driven

```javascript
// Server-side: WebSocket with Socket.io
const io = require('socket.io')(3000);

io.on('connection', (socket) => {
  console.log(`Client ${socket.id} connected`);

  // Join client to 'prices' room
  socket.join('prices');

  // Handle incoming messages from client
  socket.on('subscribe', (data) => {
    socket.join(`price-${data.productId}`);
  });

  socket.on('unsubscribe', (data) => {
    socket.leave(`price-${data.productId}`);
  });

  // Handle disconnect
  socket.on('disconnect', () => {
    console.log(`Client ${socket.id} disconnected`);
  });
});

// Broadcast price updates
function broadcastPriceUpdate(priceData) {
  io.to('prices').emit('price-update', priceData);
}

// or broadcast to specific product price room
function updateProductPrice(productId, price) {
  io.to(`price-${productId}`).emit('price-update', {
    productId,
    price
  });
}

// API endpoint updates price and broadcasts
app.post('/api/prices/:id', async (req, res) => {
  const price = req.body.price;
  await Price.update(req.params.id, price);

  updateProductPrice(req.params.id, price);

  res.json({ success: true });
});
```

### Client-side: WebSocket
```javascript
// Browser JavaScript with Socket.io
const socket = io();

socket.on('connect', () => {
  console.log('Connected to server');
  socket.emit('subscribe', { productId: 123 });
});

socket.on('price-update', (data) => {
  console.log(`Price updated: $${data.price}`);
  updateUI(data);
});

socket.on('disconnect', () => {
  console.log('Disconnected from server');
});

// Send message to server
function updateUserPreferences(prefs) {
  socket.emit('preferences-changed', prefs);
}
```

### WebSocket Pros
- Bidirectional communication
- True push (no polling)
- Low latency (<100ms)
- Binary support
- Scalable with proper infrastructure
- Real-time gaming/collaboration

### WebSocket Cons
- Requires WebSocket server
- More complex implementation
- Requires load balancer support
- More memory per connection
- Firewall issues in some networks

---

## Direct Comparison

| Feature | SSE | WebSocket |
|---------|-----|-----------|
| **Direction** | One-way (server→client) | Two-way (bidirectional) |
| **Protocol** | HTTP | TCP (upgraded from HTTP) |
| **Latency** | 100-500ms | <100ms |
| **Complexity** | Simple | Moderate |
| **Binary Support** | No (text only) | Yes |
| **Browser Support** | Modern browsers | Modern browsers |
| **Fallback** | Long-polling | Flash sockets |
| **Per-connection Memory** | Low | Medium-High |
| **Scalability** | Good (stateless HTTP) | Excellent with clustering |
| **Firewall Issues** | Rare | Possible |

---

## Architecture Patterns

### SSE with Redis Broadcasting
```javascript
// Multiple servers can broadcast through Redis
const redis = require('redis');
const pubsub = redis.createClient();
const subscriber = redis.createClient();

// Publish price updates
function broadcastPriceUpdate(priceData) {
  pubsub.publish('price-updates', JSON.stringify(priceData));
}

// Each server subscribes to updates
subscriber.subscribe('price-updates', (err, count) => {
  console.log(`Subscribed to ${count} channels`);
});

subscriber.on('message', (channel, message) => {
  const priceData = JSON.parse(message);
  // Send to all connected SSE clients on THIS server
  clients.forEach((clientRes) => {
    clientRes.write(`data: ${message}\n\n`);
  });
});
```

### WebSocket with Socket.io Clustering
```javascript
// Socket.io handles clustering automatically with Redis adapter
const io = require('socket.io')(3000);
const redis = require('socket.io-redis');

io.adapter(redis({ host: 'localhost', port: 6379 }));

// Broadcast works across all servers
function broadcastPrice(priceData) {
  io.to('prices').emit('price-update', priceData);
}

// Even if client is connected to Server B,
// and price update comes from Server A,
// the message is delivered through Redis to all servers
```

---

## When to Use Which

### Use SSE When:
- Server sends updates to clients only
- Don't need binary data
- Simplicity is priority
- Building notifications feed
- Events are infrequent (< 10/sec)
- HTTP infrastructure is already in place

```javascript
// Example: News feed updates
app.get('/api/notifications', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  const userId = req.user.id;
  sseClients.set(userId, res);

  req.on('close', () => sseClients.delete(userId));
});

// Send notification
function sendNotification(userId, notification) {
  const client = sseClients.get(userId);
  if (client) {
    client.write(`data: ${JSON.stringify(notification)}\n\n`);
  }
}
```

### Use WebSocket When:
- Need bidirectional communication
- Low latency is critical
- Building interactive applications
- Need binary data
- High event frequency (> 10/sec)
- Building games, collaboration tools

```javascript
// Example: Collaborative editor
socket.on('document-changed', (change) => {
  // Client sent update to server
  broadcastToCollaborators(change);
});

// Example: Real-time gaming
socket.on('player-moved', (position) => {
  // Client sent position update
  updateGameState(position);
});
```
