Most modern ORMs try to abstract SQL, making it dangerously easy to fall into the N+1 trap. Here is how to fix it in popular frameworks.

### Prisma (Node.js)
```typescript
// ❌ BAD: Causes N+1 if you map over articles and fetch users
const articles = await prisma.article.findMany();

// ✅ GOOD: Use eager loading (include)
const articles = await prisma.article.findMany({
  include: {
    author: true, 
  },
});
```

### Laravel Eloquent (PHP)
```php
// ❌ BAD: Lazy loading
$articles = Article::all();
foreach ($articles as $article) {
    echo $article->author->name; // Queries DB on every loop!
}

// ✅ GOOD: Eager loading (with)
$articles = Article::with('author')->get();
```

### Django ORM (Python)
```python
# ❌ BAD
articles = Article.objects.all()

# ✅ GOOD: select_related for ForeignKeys
articles = Article.objects.select_related('author').all()
```
