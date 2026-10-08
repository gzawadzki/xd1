# Architecture decision records — customer pipeline

Version 2.3 · 8 October 2026 · Condensed English edition

Related: [overview](01_pipeline_overview.md), [technical specification](02_technical_specification.md).

Supersedes the October 7 design; GitHub remains unchanged. Agreed direction does not imply deployment; proposed parameters require confirmation.

## ADR-001 — Separate manual and automatic ingestion

**Status:** Agreed. **Context:** JDBC sources lack technical-user access; MSSQL/pymssql can run automatically.

**Decision:** Manual imports produce validated READY batches. Automation selects a complete, sufficiently fresh batch and pins its identity throughout the run.

**Rejected:** Scheduled JDBC assuming available user credentials; reading the latest table without completeness/freshness checks.

**Consequences and verification:** Define per-extract freshness limits and protect selected batches from cleanup. Separate process IDs are sufficient. Reject partial/stale batches; mid-run imports must not change selected inputs.

## ADR-002 — Existing runtime and metastore

**Status:** Environment constraint. **Context:** DBR 13.3 without Unity Catalog.

**Decision:** Use Delta, the existing metastore, database.table names and available storage/permissions. Require neither UC Volumes nor newer-runtime features.

**Verification:** Test locally on DBR 13.3; current documentation alone does not prove compatibility. UC availability is separate.

## ADR-003 — Retail MVP with explicit extension points

**Status:** Agreed. **Context:** Retail MVP with approximately 15 extracts; future segments, sources, tables and columns are expected.

**Decision:** Named SQL, small Python modules, thin notebooks and explicit task dependencies. Share repetitive connection, IO and technical checks only.

**Consequences and verification:** Add explicit contracts/tasks and reviewed schema mappings; new columns do not automatically enter Platinum. Check cross-segment customer-key uniqueness. Each extract runs independently without notebook state.

## ADR-004 — Source-side aggregation

**Status:** Agreed. **Context:** Raw product/transaction volumes exceed feature requirements.

**Decision:** Bronze accepts contracted source aggregates: products by type and merchants ranked after complete window aggregation.

**Consequences and verification:** Reduced transfer, retained source workload and lost detail. Changing top five to top ten may require source history. Compare against control SQL; refreshed snapshots must remove obsolete groups.

## ADR-005 — Temporary Bronze, useful Silver history

**Status:** Direction agreed; retention proposed. **Context:** Compressed payloads plus complete parsed copies duplicate storage.

**Decision:** Keep replayable Bronze and selected fields/text in Silver, without unnecessary expanded JSON copies. Propose 3–5 days for completed automatic Bronze batches and 13 months for 12-month Silver features.

**Consequences and verification:** Parser changes may require source rereads. Protect failed/active batches, required manual inputs and repair windows. Distinguish logical record retention from physical Delta file cleanup; verify both preserve recovery.

## ADR-006 — Insert-only immutable chats

**Status:** Mechanism proposed; cursor unconfirmed. **Context:** Completed chats cannot change, but availability ordering is unknown.

**Decision:** Persist fixed extraction bounds and Bronze; parse, validate, deduplicate and insert unseen chat_id values. Commit watermark after Silver, independently of Gold. Conflicting content under one ID violates the contract.

**Rejected:** Always reading only five recent days; unguarded append; assuming event dates or increasing IDs guarantee commit order.

**Consequences and verification:** Confirm the cursor before implementation. Timestamp overlap requires bounded-lateness assumptions and reconciliation, not unconditional completeness claims. Test long outages, late arrivals, empty deltas and crashes after MERGE.

## ADR-007 — Customer-grain Gold

**Status:** Agreed. **Context:** Joins must preserve customer grain; time windows change without new records.

**Decision:** One row per customer, approximately 120 explicit features, structured event arrays and initially full recomputation. Chats: filter 12 months, rank deterministically, retain at most five.

**Rejected:** Raw one-to-many joins; updating only customers with new deltas.

**Consequences and verification:** Higher computation cost buys simpler expiry logic. Incremental optimization must match full results. Check customer uniqueness, tie ordering and expiry on empty deltas.

## ADR-008 — Platinum as the LLM contract

**Status:** Agreed; mapping pending. **Context:** Features must form seven thematic JSON objects.

**Decision:** Map validated Gold explicitly and serialize each group once. Keep customer_id, run_id, reference time and schema_version separately. Do not reread sources or recalculate business definitions.

**Rejected:** Name-based automatic grouping, string concatenation and double-encoded nested objects.

**Consequences and verification:** Approve actual group names and feature mappings. Define explicit content limits rather than silently dropping fields. Validate JSON, types, mapping coverage and prompt-inclusive size.

## ADR-009 — Separate value and feature dictionaries

**Status:** Distinction agreed; storage/versioning proposed. **Context:** Code values and technical feature names require different mappings and owners.

**Decision:** Fetch value dictionaries from MSSQL/DB2: `business_code=5` → `desc_pl="affluent"`. Pin snapshots and validate keys, descriptions, freshness and coverage. Apply classifications in Gold and presentation labels in Platinum; preserve the agreed language.

Maintain the manually authored feature dictionary in Git as CSV: Gold column, Platinum group, JSON key, full description, type, unit/window, null policy, value-dictionary reference and limits. Example: `dc_pos_30d_amt` means “Amount of debit card POS payments in the last 30 days”.

**Rejected:** Treating both mappings as source tables; repeating full definitions in every customer JSON; executable business logic in the mapping file; silently dropping unmapped codes.

**Consequences and verification:** Use short readable payload keys and generate shared LLM definitions from the feature dictionary. Version the glossary, schema and prompt compatibly. Test value coverage, feature coverage, duplicate JSON keys, column existence and version mismatches. Gold implements documented zero rules; Platinum formats values.

## ADR-010 — Resolve missing-value meaning before serialization

**Status:** Proposed. **Context:** Payload reduction must preserve meaningful absence of activity.

**Decision:** Assign 0/false only when complete inputs justify it. Omit remaining null fields using ignoreNullFields; empty groups become {}, confirmed empty lists remain [].

**Rejected:** Global fillna(0), dropping zeros and treating source failures as inactivity.

**Consequences and verification:** Each feature needs a null policy; instruct the LLM that omission does not mean zero. Test NULL/0/false separately. Reduced JSON size does not guarantee proportional disk savings.

## ADR-011 — Pinned inputs and validated publication

**Status:** Proposed. **Context:** Inputs may change mid-run; writes and status updates can fail separately.

**Decision:** Persist batch/Delta-version inputs and validate durable Gold/Platinum candidates before atomic single-table publication. Use one automatic state writer.

**Rejected:** Selecting latest inputs on every retry; publication before checks; assumed cross-table atomicity.

**Consequences and verification:** Retain inputs/candidates for repairs. Gold and Platinum may temporarily have different run_ids; LLM reads published Platinum only. Test crashes after publication and failed checks preserving prior output.

## ADR-012 — Independent LLM execution

**Status:** Proposed direction. **Context:** Prompt/model changes and generation failures should not repeat ETL.

**Decision:** Run inference separately against identified Platinum input; record input, prompt and model versions.

**Consequences and verification:** ETL is independent of model availability. Specify inference retries, limits and idempotency after choosing integration. Regeneration must not reread sources.

## ADR-013 — Manual operations, bounded retries and deferred telemetry

**Status:** Manual execution and telemetry backlog agreed; retry limits proposed.

**Decision:** Permit isolated extract runs; default to validated staging without production publication or cursor changes. Repairs reuse bounds; backfills use separate batches without changing the regular watermark. Promote deliberately; serialize target writes.

Allow two retries, three attempts total, five minutes apart, only for transient failures. Contract/configuration errors fail immediately; prevent nested retry multiplication.

**Consequences and verification:** Post-MVP, add a shared task_runs registry for attempts, timings, row counts, errors and log links, consolidating extract telemetry. Keep watermarks/manifests in MVP. Test safe reruns and retry limits; detailed event logging remains deferred.
