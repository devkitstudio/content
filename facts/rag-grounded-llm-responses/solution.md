## RAG Pipeline Architecture

RAG = Retrieve relevant documents → Augment prompt with them → Generate response

```
User Query
    ↓
[1] Embedding
    ↓
[2] Vector Search (retrieve top K docs)
    ↓
[3] Augment Prompt (add retrieved docs)
    ↓
[4] LLM Generation (with grounded context)
    ↓
Response (with citations)
```

### Step 1: Embedding

```python
from anthropic import Anthropic

def embed_text(text: str) -> list[float]:
    """Convert text to vector embedding"""
    # Use OpenAI's text-embedding-3-small or similar
    response = openai.Client().embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding
```

### Step 2: Vector Search

```python
import pinecone

class RAGRetriever:
    def __init__(self, index_name: str):
        self.index = pinecone.Index(index_name)

    def retrieve(self, query: str, top_k: int = 5) -> list[dict]:
        """Find most relevant documents"""
        query_embedding = embed_text(query)

        # Search vector database
        results = self.index.query(
            vector=query_embedding,
            top_k=top_k,
            include_metadata=True
        )

        # Extract documents with relevance scores
        documents = []
        for match in results.matches:
            documents.append({
                'content': match.metadata['text'],
                'source': match.metadata['source'],
                'score': match.score,  # 0-1 relevance
                'id': match.id
            })

        return documents

# Example usage
retriever = RAGRetriever('product-docs')
docs = retriever.retrieve('How do I enable SSO?')
# Returns: [
#   { content: 'SSO is available in Pro plan...', source: 'docs/sso.md', score: 0.92 },
#   { content: 'Enterprise features include...', source: 'docs/enterprise.md', score: 0.85 },
#   ...
# ]
```

### Step 3 & 4: Augment and Generate

```python
from anthropic import Anthropic

class GroundedChatbot:
    def __init__(self):
        self.client = Anthropic()
        self.retriever = RAGRetriever('product-docs')

    def answer(self, user_query: str) -> str:
        # Retrieve relevant documents
        documents = self.retriever.retrieve(user_query, top_k=5)

        # Filter by relevance threshold
        relevant = [doc for doc in documents if doc['score'] > 0.75]

        if not relevant:
            return "I don't have information about that."

        # Build context from retrieved docs
        context = "Based on our documentation:\n\n"
        citations = []
        for i, doc in enumerate(relevant, 1):
            context += f"[{i}] {doc['content']}\n"
            citations.append(f"[{i}] {doc['source']}")

        # Generate response with grounded context
        response = self.client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=1024,
            system="""You are a product support specialist.
You MUST:
1. Only use information from the provided documentation
2. Cite sources for each fact
3. Say "I don't know" if information isn't in the docs
4. Never make up features or pricing""",
            messages=[
                {
                    "role": "user",
                    "content": f"{context}\n\nUser question: {user_query}"
                }
            ]
        )

        answer = response.content[0].text

        # Append citations
        return f"{answer}\n\nSources:\n" + "\n".join(citations)

# Usage
chatbot = GroundedChatbot()
response = chatbot.answer("What's included in the Pro plan?")
# Output: "The Pro plan includes [list from docs]
#          Sources:
#          [1] docs/pricing.md
#          [2] docs/features.md"
```

## Advanced: Reranking for Quality

```python
from cohere import Client

class ImprovedRAG:
    def __init__(self):
        self.retriever = RAGRetriever('product-docs')
        self.reranker = Client().rerank  # Cohere reranker

    def retrieve_and_rerank(self, query: str, top_k: int = 5) -> list[dict]:
        """Retrieve candidates, then rerank with ML model"""

        # Step 1: Get more candidates (e.g., top 20)
        candidates = self.retriever.retrieve(query, top_k=20)

        # Step 2: Rerank with better model
        texts = [doc['content'] for doc in candidates]
        rerank_response = self.reranker(
            model="rerank-english-v2.0",
            query=query,
            documents=texts,
            top_n=top_k
        )

        # Step 3: Return top K reranked results
        reranked = []
        for result in rerank_response.results:
            original = candidates[result.index]
            reranked.append({
                **original,
                'rerank_score': result.relevance_score
            })

        return reranked

# Reranking improves relevance:
# - Vector search (fast): 80-85% accuracy
# - Reranked (ML-based): 92-95% accuracy
```

## Handling Hallucinations

```python
class SafeRAG:
    def __init__(self, confidence_threshold: float = 0.85):
        self.retriever = RAGRetriever('product-docs')
        self.threshold = confidence_threshold

    def answer_safely(self, query: str) -> tuple[str, bool]:
        """Return answer only if confident it's grounded"""

        docs = self.retriever.retrieve(query, top_k=5)

        # Check if we have high-quality matches
        relevant = [doc for doc in docs if doc['score'] > self.threshold]

        if not relevant:
            return ("I don't have reliable information about that. Please contact support.", False)

        # Proceed with RAG
        context = "...".join([d['content'] for d in relevant])

        response = self.client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=500,
            system="""You are a precise support specialist.
RULES:
1. ONLY answer using the provided context
2. If the context doesn't clearly answer the question, say "I don't know"
3. Never infer or extrapolate beyond the provided information
4. When unsure, admit it and suggest contacting support""",
            messages=[{"role": "user", "content": f"Context:\n{context}\n\nQuestion: {query}"}]
        )

        answer = response.content[0].text
        is_confident = any(doc['score'] > 0.9 for doc in relevant)

        return answer, is_confident

# Usage
answer, confident = rag.answer_safely("What's your refund policy?")
if not confident:
    # Add warning or escalate to human
    answer = f"⚠️ {answer}"
```
