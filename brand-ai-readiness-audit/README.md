# Brand AI-Readiness Audit

A portable Agent Skill Marketplace for auditing a public website for two connected concerns:

- **AI discoverability:** can automated systems reach, read, identify, extract, and corroborate important website information?
- **On-site engagement:** once a page is discovered, does the site's own structure provide meaningful paths to related information and useful next steps?

The input is one HTTP or HTTPS website URL. The output is one JSON audit report containing evidence-backed findings, severity and confidence, prioritized recommendations, crawl statistics, and limitations. The system is **read-only and recommend-only**: it observes public pages and reports actions; it does not log in, submit forms, change content, or apply recommendations automatically.

This repository is an Adobe University Hackathon 2026 Round 3 submission. It uses the agentskills.io-style `SKILL.md` format, a `marketplace.json` manifest, and exactly one marketplace entrypoint: `audit-orchestrator`.

## 1. Problem Statement

A website can be usable to a human visitor while still being difficult for automated systems to process reliably. Important content may be:

- inaccessible to a compliant crawler;
- discoverable only through a bounded sitemap or a narrow navigation path;
- present after client-side rendering but absent from the initial HTML;
- described without clear headings, entity names, or structured data;
- associated with the wrong organization, product, service, region, or price context;
- asserted repeatedly without enough information to verify it externally; or
- reachable as a page but not connected to meaningful next steps through the site's internal links.

The audit treats these as observable engineering problems, not as claims about the internal behavior of any particular AI product. It checks the mechanisms that affect automated discovery and interpretation: access, page identity, readable content, facts, structured representations, temporal context, evidence, and internal pathways.

The engagement side is deliberately narrower than a generic UX review. It examines observable internal navigation and page relationships. A sitemap entry is not treated as proof of engagement, and zero inbound links in a bounded crawl is not treated as conclusive proof that a page is orphaned.

## 2. What the Marketplace Does

At a high level, the marketplace performs this lifecycle:

1. Validate and normalize the target URL.
2. Retrieve `robots.txt` and discover same-origin sitemaps.
3. Crawl a bounded set of public HTML pages using a priority queue.
4. Acquire pages over HTTP first and use Playwright rendering when the HTTP response appears insufficient.
5. Build site and page context: inferred site type, brand signal, primary entities, page types, and confidence.
6. Run deterministic audits for access, machine readability, entity facts, structured data, freshness, and engagement.
7. Extract deterministic factual claims from readable page content and entity facts.
8. Semantically filter candidate claims when the local semantic classifier is available, with a deterministic continuation path if it fails.
9. Aggregate repeated observations into normalized claims and create a bounded verification task set.
10. Discover and retrieve bounded external evidence when Gemini search configuration is available.
11. Compare claims with external evidence through entity-, predicate-, and value-aware corroboration.
12. Aggregate valid findings into the internal `AuditResult` contract.
13. Generate recommendations through the LLM/deterministic recommendation chain.
14. Validate recommendations for schema, evidence grounding, scope, priority, duplicates, and read-only safety.
15. Emit one final JSON report.

### AI Discoverability

The discoverability audit covers:

- crawler access and robots limitations;
- initial HTML versus rendered content;
- page and entity classification;
- headings and extractable text;
- Product structured-data applicability and completeness;
- organization, product, and service fact scope;
- freshness signals and temporal context;
- internal consistency and external corroboration.

### On-Site Engagement

The engagement audit builds an internal link graph from accessible crawled pages and examines whether meaningful pages have useful contextual paths. It compares pages with the same inferred type, considers sitemap-only discovery, and records the limitations of the crawl subset instead of pretending to see the entire site.

## 3. Architecture Overview

The actual execution flow is:

```text
Website URL
    |
    v
URL validation and normalization
    |
    v
crawl-access: robots, sitemaps, priority crawl, acquisition
    |
    v
site-understanding: site type, brand signal, page types, entities
    |
    +--------------------------------------------------------------+
    | deterministic audits execute in parallel                     |
    |                                                              |
    |  crawl-access audit                                          |
    |  machine-readability                                         |
    |  entity-fact-audit                                           |
    |  structured-data-audit                                       |
    |  freshness-consistency                                      |
    |  engagement-audit                                            |
    +--------------------------------------------------------------+
    |
    v
entity facts + readable text claims
    |
    v
claim extraction: semantic filtering, normalization, aggregation
    |
    v
verification planner: bounded, priority-ordered tasks
    |
    v
external evidence: Gemini search grounding, retrieval, extraction,
entity resolution
    |
    v
corroboration: supported, conflicting, or unresolved comparisons
    |
    v
AuditResult: findings, statistics, site profile, crawl data, limitations
    |
    v
recommendation engine: LLM attempts, deterministic fallback, emergency actions
    |
    v
recommendation-validator
    |
    v
final audit-report.json
```

The six deterministic audit operations share an immutable crawl result and are run with `Promise.all`. Claim processing, external evidence, corroboration, result aggregation, recommendation generation, and validation then proceed in dependency order. Each major stage is wrapped by the orchestrator's `runSafely` helper so a failed specialist stage contributes a limitation and empty findings rather than necessarily destroying the complete report.

## 4. Marketplace Structure

```text
brand-ai-readiness-audit/
|-- marketplace.json
|-- package.json
|-- package-lock.json
|-- README.md
|-- run-site-audit.js
|-- inspect-claims.js
|-- audit-report.json                 # sample/current report artifact
|-- .env                              # local environment file; ignored by git
|-- .cache/                           # crawl cache; ignored by git
|-- shared/
|   |-- audit-result.js
|   |-- audit-result-contract.test.js
|   |-- evidence.js
|   |-- finding.js
|   |-- finding-contract.test.js
|   |-- gemini-client.js
|   |-- page-eligibility.js
|   |-- test-audit-result.js
|   |-- test-finding.js
|   |-- test-url.js
|   `-- url.js
`-- skills/
    |-- audit-orchestrator/
    |   |-- SKILL.md
    |   |-- references/report-schema.md
    |   |-- references/severity-model.md
    |   `-- scripts/
    |-- crawl-access/
    |   |-- SKILL.md
    |   |-- references/crawl-checklist.md
    |   `-- scripts/
    |-- site-understanding/
    |   |-- SKILL.md
    |   `-- scripts/
    |-- machine-readability/SKILL.md and scripts/
    |-- entity-fact-audit/SKILL.md and scripts/
    |-- structured-data-audit/SKILL.md and scripts/
    |-- freshness-consistency/SKILL.md and scripts/
    |-- engagement-audit/SKILL.md and scripts/
    |-- claim-extraction/SKILL.md and scripts/
    |-- external-evidence/SKILL.md and scripts/
    |-- corroboration/SKILL.md and scripts/
    |-- recommendation-engine/SKILL.md and scripts/
    `-- recommendation-validator/SKILL.md and scripts/
```

`SKILL.md` files describe the capability boundaries and intended contracts. The `scripts/` directories contain the executable implementation and focused tests. The `references/` directories contain the crawl checklist, report contract, and severity model used by the project. The root `shared/` directory contains contracts and utilities used across skills.

## 5. `marketplace.json`

`marketplace.json` is the marketplace manifest. It registers the skills and tells a host which skill receives the user request first.

The current manifest registers exactly one entrypoint:

| ID                         | Path                              | Role                                                            | Entrypoint |
| -------------------------- | --------------------------------- | --------------------------------------------------------------- | ---------- |
| `audit-orchestrator`       | `skills/audit-orchestrator`       | Composes the complete audit workflow and emits the final report | Yes        |
| `site-understanding`       | `skills/site-understanding`       | Infers site and page context                                    | No         |
| `crawl-access`             | `skills/crawl-access`             | Safely discovers and acquires public pages                      | No         |
| `machine-readability`      | `skills/machine-readability`      | Checks accessible machine-readable content                      | No         |
| `entity-fact-audit`        | `skills/entity-fact-audit`        | Extracts and compares scoped entity facts                       | No         |
| `structured-data-audit`    | `skills/structured-data-audit`    | Audits applicable JSON-LD and structured data                   | No         |
| `freshness-consistency`    | `skills/freshness-consistency`    | Detects evidence-backed temporal-context risks                  | No         |
| `corroboration`            | `skills/corroboration`            | Compares website claims with external evidence                  | No         |
| `engagement-audit`         | `skills/engagement-audit`         | Audits internal paths and link connectivity                     | No         |
| `recommendation-engine`    | `skills/recommendation-engine`    | Generates prioritized recommendations                           | No         |
| `recommendation-validator` | `skills/recommendation-validator` | Rejects unsupported or unsafe recommendations                   | No         |
| `external-evidence`        | `skills/external-evidence`        | Discovers, retrieves, and extracts bounded external evidence    | No         |
| `claim-extraction`         | `skills/claim-extraction`         | Extracts and prepares verifiable claims                         | No         |

The entrypoint imports the other skills directly from their scripts. Users normally invoke the orchestrator, while individual skills remain independently testable for development and evaluation.

## 6. Every Skill

### 6.1 `audit-orchestrator`

**Purpose and why it exists.** This is the single user-facing composition point. It keeps the marketplace easy to invoke while preserving separate specialist responsibilities. The orchestrator coordinates evidence flow, error isolation, report construction, recommendation generation, and validation; it does not independently decide whether a website is correct.

**Input.** A `targetUrl` string using HTTP or HTTPS.

**Processing.** It validates the URL, runs the crawl, understands the site, runs deterministic audits in parallel, processes claims, acquires external evidence, corroborates facts, builds `AuditResult`, generates recommendations, validates them, and maps the internal result to the final report schema.

**Output.** One report with `version`, `site`, `audited_at`, severity counts, site profile, crawl statistics, findings, recommendations, and optional limitations.

**Important details.** A global 250,000 ms timeout returns a partial timeout report. Specialist failures are recorded as limitations through `runSafely`. A finding-native emergency recommendation path remains available if LLM and normal deterministic recommendation generation fail.

**Discoverability/engagement value.** It ensures both major hackathon areas are evaluated in one reproducible workflow.

**False-positive protection and failure behavior.** It preserves scope and limitations between stages and does not turn unavailable pages or unavailable external evidence into unsupported defects.

### 6.2 `crawl-access`

**Purpose and why it exists.** Downstream audits need a trustworthy, bounded observation set. This skill owns public-page discovery, robots compliance, acquisition state, source/rendered representations, and crawl limitations so individual audits do not implement incompatible crawlers.

**Input.** A public HTTP/HTTPS URL and environment-controlled crawl settings.

**Processing.** It retrieves `robots.txt`, discovers same-origin sitemap locations, seeds the target, prioritizes candidates, follows same-origin links, applies robots rules, rate limits requests, acquires HTML over HTTP, and falls back to a headless browser when the response looks JavaScript-dependent. It extracts titles, meta descriptions, canonical URLs, headings, links, JSON-LD, text, hashes, status, discovery source, and depth.

**Output.** A `CrawlContext` containing target/origin, timestamps, limits, robots and sitemap information, pages, acquisition counts, statistics, and limitations.

**Important details.** Defaults are 30 pages, depth 4, queue capacity 500, HTTP timeout 12 seconds, browser timeout 15 seconds, and a 1,500 ms minimum request interval. The checklist reference also documents the intended safety envelope: same-origin crawling, GET/HEAD-style read-only acquisition, 5 MB maximum response guidance, no forms, no authentication, and no resource modification. URL normalization prevents duplicate visits and excludes common binary, script, stylesheet, archive, and media extensions.

**Discoverability/engagement value.** It establishes whether content can be reached by a compliant crawler and supplies the internal-link evidence used by engagement analysis.

**False-positive protection.** Robots-disallowed content is not acquired or treated as a content defect. A failed request is recorded as access state, not as proof that the page's content is missing. Sitemap-only discovery is preserved as metadata rather than interpreted as an orphan finding.

**Failure behavior.** Missing `robots.txt` permits crawling under the safe runtime policy but adds a limitation. Individual HTTP/browser failures become blocked/error or crawl-error pages. Cache failures do not turn successful acquisition into failure. A total crawl failure is converted by the orchestrator to an empty crawl with a coverage limitation.

### 6.3 `site-understanding`

**Purpose and why it exists.** Page-specific checks need context. This skill separates interpretation of the observed site from defect detection so downstream audits can apply rules only where they make sense.

**Input.** Accessible pages and their crawl metadata.

**Processing.** It detects weak site signals for ecommerce, SaaS, university, restaurant, healthcare, and media patterns; infers a site type and confidence; extracts a homepage-title brand signal; identifies primary entity types; classifies accessible pages; and counts page types.

**Output.** `siteType`, `brandName`, normalized `brand`, signals, `primaryEntities`, `pageTypeCounts`, and per-page type/confidence/scores.

**Important details.** Site signals use terminology and URL paths such as commerce language, pricing, admissions, restaurant terms, healthcare terms, and media terms. Page classification uses page text, URL patterns, structured data, purchase signals, product titles, price signals, canonical information, collection query keys, and shared page eligibility.

**Discoverability/engagement value.** It gives every later audit a page and entity scope rather than applying one generic rule to every URL.

**False-positive protection.** URL paths are weak signals and are combined with content, schema, and page-role evidence. Listing/category/search/deal and implementation-like pages can be classified away from product detail.

**Failure behavior.** If the stage fails, the orchestrator supplies an unknown site profile and empty page map, records the failure, and continues with reduced context.

### 6.4 `machine-readability`

**Purpose and why it exists.** This skill tests whether important accessible content is exposed in forms an automated extractor can observe and interpret.

**Input.** Accessible pages with source HTML/text, rendered text, headings, and URLs.

**Processing.** It compares source and rendered word counts, detects an empty client-side root shell, checks for large source/rendered text gaps, checks for no extractable rendered text, and evaluates heading structure such as missing H1 or weak heading signals where the implementation's thresholds apply.

**Output.** Machine-readability findings with metrics including source word count, rendered word count, text gap, source coverage, affected pages, and extraction method.

**Important details.** A large gap is considered when rendered text has at least 50 words and at least 60% of rendered words are absent from the source-derived text. A client-side rendering finding can also be emitted for an empty source root with rendered content.

**Discoverability/engagement value.** It distinguishes content that a browser can display from content available in the initial HTML or reliably extractable DOM.

**False-positive protection.** It analyzes only accessible pages and does not treat JavaScript usage itself as a defect. It does not report an empty-content issue when meaningful rendered text exists.

**Failure behavior.** A stage failure is isolated by the orchestrator and recorded as a limitation with no findings from that stage.

### 6.5 `entity-fact-audit`

**Purpose and why it exists.** Important facts are useful only when attached to the correct entity. This skill extracts organization, product, service, and page-level facts, then detects meaningful inconsistencies without collapsing different products into one brand identity.

**Input.** Accessible pages plus site understanding and structured data.

**Processing.** It parses Product, Organization, LocalBusiness, Corporation, Restaurant, and Service JSON-LD, infers page entities when schema is absent, extracts visible phone, email, headquarters, price, and commerce facts, normalizes values, deduplicates higher-confidence observations, and groups facts by entity and predicate.

**Output.** Facts, candidate claims, conflicts, and entity-fact findings. Facts preserve entity, predicate, value, source, source type, scope, confidence, and product context.

**Important details.** Product and service entities are namespaced separately from organizations. Product price comparisons use product-specific context such as path/title/category rather than brand alone. Different products under one brand are not automatically conflicts. Region and currency differences do not automatically create a price conflict. Organization-level phone, email, address, headquarters, and name facts remain comparable at organization scope.

**Discoverability/engagement value.** Correct entity association lets automated systems identify what a fact describes and lets corroboration compare like with like.

**False-positive protection.** The core rule is: **brand identity must not substitute for product identity**. `Brand A -> Product X -> $20` and `Brand A -> Product Y -> $40` are not an organization price contradiction. Product/listing context and product-detail evidence also control whether product facts are inferred.

**Failure behavior.** Invalid JSON-LD is skipped by fact extraction; the structured-data audit remains responsible for reporting invalid schema. A failed entity stage is isolated and its missing output is recorded as a limitation.

### 6.6 `structured-data-audit`

**Purpose and why it exists.** Structured data can improve machine-readable entity representation, but missing Product JSON-LD is meaningful only on an eligible product-detail page. This skill owns applicability and completeness checks.

**Input.** Accessible pages, page understanding, page eligibility, JSON-LD/schema objects, metadata, and visible context.

**Processing.** It parses JSON-LD, collects schema types, finds Product and Organization objects, checks product-detail eligibility, and evaluates required and recommended Product properties. It can report missing Product schema, missing core identifying properties, and incomplete recommended properties when applicable.

**Output.** Structured-data findings with schema types, missing properties, affected pages, methods, and metrics.

**Important details.** Eligibility requires a meaningful, standalone product-detail interpretation with adequate confidence. Listing/category/search/deals/cart/checkout/account/help and similar non-PDP paths are excluded. The shared page-eligibility layer and page type/entity role are consulted before `SD-001` is emitted.

**Discoverability/engagement value.** It evaluates whether important product entities have a useful machine-readable representation without penalizing catalog pages for not pretending to be one product.

**False-positive protection.** Applicability is decided before absence is reported. Product-like URL paths alone are insufficient; product evidence, page role, meaningfulness, standalone status, and confidence matter.

**Failure behavior.** Parse errors are represented for the audit logic or skipped by the relevant parser; a specialist failure becomes an orchestrator limitation.

### 6.7 `freshness-consistency`

**Purpose and why it exists.** Automated systems and visitors need temporal context for schedules, availability, deadlines, events, and similar content. This skill detects missing context only when the page provides evidence that time matters.

**Input.** Accessible page text, title, and URL.

**Processing.** It detects strong temporal signals such as events, schedules, hours, availability, deadlines, and expiration language; detects contextual news/announcement/update terms when paired with temporal language; extracts conservative date formats; and reports a finding when time-sensitive signals exist without a detected date.

**Output.** Per-page temporal signals, extracted dates, freshness relevance/reason, and `FR-001` findings where supported.

**Important details.** Supported date patterns include ISO dates, common numeric dates, and month-name formats. Strong signals can be relevant without a date; generic words such as `latest` are not enough by themselves.

**Discoverability/engagement value.** It helps machines distinguish current versus context-dependent information without claiming to prove staleness.

**False-positive protection.** **No detected date does not equal stale content.** The finding means that the page appears time-sensitive but lacks explicit temporal context useful for interpretation.

**Failure behavior.** It evaluates accessible pages only; stage failures are captured as limitations.

### 6.8 `engagement-audit`

**Purpose and why it exists.** Discoverability has a second step: useful continuation. This skill evaluates internal page relationships without becoming a broad visual UX audit.

**Input.** Accessible pages, extracted links, discovery source, and site-understanding page types.

**Processing.** It builds a normalized same-origin internal link graph, counts inbound links, groups comparable pages by inferred type, calculates median inbound-link counts, and examines weak connectivity and sitemap-only discovery conditions.

**Output.** Engagement findings such as `EN-001`, with target inbound links, comparable page counts, median inbound links, affected pages, representative pages, and limitations in the evidence notes.

**Important details.** Weak connectivity requires at least three comparable pages, zero inbound links for the target, and a positive comparable median. Sitemap-only checks require meaningful page types, at least two connected comparable pages, and exclude homepages, localization roots, and known implementation fragments.

**Discoverability/engagement value.** It identifies pages that may be difficult to reach through contextual site navigation after discovery.

**False-positive protection.** **Sitemap presence is not proof of good engagement**, and **zero inbound links in a bounded crawl is not definitive proof of an orphan page**. The audit only reports a relative weakness when the crawled comparison set supports it.

**Failure behavior.** Missing or failed pages reduce graph coverage; the resulting limitation is preserved rather than treated as a complete-site conclusion.

### 6.9 `claim-extraction`

**Purpose and why it exists.** Raw page text is not a safe verification unit. This skill converts selected observations into structured claims while keeping extraction separate from truth decisions.

**Input.** Entity facts, readable page text, site/entity context, source URLs, predicates, values, and confidence.

**Processing.** Deterministic extraction recognizes founded years, headquarters, authorized partners, product/service statements, certifications, and awards. Claims are normalized as subject/predicate/object triples, semantically filtered to remove weak candidates when possible, aggregated across observations, and sorted into bounded verification tasks.

**Output.** Candidate claims, accepted/rejected/failed semantic classifications, normalized claims with occurrence data, and verification tasks containing query text, claim keys, occurrences, and counts.

**Important details.** Verification planning caps tasks at 12 and orders them by predicate priority before using the budget. Repeated claims are deduplicated while preserving source observations and strongest confidence.

**Discoverability/engagement value.** It makes important factual assertions explicit enough to trace, verify, and compare.

**False-positive protection.** Generic marketing language is not automatically treated as a verifiable fact. Candidate, normalized, and corroborated claims remain separate states.

**Failure behavior.** If semantic filtering fails for any candidates, the orchestrator continues with the deterministic candidate set and records a semantic-classification limitation.

### 6.10 `external-evidence`

**Purpose and why it exists.** Some factual claims benefit from comparison with public sources. This skill acquires bounded evidence; it does not decide truth.

**Input.** Verification tasks and an optional Gemini search provider.

**Processing.** It builds quoted entity/predicate/value queries, asks the configured provider for candidate URLs, caps queries and candidates, retrieves HTML with GET/follow-redirect behavior, rejects non-HTML content, extracts external facts, resolves entities, and creates traceable evidence records.

**Output.** Evidence records with claim key, normalized entity, predicate, value, source, URL, title, source type, and confidence; source lists; facts considered; and limitations.

**Important details.** Discovery caps at 12 queries, 3 results per query, and 24 total candidates. Retrieval defaults to 3 pages per claim and 12 total external pages, with an 8-second fetch timeout. External fact extraction currently recognizes phone, email, price, and availability. Entity resolution uses normalized names, token similarity, and domain similarity.

**Discoverability/engagement value.** It adds traceable context for claims that automated systems may encounter outside the audited site.

**False-positive protection.** Evidence is associated with a claim key and entity match before extraction is accepted. **No evidence found is not proof that a claim is false.** External disagreement is later reported as a consistency risk, not automatic proof that the website is wrong.

**Failure behavior.** Missing provider, provider errors, unavailable pages, timeouts, unsupported content types, and absent matches produce empty or partial evidence plus limitations. The core deterministic audit remains usable.

### 6.11 `corroboration`

**Purpose and why it exists.** Evidence acquisition and evidence interpretation are separate responsibilities. This skill compares website claims and external evidence only after both exist.

**Input.** Verification tasks or facts and normalized external evidence records.

**Processing.** It compares normalized entity/predicate/value triples, counts unique supporting sources, identifies external value variants, assigns statuses, and creates findings for conflicting variants when confidence supports a consistency risk.

**Output.** Per-fact results marked `corroborated`, `conflicting`, `unverified`, or `not_checked`, source counts, confidence, variants, conflicting variants, unsupported facts, and corroboration findings.

**Important details.** Three unique sources produce confidence 0.9, two produce 0.75, one produces 0.5, and no matching sources produce 0. Different entity names, predicates, or values are not silently merged.

**Discoverability/engagement value.** It prevents a website claim from being presented as externally supported unless relevant evidence actually matches it.

**False-positive protection.** **No evidence is unresolved, not contradictory.** Different products, variants, regions, currencies, predicates, or time contexts must be comparable before disagreement is meaningful.

**Failure behavior.** Corroboration is wrapped independently; unavailable external evidence produces no corroboration findings and a limitation rather than a mass contradiction.

### 6.12 `recommendation-engine`

**Purpose and why it exists.** Findings describe problems; site owners need prioritized actions. This skill converts findings and supported opportunities into actionable recommendation objects.

**Input.** The aggregated audit result, findings, evidence, and supported opportunities.

**Processing.** It computes priority from finding severity, scope, and confidence; validates recommendation shape; deduplicates findings and recommendations; merges evidence; and supports deterministic recommendations, proactive recommendations, and an LLM-assisted recommendation path.

**Output.** Recommendation objects internally contain a finding or proactive ID, priority, action, rationale, expected impact, implementation guidance, and evidence claims. The final public report currently emits the recommendation `action` strings.

**Important details.** Medium sitewide findings with confidence at least 0.85 can become high priority; low sitewide findings with confidence at least 0.9 can become medium. Priority must not arbitrarily inflate severity. The LLM path permits two model attempts before deterministic fallback.

**Discoverability/engagement value.** It connects technical observations to concrete improvements in accessibility, clarity, entity representation, navigation, and evidence quality.

**False-positive protection.** Recommendations must originate from a finding or supported proactive opportunity; the engine is not intended to invent defects to increase output volume.

**Failure behavior.** If LLM generation fails, deterministic recommendations are attempted. If normal generation and validation fail, finding-native emergency recommendations use each finding's existing suggested action.

### 6.13 `recommendation-validator`

**Purpose and why it exists.** Generated language can introduce unsupported URLs, metrics, priorities, or unsafe actions. This skill is the final quality and safety boundary before report output.

**Input.** The aggregated audit result and generated recommendation objects.

**Processing.** It validates finding/proactive references, action schema, evidence URLs, metric claims, priority versus finding severity, duplicate keys, scope, and blocked read/write/authentication language. It returns accepted, rejected, and validation statistics.

**Output.** A validated recommendation set and rejection statistics. Only accepted recommendations enter the final report.

**Important details.** It blocks patterns involving deletion/destruction, SQL/database execution, disabling security, and login/authentication actions. It rejects unsupported URLs and metrics and prevents critical priority from being attached to a non-critical finding.

**Discoverability/engagement value.** It keeps recommendations mechanism-sound, evidence-grounded, and safe to present to a website owner.

**False-positive protection.** It rejects recommendations that cannot be traced to the audit's evidence or scope rather than allowing plausible-sounding unsupported actions.

**Failure behavior.** Rejected generated recommendations cause the orchestrator to fall back to validated emergency finding actions. Invalid recommendations are retained in internal validation information but are not emitted as final actions.

## 7. Audit Orchestrator: Detailed Lifecycle

`skills/audit-orchestrator/scripts/audit-orchestrator.js` is the single entrypoint implementation.

### 7.1 URL validation

`validateTargetUrl` requires a non-empty string, parses it with `URL`, and accepts only `http:` and `https:`. The normalized URL becomes the report's `site` value.

### 7.2 Crawl and access

The orchestrator invokes `crawlSite`. Crawl failure is isolated with an empty `CrawlContext` fallback and a coverage limitation. It then runs `auditCrawlAccess` as one of the deterministic audit stages so discovered blocked/error pages can produce an evidence-backed access finding.

### 7.3 Site understanding

`understandSite` consumes the crawl context and returns site type, confidence, brand signal, primary entity types, page type counts, and page classifications. If this stage fails, downstream stages still run with an unknown profile.

### 7.4 Deterministic audits

`auditCrawlAccess`, `auditMachineReadability`, `auditEntityFacts`, `auditStructuredData`, `auditFreshnessConsistency`, and `auditEngagement` run against the same crawl context. The latter five specialist operations are launched together with `Promise.all`; the access audit is also prepared in that deterministic stage.

### 7.5 Entity facts and claim processing

Entity facts provide structured factual observations and initial candidate claims. The semantic claim filter classifies candidates. When it fails, the orchestrator uses the candidate claims rather than abandoning the claim pipeline. Facts and accepted claims are aggregated into normalized claims.

### 7.6 Verification tasks

The verification planner filters usable claims, sorts them by predicate priority and occurrence count, and limits them to 12 tasks. Each task includes a quoted query, claim key, occurrences, and source observations.

### 7.7 External evidence

The orchestrator builds evidence queries, uses the Gemini Google Search grounding provider, discovers candidate URLs, retrieves bounded HTML pages, attaches extracted text, and extracts entity-scoped evidence. No verification tasks means external corroboration is skipped with an explicit limitation.

### 7.8 Corroboration

Corroboration runs after evidence acquisition. It compares verification tasks with evidence and returns supported, conflicting, unverified, or not-checked observations. External evidence failure is isolated and does not invalidate deterministic findings.

### 7.9 AuditResult and aggregation

`createAuditResult` validates finding contracts, groups repeated observations by finding identity, merges evidence pages and notes, preserves strongest confidence, computes severity statistics, attaches site profile/crawl data, and merges limitations.

### 7.10 Recommendations and validation

The orchestrator first asks the LLM recommendation chain for structured recommendations. The recommendation engine itself validates model schema and grounding. The orchestrator then performs final validation. If generated recommendations are absent or rejected, it uses deterministic or emergency finding-native actions.

### 7.11 Final report

The internal camelCase `suggestedAction` is mapped to the public `suggested_action` field. Recommendations are currently emitted as action strings. The report is returned to the CLI, which writes `audit-report.json` in the current working directory.

### 7.12 Timeouts and partial reports

The top-level `runAudit` races the pipeline against a 250-second timeout. On timeout it returns a report with zero findings, null site profile/crawl data, empty recommendations, and a limitation stating that the audit stopped at the global limit. This is intentionally explicit partial output, not a claim that the site has no issues.

## 8. End-to-End Example

Given:

```text
https://example.com/
```

The conceptual flow is:

```text
Target URL
  -> robots.txt and sitemap discovery
  -> bounded same-origin crawl
  -> page acquisition and source/rendered extraction
  -> page classification: homepage, product, category, article, etc.
  -> product pages distinguished from catalog/listing pages
  -> entity facts and JSON-LD facts extracted
  -> deterministic claims normalized and semantically filtered
  -> high-value claims become verification tasks
  -> bounded external candidates and pages acquired, when configured
  -> entity-aware corroboration compares matching facts
  -> deterministic findings aggregated with evidence
  -> recommendations generated and validated
  -> audit-report.json written at the repository working directory
```

The example is intentionally site-independent. The implementation uses content, URL, schema, page-role, and entity signals rather than a fixed list of Adobe pages.

## 9. Crawling and Acquisition

### HTTP-first and browser fallback

`crawl-access` uses Playwright's request context for HTTP acquisition. It parses HTML with Cheerio and records source HTML/text, title, metadata, headings, links, canonical URL, and JSON-LD. If the response is accessible but appears to be a JavaScript-only shell, it opens a headless Chromium page, waits for DOM content and a short network-idle opportunity, and records rendered content. JavaScript itself is not treated as a defect; the machine-readability audit evaluates the observable source/render relationship.

### Robots and same-origin rules

The crawler requests `/robots.txt`, builds a robots policy with `robots-parser`, and checks each candidate before acquisition. It does not bypass disallow rules. If robots cannot be retrieved, it proceeds under the runtime's safe policy and records the limitation. Only HTTP/HTTPS URLs within the target origin are queued. Binary/media/script/style/resource paths are excluded from page analysis.

### Sitemap discovery

Sitemap URLs declared in `robots.txt` are accepted when same-origin. The crawler also checks `/sitemap.xml` and `/sitemap_index.xml`, then adds eligible locations as sitemap candidates. Discovery source is preserved as `seed`, `sitemap`, or `page_link`.

### Priority and queue

The priority queue favors the homepage, sitemap, product/brand/guide paths, pricing/service/about/contact/company paths, and deprioritizes category/tag/search/account-like paths. It deduplicates normalized URLs and stops adding candidates after the queue limit.

### Crawl depth versus maximum pages

These are different controls:

- **Maximum depth** limits how many link hops away from a seed a page may be. With depth 4, the crawler can inspect the seed at depth 0 and links reached through up to four hops.
- **Maximum pages** limits the total number of page results acquired. With the default of 30, the crawler stops after 30 page attempts/results even if more pages remain within depth 4.

A site can therefore have pages within the depth limit that are not visited because the page budget, queue budget, robots policy, or prioritization stopped discovery. Increasing depth alone does not solve entity identity or evidence-quality problems.

### Runtime settings

These environment variables are read at module load time:

| Variable                        | Default | Meaning                           |
| ------------------------------- | ------: | --------------------------------- |
| `CRAWL_MAX_PAGES`               |    `30` | Maximum page results in the crawl |
| `CRAWL_MAX_DEPTH`               |     `4` | Maximum link depth                |
| `CRAWL_HTTP_TIMEOUT_MS`         | `12000` | HTTP acquisition timeout          |
| `CRAWL_BROWSER_TIMEOUT_MS`      | `15000` | Browser navigation timeout        |
| `CRAWL_MIN_REQUEST_INTERVAL_MS` |  `1500` | Minimum interval between requests |
| `CRAWL_MAX_QUEUE_CANDIDATES`    |   `500` | Maximum queued candidates         |

The crawler applies backoff after HTTP 429 or 503 responses and resets the backoff after successful accessible pages. It can use a local cache with ETag/Last-Modified metadata; cache failures are non-fatal.

### Access states and evidence coverage

Pages are represented as accessible, blocked/error, or crawl-error/no-response-like states. Failed or blocked pages can support a crawl-access finding about inability to inspect them, but their absent content is not treated as evidence that the page is empty or defective. Incomplete coverage is preserved in crawl statistics and limitations.

## 10. Site Understanding and Page Classification

Page classification exists so a product structured-data rule, freshness rule, or engagement comparison is not blindly applied to every URL.

The implementation can emit these page types from its scoring model:

- `homepage`
- `product`
- `pricing`
- `article`
- `category`
- `contact`
- `about`
- `documentation`
- `unknown`

Shared eligibility can additionally identify non-standalone roles such as listing, fragment, modal, tracking, or utility. Those roles help prevent weak URL-only classifications.

The classifier combines content terms, titles, URL patterns, Product JSON-LD, purchase signals, price signals, product-detail terms, collection query parameters, canonical information, and repeated product-card signals. For example, `/products/` by itself is not enough to establish a product-detail page; a catalog page can instead become `category` when collection evidence dominates.

## 11. Entity and Fact Model

The audit distinguishes:

- **Organization/brand:** the site-level company or named organization.
- **Product:** a specific product or software/product offering.
- **Service:** a specific service entity.
- **Category/listing:** a collection of products or content, not automatically one product.
- **Variant:** a contextual variation that may require separate comparison.
- **Regional context:** a market or locale that can affect price and availability.
- **Currency and temporal context:** dimensions that affect whether two values are comparable.

### Product identity rule

> **Brand identity must not be used as a substitute for product identity.**

A product fact is tied to product-specific signals such as its normalized product name and URL/path context. Structured Product entities are preferred when present; page titles and product-detail context provide fallback evidence. Organization facts remain attached to organization entities rather than being used as product identity.

This prevents the invalid comparison:

```text
Brand A
  Product X -> $20
  Product Y -> $40
```

from becoming:

```text
Brand A -> conflicting price
```

A price conflict is meaningful only when the observations resolve to the same product context and comparable region/currency context, are observed from distinct sources or source types, and contain incompatible normalized values. Different products, regions, or currencies do not automatically produce a contradiction.

Facts preserve source, source type, scope, confidence, and normalized value. This lets downstream claim aggregation and corroboration retain traceability instead of reducing all text to an unscoped brand-level statement.

## 12. Structured Data Audit

Structured data is inspected from JSON-LD blocks found during acquisition. The audit recognizes Product and organization-related types and evaluates Product properties such as `name`, `image`, `description`, `brand`, and `offers` according to the implementation's required/recommended distinction.

The applicability sequence is:

```text
Observed page
  -> meaningful and standalone?
  -> actual page/entity role?
  -> adequate classification confidence?
  -> Product detail page rather than listing/search/category/utility?
  -> Product JSON-LD present and complete?
  -> finding only when the issue is applicable and evidenced
```

`SD-001` is not emitted for every page without Product JSON-LD. Catalog, category, collection, search, deal, cart, checkout, account, help, and similar paths are excluded from the product-detail absence check. This eligibility-first design matters because an absent Product schema on a listing page is not the same problem as an absent Product schema on a genuine standalone product page.

Findings carry affected pages, representative pages, schema-type metrics, missing properties, extraction method, confidence, severity, and suggested action.

## 13. Machine Readability

The acquisition layer stores source HTML/text and rendered HTML/text when available. The machine-readability skill compares these representations and examines headings.

Current checks include:

- meaningful content appearing only after rendering;
- an empty client-side root shell paired with rendered content;
- a large source/rendered text gap;
- no rendered text;
- heading structure checks where heading levels are available.

The implementation's large-gap finding requires at least 50 rendered words and a rendered-minus-source gap of at least 60%. A page with low text is not automatically bad: the skill operates on accessible pages and reports evidence only when the relevant conditions are present. Browser-acquired pages may not retain the original source representation, which limits source/render comparisons for those pages and is reflected by the available evidence.

## 14. Freshness and Temporal Context

Freshness is evaluated through observable signals, not by guessing publication policy. The skill recognizes events, conferences, schedules, hours, availability, stock state, deadlines, expiration language, promotions, and contextual news/announcement/update language.

It extracts conservative date formats including ISO dates, common numeric dates, and month-name dates. A finding is emitted when the content appears time-sensitive but no explicit date or equivalent temporal context was detected.

> **Missing a detected date does not automatically prove that content is stale.**

The finding means that a reader or automated system may have difficulty determining the validity window of content that appears time-dependent. Generic words such as `latest` are intentionally insufficient without additional temporal context.

## 15. On-Site Engagement Audit

The engagement skill builds a graph from links extracted on accessible pages. It normalizes targets, ignores self-links, counts inbound sources, and joins pages to their inferred types.

The current implementation can report:

- weak connectivity when a target has zero detected inbound links, at least three comparable pages exist, and comparable pages have a positive median inbound-link count;
- a sitemap-only discovery concern when the page has no inbound link, was discovered from a sitemap, has a meaningful supported type, and enough comparable pages are connected;
- contextual limitations explaining that the graph covers only the bounded crawl.

It excludes homepages, localization roots, and known implementation fragments from sitemap-only analysis. It does not claim to measure every interaction, conversion, visual affordance, or chatbot behavior.

The important interpretations are:

- sitemap presence != proof of good engagement;
- zero inbound links in a bounded crawl != definitive proof of an orphan page.

The result is a relative, evidence-backed opportunity to add relevant contextual links from already connected pages.

## 16. Claim Extraction and Verification

The claim pipeline keeps page observations separate from truth judgments:

```text
Readable text and entity facts
  -> deterministic candidate claims
  -> normalization as subject/predicate/object
  -> semantic classification/filtering
  -> aggregation of repeated observations
  -> bounded verification tasks
  -> external evidence
  -> corroboration status
```

Deterministic claim extraction currently recognizes:

- founded or established year;
- headquarters;
- authorized or official partner;
- product/service statements such as manufacture, sell, provide, or offer;
- certifications such as ISO, SOC, or PCI wording;
- awards.

Normalization preserves subject, predicate, object, and source. Aggregation deduplicates repeated observations but preserves occurrence source, source type, value, page type, and confidence. The verification planner prioritizes predicates and limits the task set to 12, so external work is spent on a bounded set of useful claims.

A candidate claim is not a verified claim. The system does not decide truth during extraction.

## 17. External Evidence

External evidence is a complete bounded pipeline rather than an unqualified claim that the system simply searches the web:

```text
verification task
  -> quoted entity/predicate/value query
  -> Gemini Google Search grounding candidate URLs
  -> bounded candidate list
  -> read-only HTML retrieval
  -> external text/fact extraction
  -> entity resolution
  -> traceable evidence record
  -> corroboration
```

The current provider is `gemini-search-provider.js`, which uses the Google GenAI client with the `googleSearch` tool and reads grounding URLs from Gemini response metadata. Retrieval uses `fetch`, follows redirects, accepts HTML/XHTML only, applies an 8-second default timeout, and performs GET requests without authentication.

Budgets are intentionally bounded:

- up to 12 verification queries;
- up to 3 search results per query;
- up to 24 total discovered candidates;
- up to 3 retrieved pages per claim;
- up to 12 total external pages;
- up to 12 facts considered by the external evidence adapter.

External fact extraction recognizes explicit phone, email, price, and availability observations. Entity resolution normalizes names, compares token overlap, and considers exact or registrable-domain relationships. Evidence records retain claim key, entity, predicate, value, source, URL, title, source type, and confidence.

External evidence is not treated as automatic truth. A source must be relevant to the same entity and predicate. No evidence found means unresolved or unverified, not false. A failed provider or unavailable page is preserved as a limitation.

## 18. Corroboration

Corroboration compares normalized website claims with external evidence using entity, predicate, and value. It reports:

- `corroborated` when matching evidence exists;
- `conflicting` when relevant external variants disagree;
- `unverified` when external evidence exists but does not match the claim;
- `not_checked` when no external evidence was available.

Confidence is based on unique matching source count: one source is 0.5, two are 0.75, and three or more are 0.9. A source count is not a truth guarantee; it is an evidence-strength signal.

The comparison is context-aware in the sense that it requires matching normalized entity and predicate before value differences are considered. Different products, variants, currencies, regions, or time periods should not be treated as contradictions unless the claim/evidence representation makes those contexts comparable. The implementation also explicitly phrases external disagreement as a consistency risk rather than proof that the website is wrong.

## 19. Recommendation Engine

Recommendations are generated through a layered strategy:

1. LLM-assisted structured recommendations using Gemini when configured.
2. Deterministic recommendation generation from findings and supported opportunities.
3. Finding-native emergency actions using each finding's existing `suggestedAction`.

Recommendations include internal fields for origin, priority, action, rationale, expected impact, implementation guidance, and claims. Priority starts from finding severity and can be adjusted only for high-confidence sitewide scope under explicit thresholds. The engine deduplicates repeated finding/action combinations and merges evidence where appropriate.

The final CLI report currently exposes the recommendation action strings rather than the full internal recommendation objects. Proactive recommendations are supported by the internal engine when a meaningful evidence-backed opportunity is supplied; the audit does not promise that every run will produce one.

## 20. Recommendation Validator

The validator is a safety and quality boundary. It checks:

- finding or proactive origin;
- recommendation schema and required strings;
- evidence URL references;
- metric claims against actual finding metrics;
- priority against finding severity;
- duplicate recommendation keys;
- action scope and mechanism alignment;
- read-only safety.

It rejects recommendations that reference unsupported URLs or metrics, claim a critical priority without a critical finding, or contain blocked patterns such as deleting/destroying data, executing SQL/database actions, disabling security, or logging in/authenticating.

If any generated set fails final grounding validation, the orchestrator does not pass it through. It validates the emergency finding-native actions instead. This means LLM output can improve wording without being allowed to introduce unsupported or destructive instructions.

## 21. Final Report

`run-site-audit.js` writes the final report to:

```text
<current working directory>/audit-report.json
```

The report is JSON and currently has this shape:

```json
{
  "version": "1.0",
  "site": "https://example.com/",
  "audited_at": "2026-09-11T12:00:00.000Z",
  "summary": {
    "total_findings": 1,
    "critical": 0,
    "high": 1,
    "medium": 0,
    "low": 0,
    "info": 0
  },
  "siteProfile": {
    "siteType": "ecommerce",
    "primaryEntityTypes": ["Organization", "Product"],
    "pageTypeCounts": { "homepage": 1, "product": 2 }
  },
  "crawl": {
    "pagesAttempted": 3,
    "pagesAccessible": 3,
    "pagesBlocked": 0,
    "pagesFailed": 0
  },
  "findings": [
    {
      "id": "MR-001",
      "title": "Important content depends on client-side rendering",
      "description": "Meaningful page content appears only after rendering. Systems that rely on initial HTML may miss important information.",
      "severity": "high",
      "confidence": 0.9,
      "evidence": {
        "pagesChecked": 1,
        "affectedPages": ["https://example.com/product/example"],
        "representativePages": ["https://example.com/product/example"],
        "method": "Compared source HTML-derived text with rendered DOM text.",
        "metrics": {
          "sourceWordCount": 3,
          "renderedWordCount": 80,
          "textGap": 0.963
        },
        "notes": []
      },
      "suggested_action": "Expose important content in the initial HTML where practical."
    }
  ],
  "recommendations": [
    "Expose important content in the initial HTML where practical."
  ],
  "limitations": ["External corroboration was unavailable."]
}
```

The example is illustrative of the actual field names; counts, timestamps, URLs, and findings depend on the audited site.

### Top-level fields

- `version`: report format version, currently `1.0`.
- `site`: normalized target URL.
- `audited_at`: crawl timestamp used by the audit result.
- `summary`: total finding count and counts for `critical`, `high`, `medium`, `low`, and `info`.
- `siteProfile`: inferred site type, primary entity types, and page type counts; can be `null` after a site-understanding failure or timeout.
- `crawl`: attempted, accessible, blocked, and failed page counts; can be `null` in a timeout report.
- `findings`: aggregated evidence-backed issues.
- `recommendations`: validated action strings for the current public report format.
- `limitations`: optional list of coverage, semantic, external-evidence, or timeout limitations.

### Finding fields

Each finding is built from the shared finding contract and the public report maps the internal action field to `suggested_action`:

- `id`: stable rule identifier such as `CA-001`, `MR-001`, `SD-001`, or `EN-001`.
- `title`: concise issue name.
- `description`: combined problem and why-it-matters explanation when present.
- `severity`: one of `critical`, `high`, `medium`, `low`, or `info`.
- `confidence`: number from 0 to 1.
- `evidence`: pages checked, affected/representative pages, method, metrics, and notes.
- `suggested_action`: a read-only recommendation for the site owner.

`rootCause` and `verification` exist in the internal finding contract but are not currently copied into the public `findings` mapping by the orchestrator.

## 22. Severity Model

The reference model defines:

| Severity   | Meaning                                                                                     |
| ---------- | ------------------------------------------------------------------------------------------- |
| `critical` | Prevents meaningful automated discovery or blocks a major user journey                      |
| `high`     | Significantly reduces machine discoverability, factual reliability, or visitor continuation |
| `medium`   | Meaningful weakness while alternative signals remain available                              |
| `low`      | Optimization opportunity with limited immediate impact                                      |
| `info`     | Valid informational observation in the shared severity contract                             |

The shared `calculateSeverity` helper uses impact, scope, and confidence thresholds for supported callers, but individual audits also assign severity based on their specific evidence rules. The README does not treat severity as a universal numeric score.

## 23. Confidence and Limitations

Confidence is a bounded number from 0 to 1 attached to findings and observations. It expresses how strongly the implementation's evidence and classification support the result; it is not a probability that an external truth has been proved.

Limitations preserve uncertainty caused by:

- robots restrictions or missing robots retrieval;
- blocked, failed, or non-HTML pages;
- bounded page/depth/queue budgets;
- browser/source representation limits;
- semantic claim-classification failure;
- missing Gemini configuration or Gemini timeout;
- unavailable external evidence;
- global timeout or failed specialist stages.

The report records these instead of silently treating incomplete observation as a complete-site conclusion.

## 24. LLM and Gemini Configuration

The project uses `@google/genai` in two optional areas:

1. `recommendation-engine/scripts/llm-recommendation-engine.js` requests structured recommendations.
2. `external-evidence/scripts/gemini-search-provider.js` uses Gemini's Google Search grounding tool to discover public candidate evidence URLs.

Shared configuration is loaded from process environment variables. Node versions that provide `process.loadEnvFile()` can load the local `.env` file; the project does not add a separate dotenv dependency.

| Variable                | Default            | Use                                                                      |
| ----------------------- | ------------------ | ------------------------------------------------------------------------ |
| `GEMINI_API_KEY`        | none               | Required for Gemini calls; absence triggers fallback/limitation behavior |
| `GEMINI_MODEL`          | `gemini-2.5-flash` | Primary Gemini model                                                     |
| `GEMINI_FALLBACK_MODEL` | `gemini-2.0-flash` | Secondary recommendation model                                           |
| `GEMINI_TIMEOUT_MS`     | `30000`            | Recommendation request timeout                                           |

The shared Gemini client also supplies the configured model and timeout to the search provider. Do not commit secrets. A local `.env` can contain values such as:

```text
GEMINI_API_KEY=replace-with-your-key
GEMINI_MODEL=gemini-2.5-flash
GEMINI_FALLBACK_MODEL=gemini-2.0-flash
GEMINI_TIMEOUT_MS=30000
```

Gemini is not required for the deterministic crawl and audit stages. Without a key, recommendation generation falls back to deterministic/emergency actions and external evidence is unavailable or recorded as a limitation. LLM responses are parsed as JSON, schema-validated, grounding-validated, and rejected if unsupported.

## 25. Dependencies

The versions below are declared in `package.json` and locked by `package-lock.json`.

| Dependency      | Type        | Why it is used                                                                                                                  |
| --------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `@google/genai` | Runtime     | Gemini recommendation generation and Gemini Google Search grounding for external candidate discovery                            |
| `cheerio`       | Runtime     | Parses acquired HTML for text, titles, headings, links, canonical URLs, and JSON-LD                                             |
| `playwright`    | Runtime     | HTTP request context and optional headless Chromium rendering; also provides request/navigation timeouts                        |
| `robots-parser` | Runtime     | Evaluates crawler permissions from `robots.txt`                                                                                 |
| `zod`           | Runtime     | Declared runtime schema-validation dependency; the repository also contains explicit recommendation/finding contract validators |
| `eslint`        | Development | Invoked by the `lint` npm script                                                                                                |
| `vitest`        | Development | Runs the repository's `.test.js` test suite                                                                                     |

Install declared dependencies with:

```powershell
npm install
```

Playwright may require browser binaries on a fresh machine. Install the Chromium binary used by the browser fallback with:

```powershell
npx playwright install chromium
```

## 26. Environment Setup

A fresh-machine setup on Windows is:

```powershell
# 1. Open the repository directory
Set-Location "D:\adobe uni hackathon 2026\brand-ai-readiness-audit"

# 2. Install the locked JavaScript dependency graph
npm install

# 3. Install the browser used for rendered-page fallback
npx playwright install chromium

# 4. Optionally configure Gemini in .env or in the process environment
# GEMINI_API_KEY=...
# GEMINI_MODEL=gemini-2.5-flash
# GEMINI_FALLBACK_MODEL=gemini-2.0-flash
# GEMINI_TIMEOUT_MS=30000

# 5. Run the discovered Vitest tests
npm test

# 6. Audit a public site
npm run audit:site -- https://example.com/
```

The `.env` file is ignored by git. No model weights are stored in the repository.

## 27. Command Reference

### Install

```powershell
npm install
```

Installs the dependencies recorded in `package-lock.json`.

### Install browser binaries

```powershell
npx playwright install chromium
```

Installs Chromium for the browser fallback. HTTP-only audits may not need to launch it, but the acquisition code can use it when source HTML appears insufficient.

### Run an audit through the convenience CLI

```powershell
npm run audit:site -- https://example.com/
```

Runs `node run-site-audit.js`, prints progress/summary/findings/limitations, and writes `audit-report.json` in the current working directory.

### Run the orchestrator CLI directly

```powershell
npm run audit -- https://example.com/
```

Runs `node skills/audit-orchestrator/scripts/audit-orchestrator.js`. It prints the final JSON report to the console; it does not itself write the root report file.

### Run tests

```powershell
npm test
```

Runs `vitest run`. Vitest discovers the repository's `.test.js` files, including contract and regression tests for page eligibility/classification, structured-data eligibility, entity scoping, and shared contracts. The repository also contains additional `test-*.js` executable test files under skill script directories; those are useful focused tests, but their naming is not the default Vitest `.test.js` pattern.

### Run lint

```powershell
npm run lint
```

Invokes `eslint .`. The package script exists, but this checkout does not contain an ESLint configuration file at the root, so whether it completes successfully depends on the installed ESLint configuration/environment.

### Validate the marketplace

```powershell
npm run validate
```

The package script is intended to run `node scripts/validate-marketplace.js`. In the current repository inventory, `scripts/validate-marketplace.js` is missing, so this command currently fails with a missing-module error. This is documented here rather than hidden or represented as a working validation capability.

## 28. How to Run a Website Audit

Use:

```powershell
npm run audit:site -- https://example.com/
```

The command:

1. reads the URL argument;
2. calls `runAudit` from the orchestrator;
3. prints crawl/audit/recommendation progress;
4. prints total and per-severity finding counts;
5. prints up to the first two affected pages per finding in the console;
6. prints recommendation action strings and limitations; and
7. writes the full JSON response to `audit-report.json`.

The output path is resolved from the process working directory. From the repository root this is `D:\adobe uni hackathon 2026\brand-ai-readiness-audit\audit-report.json` on the current Windows workspace.

## 29. How to Read the Final Report

For a non-expert reviewer:

- Start with `summary` to see whether the audit found high-impact issues and how many findings exist.
- Use `crawl` to check whether the conclusion is based on a small or substantial accessible sample.
- Use `siteProfile` to see what site/page interpretation drove eligibility decisions.
- Open each `findings` item. `title` and `description` explain the issue; `severity` and `confidence` show priority and evidence strength.
- Read `evidence.affectedPages` and `representativePages` to see where the observation occurred.
- Read `evidence.method`, `metrics`, and `notes` to understand how the system reached the observation.
- Use `suggested_action` as the recommended change; it is reported only, never applied.
- Read `recommendations` for the final validated action list.
- Read `limitations` before interpreting “no finding” as proof that the whole site is healthy.

A report with zero findings can mean the bounded accessible sample produced no supported finding. It does not prove that every page, private area, or un-crawled path is defect-free.

## 30. Testing Strategy

The project uses Vitest for `.test.js` tests and also contains focused executable Node test scripts.

The test inventory covers:

- URL normalization and URL contracts in `shared/`;
- finding and evidence contracts;
- `AuditResult` aggregation and schema behavior;
- site understanding and page classification;
- shared page eligibility;
- product-versus-listing classification;
- entity fact scoping, including different products under the same brand, product price conflicts, region/currency context, and organization facts;
- structured-data eligibility and Product schema behavior;
- machine-readability thresholds;
- freshness signals and the no-date-does-not-equal-stale rule;
- engagement graph behavior and conservative sitemap-only handling;
- deterministic claim extraction, normalization, aggregation, semantic filtering, and verification planning;
- external candidate discovery, retrieval budgets, external fact extraction, entity resolution, Gemini search provider behavior, and failure paths;
- corroboration statuses and conflicting evidence;
- recommendation schema, priority/deduplication, proactive recommendations, Gemini connection behavior, and recommendation fallback;
- recommendation-validator grounding and safety checks;
- orchestrator claim-pipeline and recommendation-fallback behavior.

The standard check is:

```powershell
npm test
```

For a focused `.test.js` file, use for example:

```powershell
npx vitest run skills/entity-fact-audit/scripts/test-entity-facts-scoping.test.js
```

Some repository tests are named `test-*.js` rather than `*.test.js`; they can be run directly with Node when needed, for example:

```powershell
node skills/entity-fact-audit/scripts/test-entity-facts.js
node skills/audit-orchestrator/scripts/test-orchestrator.js
```

## 31. Failure Handling and Resilience

The system is designed so one unavailable dependency does not unnecessarily destroy the complete audit:

| Failure                             | Behavior                                                                                                |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Invalid/missing URL                 | The orchestrator rejects it before crawling                                                             |
| `robots.txt` disallows a page       | The page is not acquired; the limitation is preserved                                                   |
| `robots.txt` cannot be retrieved    | Crawl proceeds under safe policy and records a limitation                                               |
| HTTP non-HTML/unsuccessful response | Page becomes blocked/error state; content is not invented                                               |
| Request timeout/network failure     | Page becomes crawl-error state; other queued pages can continue                                         |
| Browser rendering failure           | HTTP result is preserved when available; browser error is recorded                                      |
| Cache failure                       | Successful acquisition remains successful                                                               |
| Site-understanding failure          | Unknown profile fallback; later stages continue                                                         |
| Individual audit-stage failure      | Empty findings plus stage-specific limitation                                                           |
| Semantic classifier failure         | Candidate claims continue through deterministic path; limitation added                                  |
| Gemini key/API/timeout failure      | External evidence or LLM recommendations fall back/limit rather than crash the full deterministic audit |
| External evidence unavailable       | Corroboration is unresolved/limited, not contradictory by default                                       |
| Invalid LLM JSON/schema             | Attempt rejected; next model/deterministic/emergency layer used                                         |
| Recommendation grounding failure    | Generated set rejected; validated finding-native actions used                                           |
| Global timeout                      | Partial report with explicit 250-second limitation                                                      |

## 32. False Positive Strategy

False-positive control is a primary design goal:

- **Eligibility before auditing:** page role, entity role, meaningfulness, and standalone status gate specialized checks.
- **Product versus brand scope:** brand/organization identity never substitutes for a specific product identity in price comparisons.
- **Comparable context:** region and currency differences do not automatically become price contradictions.
- **Structured-data applicability:** missing Product JSON-LD on a category/listing/search page is not automatically `SD-001`.
- **Access versus content:** blocked or failed pages are not treated as empty or malformed content.
- **Missing date versus stale:** temporal risk requires time-sensitive evidence; no date alone is insufficient.
- **Sitemap-only versus orphan proof:** sitemap-only discovery and zero inbound links are interpreted relative to comparable connected pages and bounded crawl coverage.
- **No evidence versus contradiction:** external search failure or no match yields unresolved/unverified status, not a false claim finding.
- **Entity resolution:** external evidence must match the relevant entity before it can support or conflict with a claim.
- **Recommendation validation:** generated actions must refer to real findings/evidence and stay read-only.
- **Limitations:** incomplete coverage remains visible instead of being converted into certainty.

These controls matter because broad heuristic coverage without scope can produce impressive-looking but unreliable reports. The project favors fewer, better-supported findings.

## 33. Safety and Guardrails

The marketplace is recommend-only and read-only:

- only public HTTP/HTTPS pages are targeted;
- crawling stays same-origin;
- `robots.txt` is respected and not bypassed;
- requests are rate-limited and bounded;
- page, depth, queue, timeout, and external-evidence budgets apply;
- acquisition uses GET-style retrieval and does not submit forms;
- no authentication or private-area access is attempted;
- no website content, resources, or configuration are modified;
- recommendations are printed/stored, never executed;
- validator blocks destructive, database-execution, security-disabling, and authentication actions in generated recommendations.

The crawler's user agent is `AIBrandAuditBot/1.0 (+https://yourdomain.com/bot-info)`. The external evidence retriever uses a separate read-only user agent string and accepts HTML/XHTML only.

## 34. Runtime and Resource Controls

The principal controls are:

| Resource                    |                     Default |
| --------------------------- | --------------------------: |
| Crawled pages               |                          30 |
| Crawl depth                 |                           4 |
| Queued candidates           |                         500 |
| HTTP timeout                |                   12,000 ms |
| Browser timeout             |                   15,000 ms |
| Request interval            |                    1,500 ms |
| Global audit timeout        |                  250,000 ms |
| Verification tasks          |                          12 |
| External queries            |                          12 |
| Results per query           |                           3 |
| External candidates         |                          24 |
| External pages per claim    |                           3 |
| Total external pages        |                          12 |
| External fetch timeout      |                    8,000 ms |
| Recommendation LLM timeout  |       30,000 ms per attempt |
| LLM recommendation attempts | primary plus fallback model |

These limits protect public sites, keep evaluation practical, prevent unbounded external requests, and make report completeness explicit.

## 35. Generalization to Unseen Websites

The implementation is pattern-based rather than hard-coded to Adobe pages or a fixed collection of websites. It adapts through:

- normalized URLs and same-origin discovery;
- content and path signals for site/page classification;
- structured Product/Organization/Service data when present;
- readable source and rendered content;
- shared page/entity eligibility;
- entity-aware fact and claim normalization;
- bounded priority crawling;
- explicit external entity resolution;
- deterministic fallbacks when optional semantic/LLM components fail.

Generalization is not perfect. Websites with unusual routing, opaque rendering, unsupported schema patterns, inaccessible content, or ambiguous entity names can remain unresolved. The design improves generalization by combining signals and preserving uncertainty rather than relying on one URL pattern or one model response.

## 36. Why This Is a Genuine Marketplace

The project separates responsibilities because they solve different problems:

```text
Crawler responsibility
    != page understanding responsibility
    != entity/fact responsibility
    != structured-data responsibility
    != external evidence responsibility
    != corroboration responsibility
    != recommendation responsibility
    != recommendation safety responsibility
```

Each capability has its own `SKILL.md`, scripts, tests, and clear input/output role. The orchestrator composes them into one user-facing workflow. This gives the marketplace a single simple entrypoint while retaining focused, independently testable skills that can evolve without turning the whole audit into one monolithic detector.

## 37. Mapping to the Hackathon Rubric

| Judging criterion                    | How this project addresses it                                                                                                                                                                                                    |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Detection accuracy                   | Deterministic evidence collection, page eligibility, entity-aware grouping, product/brand separation, structured-data gating, conservative freshness and engagement rules, and explicit limitations reduce unsupported findings. |
| Suggested-action quality             | Recommendations are tied to findings or supported opportunities, prioritized by severity/scope/confidence, and validated for evidence grounding and mechanism alignment.                                                         |
| Output design                        | The final report contains site, audit timestamp, severity-count summary, site profile, crawl counts, findings with IDs/severity/confidence/evidence/actions, recommendations, and limitations.                                   |
| Skill format and engineering hygiene | The repository contains one `marketplace.json`, 13 focused skills with `SKILL.md` files, one marked entrypoint, shared contracts, deterministic fallbacks, and no model weights.                                                 |
| Marketplace composition              | `audit-orchestrator` composes crawl, understanding, focused audits, claims, evidence, corroboration, recommendations, and validation rather than asking users to run every skill manually.                                       |
| AI discoverability                   | The system examines access, initial HTML, rendered content, headings, entities, structured data, facts, freshness, and corroboration mechanisms.                                                                                 |
| On-site engagement                   | The engagement skill builds an internal graph and evaluates meaningful contextual connectivity while accounting for crawl limits.                                                                                                |
| Generalization                       | Classification and evidence are based on content, schema, URL context, entity scope, and bounded fallback behavior rather than a hard-coded website list.                                                                        |
| Practical runtime and safety         | Priority and budgeted crawling, rate limits, timeouts, same-origin/robots controls, bounded external retrieval, and a global timeout keep operation practical and public-web-safe.                                               |
| Proactive improvements               | The recommendation engine supports evidence-backed proactive opportunities when supplied, but does not manufacture them when the audit has no meaningful basis.                                                                  |

## 38. Design Decisions and Engineering Highlights

### Deterministic core with optional LLM enhancement

Access, classification, extraction, eligibility, findings, and fallback recommendations remain deterministic. Gemini can improve recommendation wording and external candidate discovery, but its availability is not required for the core audit.

### Eligibility-first auditing

Page role and entity context are established before specialized checks. This is particularly important for Product structured data and engagement comparisons.

### Entity-aware fact grouping

Facts retain entity, predicate, source, and context. Product prices are not collapsed into organization prices, which prevents a common false contradiction.

### Evidence traceability

Findings preserve affected pages, representative pages, methods, metrics, and notes. External evidence retains URLs, titles, sources, claim keys, and confidence.

### Recommendation validation

LLM output is treated as untrusted generated data. Schema validation, finding/evidence grounding, priority checks, duplicate detection, and read-only safety checks happen before final output.

### Bounded public-web operation

Robots, same-origin rules, rate limits, page/depth/queue budgets, external evidence limits, and timeouts are explicit rather than hidden assumptions.

## 39. Known Limitations and Current Repository Issues

The implementation has meaningful boundaries:

- A bounded crawl cannot establish properties of every site page.
- A page blocked by robots, access controls, or network failure cannot be semantically audited by this crawler.
- Browser fallback may not preserve original source HTML, limiting direct source/render comparison for that page.
- Page and entity classification is heuristic and can remain unknown or ambiguous on unusual sites.
- Structured-data coverage focuses on the types and Product properties implemented in the audit, not every Schema.org type.
- Deterministic claim extraction recognizes a finite set of factual language patterns.
- External evidence depends on Gemini configuration and public HTML availability; no evidence is not a truth judgment.
- The external fact extractor currently covers phone, email, price, and availability rather than every possible predicate.
- The report's public recommendations field contains action strings, while richer internal recommendation metadata is used during validation but not emitted there.
- The root package version is `1.0.0`, while the marketplace manifest version is `2.0.0`; this metadata mismatch is present in the current repository.
- `npm run validate` references `scripts/validate-marketplace.js`, but that file is not present in the current repository, so the command currently fails until the missing script is supplied.
- `npm run lint` invokes ESLint, but no root ESLint configuration file was present in the inspected repository; completion depends on available configuration.

These are documented limitations, not silently converted into positive or negative audit claims.

## 40. Quick Start

```powershell
Set-Location "D:\adobe uni hackathon 2026\brand-ai-readiness-audit"
npm install
npx playwright install chromium

# Optional: set GEMINI_API_KEY in .env for Gemini recommendations
# and external evidence discovery.

npm test
npm run audit:site -- https://example.com/
Get-Content .\audit-report.json
```

For a deterministic-only run, omit `GEMINI_API_KEY`. The crawl and focused audits still run; recommendations use fallback behavior and external corroboration may be recorded as unavailable.

## 41. Final Summary

This marketplace turns a public website URL into an evidence-backed, bounded audit:

```text
Crawl -> Understand -> Detect -> Verify -> Corroborate
      -> Recommend -> Validate -> Report
```

Its central engineering choice is to preserve context at every step: access state, page role, entity identity, product context, evidence source, confidence, and limitations. That context is what lets the project address AI discoverability and on-site engagement without treating every missing signal as a defect, every external disagreement as proof, or every organization as one product.
