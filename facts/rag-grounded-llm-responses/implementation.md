## Complete RAG Stack

```python
from anthropic import Anthropic
import pinecone
import openai
from typing import Optional

class ProductRAGSystem:
    def __init__(self, pinecone_index: str):
        self.client = Anthropic()
        self.pinecone_index = pinecone.Index(pinecone_index)
        self.openai_client = openai.Client()

    def embed(self, text: str) -> list[float]:
        """Embed text using OpenAI"""
        response = self.openai_client.embeddings.create(
            model="text-embedding-3-small",
            input=text
        )
        return response.data[0].embedding

    def retrieve(self, query: str, top_k: int = 5) -> list[dict]:
        """Retrieve relevant documents from vector DB"""
        query_embedding = self.embed(query)

        results = self.pinecone_index.query(
            vector=query_embedding,
            top_k=top_k,
            include_metadata=True
        )

        documents = []
        for match in results.matches:
            documents.append({
                'id': match.id,
                'content': match.metadata.get('text', ''),
                'source': match.metadata.get('source', 'unknown'),
                'score': float(match.score)
            })

        return documents

    def generate(self, query: str, documents: list[dict]) -> str:
        """Generate grounded response using retrieved docs"""

        # Build context string
        context_parts = []
        for i, doc in enumerate(documents, 1):
            context_parts.append(
                f"[Source {i}: {doc['source']}]\n{doc['content']}"
            )
        context = "\n\n".join(context_parts)

        # Generate with grounded context
        response = self.client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=1024,
            system="""You are a helpful product support specialist.
You have been provided with relevant documentation to answer the user's question.

Instructions:
1. Answer ONLY using the provided documentation
2. Cite sources using [Source N] notation
3. If the answer isn't in the documentation, clearly say so
4. Be conversational but accurate
5. Never make up features or details""",
            messages=[
                {
                    "role": "user",
                    "content": f"""Documentation:
{context}

User Question: {query}"""
                }
            ]
        )

        return response.content[0].text

    def chat(self, query: str) -> dict:
        """Full RAG pipeline"""
        # Retrieve
        documents = self.retrieve(query, top_k=5)

        if not documents or all(doc['score'] < 0.5 for doc in documents):
            return {
                'answer': "I don't have information about that. Please contact support.",
                'sources': [],
                'confidence': 'low'
            }

        # Filter to relevant documents
        relevant = [doc for doc in documents if doc['score'] > 0.6]

        # Generate
        answer = self.generate(query, relevant)

        return {
            'answer': answer,
            'sources': list(set(doc['source'] for doc in relevant)),
            'confidence': 'high' if any(doc['score'] > 0.85 for doc in relevant) else 'medium'
        }

# Setup: Ingest product documentation
def ingest_documentation():
    """One-time setup: embed and index all product docs"""
    rag = ProductRAGSystem('product-docs')

    docs = [
        {
            'text': 'SSO (Single Sign-On) is available on Pro and Enterprise plans...',
            'source': 'docs/authentication.md'
        },
        {
            'text': 'Pro plan costs $50/month, includes 100 projects, priority support...',
            'source': 'docs/pricing.md'
        },
        # ... more docs
    ]

    for doc in docs:
        embedding = rag.embed(doc['text'])
        pinecone.Index('product-docs').upsert([(
            f"doc-{hash(doc['text'])}",
            embedding,
            {'text': doc['text'], 'source': doc['source']}
        )])

# Usage
rag = ProductRAGSystem('product-docs')

result = rag.chat("Does Pro plan include SSO?")
print(result['answer'])
# Output: "Yes, SSO is available on Pro and Enterprise plans... [Source 1: docs/authentication.md]"
```

## React Integration

```tsx
import React, { useState } from 'react';

export function RAGChatbot() {
  const [query, setQuery] = useState('');
  const [response, setResponse] = useState<{
    answer: string;
    sources: string[];
    confidence: string;
  } | null>(null);
  const [loading, setLoading] = useState(false);

  const handleQuery = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);

    try {
      const res = await fetch('/api/rag/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ query })
      });

      const data = await res.json();
      setResponse(data);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="rag-chatbot">
      <form onSubmit={handleQuery}>
        <input
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Ask about our product..."
          disabled={loading}
        />
        <button type="submit" disabled={loading}>
          {loading ? 'Searching...' : 'Ask'}
        </button>
      </form>

      {response && (
        <div className="response">
          <div className={`answer confidence-${response.confidence}`}>
            {response.answer}
          </div>
          {response.sources.length > 0 && (
            <div className="sources">
              <strong>Sources:</strong>
              <ul>
                {response.sources.map((source) => (
                  <li key={source}>{source}</li>
                ))}
              </ul>
            </div>
          )}
        </div>
      )}
    </div>
  );
}
```
