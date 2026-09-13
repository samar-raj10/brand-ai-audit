# Brand AI-Readiness Audit

## What this project does

This project audits a public website for AI readiness and on-site engagement.

It checks whether:

- crawlers can access and discover important pages;
- page content is available in a machine-readable form;
- products, services, and organizations are described clearly;
- structured data and time-related information are useful and consistent;
- important pages have useful internal links; and
- important website claims can be compared with public external evidence.

The audit accepts one HTTP or HTTPS URL and produces an evidence-based JSON report. It is read-only: it does not log in, submit forms, change website content, or apply recommendations.

## Dependencies and installations

Install all dependencies with:

```powershell
npm install
```

1. `@google/genai` is a runtime dependency. It provides Gemini recommendations and Google Search grounding for external evidence. Install it with `npm install @google/genai`.

2. `cheerio` is a runtime dependency. It reads HTML, text, headings, links, metadata, and JSON-LD. Install it with `npm install cheerio`.

3. `playwright` is a runtime dependency. It fetches pages and renders JavaScript-heavy pages when needed. Install it with `npm install playwright`.

4. `robots-parser` is a runtime dependency. It checks `robots.txt` rules before crawling. Install it with `npm install robots-parser`.

5. `zod` is a runtime dependency. It supports data and recommendation validation. Install it with `npm install zod`.

6. `eslint` is a development dependency. It checks code style with `npm run lint`. Install it with `npm install -D eslint`.

7. `vitest` is a development dependency. It runs the test suite with `npm test`. Install it with `npm install -D vitest`.

Playwright may also need its Chromium browser:

```powershell
npx playwright install chromium
```

## Setup and run

### 1. Open the project root

In PowerShell, open the repository directory:

```powershell
Set-Location "D:\adobe uni hackathon 2026\brand-ai-audit\brand-ai-readiness-audit"
```

Then install the packages and browser:

```powershell
npm install
npx playwright install chromium
```

### 2. Add the Gemini API key and model

Create a `.env` file in the project root. Do not commit this file.

```dotenv
GEMINI_API_KEY=your-api-key
GEMINI_MODEL=gemini-2.5-flash
GEMINI_FALLBACK_MODEL=gemini-2.0-flash
GEMINI_TIMEOUT_MS=30000
```

`GEMINI_API_KEY` enables Gemini features. `GEMINI_MODEL` selects the main model. The fallback model and timeout are optional. The deterministic crawl and audit stages still work without a Gemini key, but external evidence and LLM recommendations will be unavailable or use fallback behavior.

### 3. Run an audit

```powershell
npm run audit:site -- https://example.com/
```

This writes the final report to `audit-report.json`.

Other useful commands:

```powershell
npm test
npm run lint
npm run audit -- https://example.com/
```

## Skills

The entrypoint is `skills/audit-orchestrator/scripts/audit-orchestrator.js`. It receives the URL, calls the skills in the order needed by their data, combines their results, and writes one validated report. The specialist skills stay separate so each responsibility can be tested without running the full audit.

### 1. `audit-orchestrator`

Runs the complete audit and creates the final report. It provides one reliable entrypoint and keeps failures from one stage from stopping the whole report.

### 2. `crawl-access`

Checks `robots.txt`, discovers sitemaps, crawls same-origin pages, and records source and rendered content. It gives every other skill the same bounded and trustworthy set of observations.

### 3. `site-understanding`

Infers the site type, brand, entities, and page types. It lets later checks apply the right rules to the right pages instead of using generic assumptions.

### 4. `machine-readability`

Compares source and rendered content and checks headings and extractable text. It detects content that people can see but automated systems may not receive.

### 5. `entity-fact-audit`

Extracts facts about organizations, products, and services and checks their scope and consistency. It prevents facts about different products or entities from being incorrectly compared.

### 6. `structured-data-audit`

Checks relevant JSON-LD and other structured data, especially on eligible product pages. It makes schema findings depend on page type, avoiding missing Product data findings on category or search pages.

### 7. `freshness-consistency`

Finds time-sensitive content that lacks clear dates or temporal context. It identifies information that may be difficult to interpret without claiming that it is stale.

### 8. `engagement-audit`

Builds an internal-link graph and checks whether meaningful pages have useful paths to them. It adds the engagement view while keeping conclusions limited to the bounded crawl.

### 9. `claim-extraction`

Turns selected page facts and text into normalized claims and bounded verification tasks. It separates claim collection from truth decisions and controls the amount of external work.

### 10. `external-evidence`

Uses Gemini Search grounding and read-only retrieval to collect public evidence for selected claims. It adds traceable outside context without making external search a requirement for the core audit.

### 11. `corroboration`

Compares website claims with matching external evidence. It keeps entity, predicate, value, and source context together so missing evidence is not mistaken for a contradiction.

### 12. `recommendation-engine`

Creates prioritized actions from findings and supported opportunities. It converts technical findings into useful next steps while supporting deterministic fallback behavior.

### 13. `recommendation-validator`

Checks recommendation scope, evidence, priority, schema, and safety. It prevents generated recommendations from introducing unsupported metrics, URLs, or unsafe actions into the final report.

## Architectural Flow

### 1. `Layer 0 - Input Validation`

The orchestrator first receives a targetURL.

For example:https://example.com

It checks whether:

-URL exists or not.
-URL is a string.
-URL is valid/inalid.
-protocol is HTTP/HTTPS.

Then the actual audit begins.


### 2. `Layer 1 - Crawler`



## How the entrypoint composes the skills

The orchestrator first validates the URL and asks `crawl-access` for public page data. It then sends that shared crawl result to `site-understanding` and the deterministic audits: access, machine readability, entity facts, structured data, freshness, and engagement.

Next, entity facts and readable content go through `claim-extraction`. The resulting verification tasks are sent to `external-evidence` when Gemini is configured. `corroboration` compares the collected evidence with the claims. The orchestrator then combines all findings into the shared audit-result contract.

Finally, `recommendation-engine` creates actions and `recommendation-validator` checks them before the report is written. When an optional service or specialist stage is unavailable, the orchestrator records a limitation and keeps the rest of the report whenever possible.

## Output

The convenience command writes `audit-report.json` in the project directory. The report contains:

- the audited site and timestamp;
- finding counts and evidence-backed findings;
- inferred site and crawl information;
- validated recommendations; and
- limitations caused by bounded crawling or unavailable services.

The report is an assessment of observable public pages, not a guarantee about how every AI system will interpret the site.

## Testing

Testing was done at each important stage of the project:

1. **Dependency and setup testing:** `npm install` confirms that the declared packages can be installed. `npx playwright install chromium` confirms that the browser needed for JavaScript-rendered pages is available.

2. **Shared contract testing:** The tests in `shared/` check URL validation, finding structure, audit-result structure, evidence handling, and page eligibility. These tests protect the data contracts shared by multiple skills.

3. **Skill-level testing:** Each major skill has focused test files or executable test scripts. These cover crawling behavior, claim extraction, semantic filtering, entity scoping, structured-data eligibility, engagement findings, external evidence, corroboration, and recommendation behavior.

4. **Integration testing:** The audit orchestrator tests verify that skills are called in the correct order, that their outputs are combined correctly, and that a failed optional stage becomes a limitation instead of stopping the complete report.

5. **Recommendation testing:** Recommendation tests check deterministic fallback behavior, model fallback behavior, evidence grounding, priority rules, duplicate handling, and rejection of unsupported or unsafe actions.

6. **Project test suite:** Run `npm test` to execute the Vitest suite. This is the main automated test command.

7. **Lint testing:** Run `npm run lint` to check the JavaScript files for lint errors.

8. **End-to-end testing:** Run `npm run audit:site -- https://example.com/` with a public test URL. Confirm that the crawl completes, findings include evidence, recommendations are validated, limitations are recorded when needed, and `audit-report.json` is created.

9. **Gemini-enabled testing:** When `GEMINI_API_KEY` is configured, run the same audit and verify that Gemini-backed claim filtering, external evidence, and recommendations work. Run it without the key as well to confirm that deterministic and fallback behavior still produces a report.
