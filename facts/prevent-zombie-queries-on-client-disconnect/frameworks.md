## Cancellation Across Frameworks

### Fastify: Built-in Request Cancellation

Fastify has first-class support for request lifecycle signals:

```typescript
import Fastify from 'fastify';

const app = Fastify({ logger: true });

app.get('/heavy-report', async (request, reply) => {
  // Fastify provides request.raw (Node IncomingMessage)
  const ac = new AbortController();

  request.raw.on('close', () => {
    if (!request.raw.complete) {
      ac.abort();
      request.log.info('Client disconnected, query aborted');
    }
  });

  const data = await db.query(expensiveSQL, { signal: ac.signal });
  return data;
});

// Plugin approach: auto-attach AbortController to every request
app.addHook('onRequest', async (request) => {
  const ac = new AbortController();
  request.signal = ac.signal;

  request.raw.on('close', () => {
    if (!request.raw.complete) ac.abort();
  });
});

// Now every route handler can use request.signal
app.get('/report', async (request) => {
  return db.query(sql, { signal: request.signal });
});
```

### Koa: Middleware Pattern

```typescript
import Koa from 'koa';

const app = new Koa();

// Middleware: attach AbortSignal to context
app.use(async (ctx, next) => {
  const ac = new AbortController();
  ctx.state.signal = ac.signal;

  ctx.req.on('close', () => {
    if (!ctx.res.writableFinished) {
      ac.abort();
    }
  });

  await next();
});

// Route handler
app.use(async (ctx) => {
  const data = await db.query(sql, { signal: ctx.state.signal });
  ctx.body = data;
});
```

### NestJS: Interceptor Approach

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, takeUntil, Subject } from 'rxjs';
import { Request } from 'express';

@Injectable()
export class DisconnectInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const req = context.switchToHttp().getRequest<Request>();
    const disconnect$ = new Subject<void>();

    req.on('close', () => {
      disconnect$.next();
      disconnect$.complete();
    });

    return next.handle().pipe(takeUntil(disconnect$));
  }
}

// Usage with AbortController in service
@Injectable()
export class ReportService {
  async getHeavyReport(signal: AbortSignal) {
    return this.db.query(sql, { signal });
  }
}

@Controller('reports')
@UseInterceptors(DisconnectInterceptor)
export class ReportController {
  @Get()
  async get(@Req() req: Request) {
    const ac = new AbortController();
    req.on('close', () => ac.abort());
    return this.reportService.getHeavyReport(ac.signal);
  }
}
```

### Go: Context Cancellation (Gold Standard)

Go's `context.Context` is the original inspiration for AbortSignal patterns:

```go
func heavyReportHandler(w http.ResponseWriter, r *http.Request) {
    // r.Context() is AUTOMATICALLY cancelled when client disconnects
    ctx := r.Context()

    rows, err := db.QueryContext(ctx, "SELECT * FROM huge_table WHERE ...")
    if err != nil {
        if ctx.Err() == context.Canceled {
            log.Println("Client disconnected, query cancelled")
            return // Don't write response
        }
        http.Error(w, err.Error(), 500)
        return
    }
    defer rows.Close()
    // ... process rows
}
```

Go's approach is considered the gold standard because the HTTP server automatically ties the request context to connection lifecycle — no manual wiring needed.

### Streaming & WebSocket Disconnection

For long-lived connections (SSE, WebSockets), the same principle applies:

```typescript
// Server-Sent Events with cancellation
export async function GET(req: Request) {
  const encoder = new TextEncoder();
  let cancelled = false;

  const stream = new ReadableStream({
    async start(controller) {
      // Listen for client disconnect
      req.signal.addEventListener('abort', () => {
        cancelled = true;
        controller.close();
      });

      // Stream data from database cursor
      const cursor = db.query('SELECT * FROM events ORDER BY id').cursor(100);

      for await (const batch of cursor) {
        if (cancelled) break;
        controller.enqueue(
          encoder.encode(`data: ${JSON.stringify(batch)}\n\n`)
        );
      }
    },
  });

  return new Response(stream, {
    headers: { 'Content-Type': 'text/event-stream' },
  });
}

// WebSocket with cleanup
wss.on('connection', (ws) => {
  const ac = new AbortController();

  ws.on('close', () => ac.abort());
  ws.on('error', () => ac.abort());

  ws.on('message', async (msg) => {
    const { query } = JSON.parse(msg.toString());
    try {
      const result = await db.query(query, { signal: ac.signal });
      ws.send(JSON.stringify(result));
    } catch (err) {
      if (err.name === 'AbortError') return; // Client gone
      ws.send(JSON.stringify({ error: err.message }));
    }
  });
});
```

### Framework Comparison

| Framework | Disconnect Detection | Signal Propagation | Effort |
|-----------|--------------------|--------------------|--------|
| **Next.js / Hono** | `req.signal` (built-in) | Automatic | Zero config |
| **Fastify** | `request.raw.on('close')` | Manual AbortController | Low |
| **Express** | `req.on('close')` | Manual AbortController | Low |
| **Koa** | `ctx.req.on('close')` | Middleware pattern | Medium |
| **NestJS** | Interceptor + `req.on('close')` | RxJS `takeUntil` or AbortController | Medium |
| **Go** | `r.Context()` (built-in) | Automatic | Zero config |
