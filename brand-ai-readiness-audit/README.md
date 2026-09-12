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

## Dependencies

Install all dependencies with:

```powershell
npm install
```

| Package         | Type        | Use                                                                      | Install command             |
| --------------- | ----------- | ------------------------------------------------------------------------ | --------------------------- |
| `@google/genai` | Runtime     | Gemini recommendations and Google Search grounding for external evidence | `npm install @google/genai` |
| `cheerio`       | Runtime     | Reads HTML, text, headings, links, metadata, and JSON-LD                 | `npm install cheerio`       |
| `playwright`    | Runtime     | Fetches pages and renders JavaScript-heavy pages when needed             | `npm install playwright`    |
| `robots-parser` | Runtime     | Checks `robots.txt` rules before crawling                                | `npm install robots-parser` |
| `zod`           | Runtime     | Supports data and recommendation validation                              | `npm install zod`           |
| `eslint`        | Development | Checks code style with `npm run lint`                                    | `npm install -D eslint`     |
| `vitest`        | Development | Runs the test suite with `npm test`                                      | `npm install -D vitest`     |

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

| Skill                      | What it does                                                                                                | Why it matters to the architecture                                                                                      |
| -------------------------- | ----------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `audit-orchestrator`       | Runs the complete audit and creates the final report.                                                       | Provides one reliable entrypoint and keeps failures from one stage from stopping the whole report.                      |
| `crawl-access`             | Checks `robots.txt`, discovers sitemaps, crawls same-origin pages, and records source and rendered content. | Gives every other skill the same bounded and trustworthy set of observations.                                           |
| `site-understanding`       | Infers the site type, brand, entities, and page types.                                                      | Lets later checks apply the right rules to the right pages instead of using generic assumptions.                        |
| `machine-readability`      | Compares source and rendered content and checks headings and extractable text.                              | Detects content that people can see but automated systems may not receive.                                              |
| `entity-fact-audit`        | Extracts facts about organizations, products, and services and checks their scope and consistency.          | Prevents facts about different products or entities from being incorrectly compared.                                    |
| `structured-data-audit`    | Checks relevant JSON-LD and other structured data, especially on eligible product pages.                    | Makes schema findings dependent on page type, which avoids reporting missing Product data on category or search pages.  |
| `freshness-consistency`    | Finds time-sensitive content that lacks clear dates or temporal context.                                    | Helps the report identify information that may be difficult to interpret without claiming that it is stale.             |
| `engagement-audit`         | Builds an internal-link graph and checks whether meaningful pages have useful paths to them.                | Adds the engagement view while keeping conclusions limited to the bounded crawl.                                        |
| `claim-extraction`         | Turns selected page facts and text into normalized claims and bounded verification tasks.                   | Separates claim collection from truth decisions and controls the amount of external work.                               |
| `external-evidence`        | Uses Gemini Search grounding and read-only retrieval to collect public evidence for selected claims.        | Adds traceable outside context without making external search a requirement for the core audit.                         |
| `corroboration`            | Compares website claims with matching external evidence.                                                    | Keeps entity, predicate, value, and source context together so missing evidence is not mistaken for a contradiction.    |
| `recommendation-engine`    | Creates prioritized actions from findings and supported opportunities.                                      | Converts technical findings into useful next steps while supporting deterministic fallback behavior.                    |
| `recommendation-validator` | Checks recommendation scope, evidence, priority, schema, and safety.                                        | Prevents generated recommendations from introducing unsupported metrics, URLs, or unsafe actions into the final report. |

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
