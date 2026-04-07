# SSR vs SSG vs ISR vs CSR Trade-offs

## Comparison Table

| Metric | SSR | SSG | ISR | CSR |
|--------|-----|-----|-----|-----|
| Build Time | Fast | Slow (data fetching) | Medium | Fast |
| Page Load (FCP) | Slow | Fast | Fast | Slow |
| Time to Content | Medium | Fastest | Fast | Slowest |
| Data Freshness | Always fresh | Stale | Mostly fresh | Always fresh |
| SEO | Excellent | Excellent | Excellent | Poor |
| Server Cost | High | None (static) | Low | None |
| Cache Friendly | Medium | Excellent | Excellent | No |
| Scalability | Limited | Unlimited | High | Unlimited |
| Complexity | Medium | Low | High | Low |

## Visual Timeline Comparison

```
User Request
     │
     ├─ SSR:  Server generates HTML → Send 200-500ms → Browser render
     │        Always slow
     │
     ├─ SSG:  Pre-generated HTML → Send 50ms → Browser render
     │        Fastest, but stale
     │
     ├─ ISR:  Pre-generated HTML → Send 50ms → Background revalidate
     │        Best of both worlds
     │
     └─ CSR:  Send HTML shell → Fetch data → Render in browser
              Slowest initial load
```

## Real-World Examples

### Blog Platform

**Homepage** (Latest 10 posts):
- Strategy: ISR with 1-hour revalidation
- Reason: Mostly static, occasional updates

**Individual Blog Post**:
- Strategy: SSG with on-demand revalidation
- Reason: Post rarely changes after publishing
- Fallback: ISR in case of new posts

**Comments Section**:
- Strategy: CSR with polling
- Reason: Real-time user comments

### E-commerce Site

**Product Listing**:
- Strategy: ISR with 15-minute revalidation
- Reason: Stock/price changes frequently

**Product Details**:
- Strategy: SSG + ISR
- Reason: Stable product info, revalidate on inventory change

**Shopping Cart**:
- Strategy: CSR
- Reason: User-specific, needs real-time updates

**Admin Dashboard**:
- Strategy: SSR
- Reason: Real-time analytics, protected routes

## Cost Analysis

```
Monthly traffic: 1M requests

SSG + CDN:
- Static hosting: $10/month
- CDN: $20/month
- Total: $30/month

SSR + Server:
- Server instance: $50-200/month
- Bandwidth: $50/month
- Total: $100-250/month

ISR (hybrid):
- Static: $10/month
- Server (revalidation only): $30/month
- CDN: $15/month
- Total: $55/month
```

## Data Freshness Scenarios

```
News Site:
- Homepage (SSG, 1hr revalidate): Updated hourly
- Article (SSG, on-demand): Fresh after publish
- Comments (CSR): Real-time

Product Reviews:
- Product page (SSG, on-demand): Updates when review posted
- Review list (CSR): Real-time count

User Profile:
- Public profile (SSR): Always fresh
- Private dashboard (CSR): Secure, real-time
```

## Stale Content Risks

```
Revalidate: 3600 (1 hour)

Request at 10:05 (5 min after build):
├─ User 1: Gets fresh page (built at 10:00)
├─ User 2: Gets fresh page (built at 10:00)
└─ Background: Triggers revalidation

Request at 10:59 (59 min after build):
└─ User: Gets stale page, background revalidation starts

Request at 11:00 (revalidation complete):
└─ User: Gets fresh page

Maximum staleness: ~revalidate time
```

## ISR Fallback Strategies

```javascript
// Best practices for handling new content

// Strategy 1: Pregenerate on update event
export async function getStaticProps() {
  const data = await fetchData();
  return {
    props: { data },
    revalidate: 3600,
  };
}

// Webhook from CMS triggers revalidation
fetch('https://mysite.com/api/revalidate?secret=X&path=/products/new-item');

// Strategy 2: Longer revalidate for less popular items
export async function getStaticProps({ params }) {
  const item = await fetchItem(params.id);
  const revalidate = item.popularity > 100 ? 300 : 3600;
  return { props: { item }, revalidate };
}

// Strategy 3: Stale-while-revalidate pattern
export async function getStaticProps() {
  const data = await fetchData();
  return {
    props: { data },
    revalidate: 10, // Quick revalidation window
  };
}
```

## Migration Path

For existing SSR apps:

1. **Identify static content** → Convert to SSG
2. **Identify semi-static content** → Convert to ISR
3. **Keep dynamic content** → Keep as SSR/CSR
4. **Result**: Usually 70-80% static, 20-30% dynamic

This dramatically reduces server load and improves performance.
