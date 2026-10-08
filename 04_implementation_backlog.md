# Retail pipeline — implementation backlog

Version 1.0 · 8 October 2026 · Planned work, not implemented

Companion documents: `01_pipeline_overview.md`, `02_technical_specification.md`, `03_architecture_decisions.md` (version 2.3).

## Scope and delivery rules

Retail MVP: manual JDBC imports, automatic MSSQL/pymssql extracts, Delta on DBR 13.3 without Unity Catalog, approximately 120 Gold features and seven Platinum JSON groups. Inference is a separate integration. Exact source/feature inventories remain open.

Every task below is initially **TODO**. Dependencies refer to task IDs. Deliver each task as a reviewable change with its contract, implementation and evidence of acceptance. Do not create 15 invented source definitions: repeat the relevant extract task for each real source after inventory.

A working vertical slice comes first: customer population + products + reference data → Gold → one Platinum group → validation/publication. Then add representative increments and remaining domains. The complete seven-group retail release follows.

## Phase 1 — Confirm contracts and environment

### T01 — Inventory extracts and dependencies

**Depends on:** none.

**Deliver:** an inventory of actual sources, owner, manual/automatic access, explicit projection/query, output grain/key, volume, snapshot/incremental mode, consumers, freshness limit and empty-result policy. Mark source-side aggregates and consistency groups for manual batches.

**Accept:** every required feature domain has identified inputs; missing access, cursor or delete semantics are named blockers. Each mutable/incremental source has an agreed strategy rather than inheriting chat assumptions.

### T02 — Define retail population, features and time semantics

**Depends on:** T01.

**Deliver:** retail eligibility and customer-key contract; actual feature list; seven proposed groups; units, windows, reference time/timezone, null/default rules, history and content limits.

**Accept:** every feature has an owner and reproducible definition. Distinguish event time from ingestion availability. Confirm cross-segment key assumptions for future expansion, without implementing other segments.

### T03 — Verify DBR 13.3 execution and access

**Depends on:** T01.

**Deliver:** development databases/storage permissions, Python import/package setup, secret references and reader dependencies. Verify manual JDBC authentication and automated pymssql access separately.

**Accept:** a small authorized query and Delta write/read work in the target environment; no UC dependency, hardcoded credentials or client payloads in logs. Production locations are explicitly configured.

## Phase 2 — Build only the shared mechanics needed for MVP

### T04 — Create project layout and entry points

**Depends on:** T03.

**Deliver:** SQL, extract, Silver/Gold/Platinum, common, contract and test directories; thin manual/automatic notebook entry points. Explicit environment parameters and packaged/importable modules.

**Accept:** one task runs independently with explicit inputs; no hidden notebook state. Business logic stays in named transformations, not an executable metadata framework.

### T05 — Implement run context, batch readiness and watermark state

**Depends on:** T02, T04.

**Deliver:** minimal control tables for runs, manual batches, extract status, pinned inputs and per-source watermarks. Record run_id, batch/attempt identity, fixed as_of_ts, code/schema versions and cursor bounds. Use one automatic state writer.

**Accept:** incomplete batches cannot become input; retry reuses context; a READY batch is immutable; another arriving batch cannot replace pinned inputs mid-run. These controls are MVP, unlike detailed service metrics in F01.

### T06 — Implement bounded readers and staging

**Depends on:** T04, T05.

**Deliver:** JDBC and pymssql helpers with explicit schemas/projections, parameter handling, connection cleanup, attempt staging and complete-result validation. pymssql uses fetchmany without accumulating the whole extract on the driver.

**Accept:** empty and multi-chunk results behave correctly; interrupted attempts remain unavailable. Validate persisted data without rereading mutable sources. Source connection budgets are respected.

### T07 — Implement selective retry and isolated execution

**Depends on:** T05, T06.

**Deliver:** common error classification; configurable proposal of two automatic retries, three attempts total, five minutes apart for transient faults only. Entry point for one named extract, optional bounds, mode and batch ID.

**Accept:** parsing/schema/key/permission failures stop immediately. Nested reader/Job retries do not multiply attempts. Default manual runs stage and validate only; regular repairs reuse bounds; separate historical backfills do not move the regular watermark. Target writes cannot overlap.

## Phase 3 — Deliver the first working vertical slice

### T08 — Implement customer and product snapshot extracts

**Depends on:** T01, T02, T06.

**Deliver:** explicit customer projection, product aggregation by customer/type, source value-dictionary extracts and local contracts. Use the access path assigned in T01, not an assumed connector.

**Accept:** keys are unique, empty-snapshot behavior is explicit and refreshed results remove obsolete groups. Aggregates match source control queries for selected customers. All required members of a manual consistency group gate READY.

### T09 — Implement manual-batch selection

**Depends on:** T05, T08.

**Deliver:** select a complete READY batch within each required max_age; record its ID and source_as_of where available.

**Accept:** partial/stale/missing required inputs block publication; fresh ingestion is not confused with fresh source data. Selected manual batches are protected from cleanup.

### T10 — Build product features and initial Gold

**Depends on:** T02, T08, T09.

**Deliver:** customer-grain product feature set and explicit left joins to the retail population. Apply classification mappings and justified zeros where features acquire their meaning.

**Accept:** exactly one row per eligible customer; no join multiplication; absent products differ from unavailable sources. Reference keys and required code coverage are validated.

### T11 — Author the feature dictionary and glossary

**Depends on:** T02.

**Deliver:** manually maintained `contracts/feature_dictionary.csv`: Gold column, group, readable JSON key, definition, type, unit/window, null policy, value-dictionary reference and content limits. Start with the vertical slice, then complete the inventory.

**Accept:** mappings reference real columns; keys are unique within groups; intended coverage is complete. Full descriptions appear in a shared LLM glossary, not repeatedly in customer payloads. Dictionary revision, schema and prompt versions are compatible. No arbitrary business expressions are executed from the CSV.

### T12 — Build initial Platinum and publication path

**Depends on:** T10, T11.

**Deliver:** source-code-to-description joins, feature selection/grouping/renaming, typed structures and single JSON serialization. Persist candidates, validate, then atomically overwrite each published Delta table with fixed schema.

**Accept:** NULL is omitted, 0/false retained, empty group represented as {}; nested empty content follows explicit rules. Unknown codes cannot disappear silently. Failed checks preserve published output. Gold and Platinum carry run_id; no cross-table atomicity is assumed.

**Milestone M1:** one domain is reproducibly available as validated LLM input. This is not the full retail release.

## Phase 4 — Implement reusable incremental patterns

### T13 — Confirm cursor semantics for each incremental source

**Depends on:** T01, T02.

**Deliver:** per-source key, reliable change cursor, comparison type, fixed upper-bound rule, lookback policy, late-arrival/reconciliation plan, initial bootstrap and deletion behavior. Assign `insert_only`, `upsert` or `replace_entities`.

**Accept:** source evidence supports the ordering assumptions. Event dates or increasing IDs are not treated as guaranteed commit order. Missing cursor guarantees are recorded as blockers or explicit limitations.

### T14 — Implement incremental range and checkpoint handling

**Depends on:** T05, T06, T13.

**Deliver:** bounded extraction, persisted Bronze batch and checkpoint advancement after successful Silver application. Compute lookback from the committed cursor so outages do not truncate recovery to a fixed recent window.

**Accept:** per-table progress is independent. Test empty completed ranges, outages beyond lookback, failure before/after Silver and safe checkpoint retry. Gold failures do not rewind source progress.

### T15 — Implement insert-only chats and parser

**Depends on:** T13, T14.

**Deliver:** decoder chosen from a real anonymized sample, schema-specific parser, required-field extraction, batch dedup and insert-only chat_id writes. Preserve only useful text/fields and provenance. Reject conflicting content for an existing immutable key.

**Accept:** replay adds no duplicates; malformed records block checkpointing; source records match retained fields. Bootstrap covers required history. If IDs are available outside payloads, known chats may skip decompression, provided the agreed immutability/conflict-check policy remains satisfied.

### T16 — Implement mutable and entity-replacement extracts as needed

**Depends on:** T13, T14; actual source inventory determines applicability.

**Deliver:** explicit per-source transformations using two supported behaviors:

- Upsert by key with trustworthy version ordering and agreed delete handling.
- Replace complete sets for explicitly identified entities, including entities whose new result is empty.

**Accept:** older versions do not overwrite newer values; retries are idempotent; deletion handling prevents stale-row resurrection where applicable. Replacement removes obsolete child rows atomically with inserting current rows. Absence from an ordinary delta never implies deletion. Mark a pattern N/A if no MVP source needs it.

## Phase 5 — Complete features and orchestration

### T17 — Implement remaining snapshot/aggregate extracts

**Depends on:** T01, T06; T08 is the reference implementation.

**Deliver:** one named implementation and contract per remaining source, including complete 30-day merchant totals followed by top-five ranking.

**Accept:** deterministic ties, correct units/returns/currency semantics, freshness checks and explicit empty results. Do not derive new rankings solely from prior top five plus new purchases.

### T18 — Complete Gold domain transformations

**Depends on:** T10, T15, applicable T16, T17.

**Deliver:** all required feature sets, latest five chats within 12 months, other contracted windows and final customer assembly. Use pinned input versions and one reference time.

**Accept:** temporal boundaries and ordering are deterministic; expired events disappear even with no new data. Every feature set is customer-unique before joining; full results match independent reference calculations on controlled samples.

### T19 — Complete seven Platinum groups and payload controls

**Depends on:** T11, T12, T18.

**Deliver:** the full feature mapping, seven JSON columns, metadata, source value labels, shared definitions and explicit text/list limits.

**Accept:** every approved feature is deliberately included or excluded. Measure UTF-8 size and model-specific tokens once the model is selected, including instructions. Flag truncation; do not arbitrarily drop fields to fit a budget. Prevent schema/glossary mismatches.

### T20 — Configure manual and scheduled workflows

**Depends on:** T07, T09, T14, T19.

**Deliver:** separate manual and automatic Jobs, domain dependencies, agreed schedules/timeouts, concurrency restrictions, failure notifications and repair instructions. Capture stable Job configuration in the repository.

**Accept:** required failures block dependents; transient retries obey the total attempt limit; independent extraction does not exceed source capacity. Operators can run one extract without restarting everything or accidentally publishing production output.

### T21 — Implement retention and recovery cleanup

**Depends on:** T05, T15, T20.

**Deliver:** approved per-layer retention, cleanup eligibility and recovery rules. Proposed values: 3–5 days for completed automatic Bronze, 13 months of Silver chat history. Protect manual READY inputs still required, failed/active batches, candidates and pinned versions.

**Accept:** cleanup cannot invalidate an active run or supported repair/backfill. Logical deletion and physical Delta cleanup use distinct policies. Document when re-extraction or a new consistent rebuild is required.

### T22 — Verify and release retail MVP

**Depends on:** T19, T20, T21.

**Deliver:** acceptance evidence and an operator runbook covering retries, single-extract runs, backfills, stale manual data, publication failures and recovery.

**Accept:** exercise partial/stale batches, late arrivals, duplicate replay, parser failure, checkpoint crash, window expiry, dictionary failures, payload defaults and publication failure. Validate on DBR 13.3 and selected source records. Release only when required mappings and source contracts are resolved.

**Milestone M2:** the complete retail pipeline produces validated seven-group Platinum input. External LLM execution remains a separate integration unless explicitly added to release scope.

## Post-MVP backlog

### F01 — Shared task-run service table and operational logging

**Depends on:** T22; priority: first operational enhancement.

**Deliver:** `pipeline_ctrl.task_runs`, one record per `(run_id, task_name, attempt_no)`, containing batch, status, start/end/duration, source/target, cursor bounds, read/insert/update/delete/reject/write counts, short error and Job URL. Consolidate overlapping extract-attempt telemetry rather than duplicating it.

**Accept:** write RUNNING then SUCCESS/FAILED; rethrow errors; reconcile abandoned RUNNING records after crashes. Inapplicable metrics remain NULL, and reads are distinguished from inserted rows. Use contextual platform logs with no customer payloads/secrets; detailed event tables remain optional. Watermarks and readiness manifests remain separate authoritative controls.

### F02 — Optimize measured bottlenecks

**Depends on:** T22 and measurements, preferably F01.

**Deliver:** targeted query/layout/parallelism improvements. Consider affected-customer recomputation or richer change feeds only where justified.

**Accept:** prove equality against full recomputation, including expiry, old/new keys and empty results; document performance improvement and recovery implications.

### F03 — Add another customer segment

**Depends on:** T22 and a new segment contract.

**Deliver:** eligibility, global/composite customer-key rules, source permissions, segment-specific features/mappings and compatible Platinum schema.

**Accept:** no cross-segment key collisions or unintended population mixing; retail outputs remain correct. Share mechanics, not assumed business definitions.

### F04 — Integrate LLM execution

**Depends on:** T19 and chosen model/access contract.

**Deliver:** a separate process pinned to Platinum input, model and prompt versions, with inference-specific budget/retry/result handling.

**Accept:** model failures do not rerun ETL; regeneration preserves input provenance. Define idempotency, rate limits and response validation independently of extract retries.

## Adding a new extract or feature after MVP

Repeat T01/T02 contract work, select the T08/T14/T16/T17 implementation pattern, update dependencies and validate affected Gold/Platinum mappings. Adding a source column must not automatically expose it to the LLM. Validate schema compatibility, retain version history and extend acceptance cases before publishing.
