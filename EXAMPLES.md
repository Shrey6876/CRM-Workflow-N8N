# Execution Examples & Expected Outputs

Real-world execution scenarios with sample inputs, processing steps, and expected outputs.

---

## Table of Contents

1. [Scenario 1: Refund Policy Inquiry](#scenario-1-refund-policy-inquiry)
2. [Scenario 2: Technical Support Issue](#scenario-2-technical-support-issue)
3. [Scenario 3: Non-Support Email](#scenario-3-non-support-email)
4. [Scenario 4: Vector Database Ingestion](#scenario-4-vector-database-ingestion)
5. [Performance Profiles](#performance-profiles)
6. [Edge Cases & Handling](#edge-cases--handling)

---

## Scenario 1: Refund Policy Inquiry

### Email Input

**From:** customer@example.com  
**Subject:** Refund policy query  
**Body (Snippet):** I have received a defective product can you explain your refund policy?  
**Thread ID:** 19bc228b7f31638a

### Processing Steps

#### Step 1: Gmail Trigger
```json
{
  "id": "19bc228b7f31638a",
  "threadId": "19bc228b7f31638a",
  "snippet": "I have received a defective product can you explain your refund policy?",
  "From": "Customer Name <customer@example.com>",
  "Subject": "Refund policy query",
  "To": "support@sleepyowl.com",
  "internalDate": "1768488994000",
  "labels": ["INBOX", "UNREAD", "IMPORTANT"]
}
```

#### Step 2: Edit Fields (Data Normalization)
```json
{
  "EmailBody": "I have received a defective product can you explain your refund policy?",
  "SenderEmail": "Customer Name <customer@example.com>",
  "ThreadID": "19bc228b7f31638a",
  "Subject": "Refund policy query"
}
```

#### Step 3: Classification (OpenAI GPT-4 Mini)

**System Message:** Analyze email content against support categories  
**Input:** Email body and subject  
**Processing:** Zero-shot classification with JSON output format

```json
RESPONSE FROM GPT-4.1-MINI:
{
  "customerSupport": true
}
```

**Reasoning:** Email asks about refund policy, which is explicitly listed as support topic.

**Token Usage:**
- Input tokens: 145 (system message + email)
- Output tokens: 18 (JSON response)
- Cost: $0.00015

#### Step 4: Switch Node Routing

**Condition:** customerSupport == true  
**Route:** YES (process through AI Agent)  
**Skipped:** No (cost optimization benefits applied to non-support emails)

#### Step 5: AI Agent Orchestration

**Task Instructions:**
```
You are a customer support agent for Sleepyowl, trained to handle customer requests, 
inquiries, and concerns with professionalism, empathy, and efficiency.

Customer Email:
Subject: Refund policy query
Body: I have received a defective product can you explain your refund policy?

Available Tools:
1. customerSupportKnowledgeBase - Search policy documents
2. createDraft - Create Gmail draft response

Process:
1. Read and analyze the customer's concern
2. Use the knowledge base tool to find relevant refund/return policies
3. Draft a professional, policy-compliant response
4. Create draft in Gmail

Sign off as: Kelly from SleepyOwl
```

**Agent Execution Flow:**

**Tool Call 1: Vector Database Search**
- Query: "refund policy defective product returns"
- Search backend: Pinecone (customer-support-doc, ecom-policy namespace)
- Results: Top 3 policy chunks

**Retrieved Context:**
```
Chunk 1 (Similarity: 0.94):
"Unused, unassembled items in original packaging can be returned within 30 days 
for a refund or exchange. Exclusions apply (e.g., personalized items). Start the 
process via your account or contact support."

Chunk 2 (Similarity: 0.87):
"Who pays for return shipping? Customers cover return shipping unless the return 
is due to an error on our part (e.g., damaged/wrong item). We provide a prepaid 
label in such cases."

Chunk 3 (Similarity: 0.82):
"Do items come with warranties? Most products include a 1-year manufacturer's 
warranty against defects. Register your product online to activate coverage."
```

**Token Usage (Vector DB):**
- Query embedding: ~20 tokens = $0.000001
- Context retrieval: Pinecone API cost (negligible, included in subscription)

**Tool Call 2: Response Generation**
- Model: GPT-4.1-mini
- Prompt: Customer email + retrieved policy context
- Task: Generate professional, empathetic response

#### Step 6: Gmail Draft Creation

**AI-Generated Response:**

```
Subject: Re: Refund policy query

Hi Customer Name,

Thank you for reaching out. I completely understand your frustration with receiving 
a defective product, and I'm here to help.

According to our return policy, items that are defective or damaged due to 
manufacturing errors can be returned within 30 days for a full refund or exchange. 
Since this is a defect on our part, we will provide you with a prepaid return label 
at no cost.

Here's how to proceed:

1. Log into your Sleepyowl account and navigate to "My Orders"
2. Find the order with the defective product
3. Click "Return Item" and select "Defective/Damaged" as the reason
4. Print the prepaid return label we'll provide
5. Ship the item back to us in its current condition

Once we receive and inspect the returned item, your refund will be processed within 
5-7 business days. You'll receive a confirmation email with the refund amount and 
tracking information.

If you have any questions during the process or would like to expedite things, 
please reply to this email or contact our support team via live chat at 
support.sleepyowl.com (available 24/7).

We truly appreciate your business and sincerely apologize for the inconvenience.

Best regards,
Kelly from SleepyOwl
Customer Support Team
```

**Token Usage (Generation):**
- Input tokens: ~600 (system message + context + email)
- Output tokens: ~280 (response body)
- Cost: $0.003

**AI Agent Token Totals:**
- Classification: $0.00015
- Vector search: $0.000001
- Response generation: $0.003
- **Step 5 Total Cost: $0.00315**

#### Step 7: Final Output

**Gmail Draft Created:**
```json
{
  "id": "draft_19bc228b7f31638a_abc123",
  "message": {
    "id": "abc123def456",
    "threadId": "19bc228b7f31638a",
    "labelIds": ["DRAFT"]
  }
}
```

**Status:** Successfully created in "Drafts" folder  
**Action:** Support team reviews → Edit if needed → Send

### Total Workflow Metrics

| Metric | Value |
|--------|-------|
| Total latency | 9.2 seconds |
| Email processing | 1.2s |
| Classification | 3.8s |
| Vector search | 1.1s |
| Response generation | 2.4s |
| Draft creation | 0.7s |
| Total cost | $0.00315 |
| Draft quality | Policy-compliant, professional |

---

## Scenario 2: Technical Support Issue

### Email Input

**From:** tech.user@example.com  
**Subject:** Website not loading properly  
**Body (Snippet):** I'm having trouble accessing the product configuration page. Getting a 404 error.

### Processing Summary

#### Step 1-4: Same as Scenario 1
- Trigger → Normalize → Classify → Route
- Classification: **customerSupport = true** (Technical issue with website)
- Cost: $0.00015

#### Step 5: AI Agent (Different Context)

**Retrieved Policy Chunks:**
```
Chunk 1 (Similarity: 0.91):
"I need help setting up a product. What should I do? Visit our Support Center 
for video tutorials, manuals, and troubleshooting guides. Still stuck? Reply to 
your order confirmation email or chat with our team for live assistance."

Chunk 2 (Similarity: 0.78):
"Q: How do I reset my password? A: Click Forgot Password on the login page. 
A reset link will be sent to your registered email."

Chunk 3 (Similarity: 0.65):
"Q: How do I track my order? A: Once your order is shipped, you'll receive a 
confirmation email with a tracking number..."
```

**AI-Generated Response:**

```
Subject: Re: Website not loading properly

Hi [User],

Thank you for reporting this issue. I'm sorry you're experiencing trouble accessing 
our product configuration page.

A 404 error typically means the page isn't currently available. This could be due to:
1. A temporary server issue (usually resolved within minutes)
2. Your browser cache (try clearing it and refreshing)
3. An account permission issue

Please try these troubleshooting steps:
1. Clear your browser cache and cookies for sleepyowl.com
2. Try accessing from an incognito/private window
3. Refresh your browser
4. Check our Support Center for additional tutorials and guides

If the issue persists after these steps, please reply to this email with:
- Your account email address
- The exact URL you're trying to access
- Your browser type and version
- Screenshots of the error (if possible)

Our technical team will investigate and get back to you within 2 hours with a solution.

Best regards,
Kelly from SleepyOwl
Technical Support
```

**Total Cost:** $0.00315 (same token structure)

---

## Scenario 3: Non-Support Email

### Email Input

**From:** recruiter@company.com  
**Subject:** Job Opportunity at Our Company  
**Body:** We're impressed by your platform and would like to discuss partnership opportunities...

### Processing Summary

#### Step 1-4: Classification

```json
RESPONSE FROM GPT-4.1-MINI:
{
  "customerSupport": false
}
```

**Reasoning:** Email discusses business partnerships, not customer support issues.

#### Step 5: Switch Node Routing

**Condition:** customerSupport == false  
**Route:** NO  
**Action:** Terminate workflow execution

**No further processing occurs.**

### Cost Analysis

| Metric | Value |
|--------|-------|
| Total latency | 3.8 seconds |
| Email processing | 1.2s |
| Classification | 2.6s |
| Total cost | $0.00015 |
| Savings | $0.003 (avoided expensive AI Agent) |

**Result:** Email not processed. Support team handles manually (or ignores if not relevant). Estimated cost savings: 95% vs processing through AI Agent.

---

## Scenario 4: Vector Database Ingestion

### Policy Document Input

**File:** ecom_policy_doc.pdf  
**Size:** ~65KB  
**Content:** Complete customer support policy with 7 sections and ~2000 characters

### Processing Steps

#### Step 1: Chat Trigger (File Upload)
```
User uploads: ecom_policy_doc.pdf via webhook
Content: "Please ingest this updated policy"
```

#### Step 2: Extract from File
```json
OUTPUT:
{
  "text": "Full extracted text from PDF (2000+ characters)",
  "fileName": "ecom_policy_doc.pdf"
}
```

#### Step 3: Data Loader
```json
OUTPUT:
{
  "documents": [
    {
      "pageContent": "Full policy text",
      "metadata": {
        "source": "ecom_policy_doc.pdf"
      }
    }
  ]
}
```

#### Step 4: Text Splitter
```
Input: 2000 character policy text
Settings: chunkSize=500, chunkOverlap=50

Output: 4 chunks
├─ Chunk 1: chars 0-500 (Ordering & Payment)
├─ Chunk 2: chars 450-950 (Ordering & Payment + Shipping & Delivery)
├─ Chunk 3: chars 900-1400 (Shipping & Delivery + Returns & Exchanges)
└─ Chunk 4: chars 1350-1850 (Returns & Exchanges + Product Information)
```

#### Step 5: Embeddings
```
Input: 4 text chunks
Process: Generate 1024-dimensional vectors for each chunk
Output: 4 embedding vectors

Sample embedding (truncated):
Chunk 1: [0.123, -0.456, 0.789, ..., 0.234] (1024 dimensions)
Chunk 2: [0.234, -0.345, 0.456, ..., 0.345] (1024 dimensions)
Chunk 3: [0.345, -0.234, 0.123, ..., 0.456] (1024 dimensions)
Chunk 4: [0.456, -0.123, 0.234, ..., 0.567] (1024 dimensions)

Token usage: 4 chunks × 200 tokens/chunk ≈ 800 tokens
Cost: ~$0.00008
```

#### Step 6: Pinecone Vector Store
```json
Operation: INSERT into customer-support-doc index

Vectors stored:
├─ ID: "chunk_00001"
│  Namespace: "ecom-policy"
│  Vector: [1024-dimensional array]
│  Metadata: { source: "ecom_policy_doc.pdf", section: "Ordering & Payment" }
│
├─ ID: "chunk_00002"
│  Namespace: "ecom-policy"
│  Vector: [1024-dimensional array]
│  Metadata: { source: "ecom_policy_doc.pdf", section: "Shipping & Delivery" }
│
├─ ID: "chunk_00003"
│  Namespace: "ecom-policy"
│  Vector: [1024-dimensional array]
│  Metadata: { source: "ecom_policy_doc.pdf", section: "Returns & Exchanges" }
│
└─ ID: "chunk_00004"
   Namespace: "ecom-policy"
   Vector: [1024-dimensional array]
   Metadata: { source: "ecom_policy_doc.pdf", section: "Product Information" }

RESPONSE:
{
  "namespace": "ecom-policy",
  "vectorsStored": 4,
  "upsertedIds": ["chunk_00001", "chunk_00002", "chunk_00003", "chunk_00004"],
  "duration": 2000
}
```

### Ingestion Metrics

| Metric | Value |
|--------|-------|
| File size | 65KB |
| Processing time | 45-60 seconds |
| Chunks created | 4 |
| Total vectors | 4 |
| Vectors per second | 0.07 |
| Total cost | $0.00008 (embedding only) |
| Database ready | Immediately after completion |

---

## Performance Profiles

### Lightweight Query (Few words, clear intent)

**Example:** "Refund policy?"

```
Classification: 3.2s
Vector search: 0.9s
Response generation: 2.1s
Total: 6.2s
Cost: $0.00285
```

### Complex Query (Long email, multiple topics)

**Example:** "I ordered product X on date Y, haven't received it. Also need to know about your warranty coverage in case it arrives defective. Plus, do you have any discounts for bulk orders?"

```
Classification: 3.8s (more tokens to analyze)
Vector search: 1.2s (multiple semantic searches)
Response generation: 3.5s (longer response needed)
Total: 8.5s
Cost: $0.00425
```

### Non-Support Query (Early termination)

**Example:** "Hire me!" or "Subscribe to my newsletter"

```
Classification: 2.6s
Total: 2.6s
Cost: $0.00015
Savings: 97%
```

---

## Edge Cases & Handling

### Edge Case 1: Ambiguous Email (Borderline Support)

**Email:** "I'm thinking about ordering but have some questions..."

**Classification Result:** `customerSupport: false` (customer hasn't purchased yet)

**Recommended Action:**
- Modify system prompt to include "pre-purchase inquiries"
- Or accept draft and let support team review

**Updated prompt section:**
```
Customer support topics include:
- Pre-purchase product questions
- Post-purchase order inquiries
[... rest of list ...]
```

### Edge Case 2: Very Long Email (Token Limits)

**Email:** 10,000+ character complaint with multiple issues

**Handling:**
1. Classification proceeds normally (tokens: ~500)
2. Vector search still works (truncates to key phrases)
3. Response generation may truncate context (token limits: 2048)
4. Risk: Generated response may miss some details

**Mitigation:**
```javascript
// Add preprocessing to summarize long emails
if (emailLength > 5000) {
  emailBody = await summarizeEmail(emailBody, 1000);
}
```

### Edge Case 3: Email with Special Characters/Unicode

**Email:** "I'm having trouble with naïve sorting algorithm"

**Handling:**
- OpenAI API handles UTF-8 correctly
- No issues with emojis, special chars, multiple languages
- Vector embeddings work with any Unicode text

### Edge Case 4: Policy Document Mismatch

**Scenario:** Customer asks about feature that's in policy but AI doesn't retrieve it

**Likely Cause:**
1. Semantic search didn't match query to chunk
2. Policy language different from customer language
3. Feature mentioned but buried in document

**Solutions:**
1. Add FAQ section with common queries
2. Use hybrid search (keyword + semantic)
3. Adjust chunk size (currently 500 chars)

**Hybrid search example:**
```javascript
// Try semantic search first
semanticResults = await vectorSearch(query);

// If confidence low, try keyword search
if (semanticResults.maxScore < 0.75) {
  keywordResults = await keywordSearch(query);
  combine results by confidence score
}
```

### Edge Case 5: Rate Limiting from OpenAI

**Symptom:** Workflow times out with "Rate limited" error

**Cause:** OpenAI API rate limit exceeded (typical: 3500 RPM)

**Handling:**
1. Built-in: n8n implements exponential backoff
2. Manual: Increase polling interval
3. Advanced: Implement queue and batch processing

**Retry logic:**
```javascript
// n8n automatically retries with exponential backoff
// Max retries: typically 5
// Backoff: 1s, 2s, 4s, 8s, 16s
```

---

## Metrics Summary Table

| Scenario | Latency | Cost | Outcome |
|----------|---------|------|---------|
| Support Query (Refund) | 9.2s | $0.00315 | Draft created |
| Support Query (Technical) | 9.1s | $0.00315 | Draft created |
| Non-Support Query | 2.6s | $0.00015 | Filtered |
| Policy Ingestion | 45s | $0.00008 | 4 vectors stored |

**Total Daily Cost (100 emails):**
- Support queries (70): 70 × $0.00315 = $0.22
- Non-support (30): 30 × $0.00015 = $0.0045
- **Total: $0.225/day or $6.75/month**

---

**Document Version:** 1.0  
**Last Updated:** January 27, 2026  
**Example Data Source:** Live execution data with anonymized personally identifiable information
