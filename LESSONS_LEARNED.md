# Lessons Learned

Real problems I hit, what broke, and what I changed. This is the document that matters most for understanding how these systems evolved.

---

## AI Fabrication in Resume Generation

**The problem:** Early versions of the job scoring workflow included automated resume customization. Claude would analyze a job description and modify resume bullet points to better match. The output looked great -- until I realized Claude was inventing experience I did not have.

**What happened:** Claude inferred that if I had experience with Salesforce, I probably also had experience with specific Salesforce features mentioned in the job description. It would add phrases like "configured CPQ billing rules" when my actual experience was admin-level customization. The fabrications were plausible enough that a human reviewer might not catch them.

**The fix:** Added explicit constraints to the scoring prompt: "Score ONLY based on the provided resume content. Do NOT create, invent, or infer experience not explicitly stated. If a required skill is missing from the resume, note it as a gap rather than filling it in." This eliminated fabrication entirely.

**The lesson:** LLMs are eager to help and will fill gaps with plausible content unless you explicitly tell them not to. Any workflow that generates content representing a person needs hard guardrails against invention.

---

## Deduplication Must Happen Before Enrichment

**The problem:** The first version of the restaurant lead pipeline enriched every record it found, then deduplicated at the end. This meant paying for API calls on records that would later be thrown away as duplicates.

**What happened:** A single restaurant would appear in search results 3-4 times (different pages, different query variations). Each instance triggered a full enrichment cycle: web scrape, email search, Claude extraction. At $0.05-0.10 per enrichment, processing 1,000 records with 30% duplication meant roughly $15-30 wasted per run.

**The fix:** Implemented three-layer deduplication BEFORE enrichment:
1. URL normalization (strip tracking params, force lowercase)
2. Title + Company key matching (catches cross-source duplicates)
3. Description similarity check (catches edge cases where company names differ)

This reduced enrichment API spend by 25-35% immediately.

**The lesson:** Dedup early. The cheapest API call is the one you never make.

---

## Google Docs Templates Beat HTML Generation

**The problem:** I initially built resume generation using HTML-to-PDF conversion. The output was functional but looked obviously auto-generated. Formatting inconsistencies, weird spacing, and font rendering issues made the output unprofessional.

**What happened:** HTML/CSS rendering varies between PDF generators. What looked fine in a browser rendered differently in the PDF engine. Headers were inconsistent, margins varied, and the overall layout felt off.

**The fix:** Switched to a Google Docs template approach. A master resume document in Google Docs serves as the template. The workflow copies the template, then uses placeholder replacement to swap in customized content. The formatting stays perfect because Google Docs handles all the rendering.

**The lesson:** When the output needs to look professionally formatted, use a tool designed for formatting. Do not fight CSS-to-PDF conversion when Google Docs already solves this.

---

## Batch Size 5 Is the Sweet Spot

**The problem:** I experimented with different batch sizes for API-heavy workflows. Batch size 1 was too slow. Batch size 20 hit rate limits. Batch size 10 worked sometimes but failed unpredictably.

**What happened:** Different APIs have different rate limit windows. Serper allows 100/minute. ScraperAPI varies by plan. Anthropic has token-per-minute limits. A batch of 10 might work fine for Serper but overwhelm the Claude API when combined with large job descriptions.

**The fix:** Standardized on batch size 5 with 2-3 second delays between batches. This works within the rate limits of every API I use and provides a comfortable margin for occasional spikes.

**The lesson:** Pick a conservative batch size that works everywhere rather than optimizing per-API. The time difference between 5 and 10 is negligible over a full run, but the reliability difference is significant.

---

## n8n HTTP Request Nodes vs Dedicated API Nodes

**The problem:** Some integrations were built using generic HTTP Request nodes because I wanted more control over the request format. This worked initially but created maintenance headaches.

**What happened:** HTTP Request nodes require manual configuration of authentication headers, content types, and error handling. When Anthropic updated their API version header, I had to find and update every HTTP Request node that called their API. With a dedicated API node, the authentication is handled once in the credential configuration.

**The fix:** Use dedicated API nodes (Google Sheets, Postgres, Apify) wherever available. Reserve HTTP Request nodes for APIs that do not have n8n integrations (Serper, ScraperAPI direct).

**The lesson:** Dedicated nodes handle auth, pagination, and error responses automatically. The small loss of control is worth the massive reduction in maintenance.

---

## PostgreSQL Over Google Sheets for Anything Persistent

**The problem:** The first version of the restaurant lead system used Google Sheets as its primary data store. This worked for small datasets but became unreliable at scale.

**What happened:** Google Sheets API has rate limits (60 requests/minute for reads, 60 for writes). With 500+ records, batch updates would timeout or silently drop rows. The Sheets API also handles concurrent writes poorly -- two workflow runs could overwrite each other's data.

**The fix:** Migrated to PostgreSQL (DigitalOcean Managed Database) as the source of truth. Google Sheets is now a read-only output layer updated after database operations complete. This gave me proper ACID transactions, upsert operations, and no rate limit concerns.

**The lesson:** Google Sheets is great for output and collaboration. It is not a database. As soon as you need upserts, concurrent writes, or more than a few hundred records, use a real database.

---

## Sticky Notes Are Not Optional

**The problem:** Early workflows were built without documentation. When I came back to modify them weeks later, I could not remember what each branch of a complex IF node was doing or why certain filter conditions existed.

**What happened:** I spent more time re-understanding my own workflows than modifying them. A 10-minute change would take 45 minutes because I had to trace data flow through 15 nodes to understand context.

**The fix:** Every workflow now uses color-coded sticky note sections. Red for discovery, orange for filtering, yellow for enrichment, green for scoring, blue for output. Each section includes a brief description of what happens and why.

**The lesson:** Your future self is a different person. Document for them. The 2 minutes spent adding a sticky note saves 30 minutes of archaeology later.

---

## Pre-Filter Before AI Scoring Saves Real Money

**The problem:** The job discovery workflow initially sent every scraped job listing through Claude for scoring. At $0.003-0.01 per scoring call, processing 1,000 jobs cost $3-10 per run. Running twice daily, that added up.

**What happened:** Many of those 1,000 jobs were obviously poor fits -- wrong location, excluded titles (Sales Trainer, Customer Success), or positions at staffing agencies. A 2-second keyword check could have eliminated 40-60% of them before the expensive AI call.

**The fix:** Added a "Quick Snippet Filter" node before AI scoring that checks for:
- Remote/location match
- Title keyword inclusion (must contain at least one target term)
- Title keyword exclusion (skip if contains excluded terms)
- Company type filter (skip staffing agencies and aggregators)

This reduced AI scoring volume by roughly 50%, cutting per-run costs in half.

**The lesson:** Every dollar of API spend should go toward records that have a real chance of being useful. Deterministic pre-filtering is effectively free and should always come before expensive AI analysis.

---

## Explicit Node References in n8n Prevent Data Loss

**The problem:** n8n's default behavior passes data from the immediately preceding node. In complex workflows with merges and branches, this can result in data being silently dropped or overwritten.

**What happened:** In the restaurant enrichment pipeline, data from the "Parse Restaurant Data" node was being overwritten by the "Claude Extract Emails" response. The final merge node only had the Claude output, losing the original restaurant metadata (address, phone, state).

**The fix:** Use explicit node references instead of relying on linear data flow:
```javascript
const restaurantData = $('Parse Restaurant Data').item.json;
const claudeResponse = $input.first().json;
```

This ensures each piece of data is pulled from its source node explicitly, regardless of what the immediately preceding node contains.

**The lesson:** In any workflow with more than 5 nodes, use explicit node references. The default "previous node" behavior is convenient but fragile in complex data flows.
