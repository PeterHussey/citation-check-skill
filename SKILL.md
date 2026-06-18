---
name: citation-check-skill

description: Vision-enabled verification gate with web search. Use when users want to (1) verify slides/reports/PDFs/images against authoritative online sources, (2) validate that citations actually exist and say what's claimed, (3) check charts/graphs/tables for accuracy, (4) audit AI-generated content in doc-only mode (no external knowledge). Two modes - search mode validates against web, doc-only mode ensures everything traces to provided documents. Supports content in any language.

---

# Citation & Hallucination Checker v2

Verification tool with vision + web search. Validates every claim against authoritative sources or provided documents. Works with content in any language.

**Design principle:** Deterministic verification. Same input → Same output.

---

## Dependencies

This skill works out of the box for general use. Some features require additional setup:

| Feature | Requirement | Without it |
|---|---|---|
| Web search (all modes) | Tavily MCP or equivalent search tool active in your session | Web-dependent verification cannot run; doc-only mode still works |
| RAND publication verification | `publications-api` MCP (`mcp__publications-api__GetPublicationById`) | Falls back to rand.org web search automatically — slightly slower, same accuracy |
| Federal award verification | USAspending MCP and/or SAM.gov MCP | Falls back to usaspending.gov web search |
| Full-text academic paper access | hyperresearch CLI (`/Users/hussey/.local/pipx/venvs/hyperresearch/bin/hyperresearch fetch`) | Falls back to Paywall Resolution chain (PMC → Unpaywall → abstract-only status) |

**If you are sharing this skill with a colleague:**
- Web search and doc-only mode require no extra setup beyond Claude Code with a search-capable MCP
- RAND MCP access requires being on a RAND project session with `publications-api` configured
- hyperresearch requires the pipx install; colleagues without it get graceful fallback automatically
- The `research@rand.org` email in the Unpaywall query string can be changed to any valid institutional email

**RAND-specific routing** (claim-type routing table) is additive — colleagues at other organizations can ignore the RAND and USAspending rows or replace them with their own institutional backends. The rest of the skill is fully general.

---

## Two Verification Modes

### Mode 1: Search Verification (Default)

- Searches web for authoritative sources
- Validates citations actually exist
- Checks if cited sources say what's claimed
- Finds original data for statistics

### Mode 2: Doc-Only Verification

- User provides source document(s)
- EVERYTHING must trace to those docs
- Flags anything that appears to come from external knowledge
- Trigger: "only use this document" / "verify against the PDF only" / "don't search the web"

---

## Two-Pass Architecture

**Critical:** Always use two separate passes. Never interleave extraction and verification.

### Pass 1: Extraction Only

1. Read entire document/slides/images
2. Extract ALL claims using the Claim Extraction Rules below
3. Output numbered list: `[claim_id] | [claim_text] | [claim_type] | [location]`
4. **NO verification in this pass**
5. Present extraction to user for confirmation before proceeding

### Pass 2: Verification Only

1. Take Pass 1 output as fixed input
2. Verify each claim_id in sequential order
3. **NO re-extraction allowed** — work only with Pass 1 claims
4. Apply Status Decision Tree to each claim
5. Generate final report

This prevents "discovering new claims" mid-verification and ensures consistency.

---

## Claim Extraction Rules (Exhaustive)

Extract ONLY these claim types. Apply rules strictly — no judgment calls.

### EXTRACT as claims:

| Type            | Pattern                                                    | Example                                          |
| --------------- | ---------------------------------------------------------- | ------------------------------------------------ |
| **Statistic**   | Any number with unit/context (%, $, count, ratio, decimal) | "92.3% accuracy", "$4.7B market"                 |
| **Comparative** | X is [comparative] than Y — includes stated multipliers ("3x", "twice as likely", "four times more") | "3x faster than baseline", "four times as likely: 1.7% vs. 0.4%" |
| **Temporal**    | Time-bound assertion                                       | "In 2024, adoption reached..."                   |
| **Attribution** | Claim tied to source                                       | "According to WHO...", "Smith et al. found..."   |
| **Causal**      | X causes/leads to/results in Y                             | "This reduces latency by..."                     |
| **Existence**   | Asserts something exists/is true                           | "There are 500M users", "The model supports..."  |
| **Ranking**     | Position claims                                            | "largest", "first", "top 3"                      |
| **Quote**       | Direct quotation                                           | Any text in quotation marks attributed to source |
| **Internal data** | Statistics drawn from the study/report's own dataset — identifiable by phrases like "our data show", "participants reported", "n = X", regression coefficients, predicted probabilities, p-values | "ACE screened patients were 3x more likely: 1.5% vs. 0.5%" |

**Compound claim rule:** If a single sentence contains two or more independently verifiable components (e.g., "low-income individuals have higher ACE exposure AND less access to protective factors"), split them into separate claim entries at extraction time and note the parent sentence. Each component is verified independently — a citation may support one but not the other.

### DO NOT extract as claims:

| Type                              | Example                                            | Reason                          |
| --------------------------------- | -------------------------------------------------- | ------------------------------- |
| Definitions                       | "Machine learning is a subset of AI"               | Definitional, not factual claim |
| Opinions marked as such           | "We believe...", "In our view..."                  | Explicitly subjective           |
| Hypotheticals                     | "If adoption continues...", "Could potentially..." | Speculative                     |
| Questions                         | "What drives growth?"                              | Not an assertion                |
| Future predictions without source | "Will reach $10B by 2030"                          | Unless citing a forecast report |
| Methodology descriptions          | "We used PyTorch 2.0"                              | Process, not factual claim      |
| Acknowledgments                   | "Thanks to our collaborators"                      | Not verifiable                  |

### Extraction Output Format

```
[C01] | "Model achieves 96.555% accuracy on ImageNet" | Statistic | Slide 3, bullet 2
[C02] | "Outperforms GPT-4 by 12% on reasoning tasks" | Comparative | Slide 3, bullet 3
[C03] | "According to Chen et al. (2024), transformers scale linearly" | Attribution | Slide 5, para 1
[C04] | "Market size reached $4.7B in 2024" | Statistic + Temporal | Slide 7, chart title
```

---

## Status Decision Tree

Apply this tree to EVERY claim. Follow exactly — no shortcuts.

```
START
│
├─ Is this a CITATION claim (references a paper/report/source)?
│   ├─ YES → Go to CITATION VALIDATION
│   └─ NO → Go to STATISTIC/FACT VALIDATION
│
│
CITATION VALIDATION
│
├─ Step 1: Does the cited source exist?
│   │   Run ALL mandatory search queries (see Search Templates)
│   │
│   ├─ NO → Status: "Citation Not Found"
│   │        Issue: "Cannot locate [citation] in any database"
│   │        STOP
│   │
│   └─ YES (source exists) → Is full text accessible?
│       ├─ YES → Step 2: Does source contain the claimed topic? [continue below]
│       └─ NO (paywalled) → Apply Paywall Resolution chain
│           ├─ Full text retrieved via PMC/Unpaywall → Step 2 [continue below]
│           └─ Still inaccessible → Status: "Paywalled — abstract verified only"
│
│             Step 2: Does source contain the claimed topic?
│             │
│             ├─ NO → Status: "Misquoted"
│             │        Issue: "Source exists but does not discuss [topic]"
│             │        STOP
│             │
│             └─ YES → Step 3: Does source support the exact claim?
│                       │
│                       ├─ YES (exact match) → Status: "Verified"
│                       │                       Confidence: "exact"
│                       │
│                       ├─ YES (paraphrase, same meaning) → Status: "Verified"
│                       │                                   Confidence: "paraphrase"
│                       │
│                       ├─ PARTIALLY (missing context) → Status: "Misleading"
│                       │   Issue: "Claim omits critical context: [what's missing]"
│                       │
│                       └─ NO (contradicts) → Status: "Hallucination"
│                           Issue: "Source says [X], claim says [Y]"
│
│
STATISTIC/FACT VALIDATION
│
├─ Is this an INTERNAL DATA claim (from the document's own study/dataset)?
│   ├─ YES → Go to INTERNAL DATA VALIDATION
│   └─ NO → Step 1 below
│
│
INTERNAL DATA VALIDATION
│
├─ Step 1: Is the claim internally consistent?
│   │   Check: does the stated multiplier/qualifier match the underlying numbers?
│   │   For Comparative claims: compute ratio from the raw percentages/counts provided
│   │   e.g., "four times as likely: 1.7% vs. 0.4%" → 1.7/0.4 = 4.25x → stated "four times" understates
│   │
│   ├─ Multiplier/qualifier matches computed ratio → Status: "Verified (internal)"
│   │                                                  Confidence: "exact"
│   │
│   ├─ Multiplier understates or overstates (but raw numbers present) → Status: "Numerical Error"
│   │   Record: computed ratio, stated qualifier, suggested fix
│   │
│   └─ Raw numbers absent (only multiplier stated, no underlying figures) → Status: "Unverified (internal)"
│       Issue: "Cannot verify — underlying data not provided in document"
│
│
├─ Step 1: Can you find an authoritative source?
│   │   Run ALL mandatory search queries (see Search Templates)
│   │
│   ├─ NO (no source found) → Status: "Unverified"
│   │                          Issue: "No authoritative source found"
│   │                          STOP
│   │
│   └─ YES → Step 2: Do values match EXACTLY?
│             │
│             ├─ YES → Status: "Verified"
│             │         Confidence: "exact"
│             │         STOP
│             │
│             └─ NO → Status: "Numerical Error"
│                      Go to NUMERICAL ERROR DETAILS
│
│
NUMERICAL ERROR DETAILS (Academic Precision Mode)
│
├─ Record:
│   • Source value: [exact number from source]
│   • Claimed value: [number in document being checked]
│   • Deviation: [calculate exact difference]
│   • Source location: [page, table, section]
│
├─ Classification:
│   • ANY rounding → Numerical Error
│   • ANY truncation → Numerical Error  
│   • Significant figures mismatch → Numerical Error
│   • Unit mismatch → Numerical Error
│   • Wrong direction (e.g., increase vs decrease) → Hallucination
│
└─ Exception: If source ITSELF provides rounded figure
    • e.g., Source says "96.555% (approximately 97%)"
    • Then claiming "97%" → Verified (cite the approximation)
```

---

## Numerical Precision Rules (Academic Standard)

**Default mode: Strict academic precision. Exact numbers only.**

| Rule                 | Source      | Claim       | Status            |
| -------------------- | ----------- | ----------- | ----------------- |
| Exact match required | 96.555%     | 96.555%     | ✓ Verified        |
| Any rounding = error | 96.555%     | 97%         | ✗ Numerical Error |
| Any rounding = error | 96.555%     | 96.6%       | ✗ Numerical Error |
| Truncation = error   | 96.555%     | 96.5%       | ✗ Numerical Error |
| Sig figs must match  | 0.834       | 0.83        | ✗ Numerical Error |
| Units must match     | 96.555%     | 0.96555     | ✗ Numerical Error |
| Direction matters    | +12% growth | +15% growth | ✗ Hallucination   |
| Order of magnitude   | $4.7B       | $47B        | ✗ Hallucination   |

### Numerical Error Output Format

```markdown
### Numerical Error: [Claim ID]

| Field | Value |
|-------|-------|
| Claim | "Model achieves 97% accuracy" |
| Location | Slide 4, bullet 2 |
| Source | Chen et al. (2024), Table 3, p.8 |
| Source value | 96.555% |
| Claimed value | 97% |
| Deviation | +0.445% (rounded up) |
| Status | Numerical Error |
| Fix | Replace with: "Model achieves 96.555% accuracy" |
```

---

## Confidence Classification

| Level              | Criteria                                                   | Use when                            |
| ------------------ | ---------------------------------------------------------- | ----------------------------------- |
| **exact**          | ≥95% word overlap OR identical number with identical units | Direct quote, exact statistic       |
| **paraphrase**     | Same fact, different words, no interpretation added        | Restated finding                    |
| **interpretation** | Inference drawn from source data                           | Calculated from source, synthesized |

**Rule:** When uncertain between levels, use the MORE CONSERVATIVE option and flag for review.

---

## Claim-Type Routing

Before running search templates, identify the claim type and route to the appropriate verification backend. Routed verification is faster and more authoritative than web search for the types below.

| Claim type | How to identify | Verification backend | Notes |
|---|---|---|---|
| RAND publication finding | Cites a RAND ID (RR-xxxx, MG-xxxx, PE-xxxx, RRA-xxxx, TR-xxxx, WR-xxxx, MR-xxxx, OP-xxxx) | `publications-api` MCP: call `GetPublicationById(id)` — see RAND Verification Procedure below | Do NOT use web search for RAND claims when MCP is available — MCP is authoritative |
| Federal award / contract | Cites a PIID, award dollar amount, agency obligation figure, or USAspending data | USAspending MCP (`search_awards`, `get_award_detail`) or SAM.gov MCP (`lookup_award_by_piid`) | Use award ID or agency + year + amount to locate the specific award record |
| Academic paper with DOI or PMC ID | Cites a DOI (10.xxxx/...) or PubMed/PMC ID | `hyperresearch fetch` on the DOI or PMC URL — see Paywall Resolution below | If full text returned (>500 words body), verify against full text; otherwise apply Paywall Resolution |
| General statistic / web claim | Any other statistic, date, or factual assertion | Standard web search templates (see Mandatory Search Templates below) | Default path — use when no more specific route applies |

**Bypass rule:** If the claim type is identified and routed above, skip the Mandatory Search Templates for that claim. Record the backend used in the Sources Consulted table.

---

## RAND Verification Procedure

**Requires:** `publications-api` MCP active in the current session. This MCP is available to RAND staff within RAND project sessions where it has been configured. If you are using this skill outside that context, skip to the MCP failure fallback below.

**MCP failure modes — two distinct situations:**

| Result | Meaning | Action |
|--------|---------|--------|
| `{"data":{"publicationById":null}}` | MCP reachable but ID not in database — document may be pre-publication, not yet indexed, or the ID format is wrong | Try alternate ID formats (see Step 1); if still null after 2 attempts, fall back to rand.org web search; note "publications-api returned null — web search used" |
| `{"message":"Endpoint request timed out"}` | MCP server unreachable this session | Fall back to web search for ALL remaining RAND claims in this run; note "publications-api MCP unavailable — web search used" for the session |

Do not retry a timed-out MCP more than once per session — it will not recover mid-session.

**Step 1 — Extract the RAND ID** from the citation. Pattern: letter-prefix + digits, optionally with version suffix (e.g., `RR-2788-OSD`, `RRA4524-2`, `MG-1234`). Common ID format variants to try if first attempt returns null: `RRA2152-1` → also try `RR-A2152-1`; `RBA2152-2` → also try `RB-A2152-2`.

**Step 2 — Call `GetPublicationById(id)`**. This returns:
- `long_abstract` — full abstract (primary verification surface)
- `key_findings` — bullet-point findings (use for finding-level claim verification)
- `recommendations` — recommendations list
- `citation` — authoritative citation string (use to verify author names, year, title)
- `document_link` — PDF URL (pass to `hyperresearch fetch` for full-text verification when needed)

**Step 3 — Verify the claim:**
- **Statistic or number:** search `long_abstract` and `key_findings` for the figure; if present and matches → Verified (exact); if present but differs → Numerical Error; if absent → fetch `document_link` via hyperresearch and search full text
- **Attribution ("RAND found X"):** check `key_findings` and `long_abstract` for the stated finding; apply Status Decision Tree
- **Author/year/title:** check `citation` field directly
- **Finding not in abstract or key_findings:** fetch `document_link` via hyperresearch fetch for full-text search before returning Unverified

**Step 4 — Label source depth:** append `[full text]` if full PDF was fetched and searched, or `[abstract + key findings]` if verification stopped at MCP fields.

---

## Paywall Resolution (Academic Papers)

When an academic paper claim is routed to `hyperresearch fetch` and returns empty or near-empty content, apply this resolution chain before returning "Unverified":

**Resolution chain (in order):**
1. If PMC ID known: fetch `https://pubmed.ncbi.nlm.nih.gov/<pmcid>/` via hyperresearch fetch (or WebFetch if hyperresearch is unavailable)
2. If DOI known and step 1 failed or no PMC ID: query Unpaywall — `https://api.unpaywall.org/v2/<doi>?email=<your-institutional-email>` — parse `best_oa_location.url_for_pdf`; if present, fetch that PDF URL via hyperresearch fetch (or WebFetch)
3. If full text retrieved at any step (>500 words body): proceed with full-text verification; label the source `[full text]` in the Sources Consulted table
4. If all steps fail: do NOT return "Unverified" — return the new status below

**New status: `Paywalled — abstract verified only`**

Use when: a source exists and is identifiable, but full text is inaccessible via PMC, Unpaywall, or direct fetch.

```
Status: "Paywalled — abstract verified only"
Confidence: "abstract"
Issue: "Full text inaccessible — claim verified against abstract/metadata only. Numerical precision and exact quotes cannot be confirmed."
Fix: "Obtain full text via institutional access to complete verification."
```

**Critical distinction:**
- `Unverified` = source cannot be located in any database after exhausting all search templates
- `Paywalled — abstract verified only` = source exists and is identified, but content is gated

---

## Mandatory Search Templates

**Note:** RAND publication claims should use the RAND Verification Procedure above (publications-api MCP), not these web search templates. Federal award claims should use USAspending/SAM.gov MCPs. These templates apply to academic citations, statistics, and general web claims only.

Run ALL applicable templates. Do not stop after first result.

### For Academic Citations

```
Query 1: "[first author last name] [year] [first 3 words of title]"
Query 2: "[full paper title]" site:semanticscholar.org OR site:arxiv.org
Query 3: "[first author] [year] [venue/journal name]"
Query 4: "doi:[DOI]" (if DOI provided)
Query 5: "arxiv:[arxiv_id]" (if arXiv ID provided)
```

### For Statistics (Market size, usage numbers, etc.)

```
Query 1: "[exact number with unit] [topic] [year]"
Query 2: "[topic] [year] statistics report site:statista.com"
Query 3: "[topic] [year] report site:mckinsey.com OR site:gartner.com"
Query 4: "[topic] market size [year] site:gov OR site:edu"
Query 5: "[topic] [number] original source"
```

### For Company/Product Claims

```
Query 1: "[company name] [claim topic] press release [year]"
Query 2: site:[company domain] [claim topic]
Query 3: "[company name] [metric] official announcement"
Query 4: "[company name] [claim] SEC filing" (for public companies)
```

### For Health/Medical Claims

```
Query 1: "[claim topic] site:who.int OR site:cdc.gov OR site:nih.gov"
Query 2: "[claim] systematic review site:cochrane.org"
Query 3: "[claim] meta-analysis pubmed"
```

### For Government/Policy Claims

```
Query 1: "[policy/law name] site:gov"
Query 2: "[statistic] official statistics [country]"
Query 3: "[claim] [agency name] report"
```

---

## Source Authority Hierarchy

When multiple sources found, prefer in this order:

| Rank | Source Type                   | Examples                                          |
| ---- | ----------------------------- | ------------------------------------------------- |
| 1    | RAND institutional data (MCP) | `publications-api` MCP — authoritative RAND publication metadata, abstracts, findings |
| 1    | Federal award data (MCP)      | USAspending MCP, SAM.gov MCP — authoritative federal procurement and assistance data |
| 1    | Primary source                | Original study, official report, raw data         |
| 2    | Government/institutional      | WHO, CDC, World Bank, national statistics offices |
| 3    | Peer-reviewed publication     | Nature, Science, IEEE, ACM                        |
| 4    | Industry reports (named)      | Gartner, McKinsey, Statista (with methodology)    |
| 5    | Reputable news citing primary | NYT, Reuters citing original source               |
| 6    | Secondary compilations        | Wikipedia (check their sources)                   |

**Rule:** If only Rank 5-6 sources found, status = "Unverified" with note "Only secondary sources found"

---

## Multi-Source Verification (Search Mode)

A claim achieves "Verified" status only if:

| Condition              | Sources Required                                    |
| ---------------------- | --------------------------------------------------- |
| Primary source found   | 1 (if authoritative: .gov, peer-reviewed, official) |
| Only secondary sources | ≥2 independent sources agreeing                     |
| Sources conflict       | Status = "Unverified", note the conflict            |

---

## Tie-Breaker Rules

When uncertain, apply these rules. No judgment calls.

| Situation                                 | Rule                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| Missing date on claim                     | Assume refers to most recent year available; flag "needs date" |
| Conflicting sources                       | Use most recent authoritative source; cite both; note conflict |
| Source not found after all queries        | Status = "Unverified" (NOT "Hallucination")                  |
| Number differs due to currency conversion | Flag as "Needs clarification: currency/units"                |
| Same org, multiple reports                | Use most recent; cite with date                              |
| Claim uses "approximately" or "about"     | Still verify base number is in valid range (±10% of source)  |
| Source is paywalled                       | Apply Paywall Resolution chain (PMC → Unpaywall → flag); if still inaccessible, use status "Paywalled — abstract verified only" |
| Source is in different language           | Translate and verify; note translation                       |

---

## Visual Data Verification

For every chart, graph, table, or diagram:

### Step 1: Extract Data Points

- Read ALL values from the visual
- Record: axis labels, units, scale, legend
- Note any visual distortions (truncated axes, 3D effects, etc.)

### Step 2: Find Source

- Search mode: Run search templates for the data
- Doc-only mode: Locate in source document

### Step 3: Compare Value-by-Value

```markdown
| Visual Element | Extracted Value | Source Value | Status |
|----------------|-----------------|--------------|--------|
| Bar 1 (2022) | 45% | 45.0% | ✓ Verified |
| Bar 2 (2023) | 62% | 58.3% | ✗ Numerical Error |
| Bar 3 (2024) | 78% | Not in source | ✗ Hallucination |
```

### Step 4: Check Visual Integrity

| Check                                   | Issue Type                             |
| --------------------------------------- | -------------------------------------- |
| Y-axis starts at non-zero               | "Visual Distortion: axis manipulation" |
| 3D effects distort proportions          | "Visual Distortion: 3D exaggeration"   |
| Missing error bars when source has them | "Misleading: uncertainty omitted"      |
| Different time ranges than source       | "Misleading: cherry-picked timeframe"  |

---

## Doc-Only Mode Workflow

**Trigger phrases:**

- "only use this document"
- "don't search the web"
- "verify against the PDF only"
- "everything should be from the source"

### Step 1: Index Source Document

Build complete index before any verification:

```
SOURCE INDEX
Document: [filename]
Pages: [count]

Page 1:
- Text: [summary of content]
- Statistics: [list all numbers with context]
- Tables: [Table 1: columns X, Y, Z]
- Figures: [Figure 1: shows X]

Page 2:
...
```

### Step 2: Apply Two-Pass Architecture

Same as search mode, but verification uses ONLY the source index.

### Step 3: Trace Each Claim

```
Claim: [C01] "Model achieves 92% accuracy"
Search index for: "92", "accuracy", "performance"
├─ Found: Section 4.2, p.8 — "Our model achieves 92.1% accuracy"
│   └─ Status: Numerical Error (92% vs 92.1%)
│
OR
│
├─ Not found in index
│   └─ Status: "Not in Source"
│       Issue: "This claim cannot be traced to the provided document"
│       Likely: External knowledge / hallucination
```

### Step 4: Flag ALL External Knowledge

In doc-only mode, ANY claim not traceable to source = problem

```markdown
### External Knowledge Detected

These claims are NOT in the provided document:

| Claim ID | Claim | Status | Issue |
|----------|-------|--------|-------|
| C07 | "This method is widely adopted in industry" | Not in Source | Appears to be from model training data |
| C12 | "Published in Nature 2024" | Not in Source | Publication venue not mentioned in source |
```

---

## Output Format

**Default output: HTML report.** Generate an HTML file using the template below and save it to the same directory as the input document, named `citation-check-[document-slug]-[YYYY-MM-DD].html`. Then state the save path to the user.

If the user explicitly requests a different format, use that instead:
- **Quick (markdown):** Summary card + critical issues only (Numerical Errors, Hallucinations, Unverified, Author Attention)
- **Full (markdown):** Complete traceability report in markdown
- **JSON:** Machine-readable audit (see references/citation_schema.json)

### HTML Report Template

Generate the full HTML inline. Use only inline CSS — no external dependencies. The file must render correctly when opened directly in a browser.

**Required sections (in order):**

1. **Summary card** — overall PASS/FAIL banner (green/red), document name, date, mode, claim counts by status
2. **Action items** — only if issues exist; red-bordered box listing each Numerical Error, Hallucination, Citation Gap, and Author Attention item with its fix
3. **Verified claims table** — collapsible or full; columns: ID, Claim (truncated to ~80 chars), Source, Location, Confidence
4. **Issues detail** — one subsection per issue status with full detail rows
5. **Sources consulted** — table of all sources used

**Status color coding:**
- Verified → green row (`#f0fdf4` background, `#16a34a` text)
- Numerical Error → amber (`#fffbeb`, `#92400e`)
- Hallucination → red (`#fef2f2`, `#dc2626`)
- Unverified → amber (`#fffbeb`, `#92400e`)
- Misleading → orange (`#fff7ed`, `#c2410c`)
- Paywalled — abstract only → blue-gray (`#f0f9ff`, `#0369a1`)
- Author Attention → purple (`#faf5ff`, `#7e22ce`)
- Internal data verified → green (same as Verified)

**Summary count rows** in the summary card:

| Metric | Count |
|--------|-------|
| Total claims extracted | X |
| Verified | Y |
| Verified (internal data) | Y2 |
| Numerical Error | Z |
| Unverified | A |
| Hallucination | B |
| Misleading | C |
| Paywalled — abstract only | E |
| Citation Gap | F |
| Author Attention | G |
| Not in Source (doc-only) | D |

**Overall Status indicator:** A small inline badge — "Issues require author review" (red) or "All claims verified" (green). Do NOT use a large PASS/FAIL banner — a subtle status chip is sufficient.

---

## Critical Rules

1. **Two-pass always** — Extract first, verify second. Never interleave.
2. **Every claim gets checked** — No exceptions, no skipping "obvious" ones.
3. **Exact numbers only** — 96.555% ≠ 97% in academic mode.
4. **Find the origin** — Don't accept secondary sources citing unknown primaries.
5. **Run ALL search templates** — Don't stop after first result.
6. **Citations must be real** — Search to confirm papers/reports exist.
7. **Check what sources actually say** — A real paper can still be misquoted.
8. **In doc-only mode, flag ALL external knowledge** — Even if it's true.
9. **When uncertain, be conservative** — "Unverified" is safer than false "Verified".
10. **Follow tie-breaker rules** — No ad-hoc judgment calls.
11. **Split compound claims at extraction** — one sentence, two verifiable components = two claim entries.
12. **Check ratio math on all Comparative claims** — compute the ratio from raw numbers; stated multiplier must match within rounding of the underlying figures.

---

## Language Support

- Accepts content in any language
- Searches in the appropriate language for sources
- Reports in the same language as user's request
- Cross-language verification supported (e.g., Chinese slides citing English papers)
- When translating for verification, note: "Translated from [language]"

---

## Changelog

**v2.1** — Post-test improvements (2026-06-18)

- Added `Internal data` claim type with its own verification path (internal consistency check, ratio math)
- Added compound claim splitting rule at extraction time
- Added explicit ratio math check for all Comparative claims (stated multiplier vs. computed ratio)
- Added MCP failure mode table distinguishing null (ID not found) from timeout (MCP unreachable) with different recovery actions
- Added common RAND ID format variants to try before falling back to web search
- Changed default output to HTML report with status color coding and action items section
- Added `Citation Gap` and `Author Attention` as named output statuses
- Updated Critical Rules to include compound claim and ratio math rules

**v2.0** — Consistency update

- Added Two-Pass Architecture (extract → verify separation)
- Added exhaustive Claim Extraction Rules
- Added Status Decision Tree (deterministic classification)
- Added strict Numerical Precision Rules (academic mode)
- Added Mandatory Search Templates
- Added Multi-Source Verification requirements
- Added Tie-Breaker Rules for edge cases
- Added Confidence Classification thresholds
