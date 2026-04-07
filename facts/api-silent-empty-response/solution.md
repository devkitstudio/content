# 5 Common Causes of Silent Empty Responses

## 1. Response Streaming Not Flushed
Streaming responses that never call `end()` or `write()` leave the body empty.

```javascript
// WRONG - never sends data
app.get('/api/users', (req, res) => {
  User.find({}, (err, users) => {
    // Missing res.json(users)
  });
});

// RIGHT
app.get('/api/users', (req, res) => {
  User.find({}, (err, users) => {
    res.json(users); // Explicitly send response
  });
});
```

## 2. Middleware Consuming Body Without Re-piping
Body parsing middleware that reads the stream and doesn't restore it.

```javascript
// WRONG - body consumed, not available to route
app.use((req, res, next) => {
  let data = '';
  req.on('data', chunk => data += chunk);
  req.on('end', () => {
    // data is here but req.body is never set
    next();
  });
});

// RIGHT - use standard body parser
const express = require('express');
app.use(express.json());
app.post('/api/data', (req, res) => {
  console.log(req.body); // Works
  res.json({ received: true });
});
```

## 3. Async Handler Never Awaits Database Query
Handler resolves before the query returns, sending empty response.

```javascript
// WRONG - race condition
app.get('/api/products', (req, res) => {
  let products = [];
  Product.find().then(data => {
    products = data;
  });
  res.json(products); // Sends before promise resolves
});

// RIGHT - await the query
app.get('/api/products', async (req, res) => {
  const products = await Product.find();
  res.json(products);
});
```

## 4. Error in Serializer Silent Failures
JSON serializer errors don't throw, just return undefined.

```javascript
// WRONG - BigInt cannot serialize
app.get('/api/transaction', async (req, res) => {
  const tx = await Transaction.findById(id);
  res.json(tx); // Error: Do not know how to serialize BigInt
});

// RIGHT - convert before sending
app.get('/api/transaction', async (req, res) => {
  const tx = await Transaction.findById(id);
  res.json({
    ...tx.toObject(),
    amount: tx.amount.toString() // Convert BigInt to string
  });
});
```

## 5. Response Already Sent Check Missing
Multiple `res.json()` or `res.send()` calls; only first takes effect, others silently fail.

```javascript
// WRONG - may call res.json twice
app.get('/api/user/:id', async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) {
    res.status(404).json({ error: 'Not found' });
  }
  // Falls through and sends empty json
  res.json(user);
});

// RIGHT - use early returns
app.get('/api/user/:id', async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) {
    return res.status(404).json({ error: 'Not found' });
  }
  return res.json(user);
});
```
