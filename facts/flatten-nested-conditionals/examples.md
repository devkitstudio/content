## Real-World Refactorings

### Example 1: User Permission Check

**Before:**
```typescript
if (user != null) {
  if (user.isAdmin || user.isModerator) {
    if (user.isActive) {
      if (!user.isSuspended) {
        if (user.lastLogin > 30.days.ago) {
          return true;
        }
      }
    }
  }
}
return false;
```

**After:**
```typescript
if (!user) return false;
if (!user.isAdmin && !user.isModerator) return false;
if (!user.isActive) return false;
if (user.isSuspended) return false;
if (user.lastLogin <= 30.days.ago) return false;
return true;
```

### Example 2: File Processing

**Before:**
```typescript
function processFile(file: File) {
  if (file) {
    if (file.size > 0) {
      if (file.type.includes('image')) {
        if (file.size < 5 * 1024 * 1024) {
          resizeImage(file);
          uploadImage(file);
        }
      }
    }
  }
}
```

**After:**
```typescript
function processFile(file: File) {
  if (!file) return;
  if (file.size === 0) return;
  if (!file.type.includes('image')) return;
  if (file.size > 5 * 1024 * 1024) return;

  resizeImage(file);
  uploadImage(file);
}
```

### Example 3: Payment Processing

**Before:**
```typescript
function processPayment(order: Order) {
  if (order != null) {
    if (order.customer != null) {
      if (order.customer.paymentMethods != null && order.customer.paymentMethods.length > 0) {
        const method = order.customer.paymentMethods[0];
        if (method.isExpired === false) {
          if (order.amount > 0) {
            if (order.currency === 'USD') {
              return chargeCard(method, order.amount);
            }
          }
        }
      }
    }
  }
  return null;
}
```

**After:**
```typescript
function processPayment(order: Order): PaymentResult | null {
  // Validation
  if (!order) return null;
  if (!order.customer) return null;
  if (!order.customer.paymentMethods?.length) return null;
  if (order.amount <= 0) return null;
  if (order.currency !== 'USD') return null;

  const method = order.customer.paymentMethods[0];
  if (method.isExpired) return null;

  // Processing
  return chargeCard(method, order.amount);
}
```

### Example 4: Complex Business Logic

**Before (15 levels):**
```typescript
if (req.body) {
  if (req.user) {
    if (req.user.isAuthenticated) {
      if (req.user.permissions.includes('edit_posts')) {
        const postId = req.params.id;
        if (postId) {
          const post = db.findPostById(postId);
          if (post) {
            if (post.authorId === req.user.id) {
              if (req.body.title && req.body.content) {
                if (req.body.title.length >= 5) {
                  if (req.body.content.length >= 20) {
                    post.update(req.body);
                    res.json(post);
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

**After (3 levels):**
```typescript
// Validation middleware
const validateEditRequest = (req: Request): ValidationError | null => {
  if (!req.body) return 'No request body';
  if (!req.user?.isAuthenticated) return 'Not authenticated';
  if (!req.user.permissions.includes('edit_posts')) return 'No permission';
  if (!req.params.id) return 'Post ID required';
  if (!req.body.title || req.body.title.length < 5) return 'Title too short';
  if (!req.body.content || req.body.content.length < 20) return 'Content too short';
  return null;
};

// Route handler
app.patch('/posts/:id', (req: Request, res: Response) => {
  const error = validateEditRequest(req);
  if (error) return res.status(400).json({ error });

  const post = db.findPostById(req.params.id);
  if (!post) return res.status(404).json({ error: 'Post not found' });
  if (post.authorId !== req.user.id) return res.status(403).json({ error: 'Forbidden' });

  post.update(req.body);
  res.json(post);
});
```

## Cognitive Complexity Metric

**Before:** Cognitive complexity = 15
(Each nesting level adds 1)

**After:** Cognitive complexity = 3
(Much easier to understand)

**Rule:** Code with cognitive complexity > 10 is hard to maintain.
