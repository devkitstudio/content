# Systematic Debugging for Empty Responses

## Step 1: Add Console Logging at Route Handler
Verify the route is being hit.

```javascript
app.get('/api/data', (req, res) => {
  console.log('Route handler called'); // Should appear in logs
  const data = getData();
  console.log('Data retrieved:', data); // Check if data exists
  res.json(data);
  console.log('Response sent'); // May not appear if earlier return
});
```

## Step 2: Check Response Headers
Use network tab to inspect response status and content-type.

```bash
curl -i http://localhost:3000/api/data
# Check:
# - HTTP status (should be 200)
# - Content-Type: application/json
# - Content-Length: 0 (indicates empty body)
```

## Step 3: Add Error Handlers
Catch and log errors that might be silently failing.

```javascript
app.get('/api/data', async (req, res, next) => {
  try {
    const data = await fetchData();
    res.json(data);
  } catch (err) {
    console.error('Error in /api/data:', err); // Log the actual error
    next(err); // Pass to error handler
  }
});

// Global error handler
app.use((err, req, res, next) => {
  console.error('Unhandled error:', err);
  res.status(500).json({ error: err.message });
});
```

## Step 4: Verify Database Connection
Test that the database is actually returning data.

```javascript
app.get('/api/debug', async (req, res) => {
  try {
    const count = await User.countDocuments();
    const sample = await User.findOne();
    res.json({
      dbConnected: true,
      userCount: count,
      sample: sample
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
```

## Step 5: Check Middleware Order
Middleware that sends response before route handler executes.

```javascript
// WRONG - response sent before routes
app.use((req, res, next) => {
  res.json({ error: 'Middleware response' });
  next(); // Too late
});
app.get('/api/data', (req, res) => { /* never reached */ });

// RIGHT - middleware after routes or use next() to continue
app.get('/api/data', (req, res) => { /* executed */ });
app.use((req, res) => {
  res.status(404).json({ error: 'Not found' });
});
```

## Step 6: Monitor Network Requests
Use browser DevTools or network analyzer.

```
1. Open DevTools → Network tab
2. Make request to endpoint
3. Check:
   - Status: 200 (or expected status)
   - Response tab: should show JSON data
   - Size: should be > 0 bytes
4. Check Preview tab for parse errors
```

## Step 7: Add Request/Response Logging Middleware
Log all requests and responses for inspection.

```javascript
app.use((req, res, next) => {
  const originalJson = res.json;
  res.json = function(data) {
    console.log('Sending JSON response:', JSON.stringify(data, null, 2));
    return originalJson.call(this, data);
  };
  next();
});
```
