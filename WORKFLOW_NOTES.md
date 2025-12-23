# Workflow enhancements for grant sourcing and drafting

This document captures the intended changes for the workflow that supports Workshops for Warriors. The workflow is assumed to run in n8n.

**Airtable base/table**: US Grants.gov Tracker / Grants. Use the listed field names when present on the scraped URL: Posted Date, Last Updated Date, Original Closing Date for Applications, Current Closing Date for Applications, Archive Date, Estimated Total Program Funding, Award Ceiling, Award Floor, Eligible Applicants, Assistance Listing Number, Competition ID, Competition Title, Opportunity Package ID, Opening Date, Closing Date, and Actions. For any field scraped that is not present in Airtable, append it in a `Comments` field. Backlinks and resource URLs should also be stored to remain clickable.

## High-level flow

1. **Grant detection**
   - When a grant matching the Workshops for Warriors remit is found, the workflow updates Airtable (US Grants.gov Tracker / Grants) with the grant record and the final grant URL (including the grant ID added by the AI transformation node). Deduplicate before insert/update using Grant ID, Opportunity Number, and Funding and Opportunity Number.

2. **URL-driven enrichment**
   - After Airtable is updated with the canonical grant URL, the workflow fetches that URL and scrapes page content for grant details needed for application drafting (e.g., eligibility, deadlines, budget caps, contact info, required documents, formatting rules).
   - Capture Airtable fields when present: Posted Date, Last Updated Date, Original Closing Date for Applications, Current Closing Date for Applications, Archive Date, Estimated Total Program Funding, Award Ceiling, Award Floor, Eligible Applicants, Assistance Listing Number, Competition ID, Competition Title, Opportunity Package ID, Opening Date, Closing Date, and Actions. Any additional fields from the page that do not map to Airtable fields should be appended in the `Comments` field.

3. **Supplemental material handling**
   - If the grant page links to additional resources (supporting documents, instructions, downloadable forms), the workflow downloads and stores them in Google Drive using the provided API key. Preserve the original URL, and write the Google Drive URL back to Airtable for later use. If a file exceeds 15 MB, do not store it—only keep the URL and note in `Comments` that the file exceeded the limit.
   - Metadata to capture alongside each resource: title/filename, source URL, Google Drive URL (if stored), file type, size (if available), and a description/context extracted from surrounding text. Keep all URLs clickable.
   - If pagination or redirects expose additional URLs, capture and process those pages similarly, or at minimum record the additional URL for follow-up scraping.

4. **Analysis for drafting**
   - Extracted content and documents are analyzed to surface key details for the grant draft. The workflow should produce structured notes that include:
     - Eligibility criteria and geographic/sector restrictions.
     - Deadlines, submission portals, and formatting requirements.
     - Budget guidance, cost-share rules, and reporting expectations.
     - Scoring criteria or reviewer priorities if present.
     - Required attachments and their expected formats.
   - Each note should keep backlinks (clickable URLs) to both the original source and the stored copy (when available) so users can verify context.
   - Recommended AI models: start with OpenAI GPT-4 Turbo or Anthropic Claude 3 Sonnet for balanced cost/performance; if documents are long, consider Claude 3 Opus or GPT-4o mini with chunking. Enforce token limits by chunking text and merging summaries, and log model usage for traceability.

5. **Draft support output**
   - Store the structured notes and resource links in Airtable (or a document store) so they can be injected into the drafting step and reviewed later.

## Recommended implementation details

- **Scraping**: Use an HTTP Request node to pull the page, then HTML extract/transform nodes to capture labeled fields. Normalize URLs to absolute paths before storage. Deduplicate Airtable updates using Grant ID, Opportunity Number, and Funding and Opportunity Number before writing.
- **Document capture**: For linked files, download via HTTP Request and save to Google Drive with the provided API key; write the Drive URL back to Airtable. Skip files over 15 MB and record the original URL plus a `Comments` note about the size limit. Keep the original URL and the storage URL.
- **Analysis**: Use an AI node to summarize extracted text and documents. Feed it both the scraped fields and document text; return structured JSON that includes backlinks to originals and stored copies.
- **Error handling**: Log and mark records when a page fails to fetch or parse, and retry with backoff.
- **Security**: Avoid scraping untrusted auth-gated pages; sanitize URLs before requests.

## Open questions for the requester

At this time, all provided requirements have been incorporated. Flag any further limits (e.g., rate caps, page fetch throttles, or preferred Google Drive folder structure) before implementation.

## Branching advice

Because these changes are significant and involve new scraping, storage, and analysis behavior, create a feature branch off the original workflow to isolate testing and rollout. Merge back after validating the full end-to-end flow and data handling in a non-production Airtable/base.
