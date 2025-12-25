# Search and RAG

## Overview

The platform provides a unified search experience - one search bar for both finding documents and asking AI questions (à la Glean). This doc covers the search pipeline and RAG implementation.

**Key Package:** We use [LangChain.js](https://js.langchain.com) for the RAG pipeline. See [Tech Stack](./11-tech-stack.md) for setup details.

## Search Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SEARCH PIPELINE                                    │
│                                                                             │
│  User Query: "What are the payment terms in the Acme contract?"            │
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │   Query     │───▶│   Hybrid    │───▶│    ACL      │───▶│   Ranking   │  │
│  │   Parse     │    │   Search    │    │   Filter    │    │   + Merge   │  │
│  │             │    │             │    │             │    │             │  │
│  │ • Intent    │    │ • Keyword   │    │ • User perms│    │ • RRF/score │  │
│  │ • Entities  │    │ • Vector    │    │ • Tenant    │    │ • Dedupe    │  │
│  │ • Filters   │    │ • Both      │    │             │    │ • Top K     │  │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘  │
│                                                                             │
│                                    │                                        │
│                                    ▼                                        │
│                          ┌─────────────────┐                               │
│                          │    Response     │                               │
│                          │                 │                               │
│                          │  Search Results │                               │
│                          │       OR        │                               │
│                          │  RAG + Answer   │                               │
│                          └─────────────────┘                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Query Processing

### Intent Detection

Determine if query needs search results, AI answer, or both:

```typescript
type QueryIntent = 
  | 'search'      // "Find contracts from 2023"
  | 'question'    // "What is the termination clause?"
  | 'hybrid';     // "Payment terms in Acme contract" (find + summarize)

function detectIntent(query: string): QueryIntent {
  // Simple heuristics for MVP
  const questionPatterns = [
    /^(what|who|when|where|why|how|is|are|can|could|would|should)/i,
    /\?$/,
  ];
  
  const searchPatterns = [
    /^(find|show|list|get|search)/i,
  ];
  
  if (questionPatterns.some(p => p.test(query))) {
    return 'question';
  }
  if (searchPatterns.some(p => p.test(query))) {
    return 'search';
  }
  return 'hybrid';
}
```

### Filter Extraction

Extract structured filters from natural language:

```typescript
interface SearchFilters {
  dateRange?: { from?: Date; to?: Date };
  documentTypes?: string[];        // 'pdf', 'docx', 'email'
  authors?: string[];
  folders?: string[];
  matters?: string[];
  hasAttachments?: boolean;
}

// Examples:
// "contracts from last month" → dateRange: { from: lastMonth }
// "emails from John" → documentTypes: ['email'], authors: ['john']
// "documents in the Acme folder" → folders: ['acme']
```

## Hybrid Search

Combine keyword (BM25) and vector (semantic) search for best results.

### Keyword Search (Full-Text)

Using PostgreSQL full-text search:

```sql
-- Create search index
ALTER TABLE document_chunks 
  ADD COLUMN search_vector tsvector 
  GENERATED ALWAYS AS (to_tsvector('english', content)) STORED;

CREATE INDEX idx_chunks_search ON document_chunks USING GIN (search_vector);

-- Keyword search query
SELECT id, document_id, content,
       ts_rank(search_vector, plainto_tsquery('english', $1)) as keyword_score
FROM document_chunks
WHERE tenant_id = $2
  AND search_vector @@ plainto_tsquery('english', $1)
ORDER BY keyword_score DESC
LIMIT 50;
```

### Vector Search (Semantic)

Using pgvector for similarity search:

```sql
-- Create vector index
CREATE INDEX idx_chunks_embedding 
  ON document_chunks 
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- Vector search query
SELECT id, document_id, content,
       1 - (embedding <=> $1) as vector_score  -- Cosine similarity
FROM document_chunks
WHERE tenant_id = $2
ORDER BY embedding <=> $1
LIMIT 50;
```

### Hybrid Fusion

Combine results using Reciprocal Rank Fusion (RRF):

```typescript
interface SearchResult {
  chunkId: string;
  documentId: string;
  content: string;
  keywordRank?: number;
  vectorRank?: number;
  fusedScore: number;
}

function reciprocalRankFusion(
  keywordResults: SearchResult[],
  vectorResults: SearchResult[],
  k: number = 60  // RRF constant
): SearchResult[] {
  const scores = new Map<string, number>();
  
  // Score from keyword results
  keywordResults.forEach((result, index) => {
    const rrf = 1 / (k + index + 1);
    scores.set(result.chunkId, (scores.get(result.chunkId) || 0) + rrf);
  });
  
  // Score from vector results
  vectorResults.forEach((result, index) => {
    const rrf = 1 / (k + index + 1);
    scores.set(result.chunkId, (scores.get(result.chunkId) || 0) + rrf);
  });
  
  // Merge and sort
  const allResults = new Map<string, SearchResult>();
  [...keywordResults, ...vectorResults].forEach(r => {
    if (!allResults.has(r.chunkId)) {
      allResults.set(r.chunkId, r);
    }
  });
  
  return Array.from(allResults.values())
    .map(r => ({ ...r, fusedScore: scores.get(r.chunkId)! }))
    .sort((a, b) => b.fusedScore - a.fusedScore);
}
```

## RAG Pipeline

### Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             RAG PIPELINE                                     │
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │  Retrieve   │───▶│   Rerank    │───▶│  Context    │───▶│  Generate   │  │
│  │             │    │  (optional) │    │   Build     │    │             │  │
│  │ • Hybrid    │    │             │    │             │    │ • GPT-4.1   │  │
│  │   search    │    │ • Cross-    │    │ • Select    │    │ • Stream    │  │
│  │ • Top 50    │    │   encoder   │    │   chunks    │    │ • Citations │  │
│  │             │    │ • Top 10    │    │ • Format    │    │             │  │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Retrieval

```typescript
async function retrieve(
  query: string,
  userId: string,
  tenantId: string,
  options: RetrievalOptions = {}
): Promise<RetrievedChunk[]> {
  const { topK = 50, filters } = options;
  
  // 1. Embed query
  const queryEmbedding = await embedQuery(query);
  
  // 2. Hybrid search
  const [keywordResults, vectorResults] = await Promise.all([
    keywordSearch(query, tenantId, topK, filters),
    vectorSearch(queryEmbedding, tenantId, topK, filters),
  ]);
  
  // 3. Fuse results
  const fused = reciprocalRankFusion(keywordResults, vectorResults);
  
  // 4. ACL filter
  const permitted = await filterByACL(fused, userId, tenantId);
  
  return permitted.slice(0, topK);
}
```

### Context Building

```typescript
interface RAGContext {
  chunks: ContextChunk[];
  totalTokens: number;
}

interface ContextChunk {
  id: string;
  documentId: string;
  documentTitle: string;
  content: string;
  citationIndex: number;  // [1], [2], etc.
  metadata: {
    page?: number;
    section?: string;
  };
}

function buildContext(
  chunks: RetrievedChunk[],
  maxTokens: number = 12000
): RAGContext {
  const contextChunks: ContextChunk[] = [];
  let totalTokens = 0;
  
  for (let i = 0; i < chunks.length; i++) {
    const chunk = chunks[i];
    const chunkTokens = estimateTokens(chunk.content);
    
    if (totalTokens + chunkTokens > maxTokens) break;
    
    contextChunks.push({
      id: chunk.id,
      documentId: chunk.documentId,
      documentTitle: chunk.documentTitle,
      content: chunk.content,
      citationIndex: i + 1,
      metadata: chunk.metadata,
    });
    
    totalTokens += chunkTokens;
  }
  
  return { chunks: contextChunks, totalTokens };
}
```

### Prompt Template

```typescript
function buildPrompt(query: string, context: RAGContext): string {
  const sourcesText = context.chunks
    .map(c => `[${c.citationIndex}] ${c.documentTitle}\n${c.content}`)
    .join('\n\n---\n\n');
  
  return `You are a helpful legal research assistant. Answer the user's question based on the provided sources. 

Rules:
- Only use information from the provided sources
- Cite sources using [1], [2], etc. inline
- If the sources don't contain enough information, say so
- Be precise and professional

Sources:
${sourcesText}

Question: ${query}

Answer:`;
}
```

### Generation

```typescript
async function generateAnswer(
  query: string,
  context: RAGContext,
  options: GenerationOptions = {}
): Promise<RAGResponse> {
  const prompt = buildPrompt(query, context);
  
  const completion = await openai.chat.completions.create({
    model: 'gpt-4.1',
    messages: [
      { role: 'system', content: 'You are a legal research assistant.' },
      { role: 'user', content: prompt },
    ],
    temperature: 0.1,        // Low temp for factual responses
    max_tokens: 2000,
    stream: options.stream,
  });
  
  // Parse citations from response
  const answer = completion.choices[0].message.content;
  const citations = extractCitations(answer, context.chunks);
  
  return {
    answer,
    citations,
    sources: context.chunks.map(c => ({
      documentId: c.documentId,
      documentTitle: c.documentTitle,
      chunkId: c.id,
      citationIndex: c.citationIndex,
    })),
  };
}
```

## Streaming Responses

For better UX, stream the AI response:

```typescript
async function* streamAnswer(
  query: string,
  context: RAGContext
): AsyncGenerator<StreamChunk> {
  const prompt = buildPrompt(query, context);
  
  const stream = await openai.chat.completions.create({
    model: 'gpt-4.1',
    messages: [{ role: 'user', content: prompt }],
    stream: true,
  });
  
  // Yield citation metadata first
  yield { 
    type: 'sources', 
    sources: context.chunks.map(c => ({
      documentId: c.documentId,
      title: c.documentTitle,
      citationIndex: c.citationIndex,
    }))
  };
  
  // Stream answer tokens
  for await (const chunk of stream) {
    const content = chunk.choices[0]?.delta?.content;
    if (content) {
      yield { type: 'token', content };
    }
  }
  
  yield { type: 'done' };
}
```

## Search API

### Endpoints

```typescript
// Unified search endpoint
POST /api/search
{
  "query": "payment terms in Acme contract",
  "mode": "hybrid",           // 'search' | 'question' | 'hybrid' | 'auto'
  "filters": {
    "dateRange": { "from": "2024-01-01" },
    "documentTypes": ["pdf", "docx"]
  },
  "limit": 20
}

// Response for search mode
{
  "results": [
    {
      "documentId": "doc-123",
      "title": "Acme Corp - Services Agreement.pdf",
      "snippet": "...payment shall be due within 30 days...",
      "score": 0.92,
      "webUrl": "https://sharepoint.com/...",
      "highlights": ["payment", "30 days"]
    }
  ],
  "totalCount": 45,
  "query": { "parsed": {...}, "intent": "search" }
}

// Response for question mode
{
  "answer": "According to the Acme Services Agreement [1], payment terms are Net 30...",
  "citations": [
    {
      "index": 1,
      "documentId": "doc-123",
      "title": "Acme Corp - Services Agreement.pdf",
      "snippet": "Payment shall be due within 30 days of invoice date.",
      "webUrl": "https://sharepoint.com/...",
      "page": 5
    }
  ],
  "sources": [...],
  "query": { "parsed": {...}, "intent": "question" }
}
```

## Performance Optimization

### Query Latency Targets

| Stage | Target | Notes |
|-------|--------|-------|
| Query embedding | <100ms | Single API call |
| Hybrid search | <200ms | Depends on index size |
| ACL filtering | <50ms | Cached ACLs |
| Reranking | <300ms | Optional, skip for MVP |
| LLM generation | 1-3s | Streaming hides latency |
| **Total (search)** | <500ms | |
| **Total (RAG)** | <4s | Streaming starts earlier |

### Caching Strategy

```typescript
// Cache frequent queries (short TTL due to personalization)
const queryCache = new Redis();

async function cachedSearch(
  query: string,
  userId: string,
  tenantId: string
): Promise<SearchResult[]> {
  const cacheKey = `search:${tenantId}:${userId}:${hash(query)}`;
  
  const cached = await queryCache.get(cacheKey);
  if (cached) return JSON.parse(cached);
  
  const results = await search(query, userId, tenantId);
  
  // Short TTL - results may change with new docs
  await queryCache.setex(cacheKey, 60, JSON.stringify(results));
  
  return results;
}
```

### Index Maintenance

```sql
-- Reindex vectors periodically for better clustering
REINDEX INDEX idx_chunks_embedding;

-- Vacuum to reclaim space after deletions
VACUUM ANALYZE document_chunks;

-- Monitor index quality
SELECT * FROM pg_stat_user_indexes WHERE relname = 'document_chunks';
```
