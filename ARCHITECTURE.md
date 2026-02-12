# Architecture and Design Patterns

This document describes the recurring patterns, conventions, and architectural decisions across all systems in this portfolio.

## The 3-Layer Architecture

Every system follows a separation of concerns model designed for reliability:

### Layer 1: Directive (What to do)
Standard operating procedures written in Markdown. They define objectives, inputs, tools to use, outputs, and edge cases. Think of them as instructions you would hand to a competent mid-level employee.

### Layer 2: Orchestration (Decision-making)
n8n workflows handle intelligent routing: reading directives, calling execution tools in the right order, handling errors, and updating processes with what they learn. The orchestration layer is the glue between intent and execution.

### Layer 3: Execution (Doing the work)
Deterministic scripts and API calls that handle the actual data processing. These are reliable, testable, and fast. The key insight: push complexity into deterministic code so the orchestration layer focuses only on decision-making.

**Why this matters:**
If you do everything in one layer, errors compound. 90% accuracy per step means roughly 59% success over 5 steps. Separating concerns keeps each layer operating at its strengths.

---

## Core Pattern: Discovery to Action Pipeline

```
[Source] --> [Normalize + Dedupe] --> [Pre-Filter] --> [Enrich] --> [Score] --> [Deliver]
```

### Source
Multiple data inputs are aggregated into a single stream. Examples:
- Job Discovery: Career site feed + LinkedIn scraper + ATS boards
- Restaurant Leads: ToastTab directory + Google Maps + Google Search
- LinkedIn Prospecting: Profile search + engagement monitoring + company scraping

### Normalize and Deduplicate
Raw data is messy. Before anything else:
1. Normalize URLs (strip tracking params, force consistent format)
2. Clean company names (remove suffixes like Inc., LLC)
3. Three-layer dedup: URL match, Title+Company match, Description similarity
4. This prevents wasting expensive API calls on duplicates

### Pre-Filter (Cheap Checks First)
Apply deterministic rules before AI scoring:
- Title keyword matching (inclusion and exclusion lists)
- Location/remote status verification
- Company size or industry checks
- This typically reduces the dataset by 40-60% before any paid API call

### Enrich
Layer on additional context:
- Web scraping for company data, contact info, tech stack
- API calls for email discovery, social profiles, business details
- AI extraction for unstructured data (parsing emails from web pages, extracting structured data from job descriptions)

### Score
Two-phase scoring approach:
1. **Deterministic scoring** (weighted signals with numeric values)
2. **AI scoring** (Claude analyzes the full context for nuanced fit assessment)

Results are stored with both the numeric score and the AI reasoning, enabling human review of borderline cases.

### Deliver
Results go where they are actionable:
- Google Sheets for team collaboration and manual review
- Email digests for daily workflow integration
- PostgreSQL for persistence, querying, and pipeline tracking
- Slack or webhook notifications for real-time alerts

---

## Database Design Patterns

### Upsert with Conflict Resolution
Every INSERT uses ON CONFLICT clauses to handle duplicates gracefully:

```sql
INSERT INTO companies (name, website, domain)
VALUES ($1, $2, $3)
ON CONFLICT (domain) 
DO UPDATE SET 
  name = EXCLUDED.name,
  website = EXCLUDED.website,
  updated_at = NOW()
RETURNING id;
```

This prevents duplicate records while keeping data fresh on re-processing.

### Foreign Key Relationships
Jobs reference companies. Prospects reference accounts. Signals reference prospects and signal_definitions. These relationships enable:
- Cascading updates
- Referential integrity
- Efficient JOIN queries for scoring and reporting

### Timestamp Tracking
Every table includes:
- `created_at` -- when the record was first inserted
- `updated_at` -- last modification time
- Domain-specific timestamps (e.g., `scored_at`, `enriched_at`, `contact_found_at`)

This enables time-based queries, staleness detection, and audit trails.

---

## AI Integration Patterns

### Prompt Engineering for Structured Output
When using Claude for data extraction or scoring, prompts follow a strict pattern:

1. **Role definition** -- "You are a job fit analyst scoring roles for a specific candidate"
2. **Explicit constraints** -- "Score ONLY based on the provided resume. Do NOT fabricate experience."
3. **Output format specification** -- "Return ONLY valid JSON matching this exact schema"
4. **Examples** -- Include 1-2 examples of expected output
5. **Guardrails** -- "If information is missing, score conservatively rather than guessing"

The constraint against fabrication is critical. Early iterations had Claude creating false experience claims in resume generation. Adding explicit "do not fabricate" instructions resolved this entirely.

### Cost Optimization
AI API calls are the most expensive operation in any workflow. Cost management strategies:

- **Pre-filter aggressively** -- Only send pre-qualified records to Claude
- **Batch processing** -- 5-item batches with 2-3 second delays
- **Token management** -- Keep prompts concise; strip HTML and boilerplate before sending
- **Model selection** -- Use Sonnet for extraction tasks, reserve Opus for complex analysis
- **Caching** -- Store AI results in PostgreSQL to avoid re-processing

### Sanitization Before AI
Before sending data to Claude:
- Strip HTML tags and excessive whitespace
- Remove boilerplate (navigation, footers, cookie notices)
- Truncate to relevant sections (first 3000 chars of job descriptions)
- This reduces token usage by 60-80% while preserving all decision-relevant content

---

## Error Handling Patterns

### Continue On Fail
n8n nodes that interact with external APIs use "Continue On Fail" settings. This prevents a single API timeout from killing an entire batch run. Failed items are logged and can be retried.

### Retry Logic
Transient failures (rate limits, timeouts) use retry with exponential backoff:
- First retry: 2 seconds
- Second retry: 5 seconds
- Third retry: 10 seconds
- After 3 failures: log and skip

### Graceful Degradation
If enrichment fails for a record, the record still progresses through the pipeline with whatever data is available. A missing email does not prevent scoring. A failed web scrape does not block the entire batch.

---

## Batch Processing Pattern

Processing large datasets requires discipline:

```
[Full Dataset] --> [Split into Batches of 5] --> [Process Batch] --> [Delay 2-3s] --> [Next Batch]
```

Why batches of 5:
- Most APIs allow 5-10 concurrent requests
- Small enough that a single batch failure is cheap to retry
- Large enough that processing completes in reasonable time
- The delay between batches prevents rate limit violations

Why 2-3 second delays:
- Serper API: 100 requests/minute limit
- ScraperAPI: Varies by plan, but 2s delay keeps well under limits
- Anthropic API: Token-per-minute limits; small delays prevent quota exhaustion
- Being conservative with delays is always cheaper than debugging rate limit errors

---

## Workflow Organization in n8n

### Color-Coded Sticky Notes
Complex workflows use colored sticky note sections:
- **Red** -- Discovery and data ingestion
- **Orange** -- Filtering and pre-qualification
- **Yellow** -- Enrichment and API calls
- **Green** -- Scoring and classification
- **Blue** -- Output and delivery

This visual organization makes debugging fast. When something breaks, you can immediately locate which phase failed.

### Naming Conventions
Nodes follow a verb-first naming pattern:
- "Generate Search Queries" (not "Search Query Generator")
- "Filter Pending" (not "Pending Filter")
- "Score Role" (not "Role Scorer")
- "Write to Google Sheets" (not "Google Sheets Writer")

This makes the workflow read like a sequence of actions when scanning left to right.

---

## Security Practices

### Credential Management
- API keys stored in n8n credential objects (encrypted at rest)
- Database connection strings use n8n's built-in Postgres credential type
- No hardcoded secrets in workflow JSON (sanitized before export)
- OAuth tokens managed through n8n's token refresh mechanisms

### Data Handling
- Personal data (emails, phone numbers) stored in PostgreSQL with access controls
- Google Sheets used for operational data only, not as a database
- Temporary files cleaned up after processing
- No PII logged in workflow execution history
