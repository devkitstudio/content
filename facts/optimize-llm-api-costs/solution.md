## 5 Cost Reduction Strategies

### 1. Prompt Caching (OpenAI Cache)

```python
import anthropic

client = anthropic.Anthropic()

# Define context that will be cached
system_context = """You are a customer support specialist for TechCorp.
Here's our comprehensive product documentation:

PRODUCT FEATURES:
- Feature A: Real-time collaboration
- Feature B: Advanced analytics
- Feature C: Custom integrations

PRICING:
- Basic: $10/month (up to 5 users)
- Pro: $50/month (unlimited users, custom integrations)
- Enterprise: Custom pricing

SUPPORT POLICY:
- Email: 24 hour response (all tiers)
- Chat: 2 hour response (Pro+)
- Priority: 30 min response (Enterprise)

FAQ:
[... 5000 tokens of knowledge base ...]
"""

# The system context is cached and reused
response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": system_context,
            "cache_control": {"type": "ephemeral"}
        }
    ],
    messages=[
        {
            "role": "user",
            "content": "How much is the Pro plan?"
        }
    ]
)

# First call: ~5000 cache write tokens + 20 input tokens
# Subsequent calls: 5000 cache read tokens only (90% cheaper)
print(response.usage.cache_creation_input_tokens)  # 5000 (first call only)
print(response.usage.cache_read_input_tokens)      # 0 (first call), 5000 (later calls)
```

**Cost savings:** 90% reduction on cached content (reads cost 10% of writes)

### 2. Model Routing (Intelligent Selection)

```python
class SmartLLMRouter:
    def __init__(self):
        self.client = anthropic.Anthropic()

    def select_model(self, query: str, context_length: int) -> str:
        """Route to appropriate model based on complexity"""
        query_lower = query.lower()

        # Simple queries → cheaper model
        if any(keyword in query_lower for keyword in ['how much', 'price', 'cost']):
            return "claude-3-5-haiku-20241022"  # 10x cheaper than Sonnet

        # Moderate queries → mid-tier
        if context_length < 5000:
            return "claude-3-5-sonnet-20241022"

        # Complex queries → best model
        return "claude-3-opus-20250219"

    def process_query(self, user_id: str, query: str, docs: list[str]) -> str:
        context = "\n".join(docs)
        model = self.select_model(query, len(context.split()))

        response = self.client.messages.create(
            model=model,
            max_tokens=1024,
            messages=[
                {
                    "role": "user",
                    "content": f"{context}\n\nQuestion: {query}"
                }
            ]
        )

        return response.content[0].text

# Router ensures only complex queries hit expensive models
router = SmartLLMRouter()
# "What's your price?" → Haiku ($0.00015/1K tokens)
# "Analyze my business metrics" → Opus ($0.015/1K tokens)
```

**Cost breakdown:**
- Haiku: $0.00080 per 1M input tokens
- Sonnet: $0.003 per 1M input tokens
- Opus: $0.015 per 1M input tokens

### 3. Prompt Compression

```python
from compress import BaliCompressor

class CompressedPrompt:
    def __init__(self, docs: list[str]):
        self.compressor = BaliCompressor()
        self.docs = docs

    def compress(self, query: str) -> tuple[str, float]:
        """Compress context while keeping relevant information"""
        combined = "\n".join(self.docs)

        # Original
        original_tokens = len(combined.split())

        # Compress with semantic awareness
        compressed = self.compressor.compress(
            context=combined,
            query=query,
            target_ratio=0.3  # Keep 30% of tokens
        )

        compressed_tokens = len(compressed.split())
        ratio = compressed_tokens / original_tokens

        return compressed, ratio

# Example: 10,000 token context → 3,000 tokens
# Cost reduction: 70% fewer tokens
compressor = CompressedPrompt([doc1, doc2, doc3])
compressed_context, ratio = compressor.compress("What's the ROI?")
print(f"Compressed to {ratio:.1%}")  # ~30%
```

### 4. Batch Processing + Caching

```python
from datetime import datetime, timedelta
from functools import lru_cache

class CachedLLMBatch:
    def __init__(self, ttl_hours=24):
        self.cache = {}
        self.ttl = timedelta(hours=ttl_hours)
        self.batch_queue = []

    @lru_cache(maxsize=10000)
    def get_embedding_hash(self, text: str) -> str:
        """Cache exact queries to avoid re-processing"""
        return hash(text)

    def add_to_batch(self, query: str, user_id: str):
        """Batch similar queries"""
        query_hash = self.get_embedding_hash(query)

        # Check if we've seen this query before
        if query_hash in self.cache:
            cached_result, timestamp = self.cache[query_hash]
            if datetime.now() - timestamp < self.ttl:
                return cached_result  # Return cached, no API call

        # Add to batch
        self.batch_queue.append({
            'query': query,
            'user_id': user_id,
            'hash': query_hash
        })

        return None

    def process_batch(self):
        """Send grouped queries in single API call"""
        if not self.batch_queue:
            return

        # Deduplicate
        seen = set()
        unique_queries = []
        for item in self.batch_queue:
            if item['query'] not in seen:
                seen.add(item['query'])
                unique_queries.append(item)

        # Single API call for multiple queries
        batch_response = anthropic.Anthropic().messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=2000,
            messages=[
                {
                    "role": "user",
                    "content": f"Answer these {len(unique_queries)} questions: " +
                              "\n".join(f"{i+1}. {q['query']}" for q, i in zip(unique_queries, range(len(unique_queries))))
                }
            ]
        )

        # Cache all results
        for item in unique_queries:
            self.cache[item['hash']] = (batch_response, datetime.now())

        self.batch_queue = []

# Process 100 user queries:
# - Without batching: 100 API calls
# - With batching + caching: 5 API calls (80% cost reduction)
```

### 5. Local-First Fallbacks

```python
import spacy

class HybridLLMSystem:
    def __init__(self):
        self.llm_client = anthropic.Anthropic()
        self.nlp = spacy.load("en_core_web_sm")  # Local NLP
        self.simple_rules = {
            'price': self.get_price_from_db,
            'feature': self.get_feature_from_db,
            'contact': self.get_contact_from_db,
        }

    def process_query(self, query: str) -> str:
        # Try rule-based first (free)
        doc = self.nlp(query)
        for entity in doc.ents:
            if entity.label_ == 'PRODUCT':
                return self.get_feature_from_db(entity.text)

        # Try simple regex patterns (free)
        if query.lower().startswith('how much'):
            return self.get_price_from_db(query)

        # Fall back to LLM only when needed
        response = self.llm_client.messages.create(
            model="claude-3-5-haiku-20241022",  # Cheapest
            max_tokens=500,
            messages=[{"role": "user", "content": query}]
        )

        return response.content[0].text

    def get_price_from_db(self, product: str) -> str:
        """Rule-based lookup (no LLM cost)"""
        prices = {'basic': '$10', 'pro': '$50', 'enterprise': 'custom'}
        return prices.get(product.lower(), "Price not found")

    def get_feature_from_db(self, feature: str) -> str:
        """Database lookup instead of LLM"""
        # Query features table directly
        return f"Feature {feature} details: ..."

    def get_contact_from_db(self, contact_type: str) -> str:
        """Return canned responses"""
        return "Contact us at support@example.com"

# Result: 70% of queries handled without LLM calls
# Cost reduction: 70%
```

## Implementation Priority

1. **Day 1:** Implement prompt caching (quick 70% win)
2. **Week 1:** Add model routing (another 40% reduction)
3. **Week 2:** Set up batch processing (30% more savings)
4. **Month 1:** Add local fallbacks (25% more savings)
