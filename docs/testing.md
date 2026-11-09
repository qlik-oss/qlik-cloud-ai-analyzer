# Testing: Answers feedback, knowledge base document usage and new sheets

This page covers how to test the changes that add Qlik Answers feedback, knowledge base document citations and the knowledge base document inventory to the AI Analyzer, plus four new sheets built on that data.

## What changed

### Load script

Three extract steps were added to the **Extract QVDs** tab, after the messages extract:

| Step | Source endpoint | QVD written |
| --- | --- | --- |
| 7b. Answers feedback | `GET /api/v1/cloud-assistants/threads/messages` (AI messages, `type eq "ai"`) | `ai_monitoring_Extract_AgenticAnswersFeedback.qvd` |
| 7c. Knowledge base citations | Same call, reading the citation sources on each AI answer | `ai_monitoring_Extract_KBDocumentCitations.qvd` |
| 7d. Knowledge base documents | `GET /api/v1/knowledgebases/{id}/documents`, once per knowledge base in `ai_monitoring_Extract_Knowledgebases.qvd` | `ai_monitoring_Extract_KBDocuments.qvd` |

Two tabs were added before **Multi-Tenant Logic**:

* **Feedback** builds the `Feedback` table (vote, reason, comment, date, sentiment) and links it to the fact table through the thread and human message.
* **Document Usage** builds `KB_Documents_final`, with one row per document in a knowledge base. It carries citation counts, last cited date, last indexed date, size, type and a usage status of `Cited`, `Never cited` or `Not in KB anymore`.

How the new steps behave:

* **Rolling window.** Feedback and citations are read for AI answers created in the last `vu_initial_days_back` days (90 by default) on every reload. A thumbs vote can land days after the answer, so the window is re-read on every reload and late votes get picked up.
* **Incremental merge.** When `vIncrementalMode = 1`, rows older than the window are kept from the previous QVD (`where not Exists(...)`), so history keeps growing past 90 days.
* **Tenant-prefixed keys.** Feedback, citation and knowledge base keys use the same `'$(vTenantId)|'` prefix as the rest of the app.
* **Empty-safe.** Each QVD is stored only when the step returned rows. Step 7d is skipped when the tenant has no knowledge bases.

### Sheets

| Sheet | What it answers |
| --- | --- |
| **KB Document Monitoring** | How many files sit in knowledge bases, how many have never been cited by an answer, the share that has been cited, citations per knowledge base, and a file list with times cited and last cited date. |
| **Feedback Over Time** | Thumbs up vs down by month, the share of Answers questions that got a vote, thumbs-down reasons and which apps the votes trace back to. |
| **Answers Health** | Answers error rate overall and by month, where errors happen, questions per active user and the thumbs-down share. |
| **Estate Hygiene** | Assistants and knowledge bases that are candidates for cleanup, knowledge bases by status and duplicate knowledge base names. |

## Test setup

1. Import `assets/AI-Analyzer.qvf` into a tenant that has Qlik Answers in use: at least one assistant with a knowledge base and some answered questions. The test is more useful if a few answers carry a thumbs up or down.
2. Fill in the four Config variables as described in the README. Use a dedicated QVD folder for the test so existing QVDs from another copy of the app are not read in.
3. Reload with `vIncrementalMode = 0`.
4. Reload a second time with `vIncrementalMode = 1`.

## Test cases

| # | Check | How | Expected |
| --- | --- | --- | --- |
| 1 | Full reload succeeds | Reload with `vIncrementalMode = 0` | Reload succeeds. The log has trace lines for 7b, 7c and 7d with row counts, and `0 KBs failed` for 7d. |
| 2 | Incremental reload succeeds | Reload again with `vIncrementalMode = 1` | Reload succeeds. Row counts for `Feedback`, citations and documents are the same as or higher than test 1. Nothing is duplicated. |
| 3 | Tenant with no knowledge bases | Reload on a tenant without knowledge bases | Reload succeeds. Step 7d is skipped and the KB Document Monitoring sheet shows zeros, not errors. |
| 4 | File count matches the API | Compare **Files in knowledge bases** with the distinct (knowledge base, file name) pairs returned by `/api/v1/knowledgebases/{id}/documents` across all knowledge bases | Equal. |
| 5 | Cited files match the API | Compare **Share of files cited** and **Citations to current files** with the cross-check script below | Equal, for the same `vu_initial_days_back` window. |
| 6 | Removed files are flagged | Find a cited file that was later removed from its knowledge base | Shows in the file table with status `Not in KB anymore`, and is left out of the KPI denominators. |
| 7 | Feedback count matches | Compare **Feedback received** with the number of AI messages in the window that carry a feedback object | Equal. |
| 8 | Feedback links to apps | Open **Feedback Over Time** and check **Votes traced to app** | Every vote on an app-scoped question shows its app. Votes on assistant-only questions show no app. |
| 9 | Late vote is caught | Vote on an answer that is a few days old, then reload | The new vote appears after the reload. |
| 10 | Charts render | Open every new sheet | No chart shows a calculation error. |

## Cross-checking against the API

Tests 4, 5 and 7 compare the app with the REST API. The method:

1. Page through `GET /api/v1/cloud-assistants/threads/messages?limit=100&filter=type eq "ai" and createdAt ge "<today minus vu_initial_days_back>T00:00:00Z"` and follow `links.next.href` until it is empty.
2. For each message, read `content[].card.citations[].sources[]`. Keep the sources that have a `knowledgebaseId`, and collect `(knowledgebaseId, documents[].source)` pairs with the message id.
3. Call `GET /api/v1/knowledgebases?limit=100`, then `GET /api/v1/knowledgebases/{id}/documents` for each one, and collect `(id, source)` pairs as the inventory.
4. Cited files = inventory pairs that have at least one citing message. Citations to current files = the number of citing messages summed over those files. Pairs cited but missing from the inventory are the `Not in KB anymore` rows.

Reference run during development, with a 90 day window:

| Measure | API | App |
| --- | --- | --- |
| Distinct files in knowledge bases | 439 | 439 |
| Files cited at least once | 4 | 4 |
| Citations to current files | 11 | 11 |
| Cited files no longer in a knowledge base | 4 (71 citations) | 4 |
| Total citations in the window | 82 | 82 |

## Known limitations

* **Multi-tenant parent apps do not pull the new tables yet.** The new QVDs are not in `tables_to_pull`, so a parent app in multi-tenant mode will not show feedback or document data from child tenants.
* **The window follows `vu_initial_days_back`.** On a first full reload, feedback and citations go back 90 days by default. Raise the variable to load more history on the first run.
* **The document endpoint is called once per knowledge base.** On tenants with many knowledge bases this adds reload time. Failed calls are retried and counted in the `KBs failed` trace line.
* **Duplicate file names in one knowledge base are counted once.** The API can return the same file name twice in one knowledge base. The app counts distinct names, matching how citations reference documents.
