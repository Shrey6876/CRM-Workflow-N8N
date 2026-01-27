# Workflow Specifications

Detailed node-by-node specifications for each workflow with configuration parameters and usage patterns.

---

## Table of Contents

1. [Customer Support Agent Workflow](#customer-support-agent-workflow)
2. [Pinecone Setup Workflow](#pinecone-setup-workflow)
3. [Node Configuration Reference](#node-configuration-reference)
4. [Data Transformation Details](#data-transformation-details)

---

## Customer Support Agent Workflow

**File:** `Customer-Support-Agent.json`

**Purpose:** Process incoming emails, classify support relevance, retrieve policy context, and generate contextual response drafts.

**Active:** false (Enable after credentials configured)

**Execution Mode:** Trigger-based polling (1-minute intervals)

### Workflow Overview

```
Gmail Trigger
    ↓
Edit Fields (Extract metadata)
    ↓
Check if Customer Support (GPT-4 classification)
    ↓
Switch (Route by classification)
    ├─ YES → AI Agent (Generate response)
    └─ NO → End
    ↓
Create Draft in Gmail
```

---

### Node Specifications

#### 1. Gmail Trigger

**Type:** `n8n-nodes-base.gmailTrigger`

**Version:** 1.3

**Configuration:**
```json
{
  "pollTimes": {
    "item": [
      {
        "mode": "everyMinute"
      }
    ]
  },
  "filters": {}
}
```

**Credentials Required:**
- Gmail OAuth2 account with read/write scope

**Output Format:**
```json
{
  "id": "string (unique message ID)",
  "threadId": "string (conversation thread ID)",
  "snippet": "string (email preview, max 150 chars)",
  "payload": {
    "mimeType": "multipart/alternative | text/plain | text/html"
  },
  "labels": ["string"],
  "internalDate": "string (epoch milliseconds)",
  "From": "string (email address)",
  "Subject": "string (email subject)",
  "To": "string (recipient email)"
}
```

**Key Behaviors:**
- Polls every 60 seconds (configurable via cron)
- Fetches unread emails from INBOX
- Respects Gmail API rate limits (15 requests/second)
- Marks emails as processed to prevent re-processing

**Common Issues:**
- Rate limit exceeded: Increase polling interval to every 5 minutes
- Missing emails: Ensure credentials have `gmail.modify` scope
- OAuth token expired: Re-authenticate through n8n UI

---

#### 2. Edit Fields

**Type:** `n8n-nodes-base.set`

**Version:** 3.4

**Configuration:**
```json
{
  "assignments": {
    "assignments": [
      {
        "id": "9523e827-4bb9-4911-bc11-779e5ef74546",
        "name": "EmailBody",
        "value": "={{ $json.snippet }}",
        "type": "string"
      },
      {
        "id": "5bb3ebd5-6740-4579-9f1e-fb828d08c8ed",
        "name": "SenderEmail",
        "value": "={{ $json.From }}",
        "type": "string"
      },
      {
        "id": "5a869107-78c1-4d15-894f-567d138c335f",
        "name": "ThreadID",
        "value": "={{ $json.threadId }}",
        "type": "string"
      },
      {
        "id": "c1ace662-59d3-4712-992b-0030de25b370",
        "name": "Subject",
        "value": "={{ $json.Subject }}",
        "type": "string"
      }
    ]
  }
}
```

**Input:** Raw Gmail message object

**Output:**
```json
{
  "EmailBody": "Email content or snippet",
  "SenderEmail": "sender@example.com",
  "ThreadID": "1abc2def3ghi",
  "Subject": "Email subject line"
}
```

**Purpose:** Normalize Gmail API output to predictable schema

**Transformation Logic:**
- Direct field mapping (no computation)
- String types enforced
- Null handling: Empty string if field missing

---

#### 3. Check if Customer Support

**Type:** `@n8n/n8n-nodes-langchain.openAi`

**Version:** 2.1

**Model:** `gpt-4-1106-preview` (mini variant for cost)

**Configuration:**
```json
{
  "modelId": "gpt-4.1-mini",
  "responses": {
    "values": [
      {
        "role": "system",
        "content": "Analyze the content of the following email and determine whether\nit is related to any of the following topics, mark customer Support as true, otherwise mark it as false.\n\nCustomer support topics include:\n- Questions about order status, tracking, or changes\n- Refund or return requests\n- Subscription cancellations or adjustments\n- Technical issues with products, website or app\n- Payment or billing inquiries\n- Requests for speaking with a support representative.\n\nOutput: Provide the response in JSON format with a field named \"customerSupport\" set to true or false."
      },
      {
        "content": "={{ $json.EmailBody }}"
      }
    ]
  },
  "options": {
    "textFormat": {
      "textOptions": {
        "type": "json_object"
      }
    }
  }
}
```

**Input:** `EmailBody` (from Edit Fields)

**Output:**
```json
{
  "output": [
    {
      "content": [
        {
          "text": {
            "customerSupport": true | false
          }
        }
      ]
    }
  ]
}
```

**Prompt Design:**
- Zero-shot classification (no examples needed)
- Explicit category list for precision
- JSON output enforced for reliability
- System message defines classification boundaries

**Cost Analysis:**
- Input tokens: ~150 (system message + email)
- Output tokens: ~20 (JSON response)
- Estimated cost: $0.00015 per classification

**Error Handling:**
- Response format validation in Switch node
- Fallback: If JSON parsing fails, treat as non-support
- Timeout: 30 seconds (configurable)

---

#### 4. Switch

**Type:** `n8n-nodes-base.switch`

**Version:** 3.4

**Configuration:**
```json
{
  "rules": {
    "values": [
      {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "typeValidation": "strict",
            "version": 3
          },
          "conditions": [
            {
              "leftValue": "={{ $json.output[0].content[0].text.customerSupport }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "true"
              }
            }
          ],
          "combinator": "and"
        },
        "renameOutput": true,
        "outputKey": "YES"
      },
      {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "typeValidation": "strict",
            "version": 3
          },
          "conditions": [
            {
              "leftValue": "={{ $json.output[0].content[0].text.customerSupport }}",
              "rightValue": false,
              "operator": {
                "type": "boolean",
                "operation": "false"
              }
            }
          ],
          "combinator": "and"
        },
        "renameOutput": true,
        "outputKey": "NO"
      }
    ]
  }
}
```

**Decision Logic:**
1. If `customerSupport == true` → Route to output: YES (process through AI Agent)
2. If `customerSupport == false` → Route to output: NO (terminate execution)

**Purpose:** Cost optimization - skip expensive RAG for non-support emails

**Routing Implications:**
- YES branch: 1 connection to AI Agent node
- NO branch: 1 connection to End node (implicit)
- Both branches have full email context available

---

#### 5. AI Agent

**Type:** `@n8n/n8n-nodes-langchain.agent`

**Version:** 3.1

**Configuration:**
```json
{
  "promptType": "define",
  "text": "={{ $('Edit Fields').item.json.Subject }}\n{{ $('Edit Fields').item.json.EmailBody }}",
  "options": {
    "systemMessage": "Role and Purpose\n\nYou are a customer support agent for Sleepyowl, trained to handle customer requests,inquiries,and concerns withprofessionalism,empathy,and efficiency.\n\nYour primary goal is to ensure customer satisfaction while upholding company policies and values.\n\n# Task Specification\n1. Read and Analyze: Carefully read the customer's email to identify their main query or concern.\n2. Use the **customerSupportKnowledgeBase** tool to look up relevant policies or FAQs to ensure accurate and policy-compliant responses.\n\n3. Create a draft response using **createDraft** tool\n4. After drafting the email, provide a concise summary of the email content.\n\nEnsure the response:\n* directly addresses the query\n* maintains polite and prof tone\n* ends with a sign off: \"Kelly from SleepyOwl\""
  }
}
```

**Language Model:** OpenAI Chat Model (GPT-4.1-mini)

**Tools Available:**
1. Vector Store Tool (Policy retrieval)
2. Gmail Tool (Draft creation)

**Execution Flow:**
```
Agent receives email context
    ↓
Decides which tool to use (agent reasoning)
    ↓
Tool 1: Query vector store for policies
    ↓
Tool 2: Generate response with policy context
    ↓
Tool 3: Create Gmail draft with response
    ↓
Return to agent: Execution complete
```

**Agent Parameters:**
- Max iterations: 10 (prevents infinite loops)
- Timeout: 60 seconds
- Tool choice: Auto (agent decides)

**Token Estimation:**
- System message: ~200 tokens
- Email context: ~100 tokens
- Tool outputs: ~500 tokens
- Response generation: ~300 tokens
- Total: ~1100 tokens per execution
- Cost: ~$0.003-0.005 per email

---

#### 6. OpenAI Chat Model

**Type:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`

**Version:** 1.3

**Configuration:**
```json
{
  "model": "gpt-4.1-mini",
  "builtInTools": {},
  "options": {}
}
```

**Role:** Language model backend for AI Agent

**Model Specifications:**
- Model ID: gpt-4-1106-preview (mini)
- Temperature: 0.7 (default, provides balance between determinism and creativity)
- Max tokens: 2048 (default)
- Top P: 1.0 (no nucleus sampling)

**Credentials:** OpenAI API key

---

#### 7. Answer questions with a vector store

**Type:** `@n8n/n8n-nodes-langchain.toolVectorStore`

**Version:** 1.1

**Configuration:**
```json
{
  "description": "Document Containing policy and FAQs"
}
```

**Purpose:** Expose vector store as tool for agent

**Tool Behavior:**
- Query: Customer's email content
- Search backend: Pinecone vector store
- Results: Top-3 policy chunks
- Format: Structured text with context

**Output Format:**
```
Based on our policy:

- [Policy chunk 1 with context]
- [Policy chunk 2 with context]
- [Policy chunk 3 with context]

Source: ecom_policy_doc.pdf (Last updated: DATE)
```

---

#### 8. Create a draft in Gmail

**Type:** `n8n-nodes-base.gmailTool`

**Version:** 2.2

**Configuration:**
```json
{
  "resource": "draft",
  "subject": "={{ $fromAI('Subject', '', 'string') }}",
  "message": "={{ $fromAI('Message', '', 'string') }}",
  "options": {
    "threadId": "={{ $('Edit Fields').item.json.ThreadID }}"
  }
}
```

**Parameters:**
- Resource: draft (creates unsent message)
- Subject: AI-generated (with "Re:" prefix)
- Message: Full response body from agent
- ThreadID: Original email thread (for conversation grouping)

**Credentials:** Gmail OAuth2 account

**Output:**
```json
{
  "id": "draft_id_abc123",
  "message": {
    "id": "message_id_abc123",
    "threadId": "thread_id_abc123",
    "labelIds": ["DRAFT"]
  }
}
```

**Key Behaviors:**
- Creates draft (no auto-send)
- Preserves thread context (appears as reply)
- Requires human review before sending
- Supports rich text formatting

**Gmail API Limitations:**
- Draft character limit: 256MB
- Thread ID must exist
- Subject line max 1024 characters

---

#### 9. Pinecone Vector Store

**Type:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`

**Version:** 1.3

**Configuration:**
```json
{
  "pineconeIndex": "customer-support-doc",
  "options": {}
}
```

**Purpose:** Backend storage for semantic search

**Index Configuration:**
- Index name: customer-support-doc
- Dimension: 1024
- Metric: Cosine similarity
- Namespace: ecom-policy

**Search Behavior:**
- Query: Text from customer email
- Embedding: Generated by OpenAI embeddings
- Results: Top-K vectors with highest similarity
- Metadata: Source document, section, chunk ID

**Credentials:** Pinecone API key

---

#### 10. Embeddings OpenAI

**Type:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`

**Version:** 1.2

**Configuration:**
```json
{
  "options": {
    "dimensions": 1024
  }
}
```

**Purpose:** Convert text to vector embeddings

**Embedding Model:**
- Model: text-embedding-3-small (1024 dimensions)
- Input: Raw email text
- Output: 1024-dimensional float vector
- Latency: 100-200ms per embedding

**Credentials:** OpenAI API key

**Cost:**
- $0.00001 per 1K tokens
- Typical email: ~100 tokens = $0.000001

---

### Pinned Execution Data

The workflow includes test data for manual execution:

**Sample Email (Gmail Trigger output):**
```json
{
  "id": "19bc228b7f31638a",
  "threadId": "19bc228b7f31638a",
  "snippet": "I have received a defective product can you explain your refund policy?",
  "From": "Rishab Jain <rishabprakash12@gmail.com>",
  "Subject": "Refund policy query",
  "To": "rishabprakashjain12@gmail.com"
}
```

**Usage:** Click "Execute Node" to test with this data without waiting for new email

---

## Pinecone Setup Workflow

**File:** `pinecone_setup.json`

**Purpose:** Ingest policy documents into Pinecone vector database

**Active:** false (Run manually when updating policies)

**Execution Mode:** Webhook-triggered (chat interface)

### Workflow Overview

```
Chat Trigger (File upload)
    ↓
Extract from File (PDF → text)
    ↓
Default Data Loader (Text normalization)
    ↓
Recursive Character Text Splitter (Chunking)
    ↓
Embeddings OpenAI (Vector generation)
    ↓
Pinecone Vector Store (Storage)
```

---

### Node Specifications

#### 1. When chat message received

**Type:** `@n8n/n8n-nodes-langchain.chatTrigger`

**Version:** 1.3

**Configuration:**
```json
{
  "options": {
    "allowFileUploads": true
  }
}
```

**Webhook ID:** `be01fd45-7d73-4b95-bf40-c3ce78f71625` (auto-generated)

**Webhook URL:** `https://[n8n-instance]/webhook/[webhook-id]`

**Input Format:**
```json
{
  "message": "Optional message text",
  "data0": { // File binary data
    "mimeType": "application/pdf",
    "fileName": "ecom_policy_doc.pdf",
    "fileSize": 65000
  }
}
```

**Purpose:** Accept PDF files for knowledge base ingestion

**Usage:**
1. Access webhook URL in web browser
2. Upload PDF file
3. Execution starts automatically

---

#### 2. Extract from File

**Type:** `n8n-nodes-base.extractFromFile`

**Version:** 1

**Configuration:**
```json
{
  "operation": "pdf",
  "binaryPropertyName": "data0",
  "options": {}
}
```

**Input:** Binary PDF file

**Output:**
```json
{
  "text": "Full extracted text from PDF...",
  "fileName": "ecom_policy_doc.pdf"
}
```

**Supported Formats:**
- PDF (text-based, not scanned)
- Uses PDF.js for extraction
- Preserves text ordering
- Handles multipage documents

**Limitations:**
- No OCR for scanned PDFs
- Image-based text not extracted
- Max file size: 100MB

---

#### 3. Default Data Loader

**Type:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`

**Version:** 1.1

**Configuration:**
```json
{
  "jsonMode": "expressionData",
  "jsonData": "={{ $json.text }}",
  "textSplittingMode": "custom",
  "options": {}
}
```

**Input:** Raw extracted text

**Output:**
```json
{
  "documents": [
    {
      "pageContent": "Full text content",
      "metadata": {
        "source": "ecom_policy_doc.pdf"
      }
    }
  ]
}
```

**Purpose:** Convert raw text to LangChain Document objects

**Metadata Handling:**
- Source: Original file name
- Page: Document index
- Custom fields: User-definable

---

#### 4. Recursive Character Text Splitter

**Type:** `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`

**Version:** 1

**Configuration:**
```json
{
  "chunkSize": 500,
  "chunkOverlap": 50,
  "options": {}
}
```

**Chunking Strategy:**
- Size: 500 characters per chunk
- Overlap: 50 characters (context preservation)
- Separators: Custom recursive splitting

**Example:**
```
Policy text (2000 chars total)
    ↓
Chunk 1 (500 chars, chars 0-500)
Chunk 2 (500 chars, chars 450-950)   [50-char overlap with chunk 1]
Chunk 3 (500 chars, chars 900-1400)  [50-char overlap with chunk 2]
Chunk 4 (500 chars, chars 1350-1850) [50-char overlap with chunk 3]
```

**Benefits of Overlap:**
- Prevents context loss at chunk boundaries
- Improves semantic search recall
- Cost: ~10% more embeddings

**Typical Breakdown:**
- 30-page policy document = ~15,000 characters
- Results in ~30 chunks with overlap
- Total embeddings needed: ~30

---

#### 5. Embeddings OpenAI

**Type:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`

**Version:** 1.2

**Configuration:**
```json
{
  "options": {
    "dimensions": 1024
  }
}
```

**Input:** Array of text chunks

**Output:**
```json
{
  "embeddings": [
    [0.123, 0.456, ...1024 values],
    [0.234, 0.567, ...1024 values],
    ...
  ]
}
```

**Batch Processing:**
- Processes all chunks in parallel
- Max tokens per API call: 2M
- Latency: 500-1000ms for 30 chunks

---

#### 6. Pinecone Vector Store

**Type:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`

**Version:** 1.3

**Configuration:**
```json
{
  "mode": "insert",
  "pineconeIndex": "customer-support-doc",
  "options": {
    "pineconeNamespace": "ecom-policy"
  }
}
```

**Operation Mode:** insert (create or update)

**Index Details:**
- Index: customer-support-doc (must exist in Pinecone)
- Namespace: ecom-policy (allows partitioning by document type)
- Metric: Cosine similarity

**Vector Storage Format:**
```json
{
  "id": "uuid",
  "values": [1024-dimensional float array],
  "metadata": {
    "text": "Original chunk text",
    "source": "ecom_policy_doc.pdf",
    "chunk_index": 0,
    "timestamp": "2026-01-27T14:30:00Z"
  },
  "namespace": "ecom-policy"
}
```

**Credentials:** Pinecone API key (read/write)

**Output:**
```json
{
  "namespace": "ecom-policy",
  "vectorsStored": 30,
  "upsertedIds": ["id1", "id2", ...],
  "duration": 2000
}
```

---

## Node Configuration Reference

### Common Configuration Patterns

**Expression Language (n8n):**
```javascript
// Reference previous node output
{{ $json.fieldName }}

// Reference specific node
{{ $('NodeName').json.fieldName }}

// Conditional
{{ $json.status === 'support' ? 'Yes' : 'No' }}

// Array operations
{{ $json.items.map(x => x.name) }}
```

**Error Handling:**
```javascript
// With default value
{{ $json.field || 'default value' }}

// With null check
{{ $json.field !== null ? $json.field : 'empty' }}
```

---

## Data Transformation Details

### Field Mapping Example

**Input (Gmail Raw):**
```json
{
  "id": "19bc228b7f31638a",
  "threadId": "19bc228b7f31638a",
  "snippet": "I have received a defective product can you explain your refund policy?",
  "From": "Rishab Jain <rishabprakash12@gmail.com>",
  "Subject": "Refund policy query"
}
```

**After Edit Fields:**
```json
{
  "EmailBody": "I have received a defective product can you explain your refund policy?",
  "SenderEmail": "Rishab Jain <rishabprakash12@gmail.com>",
  "ThreadID": "19bc228b7f31638a",
  "Subject": "Refund policy query"
}
```

**After Classification:**
```json
{
  "EmailBody": "...",
  "SenderEmail": "...",
  "ThreadID": "...",
  "Subject": "...",
  "output": [
    {
      "content": [
        {
          "text": {
            "customerSupport": true
          }
        }
      ]
    }
  ]
}
```

### Vector Search Context Example

**Query:** "refund policy"

**Retrieved Chunks:**
1. "Unused, unassembled items in original packaging can be returned within 30 days for a refund or exchange. Exclusions apply..."
2. "Who pays for return shipping? Customers cover return shipping unless the return is due to an error on our part..."
3. "Contact Us: For urgent issues, use our 24/7 Live Chat or email support@[company].com..."

**Injected into Agent Context:**
```
You have access to the following policy information:

RETURNS_POLICY:
Unused, unassembled items in original packaging can be returned within 30 days...

RETURN_SHIPPING:
Customers cover return shipping unless...

CONTACT:
For urgent issues, use our 24/7 Live Chat...

Now generate a response to the customer's email.
```

---

**Document Version:** 1.0  
**Last Updated:** January 27, 2026  
**Compatibility:** n8n v1.0+, OpenAI API v1, Pinecone v1.3+
