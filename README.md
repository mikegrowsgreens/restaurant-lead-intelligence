# Restaurant Lead Intelligence Platform

End-to-end lead generation and enrichment pipeline for the restaurant vertical. Built at Shipday to transform raw restaurant data into qualified, actionable leads with verified contact information.

## Problem

I was handed lead lists of 10,000+ restaurants with no qualification, no prioritization, and 60%+ missing contact information. Manual research took 4-5 minutes per lead. At that rate, properly qualifying the list would take 100+ hours of pure data entry. Meanwhile, I was expected to be on the phone selling.

The lists also had no ICP filtering. They included every type of restaurant from fine dining to food trucks, with no way to identify which ones were actually good prospects for a delivery logistics platform.

## Solution

A multi-workflow system that handles discovery, enrichment, scoring, and delivery:

### Workflow 1: ToastTab Lead Generator (`toasttab-lead-gen.json`)
- Uses Serper API to search Google for `site:toasttab.com/local` across target states
- ToastTab presence is itself a buying signal: it means the restaurant has a modern POS system and is likely technology-forward
- Extracts and deduplicates restaurant URLs from search results
- Scrapes each ToastTab page for business name, address, phone, cuisine type, website
- Searches Google for restaurant email addresses using business name + city + "contact email"
- Uses Claude API to intelligently extract email addresses from search result snippets
- Writes enriched leads to Google Sheets with status tracking
- Covers 6+ states: CT, MA, MD, MN, WI, IN

### Workflow 2: Email Enrichment Pipeline (`email-enrichment.json`)
- Reads restaurants from the master Google Sheet that need email enrichment
- Runs multi-source search: Google results, Facebook pages, official website scraping
- Claude extracts and validates email addresses from unstructured web content
- Confidence scoring based on email domain match and prefix type
- Prioritizes business emails (info@, contact@, orders@) over generic ones
- Updates the master sheet with results, confidence scores, and enrichment timestamps

### Workflow 3: Google Maps Discovery (`google-maps-scraper.json`)
- Alternative discovery method using Google Maps/Places API
- Searches for restaurants by cuisine type and location
- Extracts business details, ratings, review counts, operating hours
- Feeds into the same enrichment pipeline

## Scoring Algorithm

The 150-point scoring system evaluates 12+ operational signals:

| Signal | Weight | What It Indicates |
|--------|--------|-------------------|
| Cuisine type (Pizza/Italian) | High | Delivery-heavy cuisine = higher platform need |
| Google rating (below 4.0) | Medium | Potential operational challenges = pain point |
| Review count (high volume) | Medium | Established business with customer base |
| Third-party delivery presence | High | Already doing delivery = easier adoption |
| Delivery volume indicators | High | High volume = more revenue potential |
| POS system type | Medium | Tech-forward = more likely to adopt |
| Driver status | High | "Hiring drivers" = active delivery operation |
| Location count | Medium | Multi-location = larger deal potential |
| Website quality | Low | Professional online presence = business maturity |
| Pain point indicators | High | Mentions of delivery challenges in reviews |
| Years in business | Low | Established = more stable revenue |
| Operating hours | Low | Late-night/extended hours = delivery demand |

## Email Extraction Methodology

The Claude-powered email extraction handles cases that regex cannot:

- Obfuscated emails: "info [at] restaurant [dot] com"
- Context-dependent extraction: Distinguishing business emails from platform emails
- Confidence assessment: Domain match to business name increases confidence
- Exclusion logic: Filters out noreply@, support@wix.com, @facebook.com, test@, example.com
- Fallback: If Claude extraction fails, raw regex serves as backup

Discovery rate: 65-80% of restaurants processed yield at least one verified email address.

## Results

- Processed 1,000+ restaurants across CT and MA markets
- 600+ verified contacts extracted for targeted outreach
- $6.49 average cost per state run
- 15-20 hours/week saved on manual research
- 85%+ data confidence on enriched records
- System scales to any state by adding search queries (no code changes needed)

## Files

| File | Description |
|------|-------------|
| `workflows/toasttab-lead-gen.json` | ToastTab discovery via Google Search |
| `workflows/email-enrichment.json` | Production email enrichment pipeline |
| `workflows/google-maps-scraper.json` | Google Maps based discovery |
| `database/schema.sql` | PostgreSQL table definitions |
| `docs/scoring-algorithm.md` | Full 150-point scoring breakdown |
| `docs/email-extraction.md` | AI extraction methodology and examples |

## Transferability

This architecture is vertical-agnostic. To adapt it to a different market:

1. **Change the discovery source** -- Replace ToastTab with G2, Crunchbase, or any directory
2. **Adjust ICP criteria** -- Replace cuisine type with tech stack, replace review count with employee count
3. **Update enrichment sources** -- Different verticals have different data sources
4. **Modify scoring weights** -- Each market has different signals that matter

The pipeline structure (Discover, Enrich, Score, Deliver) stays exactly the same.

## Setup

1. Import workflow JSON files into your n8n instance
2. Configure credentials: Serper API, Anthropic, Google Sheets, ScraperAPI
3. Create a Google Sheet with the expected column structure (see schema)
4. Update target states and search queries in the "Generate Search Queries" node
5. Run manually for first batch, then enable scheduling
