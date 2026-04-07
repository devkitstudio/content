# Next.js Rendering Strategy Decision Tree

## Decision Flow

```
Does the page need real-time data?
├─ YES → Does it need SEO?
│   ├─ YES → SSR (getServerSideProps)
│   └─ NO → Client-side (useEffect + API)
│
├─ NO → Does it change frequently?
│   ├─ YES (e.g., 10x/day) → ISR (revalidate: 3600)
│   └─ NO (e.g., once/week) → SSG (getStaticProps)
│
└─ NOT SURE → Start with ISR, adjust based on metrics
```

## Static Site Generation (SSG)

Use for: Blog posts, documentation, landing pages that rarely change

```javascript
// pages/blog/[slug].js
export async function getStaticPaths() {
  const posts = await getAllBlogPosts(); // Fetch at build time
  return {
    paths: posts.map(p => ({ params: { slug: p.slug } })),
    fallback: 'blocking', // New posts → build on demand
  };
}

export async function getStaticProps({ params }) {
  const post = await getBlogPost(params.slug);

  return {
    props: { post },
    revalidate: 3600, // Revalidate every hour
  };
}

export default function BlogPost({ post }) {
  return <article>{post.content}</article>;
}
```

**Characteristics:**
- Built at deploy time
- Cached at CDN globally
- Fastest page load (just HTML + CSS)
- Good for SEO
- Old data until revalidation

## Server-Side Rendering (SSR)

Use for: User-specific data, real-time dashboards, pages that need fresh data at every request

```javascript
// pages/dashboard.js
export async function getServerSideProps(context) {
  // Runs on every request
  const user = await getUser(context.req.headers.cookie);
  const data = await fetchDashboardData(user.id);

  if (!user) {
    return {
      redirect: {
        destination: '/login',
        permanent: false,
      },
    };
  }

  return {
    props: { user, data },
  };
}

export default function Dashboard({ user, data }) {
  return <div>Welcome {user.name}! {data}</div>;
}
```

**Characteristics:**
- Renders on every request
- Always fresh data
- Slower than SSG (server processing time)
- Good for SEO
- Can redirect/authenticate

## Incremental Static Regeneration (ISR)

Use for: E-commerce, news, content that updates moderately

```javascript
// pages/products/[id].js
export async function getStaticPaths() {
  // Pregenerate popular products
  const popularProducts = await getTopProducts(100);

  return {
    paths: popularProducts.map(p => ({ params: { id: p.id } })),
    fallback: 'blocking', // New products: generate on first request
  };
}

export async function getStaticProps({ params }) {
  const product = await getProduct(params.id);

  return {
    props: { product },
    revalidate: 60, // Revalidate every 60 seconds
  };
}

export default function Product({ product }) {
  return (
    <div>
      <h1>{product.name}</h1>
      <p>${product.price}</p>
    </div>
  );
}
```

**Characteristics:**
- Pregenerated at build time
- Revalidated in background (stale-while-revalidate)
- Fast page loads + relatively fresh data
- Great for SEO
- Graceful fallback for new content

## Client-Side Rendering

Use for: User dashboards, real-time data, authenticated content that doesn't need SEO

```javascript
// pages/notifications.js
import { useEffect, useState } from 'react';

export default function Notifications() {
  const [notifications, setNotifications] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchNotifications = async () => {
      const res = await fetch('/api/notifications');
      const data = await res.json();
      setNotifications(data);
      setLoading(false);
    };

    fetchNotifications();
    // Optional: poll for updates
    const interval = setInterval(fetchNotifications, 5000);
    return () => clearInterval(interval);
  }, []);

  if (loading) return <div>Loading...</div>;
  return <div>{notifications?.map(n => <Notification key={n.id} {...n} />)}</div>;
}
```

**Characteristics:**
- Load shell → fetch data in browser
- No SEO benefits
- No stale data
- Slower initial load
- Good for authenticated/real-time content

## Hybrid Approach (Recommended)

```javascript
// pages/index.js
export async function getStaticProps() {
  // Static homepage content
  const posts = await getLatestPosts(5);

  return {
    props: { posts },
    revalidate: 3600, // Hourly revalidation
  };
}

export default function Home({ posts }) {
  const [recommendations, setRecommendations] = useState(null);

  // Client-side fetch for personalized content
  useEffect(() => {
    fetch('/api/recommendations').then(r => r.json()).then(setRecommendations);
  }, []);

  return (
    <div>
      <section>
        {/* SSG content: fast, cacheable, good SEO */}
        <h1>Latest Posts</h1>
        {posts.map(p => <PostCard key={p.id} post={p} />)}
      </section>

      <section>
        {/* CSR content: personal recommendations */}
        <h2>Your Recommendations</h2>
        {recommendations?.map(r => <RecommendationCard key={r.id} {...r} />)}
      </section>
    </div>
  );
}
```

## Performance Tips

```javascript
// Use revalidate strategically
export async function getStaticProps() {
  const data = await fetchData();

  // Revalidate based on data type
  const revalidateTime = {
    'news': 300, // 5 minutes
    'products': 3600, // 1 hour
    'docs': 86400, // 1 day
  };

  return {
    props: { data },
    revalidate: revalidateTime[data.type],
  };
}
```

## On-Demand Revalidation

```javascript
// pages/api/revalidate.js
export default async function handler(req, res) {
  // Verify secret
  if (req.query.secret !== process.env.REVALIDATE_SECRET) {
    return res.status(401).json({ message: 'Invalid token' });
  }

  try {
    // Revalidate specific path
    await res.revalidate('/blog/my-post');
    return res.json({ revalidated: true });
  } catch (err) {
    return res.status(500).send('Error revalidating');
  }
}
```
