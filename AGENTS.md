# AGENTS.md — Internal QA Tool Hub

## 1. Purpose
Build an internal, browser-based Tool Hub that launches two independent existing tools: Banner Preview and QA Dashboard. Preserve existing working behavior and incrementally add persistent QA workspaces, spreadsheet comparison, and portable reports.

This file describes the **target design**, not a claim that all features already exist. Inspect the actual code before editing. Ask for missing API schemas, DOM structure, and business validation rules; do not invent them.

## 2. Current environment and known behavior
- Runtime: Google Chrome DevTools **Snippets** executed in the context of an authenticated internal company website.
- Stack: plain HTML, CSS, JavaScript. No mandatory React, backend, or build step.
- Existing behavior: user enters search criteria; JavaScript calls internal APIs, parses JSON, and renders a preview/result UI; Excel export exists; some automatic QA checks exist.
- Separate existing tools: Banner Preview and QA Dashboard. Their actual implementation and integration points must be inspected.
- Some QA judgments require manual review or comparison with operations-managed Excel spreadsheets.
- Company security constraints, exact APIs, data classification, and external library availability are **unknown** until verified.

## 3. Architectural decisions
1. A lightweight **Tool Hub** presents independent launch/navigation entries for Banner Preview and QA Dashboard.
2. **Banner Preview** only fetches and visualizes banner information. Do not force it to perform QA or persist QA results.
3. **QA Dashboard** handles API-based automatic checks, spreadsheet comparisons, manual PASS/FAIL review, and workspace persistence.
4. Do not attempt to programmatically run another saved DevTools Snippet by name; there is no supported webpage JavaScript API for this. Extract existing Snippet behavior into callable functions/modules and register them or bundle their code into a single bootstrap Snippet.
5. Keep tool UIs isolated; define mount/unmount (or init/destroy) lifecycle, avoid duplicate event listeners, conflicting element IDs, and uncontrolled globals.
6. Use **IndexedDB** for local persistence of multiple QA workspaces. It is origin- and browser-profile-scoped, not a shared server database or guaranteed durable backup.
7. Export self-contained, read-only HTML reports for other teams. No server is required to view a report. No editing or live API calls are needed in the exported report.
8. Use JSON for full workspace backup/restore and Excel for tabular analysis/reporting. These are distinct from HTML sharing.

## 4. Target user flows
### Tool Hub
- Open hub -> choose Banner Preview or QA Dashboard -> navigate back or switch tools.
- Navigation presentation (full-screen switch vs modal) remains undecided; do not hard-code this assumption prematurely.

### QA workspace
- Create named workspace with stable unique ID and optional target/query metadata.
- Enter criteria -> call approved internal API -> parse and render data.
- Run automatic validation; import local `.xlsx` as needed; normalize/map columns and compare to API data.
- Manually mark checks PASS / FAIL / NOT_CHECKED (optionally BLOCKED), enter notes/evidence.
- Save workspace locally; close/reopen and restore prior state without automatically refreshing API results.
- Explicit refresh creates a new timestamped API snapshot; rerun QA without destroying prior run history.
- List, rename, duplicate, delete, and export workspaces.
- Export a read-only HTML report with search/filter, report generation timestamp, QA run timestamp, summary, and error details.

## 5. Proposed IndexedDB schema (v1, refine after inspecting code)
Database: `QA_WORKSPACE_DB`, version `1`.

| Object store | Key | Recommended indexes | Contents |
| --- | --- | --- | --- |
| `workspaces` | `id` | `updatedAt`, `status` | name, query, timestamps, current run/snapshot references |
| `api_snapshots` | `id` | `workspaceId`, `fetchedAt` | sanitized API response, query, source metadata |
| `excel_datasets` | `id` | `workspaceId`, `importedAt` | file metadata, parsed normalized rows, column mapping |
| `qa_runs` | `id` | `workspaceId`, `startedAt` | execution time, rules version, snapshot/dataset IDs, summary |
| `qa_results` | `id` | `workspaceId`, `runId` | check ID, type, auto verdict, manual verdict, effective verdict, notes, expected/actual, timestamp |

- Use a stable workspace ID (e.g. `crypto.randomUUID()` where supported).
- Keep automatic and manual judgments separate; never overwrite automatic results when a human changes a verdict.
- Store API and Excel source snapshots separately from derived QA results so comparisons are reproducible.
- Do not persist tokens, cookies, credentials, or unnecessary sensitive fields.
- Design indexes for workspace-based retrieval and cascading deletion through explicit IndexedDB transactions.
- Version schema migrations with `onupgradeneeded`; avoid deleting old stores during upgrades.
- Use atomic transactions where related records must be consistent; show save failures to the user.
- Guard against stale writes from multiple tabs (revision number or optimistic concurrency).

## 6. Data contracts (illustrative, not actual API schemas)
```js
const workspace = {
  id: 'uuid', name: 'Banner QA', status: 'IN_PROGRESS',
  query: { bannerId: '12345' },
  currentRunId: null, currentSnapshotId: null,
  createdAt: 'ISO-8601', updatedAt: 'ISO-8601', revision: 1
};
const result = {
  id: 'uuid', workspaceId: 'uuid', runId: 'uuid',
  checkId: 'banner-text', type: 'MANUAL',
  autoVerdict: null, manualVerdict: 'FAIL', effectiveVerdict: 'FAIL',
  expected: 'expected text', actual: 'observed text',
  note: 'Mismatch', checkedAt: 'ISO-8601'
};
```
Define a documented precedence policy for `effectiveVerdict` after product-owner confirmation. Do not assume an automatic PASS can override a manual FAIL.

## 7. Spreadsheet import and comparison
- Provide user-initiated file selection; parse `.xlsx` locally in the browser with an approved library (e.g. SheetJS), subject to internal CSP/security review.
- Do not upload spreadsheet contents to any external service.
- Ask for the matching key(s), worksheet, column mapping, and expected field semantics.
- Normalize numeric strings, dates/timezones, whitespace, booleans, missing values, and IDs carefully. Preserve leading zeros in identifiers.
- Classify comparisons as MATCH / MISMATCH / MISSING_IN_API / MISSING_IN_EXCEL / NOT_COMPARABLE, then map to QA verdicts according to agreed rules.
- Display side-by-side expected (Excel) and actual (API) values, with clear mismatch reasons.

## 8. Exports and imports
- **JSON workspace backup:** schema version + workspace + related snapshots, datasets, runs, results. Validate imported structure and size; regenerate colliding IDs and repair references; never execute imported content.
- **Excel export:** structured QA result tables and comparison details; keep existing export behavior intact unless explicitly changed.
- **HTML report:** one portable `.html` file containing only approved, sanitized report data and self-contained styling/scripts; read-only, no IndexedDB or live API dependency. Include title, target ID, QA time, export time, counts, filters, and FAIL details.
- Escape untrusted strings before injecting HTML; avoid `innerHTML` with unsanitized API, Excel, or notes data. Exclude secrets and unneeded internal payloads.
- Reports are point-in-time snapshots. New QA results require a new export; no server-side synchronization is implied.

## 9. Suggested code organization
```text
bootstrap-snippet.js     # one entrypoint usable in DevTools Snippets
hub.js                   # tool launcher and navigation
modules/banner-preview.js
modules/qa-dashboard.js
qa/automatic-checks.js
qa/manual-review.js
qa/excel-compare.js
storage/indexeddb.js
storage/workspace-repository.js
io/excel-import.js
io/excel-export.js
io/json-backup.js
io/html-report.js
```
This is a **logical** structure. If DevTools Snippets cannot load separate files under company CSP constraints, bundle into one deployable Snippet while preserving module boundaries in source control. Never assume local filesystem imports work in a page context.

## 10. Implementation order
1. Inventory existing Snippets, API entrypoints, DOM dependencies, global variables, and current Excel export; record a baseline.
2. Confirm IndexedDB access, data-retention policy, approved libraries, and export restrictions in the actual company environment.
3. Extract existing tools into callable modules without altering their user-visible behavior.
4. Build Tool Hub and tool lifecycle/navigation.
5. Implement IndexedDB schema and CRUD for multiple QA workspaces; test reload/restore.
6. Persist QA state and separate automatic/manual verdicts; add run/snapshot history.
7. Add Excel import, mapping, normalization, and API comparison.
8. Add JSON backup/restore, Excel QA export, and read-only HTML report export.
9. Test performance, errors, migrations, security, and data-loss scenarios.

## 11. Acceptance criteria
- Existing Preview, QA, and Excel export behavior still works.
- Tool Hub opens either tool independently and returns safely to the hub.
- At least two distinct QA workspaces can be created, saved, listed, reopened, and updated independently.
- QA results and notes survive Snippet rerun and browser page reload on the same origin/profile.
- Restoring a workspace does not silently replace its stored API snapshot with a fresh API response.
- Imported Excel rows can be matched against API results and mismatches clearly explained.
- Manual verdicts persist independently from automatic results.
- JSON export/import restores the complete workspace and handles ID collisions.
- Exported HTML opens without a server or company API and is read-only; it displays the correct report snapshot.
- No credentials or prohibited sensitive fields are persisted or exported.
- Failures (storage quota, denied storage, parse errors, network failures) produce understandable user feedback.

## 12. Agent operating rules
- **First inspect and explain the existing code.** Do not replace working Snippets wholesale.
- Before each change, state affected files/functions, intended behavior, and potential regressions.
- Make small, testable changes; prefer native JavaScript and minimal dependencies.
- Never invent internal API endpoints, credentials, selectors, Excel column names, or QA business rules.
- Respect internal security, authentication, CSP, and data-handling restrictions; do not bypass them.
- Never send company data to third-party endpoints or external AI services without authorization.
- Prefer explicit user action for API refresh and exports; do not make unintended mutating API calls.
- Distinguish **implemented**, **planned**, and **unverified** functionality in all summaries.
- After each implementation step, provide a test checklist and any known limitations.

## 13. Questions to resolve with the developer
1. What are the current Preview/QA Snippet source files and entrypoints?
2. Are both tools executed on the same origin and compatible with one hub?
3. Which API schemas and QA rules are authoritative?
4. Which Excel sheet/columns are used, and what keys join Excel to API data?
5. Is local persistence of these fields and exporting HTML/JSON/Excel permitted?
6. Are external libraries permitted, and does CSP prevent loading them?
7. Should switching tools unmount the previous UI or preserve it in memory?
8. What counts as a final PASS when automatic and manual verdicts differ?

## 14. Immediate next task for the agent
Review the supplied existing Snippet code and produce: (a) current architecture map, (b) risks and shared dependencies, (c) minimum-change refactoring plan for a Tool Hub, and (d) first implementation patch with test steps. Do not start by rewriting the whole application.
