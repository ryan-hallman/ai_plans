# Live Data Pipelines Design

## Goal

Replace seed-only data loading with live, scheduled pipelines that continuously feed the system with fresh KPI values, monitoring findings, and intelligence signals. The downstream consumption layer (morning brief, triage, board prep, copilot) already queries these tables — the gap is entirely on the supply side.

## Architecture

All pipelines run as Celery tasks on a single worker pool, scheduled by Celery Beat. A transactional outbox (already implemented) publishes domain events to registered subscribers, also via Celery tasks. This gives one operational model for all background work: scheduling, retries, monitoring, and failure handling.

## Tech Stack

- **Celery** (already configured) + **Celery Beat** for scheduling
- **Redis** (already running) as broker and result backend
- **NewsAPI.org** for news intelligence
- **SEC EDGAR** (existing `EdgarClient`) for regulatory filings
- **Anthropic Claude** (existing SDK) for LLM-assisted KPI column mapping
- **PostgreSQL** transactional outbox (existing `shared.domain_events`)

---

## 1. Celery Beat Infrastructure

### Beat Schedule

| Task | Interval | Description |
|------|----------|-------------|
| `monitor.sweep` | 30 min | Query all active watch targets, fan out individual check tasks |
| `outbox.drain` | 30 sec | Poll `shared.domain_events` where `published_at IS NULL`, dispatch subscribers |
| `kpi.check_overdue` | Daily 8:00 UTC | Check for KPI owners who haven't reported on cadence, create reminders |

### Worker Configuration

- Single worker pool handles all queues.
- `task_acks_late=True` (already set) for crash recovery.
- Per-task time limits:
  - `monitor.check_target`: `task_soft_time_limit=300` (5 min, SEC fetches can be slow)
  - `outbox.drain`: `task_soft_time_limit=30`
  - `kpi.check_overdue`: `task_soft_time_limit=60`
- Autodiscover expands to:
  - `prescient.monitoring.infrastructure.tasks`
  - `prescient.kpis.infrastructure.tasks`
  - `prescient.shared.infrastructure.tasks`

### Docker Services

Two new services in `docker-compose.yml`:

- **worker** — `celery -A prescient.celery_app worker --loglevel=info`
- **beat** — `celery -A prescient.celery_app beat --loglevel=info`

Both depend on Redis + Postgres health checks. Worker shares the same image and volume mounts as the API service.

---

## 2. Monitor Runner Pipeline

### Sweep Flow

```
Beat (30 min) -> monitor.sweep task
  -> query all active monitor_targets from DB
  -> for each target, dispatch monitor.check_target(target_id) as Celery task
```

```
monitor.check_target(target_id):
  -> load target from DB
  -> resolve source adapter(s) from target config
  -> create MonitorRun row (status=running)
  -> for each adapter:
      -> adapter.check(target, since=last_checked_at) -> list[RawFinding]
  -> deduplicate findings by content_hash against existing findings
  -> persist new MonitorFinding rows
  -> update MonitorRun (status=completed, items_found=N)
  -> append DomainEvent("monitoring.finding.discovered") to outbox for each new finding
  -> update target.last_checked_at
```

### Source Adapter Protocol

```python
class MonitorSourceAdapter(Protocol):
    async def check(self, target: MonitorTarget, since: datetime | None) -> list[RawFinding]: ...
```

`RawFinding` is a dataclass with: `title`, `summary`, `finding_type`, `severity`, `source_url`, `occurred_at`, `raw_payload`.

### Concrete Adapters

**SecFilingAdapter** — Wraps the existing `EdgarClient`. Given a target with a CIK in config, checks for filings newer than `since`. Returns `RawFinding` with `finding_type=FILING`, title like "10-Q filed 2026-04-10", `source_url` pointing to SEC. Reuses `EdgarClient.list_filings()` — no new SEC integration code needed.

**NewsApiAdapter** — Calls NewsAPI.org `GET /v2/everything` with a query built from target name + keywords in config. Filters by `from` date using `since`. Returns `RawFinding` with `finding_type=NEW_DOCUMENT`. Severity auto-classified by keyword heuristic:
- Executive departure/hire keywords -> IMPORTANT
- Product launch, acquisition, funding keywords -> NOTABLE
- General mentions -> INFO

LLM-based severity classification is a future enhancement.

**UrlChangeAdapter** — Fetches a URL via HTTP, hashes the response body (SHA256), compares to hash stored in `target.last_content_hash`. If changed, returns a `RawFinding` with `finding_type=CUSTOM` and a summary noting the URL changed. Covers pricing pages, changelogs, competitor feature pages without needing bespoke parsers.

### Adapter Resolution

Target config includes a `sources` list specifying which adapters to run:

```json
{
  "target_type": "COMPANY",
  "target_id": "<company-uuid>",
  "config": {
    "company_name": "Echelon",
    "cik": "0001868726",
    "sources": ["sec_filing", "news_api"],
    "news_keywords": ["Echelon Fitness", "connected fitness"]
  }
}
```

If `sources` is omitted, defaults are inferred from `target_type`:
- COMPANY -> `["sec_filing", "news_api"]`
- PRODUCT -> `["news_api", "url_change"]`
- MARKET -> `["news_api"]`
- PERSON -> `["news_api"]`
- TOPIC -> `["news_api"]`

### Finding Deduplication

Each finding gets a `content_hash`: SHA256 of `source_url + title + finding_type`. Before persisting, check for existing findings with the same hash on the same target. Duplicates are silently dropped.

### Operational Endpoint

`GET /monitoring/runs?target_id=X` — returns execution history for a target. Each `MonitorRun` records `started_at`, `finished_at`, `status`, `items_found`, `error`. Useful for debugging "why didn't I get a finding?"

---

## 3. KPI Intake Engine

A "map once, ingest forever" engine for importing KPI data from CSV files. Uses LLM-assisted column mapping on first upload, saves the mapping as a reusable template, and auto-applies it on subsequent uploads from the same source format.

### Intake Template Model

```python
class IntakeTemplate:
    id: UUID
    scope: "system" | "organization"       # system = shared across all orgs
    organization_id: str | None            # null for system-scoped
    source_system: str | None              # "quickbooks", "xero", "stripe", "sage", etc.
    report_type: str | None                # "profit_and_loss", "balance_sheet", "mrr_summary"
    name: str                              # display name
    header_fingerprint: str                # SHA256 of sorted, lowercased, trimmed headers
    column_mappings: list[ColumnMapping]
    period_config: PeriodConfig
    kpi_mapping_overrides: dict | None     # org-level overrides on top of system template
    created_by: str
    last_used_at: datetime | None

class ColumnMapping:
    source_column: str                     # "Net Rev ($M)" — the header in the CSV
    target_kpi_id: str                     # "revenue" — maps to kpi_definitions
    unit: str                              # "USD_MILLIONS"
    transform: str | None                  # "multiply_1000", "divide_100", "negate", null
    skip: bool                             # true if column should be ignored

class PeriodConfig:
    period_column: str                     # "Quarter" or "Date"
    period_format: str                     # "quarter_label" (Q1 FY2025), "date_range", "month"
    fy_end_month: int                      # 6 for June FY companies
```

### Template Tiers

**System templates** — shared across all orgs. Identified by `source_system` + `report_type`. When an operator confirms a mapping and the header fingerprint matches a known system format (e.g., QuickBooks P&L), the template is saved at system scope. Every subsequent org that uploads the same format gets it for free.

**Organization templates** — for bespoke formats unique to one company. "Peloton Board Deck Q4 Format", "Internal FP&A Spreadsheet". Only match for the owning org.

### Upload Flow

```
Operator uploads CSV
  -> parse headers + sample rows (first 5)
  -> compute header fingerprint (SHA256 of sorted, lowercased, trimmed headers)
  -> check for matching template:
      1. org templates by fingerprint
      2. system templates by fingerprint
      3. if system match but org has different KPI IDs, apply with kpi_mapping_overrides
  -> if template found:
      -> show operator: "Looks like your [template.name]. Use saved mapping?"
      -> operator confirms -> parse all rows -> validate -> ingest
  -> if no template match:
      -> send to LLM: headers + sample rows + org's KPI definitions
      -> Claude proposes column mappings + period detection
      -> operator reviews in UI: confirm, correct, or skip columns
      -> save confirmed mapping as new IntakeTemplate (system or org scope)
      -> parse all rows -> validate -> ingest
```

### LLM Mapping

The prompt sends to Claude:
- The org's KPI definitions (id, label, unit, description)
- CSV headers
- 5 sample data rows
- Instructions: "Map each CSV column to a KPI definition. Return JSON with `column_mappings` and `period_config`. If a column doesn't map to any KPI, set `skip: true`. If you're unsure, set `uncertain: true` with your best guess so the operator can confirm."

Response is validated against the `ColumnMapping` schema before presenting to the operator.

### Ingestion

For each mapped row, the engine calls the existing `ReportKpiValue` use case internally (not via HTTP). This means anomaly detection fires automatically for every ingested value.

Batch results returned to the operator:
- N values ingested successfully
- M anomalies detected (with links to triage)
- K rows skipped (with reasons: unmapped column, invalid value, duplicate period)

### KPI Value Deduplication

Before ingesting, check for existing values with the same `organization_id + kpi_id + period_start + period_end`. If a value already exists:
- If the new value is identical, skip silently
- If the new value differs, update the existing value and note it in the batch results

### Template Fingerprinting

SHA256 of sorted, lowercased, trimmed column headers joined by `|`. Example: headers `["Net Rev ($M)", "Quarter", "EBITDA"]` become fingerprint of `"ebitda|net rev ($m)|quarter"`.

### API Surface

- `POST /kpis/intake/upload` — multipart CSV upload, returns proposed mapping (from template or LLM)
- `POST /kpis/intake/confirm` — operator confirms/corrects mapping, triggers batch ingestion
- `GET /kpis/intake/templates` — list saved templates for org (includes system templates)
- `DELETE /kpis/intake/templates/{id}` — remove a stale org template (system templates require admin)

### Webhook & Connector Stubs

Protocol defined but not implemented:

```python
class KpiIntakeAdapter(Protocol):
    async def fetch(self, config: dict) -> list[RawKpiRow]: ...
```

Future adapters (QuickBooks, Stripe, Xero) implement this protocol. A sweep task similar to the monitor runner would call them on cadence. The `IntakeTemplate` model already supports `source_system` for this purpose.

---

## 4. Outbox Publisher & Event Subscribers

### Outbox Drain Task

```
Beat (30 sec) -> outbox.drain task
  -> SELECT * FROM shared.domain_events
     WHERE published_at IS NULL
     ORDER BY occurred_at ASC
     LIMIT 100
  -> for each event:
      -> look up subscribers by event_type in registry
      -> dispatch each subscriber as a Celery task
      -> SET published_at = now()
      -> commit
```

The drain marks events as published after dispatching subscriber tasks to the Celery queue — not after subscribers complete. If a subscriber fails, Celery's own retry handles it. The outbox's job is to reliably get events into the queue.

### Subscriber Registry

A dict mapping event types to handler task functions:

```python
EVENT_SUBSCRIBERS: dict[str, list[Callable]] = {
    "kpi.anomaly_detected": [create_triage_question_from_anomaly],
    "monitoring.target.created": [run_immediate_check],
}
```

Each handler runs as its own Celery task with independent retry (exponential backoff, max 3 retries).

### Subscribers Implemented

**`create_triage_question_from_anomaly`** — When a KPI anomaly is detected, creates a triage question using the anomaly's `enrichment_question` field (which the anomaly detector already generates). Closes the loop: KPI uploaded -> anomaly detected -> triage question appears for operator. Uses a published query interface to create the triage question — does not import triage tables directly.

**`run_immediate_check`** — When a watch target is created, dispatches `monitor.check_target` immediately instead of waiting for the next 30-minute sweep. The operator gets their first findings within seconds of adding a target.

### Events Emitted

The existing codebase doesn't call `append_event` anywhere yet. We wire it into:

- **`ReportKpiValue` use case** — emits `kpi.value_reported`, and `kpi.anomaly_detected` if anomaly found
- **`monitor.check_target` task** — emits `monitoring.finding.discovered` for each new finding
- **`AddWatchTarget` use case** — emits `monitoring.target.created`

Event payloads include enough context for subscribers to act without re-querying:

```python
# kpi.anomaly_detected payload
{
    "anomaly_id": "...",
    "organization_id": "...",
    "kpi_id": "revenue",
    "anomaly_type": "sudden_change",
    "severity": "important",
    "description": "Revenue dropped 28% vs trailing average",
    "enrichment_question": "Revenue declined significantly. What drove this change?"
}

# monitoring.finding.discovered payload
{
    "finding_id": "...",
    "target_id": "...",
    "owner_tenant_id": "...",
    "finding_type": "filing",
    "title": "10-Q filed 2026-04-10",
    "severity": "notable",
    "source_url": "https://..."
}

# monitoring.target.created payload
{
    "target_id": "...",
    "owner_tenant_id": "...",
    "target_type": "COMPANY",
    "target_name": "Echelon"
}
```

---

## 5. File Structure

### New Files

```
apps/api/src/prescient/
  celery_app.py                                    # MODIFY — add beat_schedule, expand autodiscover
  config/base.py                                   # MODIFY — add newsapi_api_key

  shared/infrastructure/tasks/
    outbox_drain.py                                # drain task + subscriber registry

  monitoring/infrastructure/
    tasks/
      sweep.py                                     # monitor.sweep + monitor.check_target tasks
      adapters/
        base.py                                    # MonitorSourceAdapter protocol + RawFinding
        sec_filing.py                              # wraps EdgarClient
        news_api.py                                # wraps NewsAPI.org
        url_change.py                              # generic URL change detection
    tables/
      monitor_target.py                            # MODIFY — add last_checked_at, last_content_hash

  kpis/
    api/routes.py                                  # MODIFY — add intake endpoints
    application/use_cases/
      intake_csv.py                                # LLM-assisted mapping + batch ingestion
    domain/entities/
      intake_template.py                           # IntakeTemplate, ColumnMapping, PeriodConfig
    infrastructure/
      tables/
        intake_template.py                         # IntakeTemplateRow ORM
      repositories/
        intake_template_repository.py
      tasks/
        check_overdue.py                           # daily overdue KPI reminder task

  events/
    subscribers/
      anomaly_to_triage.py                         # kpi.anomaly_detected -> triage question
      target_immediate_check.py                    # monitoring.target.created -> immediate sweep

infrastructure/
  docker-compose.yml                               # MODIFY — add worker + beat services
```

### Migration

One new Alembic migration adding:
- `kpis.intake_templates` table (id, scope, organization_id, source_system, report_type, name, header_fingerprint, column_mappings JSONB, period_config JSONB, kpi_mapping_overrides JSONB, created_by, last_used_at, created_at, updated_at)
- `last_checked_at` (timestamptz) and `last_content_hash` (varchar 64) columns on `monitoring.monitor_targets`
- `content_hash` (varchar 64) column on `monitoring.monitor_findings`

### Modified Existing Files

- `celery_app.py` — beat schedule dict + expanded autodiscover list
- `config/base.py` — `newsapi_api_key: str = Field(default="")`
- `ReportKpiValue` use case — call `append_event` for `kpi.value_reported` and `kpi.anomaly_detected`
- `AddWatchTarget` use case — call `append_event` for `monitoring.target.created`
- `docker-compose.yml` — add `worker` and `beat` services
- `knowledge_mgmt/api/routes.py` — migrate `_trigger_ingestion` from BackgroundTasks to Celery task

### Boundary Discipline

All new code stays within its bounded context. Cross-context communication uses:
- Published query interfaces (e.g., subscriber `anomaly_to_triage` uses a triage query interface to create questions)
- Domain events via the outbox (no direct imports across contexts)
- The `events/subscribers/` directory is a composition root — it imports from published interfaces only

---

## 6. Testing & Operational Visibility

### Testing Strategy

- **Unit tests for each monitor adapter** — mock the external API (NewsAPI HTTP responses, SEC filing list), verify raw responses parse into `RawFinding` correctly, dedup logic works, error handling returns empty list (not crash)
- **Unit tests for intake engine** — mock the LLM response, verify column mapping parsing, template fingerprinting, batch ingestion calling `ReportKpiValue` per row, dedup on existing values
- **Unit tests for outbox drain** — verify events dispatch to correct subscribers, `published_at` gets set, idempotency on re-drain (already-published events skipped)
- **Integration tests for sweep task** — real Celery worker with test DB, verify target -> adapter -> finding -> outbox event -> subscriber
- **Template matching tests** — verify fingerprint generation, system vs org resolution order, override application, edge cases (reordered columns, extra whitespace)

### Failure Modes

| Scenario | Behavior |
|----------|----------|
| NewsAPI rate limit or down | Adapter returns empty list, MonitorRun status=failed with error message, next sweep retries |
| SEC EDGAR slow or timeout | `task_soft_time_limit=300` on check_target, task killed gracefully, MonitorRun marked failed |
| LLM mapping fails | Return error to operator in upload response, don't save template, operator retries |
| Outbox drain finds no events | No-op, task returns immediately |
| Subscriber task fails | Celery retry with exponential backoff, max 3 retries, then dead-letter |
| Duplicate CSV upload | Template fingerprint matches, identical KPI values deduplicated by org+kpi+period |
| Celery worker crashes mid-task | `task_acks_late=True` ensures unfinished task is re-delivered on worker restart |
| Beat scheduler goes down | Missed schedules run on next Beat startup (catch-up behavior) |
