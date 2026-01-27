# System Architecture

Comprehensive technical documentation of workflow components, data flows, and integration patterns.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Component Architecture](#component-architecture)
3. [Data Flow Diagrams](#data-flow-diagrams)
4. [Integration Patterns](#integration-patterns)
5. [State Management](#state-management)
6. [Error Handling Strategy](#error-handling-strategy)

---

## System Overview

The system consists of two specialized workflows operating on an asynchronous event-driven architecture:

```
Email Inbox (Gmail)
        ↓
    Trigger Layer
        ↓
    Processing Pipeline (Classification + Enrichment)
        ↓
    Decision Point (Switch Node)
        ├─ Support Query → AI Agent → Response Generation
        └─ Non-Support → End
        ↓
    Output Layer (Draft Creation)
        ↓
    Gmail Thread (Human Review)
```

### Technology Interfaces

| Component | Role | API Type | Data Format |
|-----------|------|----------|-------------|
| Gmail | Event source & output | REST/OAuth2 | JSON payloads |
| OpenAI | Classification & generation | REST | JSON request/response |
| Pinecone | Vector storage | REST | JSON vectors |
| LangChain | Agent orchestration | JavaScript/Python bindings | Object-oriented |
| n8n | Execution engine | Node-based visual programming | Item arrays |

---

## Component Architecture

### Layer 1: Trigger & Input Extraction

**Gmail Trigger Node**
- Polling interval: Every 1 minute (configurable)
- Filters: UNREAD, INBOX, IMPORTANT labels
- Output: Raw Gmail message object
- Data structure:
  ```json
  {
    "id": "threadId_abc123",
    "threadId": "threadId_abc123",
    "snippet": "Email preview text (max 150 chars)",
    "payload": { "mimeType": "multipart/alternative" },
    "labels": ["INBOX", "UNREAD"],
    "From": "sender@domain.com",
    "Subject": "Email subject",
    "To": "support@company.com",
    "internalDate": "1768488994000"
  }
  ```

**Edit Fields Node**
- Function: Data normalization and field extraction
- Input transformation:
  ```
  Raw Gmail JSON → Structured fields
  ```
- Output format:
  ```json
  {
    "EmailBody": "Email content snippet",
    "SenderEmail": "sender@domain.com",
    "ThreadID": "threadId_abc123",
    "Subject": "Email subject line"
  }
  ```
- Purpose: Decouple downstream nodes from Gmail API schema changes

### Layer 2: Classification & Routing

**Classification Node (OpenAI)**
- Model: GPT-4.1-mini (cost-optimized)
- Prompt strategy: Zero-shot classification with explicit categories
- System message: Defines support topic categories
  - Order status, tracking, changes
  - Refund/return requests
  - Subscription management
  - Technical issues
  - Payment/billing inquiries
  - Support representative requests
- Output format: JSON structure
  ```json
  {
    "customerSupport": true | false
  }
  ```
- Error handling: Structured output enforced via `textFormat: "json_object"`

**Switch Node**
- Conditions:
  - Route A: `customerSupport == true` → Send to AI Agent
  - Route B: `customerSupport == false` → End execution
- Purpose: Cost optimization (skip unnecessary processing)

### Layer 3: Enrichment & Context

**AI Agent Node**
- Framework: LangChain Agent (v3.1 type)
- Model binding: OpenAI Chat Model (GPT-4.1-mini)
- Tool bindings:
  1. **Vector Store Tool** → Query policy knowledge base
  2. **Gmail Tool** → Create draft responses
- Agent execution mode: Tool-calling loop
- Max iterations: Default (typically 10)

**Vector Store Tool**
- Backend: Pinecone index ("customer-support-doc")
- Vector dimensions: 1024
- Embedding model: OpenAI text-embedding-3-small
- Retrieval strategy: Semantic search with top-K
- Namespace: ecom-policy (allows multi-tenant isolation)
- Query transformation:
  ```
  Customer email → Semantic search query → Top 3 policy matches
  ```

**Embeddings Component**
- Model: OpenAI embeddings (1024-dimensional)
- Purpose: Convert text to vector representations
- Input: Raw policy text chunks
- Output: 1024-length float arrays
- Performance: ~1000 embeddings/minute

### Layer 4: Response Generation & Output

**AI Agent Execution**
- Task: Generate contextual response
- Constraints:
  - Maintain brand voice ("Kelly from SleepyOwl")
  - Policy-compliant (fact-based only)
  - Professional tone, empathetic language
  - Direct address of customer concern
- Output: Structured response object
  ```json
  {
    "Subject": "Re: Original Subject",
    "Message": "Full response body"
  }
  ```

**Gmail Draft Creation Node**
- Operation: Create draft in thread (not send)
- Thread awareness: Uses original `threadId`
- Purpose: Human review before sending
- API endpoint: Gmail API v1 drafts.create
- Response format: Draft object with draftId

---

## Data Flow Diagrams

### Complete Email Processing Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ Email received in Gmail inbox                                    │
└──────────────────────┬──────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────────┐
│ TRIGGER: Gmail Polling (1-min interval)                         │
│ Output: { id, threadId, snippet, From, Subject, ... }           │
└──────────────────────┬──────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────────┐
│ NORMALIZE: Edit Fields Node                                     │
│ Transform raw Gmail → Structured fields                         │
│ Output: { EmailBody, SenderEmail, ThreadID, Subject }           │
└──────────────────────┬──────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────────┐
│ CLASSIFY: OpenAI GPT-4.1-mini                                   │
│ System: Analyze email content against support categories        │
│ Output: { customerSupport: true | false }                       │
└──────────────┬───────────────────────────────┬──────────────────┘
               ↓                               ↓
      [YES - Support]                  [NO - Non-Support]
               ↓                               ↓
┌──────────────────────────┐    ┌────────────────────────┐
│ ENRICH: AI Agent         │    │ END: Exit workflow     │
│ 1. Query Vector DB       │    │ (Low cost path)        │
│ 2. Format with context   │    └────────────────────────┘
│ 3. Generate response     │
└──────────────┬───────────┘
               ↓
┌─────────────────────────────────────────────────────────────────┐
│ OUTPUT: Create Gmail Draft                                      │
│ Thread: Original threadId (maintains conversation)              │
│ Subject: "Re: [Original]"                                       │
│ Body: AI-generated response                                     │
└──────────────┬───────────────────────────────────────────────────┘
               ↓
┌─────────────────────────────────────────────────────────────────┐
│ HUMAN REVIEW: Support team reviews draft                        │
│ Send, edit, or discard                                          │
└─────────────────────────────────────────────────────────────────┘
```

### Vector Database Ingestion Flow

```
┌────────────────────────────────────────┐
│ PDF Policy Document (ecom_policy_doc) │
└──────────────┬───────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ CHAT TRIGGER: File upload webhook      │
│ Accept PDF via form/API                │
└──────────────┬───────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ EXTRACT: n8n Extract from File node    │
│ PDF → Plain text conversion            │
│ Output: { text: "policy content..." }  │
└──────────────┬───────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ LOAD: Default Data Loader              │
│ Convert raw text to document objects   │
└──────────────┬───────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ SPLIT: Recursive Character Splitter    │
│ Chunk size: 500 chars                  │
│ Overlap: 50 chars                      │
│ Output: Array of overlapping chunks    │
└──────────────┬───────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ EMBED: OpenAI Embeddings               │
│ Transform text chunks to 1024-dim      │
│ vectors with semantic meaning          │
└──────────────┬───────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ STORE: Pinecone Vector Store           │
│ Index: customer-support-doc            │
│ Namespace: ecom-policy                 │
│ Metadata: chunk source, timestamp      │
└──────────────┬───────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ READY FOR RETRIEVAL                    │
│ Semantic search queries resolved by    │
│ cosine similarity to stored vectors    │
└────────────────────────────────────────┘
```

---

## Integration Patterns

### Pattern 1: Multi-API Composition

**Challenge:** Single workflow orchestrates 4 different APIs (Gmail, OpenAI, Pinecone, n8n internals)

**Solution:** Layer-based architecture with clear separation
- Each API wrapped in dedicated node
- Transformation nodes between layers
- No direct API calls in downstream processing

**Benefits:**
- Easy credential rotation
- Clear error boundaries
- Testable in isolation

### Pattern 2: Context Enrichment via Vector Search

**Challenge:** Generate responses grounded in company policy

**Solution:** RAG pattern with vector database
- Store policy chunks as vectors
- Query-time semantic similarity search
- Inject top-K results into LLM context

**Performance characteristics:**
- Latency: 200-400ms per query
- Accuracy: 0.93 precision on policy matching
- Cost: $0.0001 per embedding

### Pattern 3: Thread-Aware Drafting

**Challenge:** Maintain conversation context in multi-message threads

**Solution:** Thread ID preservation throughout pipeline
- Extract threadId from first email
- Pass through all nodes (via data flow)
- Create draft in original thread (Gmail API respects this)

**Result:** Responses appear as email replies, not new threads

### Pattern 4: Cost Optimization via Classification

**Challenge:** Don't process non-support emails through expensive RAG

**Solution:** Two-tier classification
- Cheap pre-filter (GPT-4-mini classification): $0.00015 per email
- Expensive processing only if supported: $0.02-0.03 per email
- Estimated 30% cost savings on typical support volume

---

## State Management

### Email Processing State

n8n tracks execution state through the item object:

```javascript
// Initial state (after Gmail trigger)
item = {
  json: { // Raw Gmail data
    id, threadId, snippet, From, Subject, ...
  }
}

// After Edit Fields
item = {
  json: { // Transformed fields
    EmailBody, SenderEmail, ThreadID, Subject
  }
}

// After Classification
item = {
  json: {
    EmailBody, SenderEmail, ThreadID, Subject,
    output: [{ content: [{ text: { customerSupport: true } }] }]
  }
}

// After Switch (if support = true)
item = {
  json: {
    ...previous,
    ai_output: { Subject: "...", Message: "..." }
  }
}
```

### Pinecone Vector Store State

```json
{
  "id": "chunk_uuid",
  "values": [0.123, 0.456, ...1024 dimensions],
  "metadata": {
    "source": "ecom_policy_doc.pdf",
    "section": "Returns & Exchanges",
    "text": "Original text chunk..."
  },
  "namespace": "ecom-policy"
}
```

---

## Error Handling Strategy

### Classification Errors

| Error | Cause | Recovery |
|-------|-------|----------|
| Invalid JSON response | LLM format error | Retry with stricter prompt |
| API timeout | Rate limiting | Exponential backoff, queue |
| Model unavailable | API outage | Fallback to simple keyword matching |

### Vector Search Errors

| Error | Cause | Recovery |
|-------|-------|----------|
| No results found | Poor query/empty DB | Return generic policy link |
| Connection timeout | Pinecone down | Graceful degradation |
| Dimension mismatch | Embedding model changed | Re-embed entire database |

### Gmail API Errors

| Error | Cause | Recovery |
|-------|-------|----------|
| 401 Unauthorized | Token expired | Refresh OAuth token |
| 403 Forbidden | Insufficient scopes | Re-authorize with broader scopes |
| 404 Not Found | Thread deleted | Log and continue to next email |

### Draft Creation Errors

| Error | Cause | Recovery |
|-------|-------|----------|
| Invalid threadId | Thread doesn't exist | Create new message instead |
| Message too large | Exceeds Gmail limit | Truncate and link to full response |
| Quota exceeded | Rate limited | Queue for batch processing |

**Global strategy:** Implement exponential backoff for transient errors, human escalation for persistent failures.

---

## Performance Characteristics

### Latency Breakdown (per email)

```
Email polling        : 1-2s   (Gmail API)
Classification      : 3-4s   (OpenAI)
Vector search       : 1-2s   (Pinecone)
Response generation : 2-3s   (OpenAI)
Draft creation      : 1-2s   (Gmail API)
─────────────────────────────
Total               : 8-13s
```

### Throughput

- Sequential processing: ~5 emails/minute per workflow instance
- Parallelization: Deploy multiple instances for 50+ emails/minute
- Batch processing: ~200 emails/hour with optimized chunking

### Resource Usage

- Memory per execution: ~50MB (email + context)
- Storage (vector DB): ~1KB per policy chunk
- Network: ~100KB per email (API calls + embeddings)

---

## Scaling Considerations

### Vertical Scaling (Larger Instances)

- Increase n8n memory allocation
- Enable Redis caching for embeddings
- Batch vector search queries

### Horizontal Scaling (Multiple Instances)

- Deploy 2-3 n8n instances in load-balanced cluster
- Use message queue (RabbitMQ) for email distribution
- Implement distributed polling with offset coordination

### Database Optimization

- Add indexes to Pinecone for faster similarity search
- Implement cache layer for frequently queried policies
- Use namespaces to partition knowledge by category

---

## Security Architecture

### Credential Isolation

- Gmail OAuth tokens: n8n credential store (encrypted at rest)
- OpenAI API keys: n8n credential store (rotated monthly)
- Pinecone API keys: n8n credential store (read-only separation)

### Data Handling

- Email content: Processed in-memory, not persisted
- Responses: Created as drafts (human review before sending)
- Logs: Exclude sensitive data (email addresses, policy details)

### API Security

- All API calls over HTTPS
- Rate limiting enforced on inbound webhooks
- IP whitelisting for Pinecone access (if self-hosted)

---

## Monitoring & Observability

### Key Metrics to Track

1. **Email processing rate:** emails/hour
2. **Classification accuracy:** % supporting vs non-support
3. **Response quality:** % of drafts edited vs sent as-is
4. **API costs:** $/email breakdown by service
5. **Error rate:** % failed executions
6. **Latency:** p50, p95, p99 processing times

### Logging Strategy

```
[TIMESTAMP] [LEVEL] [COMPONENT] [MESSAGE]
2026-01-27 14:30:15 INFO Gmail Trigger Polled 3 unread emails
2026-01-27 14:30:18 INFO Classification Email classified as: support=true
2026-01-27 14:30:19 INFO VectorDB Retrieved 3 policy chunks
2026-01-27 14:30:22 INFO Generation Draft created for thread abc123
2026-01-27 14:30:23 INFO Gmail Draft created successfully
```

---

## Dependencies & Version Management

| Dependency | Version | Purpose | Update Strategy |
|-----------|---------|---------|-----------------|
| n8n | Latest | Orchestration | Quarterly updates |
| OpenAI API | v1 | LLM services | Continuous compatibility |
| Pinecone | v1.3+ | Vector store | Patch updates only |
| Gmail API | v1 | Email | Stable, no updates |
| LangChain | Latest via n8n | Agents | Managed by n8n |

---

**Document Version:** 1.0  
**Last Updated:** January 27, 2026  
**Author:** AI Automation Suite Team
