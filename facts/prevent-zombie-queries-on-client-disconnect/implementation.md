Here is how to properly link a client's disconnection to a Database query in modern Node.js.

### 1. Using Standard Fetch APIs (Next.js / Hono)
Modern edge frameworks expose the standard Web `Request` object. This object has a built-in `.signal` property that automatically aborts when the client disconnects!

```typescript
export async function GET(req: Request) {
  try {
    // 1. Just fetch external data using the built-in signal!
    const res = await fetch('http://heavy.internal.api/data', { 
      signal: req.signal // Cascade the abort!
    });
    
    // 2. Pass it to supported Database Drivers
    const data = await myDbDriver.query("SELECT * FROM massive_log", {
      abortSignal: req.signal
    });

    return Response.json(data);
  } catch (err) {
    if (err.name === 'AbortError') {
      console.log('Client disconnected. Query aborted safely!');
    }
  }
}
```

### 2. Using Express.js `req.on('close')`
If you use classic Express, you must manually intercept the socket closure and trigger an `AbortController`.

```javascript
app.get('/heavy-report', async (req, res) => {
  const ac = new AbortController();
  
  // Listen for absolute client disconnect
  req.on('close', () => {
    if (!res.headersSent) {
      ac.abort(); // Fire the kill switch!
    }
  });

  try {
    // Pass the kill switch to your heavy operation
    const data = await executeHeavyDatabaseQuery({ signal: ac.signal });
    res.json(data);
  } catch (err) {
    if (err.name === 'AbortError') return; // Handled
    res.status(500).send(err.message);
  }
});
```

> [!WARNING]  
> Not all ORMs natively support `AbortSignal` for killing active queries yet (e.g. Prisma still has open RFPs for native cancellation). If your ORM doesn't support it, you may have to listen to the `abort` event and manually issue a `pg_cancel_backend(pid)` command to your Postgres database!
