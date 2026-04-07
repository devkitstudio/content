# Complete Implementation: SSE vs WebSocket

## Full SSE Implementation (Express + EventSource)

```javascript
const express = require('express');
const app = express();

// Store active SSE clients
const sseClients = new Map();

// SSE endpoint
app.get('/api/price-stream', (req, res) => {
  // Set proper headers for EventSource
  res.writeHead(200, {
    'Content-Type': 'text/event-stream; charset=utf-8',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'X-Accel-Buffering': 'no', // Disable nginx buffering
    'Access-Control-Allow-Origin': '*'
  });

  const clientId = `${req.user?.id}-${Date.now()}`;
  const client = {
    id: clientId,
    res,
    userId: req.user?.id,
    connected: new Date()
  };

  sseClients.set(clientId, client);
  console.log(`SSE client connected: ${clientId}. Total: ${sseClients.size}`);

  // Send initial message
  res.write('data: {"status":"connected","timestamp":"' + new Date().toISOString() + '"}\n\n');

  // Send heartbeat every 30 seconds to prevent timeout
  const heartbeatInterval = setInterval(() => {
    res.write(': heartbeat\n\n');
  }, 30000);

  // Handle client disconnect
  req.on('close', () => {
    clearInterval(heartbeatInterval);
    sseClients.delete(clientId);
    console.log(`SSE client disconnected: ${clientId}. Total: ${sseClients.size}`);
  });

  req.on('error', (err) => {
    console.error(`SSE error for ${clientId}:`, err);
    clearInterval(heartbeatInterval);
    sseClients.delete(clientId);
  });
});

// Broadcast price update to all SSE clients
function broadcastPriceUpdate(priceData) {
  const message = `data: ${JSON.stringify(priceData)}\n\n`;
  let successCount = 0;
  let errorCount = 0;

  sseClients.forEach((client, clientId) => {
    try {
      client.res.write(message);
      successCount++;
    } catch (err) {
      console.error(`Failed to send to ${clientId}:`, err);
      sseClients.delete(clientId);
      errorCount++;
    }
  });

  console.log(`Broadcast: ${successCount} success, ${errorCount} failed`);
}

// API endpoint
app.post('/api/prices/:id', async (req, res) => {
  try {
    const price = req.body.price;
    const id = req.params.id;

    // Update database
    const updatedPrice = await Price.findByIdAndUpdate(id, { price });

    // Broadcast to all SSE clients
    broadcastPriceUpdate({
      id,
      price,
      timestamp: new Date().toISOString(),
      updatedAt: updatedPrice.updatedAt
    });

    res.json({ success: true, price: updatedPrice });
  } catch (err) {
    console.error('Price update error:', err);
    res.status(500).json({ error: err.message });
  }
});

// Metrics endpoint
app.get('/api/sse-metrics', (req, res) => {
  res.json({
    connectedClients: sseClients.size,
    clients: Array.from(sseClients.values()).map(c => ({
      id: c.id,
      userId: c.userId,
      connectedFor: Date.now() - c.connected.getTime()
    }))
  });
});

app.listen(3000);
```

## Full WebSocket Implementation (Socket.io)

```javascript
const express = require('express');
const http = require('http');
const socketIo = require('socket.io');
const redis = require('socket.io-redis');

const app = express();
const server = http.createServer(app);
const io = socketIo(server, {
  cors: {
    origin: '*',
    methods: ['GET', 'POST']
  },
  transports: ['websocket', 'polling']
});

// For clustering: use Redis adapter
const redisAdapter = require('socket.io-redis');
io.adapter(redisAdapter({
  host: process.env.REDIS_HOST || 'localhost',
  port: process.env.REDIS_PORT || 6379
}));

// Track subscriptions
const subscriptions = new Map(); // clientId -> Set<productId>

io.on('connection', (socket) => {
  console.log(`Client connected: ${socket.id}`);

  // Initialize subscription tracking for this socket
  subscriptions.set(socket.id, new Set());

  // Client wants to subscribe to price updates
  socket.on('subscribe-price', (data) => {
    const { productId } = data;
    socket.join(`price-${productId}`);

    const subs = subscriptions.get(socket.id);
    subs.add(productId);

    console.log(`Client ${socket.id} subscribed to price-${productId}`);

    // Send current price
    socket.emit('current-price', {
      productId,
      price: getCurrentPrice(productId),
      timestamp: new Date().toISOString()
    });
  });

  // Client wants to unsubscribe
  socket.on('unsubscribe-price', (data) => {
    const { productId } = data;
    socket.leave(`price-${productId}`);

    const subs = subscriptions.get(socket.id);
    subs.delete(productId);

    console.log(`Client ${socket.id} unsubscribed from price-${productId}`);
  });

  // Handle disconnect
  socket.on('disconnect', () => {
    subscriptions.delete(socket.id);
    console.log(`Client disconnected: ${socket.id}`);
  });

  socket.on('error', (err) => {
    console.error(`Socket error for ${socket.id}:`, err);
  });
});

// Broadcast price update
function broadcastPriceUpdate(productId, priceData) {
  io.to(`price-${productId}`).emit('price-update', {
    productId,
    price: priceData.price,
    timestamp: new Date().toISOString()
  });
}

// API endpoint
app.post('/api/prices/:id', async (req, res) => {
  try {
    const price = req.body.price;
    const id = req.params.id;

    // Update database
    const updatedPrice = await Price.findByIdAndUpdate(id, { price });

    // Broadcast to all WebSocket clients subscribed to this product
    broadcastPriceUpdate(id, { price });

    res.json({ success: true, price: updatedPrice });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Metrics
app.get('/api/ws-metrics', (req, res) => {
  const clients = Array.from(io.sockets.sockets.values()).map(socket => ({
    id: socket.id,
    subscriptions: Array.from(subscriptions.get(socket.id) || [])
  }));

  res.json({
    connectedClients: io.sockets.sockets.size,
    clients
  });
});

server.listen(3000);
```

## Client-side: SSE

```javascript
// HTML
<!DOCTYPE html>
<html>
<head>
  <title>SSE Price Updates</title>
</head>
<body>
  <h1>Price Updates (SSE)</h1>
  <div id="prices"></div>

  <script>
    // Create EventSource
    const eventSource = new EventSource('/api/price-stream');
    const pricesDiv = document.getElementById('prices');

    // Handle incoming messages
    eventSource.addEventListener('message', (event) => {
      const data = JSON.parse(event.data);

      if (data.status === 'connected') {
        pricesDiv.innerHTML += `<p>Connected at ${data.timestamp}</p>`;
      } else if (data.id) {
        const p = document.createElement('p');
        p.textContent = `Product ${data.id}: $${data.price.toFixed(2)}`;
        pricesDiv.appendChild(p);
      }
    });

    // Handle errors
    eventSource.onerror = (error) => {
      console.error('SSE error:', error);
      if (eventSource.readyState === EventSource.CLOSED) {
        pricesDiv.innerHTML += '<p>Connection closed. Attempting reconnect...</p>';
      }
    };

    // Cleanup on page unload
    window.addEventListener('beforeunload', () => {
      eventSource.close();
    });
  </script>
</body>
</html>
```

## Client-side: WebSocket

```javascript
// HTML
<!DOCTYPE html>
<html>
<head>
  <title>WebSocket Price Updates</title>
  <script src="https://cdn.socket.io/4.5.4/socket.io.min.js"></script>
</head>
<body>
  <h1>Price Updates (WebSocket)</h1>

  <div>
    <input type="number" id="productId" placeholder="Product ID">
    <button onclick="subscribeToPrice()">Subscribe</button>
    <button onclick="unsubscribeFromPrice()">Unsubscribe</button>
  </div>

  <div id="prices"></div>

  <script>
    const socket = io();
    let currentProductId = null;
    const pricesDiv = document.getElementById('prices');

    socket.on('connect', () => {
      pricesDiv.innerHTML += '<p>Connected to server</p>';
    });

    socket.on('current-price', (data) => {
      const p = document.createElement('p');
      p.textContent = `Current: Product ${data.productId} - $${data.price.toFixed(2)}`;
      pricesDiv.appendChild(p);
    });

    socket.on('price-update', (data) => {
      const p = document.createElement('p');
      p.textContent = `Updated: Product ${data.productId} - $${data.price.toFixed(2)}`;
      pricesDiv.appendChild(p);
    });

    socket.on('disconnect', () => {
      pricesDiv.innerHTML += '<p>Disconnected from server</p>';
    });

    function subscribeToPrice() {
      currentProductId = parseInt(document.getElementById('productId').value);
      socket.emit('subscribe-price', { productId: currentProductId });
    }

    function unsubscribeFromPrice() {
      if (currentProductId) {
        socket.emit('unsubscribe-price', { productId: currentProductId });
      }
    }
  </script>
</body>
</html>
```

## Performance Testing

```javascript
// Load test SSE vs WebSocket
const axios = require('axios');
const io = require('socket.io-client');

async function loadTestSSE(numClients) {
  const start = Date.now();
  const clients = [];

  for (let i = 0; i < numClients; i++) {
    const es = new EventSource('/api/price-stream');
    clients.push(es);
  }

  // Wait for all to connect
  await new Promise(resolve => setTimeout(resolve, 2000));

  // Broadcast message
  await axios.post('/api/prices/1', { price: 99.99 });

  // Measure time
  const elapsed = Date.now() - start;
  console.log(`SSE: ${numClients} clients, ${elapsed}ms`);

  clients.forEach(es => es.close());
}

async function loadTestWebSocket(numClients) {
  const start = Date.now();
  const sockets = [];

  for (let i = 0; i < numClients; i++) {
    const socket = io('http://localhost:3000');
    socket.on('connect', () => {
      socket.emit('subscribe-price', { productId: 1 });
    });
    sockets.push(socket);
  }

  // Wait for all to connect
  await new Promise(resolve => setTimeout(resolve, 2000));

  // Broadcast message
  await axios.post('/api/prices/1', { price: 99.99 });

  // Measure time
  const elapsed = Date.now() - start;
  console.log(`WebSocket: ${numClients} clients, ${elapsed}ms`);

  sockets.forEach(socket => socket.close());
}
```
