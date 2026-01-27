# Setup & Deployment Guide

Complete instructions for configuring, testing, and deploying both workflows to production.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Credential Setup](#credential-setup)
3. [Vector Database Initialization](#vector-database-initialization)
4. [Workflow Import](#workflow-import)
5. [Testing & Validation](#testing--validation)
6. [Production Deployment](#production-deployment)
7. [Troubleshooting](#troubleshooting)
8. [Monitoring & Maintenance](#monitoring--maintenance)

---

## Prerequisites

### System Requirements

- n8n instance (Cloud or Self-hosted)
- Active internet connection
- Storage: ~2GB for vector database cache

### Required Accounts & API Keys

| Service | Requirement | Cost | Setup Time |
|---------|-------------|------|------------|
| OpenAI | API key, GPT-4 mini access | ~$0.02-0.05/email | 5 minutes |
| Pinecone | Starter (free) tier, API key | Free, upgradable | 10 minutes |
| Gmail | Google account, OAuth credentials | Free | 15 minutes |
| n8n | Cloud or self-hosted instance | Free Cloud, or self-hosted | Already set up |

### Skill Level

- Familiarity with n8n node-based workflows
- Basic understanding of API authentication
- JSON configuration editing capability

---

## Credential Setup

### Step 1: OpenAI API Key

**Obtain API Key:**
1. Visit https://platform.openai.com/account/api-keys
2. Create new API key (or use existing)
3. Copy key to secure location

**Grant API Access:**
- Ensure account has GPT-4 mini access (included with most OpenAI accounts)
- Set spending limits in https://platform.openai.com/account/billing/limits
- Recommended: $10/month limit for safe experimentation

**Add to n8n:**
1. Go to n8n → Credentials
2. Create new → OpenAI API
3. Paste API key
4. Test connection (should succeed immediately)

**Verification:**
```bash
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer YOUR_API_KEY" | grep gpt-4
```

---

### Step 2: Pinecone Vector Database

**Create Pinecone Account:**
1. Visit https://www.pinecone.io/
2. Sign up (free tier available)
3. Create new organization
4. Create index: 
   - Name: `customer-support-doc`
   - Dimension: 1024
   - Metric: Cosine

**Obtain API Key:**
1. Dashboard → API Keys
2. Create new API key (or use default)
3. Copy API key and environment (e.g., "us-west1-gcp")

**Add to n8n:**
1. Go to n8n → Credentials
2. Create new → Pinecone API
3. Paste API key
4. Enter environment (us-west1-gcp, us-east1-gcp, etc.)
5. Test connection

**Verification:**
```bash
curl -i -X GET https://api.pinecone.io/indexes \
  -H "Api-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json"
```

---

### Step 3: Gmail OAuth2

**Create Google Cloud Project:**
1. Visit https://console.cloud.google.com/
2. Create new project: "n8n Automation"
3. Enable APIs:
   - Gmail API
   - Google Drive API (for file uploads)

**Create OAuth Credentials:**
1. APIs & Services → Credentials
2. Create OAuth 2.0 Client ID
3. Application type: Web application
4. Authorized redirect URIs:
   - `https://your-n8n-instance.com/rest/oauth2/callback`
   - `http://localhost:5678/rest/oauth2/callback` (if self-hosted)

**Add to n8n:**
1. Go to n8n → Credentials
2. Create new → Gmail OAuth2
3. Paste Client ID and Client Secret
4. Paste authorization URI and other OAuth URLs
5. Click "Connect" to authorize (will redirect to Gmail login)

**Scopes Required:**
- `https://www.googleapis.com/auth/gmail.modify` (read/write emails)
- `https://www.googleapis.com/auth/gmail.readonly` (read only, more secure if available)

**Verification:**
- Authorized successfully if n8n shows green checkmark
- Can manually trigger workflow to verify email access

---

### Step 4: Verify All Credentials

| Credential | Test | Expected Result |
|-----------|------|-----------------|
| OpenAI API | List models | Returns GPT-4 models |
| Pinecone API | List indexes | Returns customer-support-doc index |
| Gmail OAuth | Read inbox | Returns list of emails |

---

## Vector Database Initialization

### Prepare Policy Document

**File:** `ecom_policy_doc.pdf` (provided)

**Document Checklist:**
- [ ] PDF is text-based (not scanned/image)
- [ ] Content is searchable
- [ ] File size < 100MB
- [ ] All policy sections included

**If document needs updating:**
1. Edit policy document
2. Export as PDF
3. Ensure text layer is preserved (not image-only)

### Initialize Pinecone Index

**Option A: Using Provided Workflow (Recommended)**

1. Import `pinecone_setup.json` to n8n
2. Open workflow in editor
3. Enable workflow (toggle switch)
4. Access webhook URL:
   ```
   https://[n8n-instance]/webhook/be01fd45-7d73-4b95-bf40-c3ce78f71625
   ```
5. Upload `ecom_policy_doc.pdf`
6. Wait for execution to complete (30-60 seconds)
7. Check Pinecone dashboard: Should show ~30 vectors in customer-support-doc

**Option B: Using Pinecone Web UI**

1. Visit Pinecone dashboard
2. Index → Data
3. Upload vectors manually (requires formatting as JSONL)
4. More complex, not recommended for initial setup

### Verify Vector Storage

**Check vector count:**
```bash
curl -i -X GET \
  https://api.pinecone.io/indexes/customer-support-doc \
  -H "Api-Key: YOUR_API_KEY"
```

**Expected response includes:**
```json
{
  "name": "customer-support-doc",
  "dimension": 1024,
  "vectorCount": 30,
  "namespaces": {
    "ecom-policy": { "vectorCount": 30 }
  }
}
```

---

## Workflow Import

### Import Customer Support Agent Workflow

**Step 1: Access n8n**
1. Open n8n instance
2. Workflows → New Workflow
3. Menu (top-left) → Import Workflow

**Step 2: Load Workflow JSON**
1. Click "Select File"
2. Choose `Customer-Support-Agent.json`
3. Confirm import

**Step 3: Update Credentials**

The workflow references credentials by ID. After import:

1. **Gmail Trigger node:**
   - Click node → Change credentials
   - Select your configured Gmail OAuth2 credential
   - Save

2. **OpenAI nodes (3 total):**
   - Find each node: "Check if Customer Support", "OpenAI Chat Model", "OpenAI Chat Model1"
   - Click each → Change credentials
   - Select your configured OpenAI API credential
   - Save

3. **Pinecone nodes (2 total):**
   - Find: "Pinecone Vector Store" and related nodes
   - Click each → Change credentials
   - Select your configured Pinecone API credential
   - Save

**Step 4: Verify All Connections**
- All nodes should have green checkmarks (no credential errors)
- No warning icons visible

### Import Pinecone Setup Workflow

Repeat same process for `pinecone_setup.json`:
1. Menu → Import Workflow
2. Select `pinecone_setup.json`
3. Update OpenAI and Pinecone credentials
4. Save

---

## Testing & Validation

### Test 1: Manual Classification (No Email Needed)

1. Open Customer Support Agent workflow
2. Click "Gmail Trigger" node
3. Click "Test" button
4. Use pinned data (Refund policy query)
5. Check "Edit Fields" output → Should show extracted fields
6. Check "Check if Customer Support" output → Should show `customerSupport: true`

**Expected output:**
```json
{
  "customerSupport": true
}
```

### Test 2: Full Workflow Execution

**Option A: Send Real Email (Recommended)**
1. Send email to your configured Gmail inbox from another account
2. Subject: "Refund policy inquiry"
3. Body: "I received a defective product. What is your refund policy?"
4. Wait up to 2 minutes for workflow to trigger
5. Check Gmail drafts folder
6. Should see draft response with policy information

**Option B: Trigger Manually**
1. Click "Gmail Trigger" node
2. Modify pinned data with your email
3. Click "Test with Pinned Data"
4. Follow execution through all nodes
5. Check final output in "Create a draft in Gmail" node

### Test 3: Vector Database Search

1. Open Pinecone Setup workflow
2. Manually trigger with test email
3. Search for: "return policy"
4. Check Pinecone dashboard
5. Should show vectors added to ecom-policy namespace

---

## Production Deployment

### Pre-Deployment Checklist

- [ ] All credentials configured and tested
- [ ] Vector database initialized with policy document
- [ ] Both workflows imported
- [ ] Manual tests passed
- [ ] Error logging configured (optional but recommended)
- [ ] Slack notifications enabled (for failures)
- [ ] Gmail draft folder monitored daily

### Enable Workflows

1. Open Customer Support Agent workflow
2. Click toggle "Active" (top-right)
3. Confirm: Workflow will start polling for emails
4. Repeat for Pinecone Setup if using automated ingestion

### Monitor Initial Execution

**First hour checklist:**
- [ ] Monitor workflow executions in Dashboard
- [ ] Check Gmail drafts folder
- [ ] Verify no error logs
- [ ] Test with manual email

**First day checklist:**
- [ ] Review draft quality
- [ ] Measure response time (8-15 seconds expected)
- [ ] Check OpenAI API usage
- [ ] Verify all integrations still connected

### Configure Alerts

**Set up failure notifications (optional):**

1. Open workflow settings
2. Workflows → Settings → Notifications
3. Enable: "Workflow execution failed"
4. Notification method: Email or Slack
5. Recipients: Support team

**Example Slack integration:**
```
Workflow failed: Customer Support Agent
Reason: Rate limit exceeded
Time: 2026-01-27 15:30:00 UTC
Retry in: 5 minutes
```

---

## Troubleshooting

### Workflow Won't Trigger

**Symptom:** Gmail Trigger shows "No emails to process" despite unread emails in inbox

**Diagnosis:**
1. Check Gmail OAuth token expiration
2. Verify filter settings (INBOX, UNREAD labels)
3. Check Gmail API rate limits

**Solutions:**
1. Re-authenticate Gmail in credentials
2. Manual trigger via "Test with Pinned Data"
3. Increase polling interval to avoid rate limits

---

### Classification Always Returns false

**Symptom:** All emails classified as non-support

**Diagnosis:**
1. Check OpenAI prompt (may be too restrictive)
2. Verify GPT-4-mini access
3. Check API response format

**Solutions:**
1. Update system message in "Check if Customer Support" node
2. Test with simple prompt: "Is this a support request? Answer: true/false"
3. Add logging node to inspect GPT response

---

### Vector Search Returns No Results

**Symptom:** AI Agent generates generic responses (not policy-based)

**Diagnosis:**
1. Vector database not initialized
2. Wrong namespace (ecom-policy)
3. Email queries don't match policy content

**Solutions:**
1. Run Pinecone Setup workflow again
2. Check namespace in Pinecone Vector Store node
3. Re-ingest with broader policy document

**Debug Query:**
```bash
# Test vector search directly
curl -i -X POST \
  https://api.pinecone.io/query \
  -H "Api-Key: YOUR_API_KEY" \
  -d '{
    "vector": [0.1, 0.2, ...1024 dims],
    "topK": 3,
    "namespace": "ecom-policy"
  }'
```

---

### Gmail Draft Not Created

**Symptom:** Workflow runs successfully but no draft appears in Gmail

**Diagnosis:**
1. Check Gmail credentials have correct scopes
2. Verify threadId is valid
3. Check n8n execution logs

**Solutions:**
1. Re-authorize Gmail with correct scopes (including `gmail.modify`)
2. Test with fresh email (new threadId)
3. Add debugging node: Log item before draft creation

---

### High Cost (>$0.10 per email)

**Symptom:** OpenAI costs exceed budget

**Diagnosis:**
1. Switch is not routing non-support emails (costs $0.02 each)
2. Long emails increase token count
3. Multiple API calls happening

**Solutions:**
1. Verify Switch node conditions are correct
2. Summarize long emails before classification
3. Cache frequently asked questions to reduce agent calls

**Cost breakdown:**
- Classification: $0.00015 per email
- Vector search: $0.00001 (embedding)
- Response generation: $0.002-0.005
- **Total: $0.002-0.006 (non-support: $0.00015)**

---

### Slow Response Time (>30 seconds)

**Symptom:** Emails take >30s to generate drafts

**Diagnosis:**
1. API latency from OpenAI/Pinecone
2. Network connectivity issues
3. n8n resource constraints

**Solutions:**
1. Monitor API latency independently
2. Check n8n system resources
3. Increase polling interval if processing falls behind

---

## Monitoring & Maintenance

### Daily Tasks (5 minutes)

- Review drafted emails in Gmail Drafts folder
- Check for any failed executions in n8n Dashboard
- Spot-check 2-3 responses for quality

### Weekly Tasks (15 minutes)

1. Review metrics:
   - Total emails processed
   - Classification accuracy
   - Average response time
   - API cost spent

2. Check API quotas:
   - OpenAI: Usage % of monthly limit
   - Gmail: API quota (should be <15 req/sec)
   - Pinecone: Vector storage % of tier

3. Update policy document if needed:
   - Export updated policy as PDF
   - Run Pinecone Setup workflow
   - Verify vectors added

---

### Monthly Tasks (30 minutes)

1. **Cost Analysis**
   - Review OpenAI invoice
   - Calculate per-email cost
   - Optimize if >$0.01 per email

2. **Performance Review**
   - Accuracy rate (% of responses useful)
   - Response time trends
   - Error rate

3. **Policy Update**
   - Confirm policy document is current
   - Add new FAQs if not captured
   - Archive old/obsolete policies

4. **Credential Rotation**
   - Rotate API keys (optional but recommended)
   - Update n8n credentials
   - Test all workflows still function

---

### Logging Setup (Optional but Recommended)

**Add debug node after each layer:**

```json
{
  "type": "n8n-nodes-base.logWriter",
  "position": [x, y+100],
  "parameters": {
    "message": "= Layer: [NAME], Output: {{ JSON.stringify($json) }}"
  }
}
```

**Monitor logs via:**
- n8n Logs tab
- n8n Cloud logging dashboard
- Or export to Slack/email

---

### Performance Optimization

**If processing >100 emails/day:**

1. Deploy multiple n8n instances
2. Load balance Gmail trigger across instances
3. Use message queue for email buffering
4. Implement Redis caching for vector search results

**Code snippet for distributed architecture:**
```
Load Balancer
    ├─ n8n Instance 1 (Emails 1-50)
    ├─ n8n Instance 2 (Emails 51-100)
    └─ n8n Instance 3 (Emails 101-150)
         ↓
    Shared Redis Cache (Recent vectors)
         ↓
    Pinecone (Persistent storage)
```

---

### Backup & Recovery

**Backup workflows:**
```bash
# Export workflow JSON (keep in git)
curl -X GET \
  https://[n8n-instance]/rest/workflows/[workflow-id] \
  -H "Authorization: Bearer [api-key]"
```

**Backup vector database:**
```bash
# Export from Pinecone API or use UI
curl -i -X POST \
  https://api.pinecone.io/export \
  -H "Api-Key: YOUR_API_KEY"
```

**Restore procedure:**
1. Re-import workflow JSON
2. Update credentials
3. Re-ingest policy document if needed

---

**Document Version:** 1.0  
**Last Updated:** January 27, 2026  
**Maintenance Schedule:** Weekly review recommended
