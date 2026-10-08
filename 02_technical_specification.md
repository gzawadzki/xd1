# Customer pipeline — technical specification

Version 2.3 · 8 October 2026 · Implementation design, not a deployed system

Related: [overview](01_pipeline_overview.md), [ADRs](03_architecture_decisions.md).

## 1. Runtime and code

Target DBR 13.3 / Spark 3.4.1 [S1], existing metastore and `database.table` names. No UC Volumes or newer-runtime dependencies. Missing Unity Catalog is an environment constraint, not implied by the runtime version.

Use named SQL, manual/automatic extract modules, Silver/Gold/Platinum transformations, shared IO/checks and thin task notebooks. Install modules explicitly. Separate environment databases; never depend on another notebook's state.

## 2. Execution and handoff

**Manual:** create a WRITING batch; extract into attempt-specific staging; validate all required members; mark READY. READY data is immutable. Failed/incomplete attempts remain unavailable; retries cannot combine fragments from different attempts.

**Automatic:** create run_id, fixed as_of_ts and code version; select an acceptable READY batch; persist its identity; execute MSSQL extracts and Silver updates; pin calculation inputs, including Delta versions; build, validate and publish candidates.

Tasks reuse pinned inputs rather than selecting “latest” again. Initially select one complete batch for related manual extracts; independent consistency groups may use separate batches. Set max_age per extract and retain source_as_of where available: recent ingestion does not prove recent source data. Missing acceptable inputs block publication.

## 3. Control tables

| Table | Logical key | Main fields |
|---|---|---|
| manual_batches | manual_batch_id | status, started_at, ready_at, source_as_of, code_version |
| extract_runs | batch_id, extract_name | attempt_id, status, table_name, row_count, lower_cursor, upper_cursor, error |
| runs | run_id | as_of_ts, status, code_version, schema_version, start/end times |
| run_inputs | run_id, input_name | batch_id, table_name, delta_version, source_as_of |
| watermarks | source_id | cursor_type/value, committed_batch_id, updated_at |

Enforce logical uniqueness in application checks. Use one active automatic run and one state writer. Never store credentials or executable business logic in control tables.

## 4. Extract contracts and readers

Each contract specifies source/adapter, SQL, columns/types, grain/key, snapshot/incremental mode, availability cursor, update/delete behavior, units/timezone, empty-result policy, freshness and consumers.

Inventory contracts for all sources. Customers/products use snapshots; merchant rankings follow complete window aggregation; immutable chats use insert-only increments. Mutable sources need separate update/delete rules. Use explicit projections, never SELECT *.

**pymssql:** bind values, use fetchmany and explicit Spark schemas; write chunks into attempt staging rather than accumulating them on the driver. Commit only complete extracts. Empty results retain schema and completion status.

**JDBC:** budget connections across extracts. Partition bounds divide work, not filter rows [S2]. Separate connections do not guarantee a shared snapshot; use source staging/snapshot when required. Persist extraction before counts/checks to avoid repeated reads of changing data.

## 5. Immutable chat increments

Source availability ordering is a prerequisite. Conversation timestamps and increasing IDs do not prove commit order. `source_available_at` is a semantic requirement, not a confirmed source column.

Prefer a reliable availability cursor. Timestamp extraction needs fixed bounds, overlap and reconciliation; arbitrary late commits still require stronger source guarantees or CDC/CT. Overlap is independent of Bronze retention.

1. Read committed cursor C; persist bounds L/U, optionally L = C minus overlap.
2. Extract `[L,U)` and persist the complete raw batch.
3. Decompress, parse and validate identifiers, customer and time.
4. Deduplicate identical rows; reject conflicting content for the same chat_id.
5. Insert only previously unseen chat_id values into Silver.
6. After successful persistence, commit the batch and advance to U.

A completed empty extract may advance to U under the source contract. Parsing errors block the batch; never silently discard rows. Insert-only MERGE still requires source deduplication [S3].

**Recovery:** before Silver, replay persisted Bronze; after Silver but before watermark, repeat the idempotent insert and commit; after Gold failure, reuse pinned inputs without rolling back extraction. Confirm the real cursor before implementation.

## 6. Parsing and retention

Choose decompression only after inspecting a sample; neither codec nor JSON structure is known. Silver stores required fields, not the full expanded JSON alongside them.

| Field | Type / meaning |
|---|---|
| chat_id | STRING; globally unique or combined with source_id |
| customer_id | STRING |
| chat_ended_at | Normalized TIMESTAMP |
| topic, text | STRING; required-text and message-joining rules in contract |
| source_available_at | TIMESTAMP, if provided |
| loaded_at, parser_version | Ingestion provenance |
| payload_hash | Optional canonical-content hash for conflict detection |

Proposed Bronze retention: 3–5 days after successful processing, excluding active, failed or still-required batches. Keep manual batches through replacement and dependent repair periods. Proposed Silver retention: 13 months for 12-month features; preserve agreed backfills.

Business windows use chat_ended_at; ingestion uses the source cursor. Logical Delta deletion does not immediately reclaim old files. Physical cleanup must preserve pinned versions and recovery retention, not blindly follow Bronze's proposed age limit.

## 7. Gold

Use an explicit customer population and unique customer_id in every feature set before joining. Monetary fields use agreed DECIMAL types. Derive all boundaries from one as_of_ts: Europe/Warsaw business time, normalized UTC storage. Specify calendar-month and day windows explicitly; 12 months is not necessarily 365 days.

For chats, filter `[window_start, as_of_ts)`, apply row_number per customer ordered by chat_ended_at DESC, chat_id DESC, select five, then collect and explicitly sort structs by rank. Validate keys beforehand; after the customer join, fill confirmed absence with a typed empty array. collect_list alone does not guarantee order.

Initially rebuild Gold from required history even with empty deltas: events expire. Merchant top five requires complete window totals, not previous top five plus new purchases.

## 8. Value dictionaries and nulls

Fetch **value dictionaries** from MSSQL/DB2 through their configured manual/automatic extract path. They translate values, e.g. `business_code=5` to the matching `desc_pl="affluent"`. Pin dictionary snapshots with other run inputs and validate freshness, unique typed keys, nonempty descriptions and coverage of non-null codes. Historical mappings require effective-date rules.

Apply classification mappings during Gold calculations; presentation labels during Platinum preparation. Preserve Gold codes where useful. Unknown codes block required output unless an explicit fallback exists; they must not silently become omitted nulls. Preserve the agreed description language; English documentation does not imply translating desc_pl.

| Case | Result |
|---|---|
| No events, complete source | Count 0 |
| Undefined average or last-event date | NULL |
| Known inactive flag | false |
| Unavailable required source/join | Error, not zero |
| Confirmed empty event list | [] |

Apply defaults where missing values arise, using an explicit column list. No global fillna(0).

## 9. Platinum

Maintain a separate **feature dictionary**, authored manually as `contracts/feature_dictionary.csv` and versioned in Git. It explains technical variable names and controls explicit selection/grouping/renaming, not arbitrary business calculations.

| gold_column | platinum_group | json_key | description | unit | null_policy | value_dictionary |
|---|---|---|---|---|---|---|
| dc_pos_30d_amt | transactions | debit_card_pos_amount_30d | Amount of debit card POS payments in the last 30 days | PLN | zero_if_no_events | — |
| business_code | customer_info | customer_segment | Customer business segment | — | omit | business_codes |

Also record data_type, window and content_limit where relevant. `zero_if_no_events` documents a Gold rule, implemented only with complete inputs. Obtain the actual feature inventory before filling all mappings.

Validate Gold column existence, complete intended coverage, allowed groups, unique `(platinum_group,json_key)`, descriptions, types and referenced dictionaries. Generate a shared glossary from this file: full definitions appear once in LLM instructions, not in every customer payload. Pin the feature-dictionary revision with schema_version and prompt version; verify compatibility before inference. Short readable JSON keys preserve meaning without repeating definitions.

Output: customer_id, run_id, reference time, schema_version and seven JSON STRING columns. Build STRUCT/ARRAY values first; serialize each group once [S4].

```python
payload = F.to_json(F.struct(
    F.col("app_logins_30d"),
    F.col("last_app_login"),
    F.col("uses_mobile_app")
), options={"ignoreNullFields": "true"})
```

Omit null object fields; preserve 0/false; empty groups become `{}`. Define empty nested objects, null array elements and empty arrays separately: the option does not recursively remove all empty content. Avoid double encoding subgroups as strings.

Tell the LLM that absent fields mean unavailable values, not zero. Customer identifiers need not enter prompts. Measure UTF-8 bytes and model-specific tokens including instructions; characters are not tokens. Set explicit text/list limits and indicate truncation. Smaller JSON does not guarantee proportional Delta storage savings.

## 10. Publication and verification

Manual workflow: initialize → JDBC extracts → validate group → READY.
Automatic workflow: initialize/select inputs → MSSQL → Silver → domain features → Gold candidate/DQ → Platinum candidate/DQ → publication. Dependencies are domain-specific [S5].

Publish validated, persisted candidates using atomic single-table Delta overwrite with fixed schemas. Gold and Platinum are not one transaction; retain run_id and let LLM consumers read only published Platinum. Retry publication from the same candidate. Exceptions remain failures; retry transient faults only. Missing pinned versions require a new consistent rebuild.

Test on DBR 13.3; this design is not an executed implementation.

Required tests: missing feature mappings, duplicate JSON keys, glossary/schema version mismatch; incomplete/stale manual batches; mid-run replacement; outage exceeding Bronze retention; duplicate replay; changed immutable content; parsing failure; crash between MERGE/watermark; empty delta with window expiry; deterministic ties; unknown/duplicate reference codes; NULL versus 0/false; empty group `{}`; failed validation preserving published output.

## 11. MVP evolution and operations

Scope: retail MVP. Add segments through explicit eligibility/grain contracts; confirm customer_id uniqueness across segments before combining them. Add sources/extracts through named SQL, contracts and tasks; add columns through reviewed schema and feature-dictionary changes. New columns do not automatically enter LLM payloads. Keep shared mechanics small; avoid premature segment frameworks.

Expose manual execution of one named extract with optional range overrides and a batch ID. Default: staging and validation only, without moving production watermarks or publishing Gold/Platinum. Normal incremental repairs reuse pinned bounds and idempotent writes. Explicit backfills use separate batches and do not rewind/advance the regular cursor; rebuild dependents deliberately after successful promotion. Prevent concurrent writes to the same target.

Proposed retry policy: at most two automatic retries (three attempts total), five minutes apart, for transient connection/time-out failures. Schema, parsing, key, permission and configuration failures stop immediately. Enforce classification in shared retry handling; do not stack reader and Job retries. Manual reruns are explicit new attempts, not an unlimited automatic loop.

Post-MVP backlog: `pipeline_ctrl.task_runs`, one row per `(run_id,task_name,attempt_no)`, recording batch, status, start/end/duration, source/target, range, read/insert/update/delete/reject/write counts, short error and Job URL. Inapplicable metrics stay NULL. Log RUNNING then SUCCESS/FAILED; rethrow errors and reconcile abandoned RUNNING entries after crashes. Start with contextual task logs, not a separate event table. This registry replaces overlapping extract-attempt telemetry, not watermarks or READY manifests. Essential run/cursor/batch controls remain MVP requirements.

## 12. Outstanding inputs and references

Confirm source/feature inventory, seven group names, codec/JSON structure, actual cursor, source retention, freshness/cleanup limits, window definitions, references/fallbacks, model budgets, storage and permissions.

- [S1](https://learn.microsoft.com/en-us/azure/databricks/release-notes/runtime/13.3lts): runtime.
- [S2](https://spark.apache.org/docs/3.5.6/sql-data-sources-jdbc.html): JDBC semantics; validate on Spark 3.4.1.
- [S3](https://docs.databricks.com/aws/en/delta/merge): MERGE; exclude newer-runtime extensions.
- [S4](https://dlcdn.apache.org/spark/docs/3.4.3/configuration.html): JSON configuration.
- [S5](https://docs.databricks.com/aws/en/jobs/run-if): dependencies; [parameters](https://docs.databricks.com/aws/en/jobs/parameters).
