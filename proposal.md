# GSoC Proposal: Legacy Incremental Backup with Tablespace Support (PostgreSQL 14-16)

## Title

Complete tablespace-aware incremental backup support for PostgreSQL 14-16 in `pgmoneta`, with robust backup/restore test coverage and optional performance improvements.

## Detailed Description

`pgmoneta` already contains a legacy incremental backup path for PostgreSQL 14-16 and a restore/combine pipeline that handles incremental chains. The remaining gap is complete **tablespace-aware legacy incremental backup** behavior.  

The project will complete this capability end-to-end by:

1. correctly identifying changed tablespace blocks from WAL-derived summaries,
2. mapping tablespace relation files to the same relation-identity model used by incremental logic,
3. transferring only modified blocks from cluster to backup storage using existing server APIs and extension support,
4. validating that restore remains correct for these new artifacts,
5. building strong scenario coverage, especially for timeline changes and active-write workloads.

This approach keeps implementation aligned with existing PostgreSQL/pgmoneta behavior and avoids reinventing change tracking.

Current scope clarification:

1. legacy incremental backup path for PG14-16 exists, but tablespace support is incomplete,
2. incremental restore workflow already works and is expected to need minimal or no changes, though compatibility will be explicitly validated.

## Quick Code Anchors (for clarity)

Only short anchors are included here to show where work is centered.

### 1) Legacy vs modern dispatch

```c
if (config->common.servers[server].version < 17)
{
   return incr_backup_execute_14_to_16(name, nodes);
}
```

File: `src/libpgmoneta/wf_backup_incremental.c`

### 2) Tablespace work plugs into existing changed-block lookup

```c
brtentry = pgmoneta_brt_get_entry(summarized_brt, &rlocator, frk, &limit_block);
```

File: `src/libpgmoneta/wf_backup_incremental.c`

### 3) Restore already consumes tablespace metadata

```c
for (uint64_t i = 0; i < bck->number_of_tablespaces; i++)
{
   tsoid = parse_oid(bck->tablespaces_oids[i]);
   ...
}
```

File: `src/libpgmoneta/restore.c`

These anchors reflect the implementation strategy: extend legacy backup-side tablespace incremental logic while preserving restore compatibility.

---

## List of Deliverables

### D1. Tablespace-aware legacy incremental implementation

Implement complete handling of `pg_tblspc` relation files in the PostgreSQL 14-16 incremental backup path.

### D2. WAL/BRT-driven tablespace block selection

Use existing WAL summary and BRT infrastructure to identify modified blocks for tablespace-backed relations.

### D3. Correct modified-block transfer and file emission

Use existing read/copy mechanisms (`pg_read_binary_file` path and extension-backed file discovery) to materialize incremental files for tablespace relations.

### D4. Restore compatibility validation (and minimal adaptation if needed)

Verify that current incremental restore/combine flow consumes newly generated legacy tablespace incrementals correctly; update workflow only if required.

### D5. Comprehensive backup+restore test suite

Add real-world and extreme cases, including:

1. incremental backups after timeline changes,
2. incremental backups under active data updates,
3. truncation/segment/fork edge behavior,
4. multi-step chain restore validation.

### D6. PG14-16 version coverage in test execution

Ensure version-aware scenario runs and stable coverage for PostgreSQL 14, 15, and 16.

### D7. Optional performance improvements

Propose and, where feasible, prototype optimizations for sequential pipeline bottlenecks (parallelism/worker pool, memory reuse, copy path optimization).

### D8. Documentation updates

Document behavior, constraints, and scenario coverage for legacy incremental tablespace support.

---

## Approach

This section is intentionally structured around the three priority areas of the implementation.

### A) Tablespace implementation sub-problems (core implementation)

### A1. Parse and classify tablespace relation files in legacy incremental flow

Target:

- `src/libpgmoneta/wf_backup_incremental.c`

Plan:

1. extend relation-file interpretation for `pg_tblspc` paths,
2. derive `spcOid`, `dbOid`, `relfilenode`, `fork`, `segno`,
3. route these files into the same incremental decision path already used for relation files.

Result:

tablespace relations participate in block-level incremental decisions instead of bypass behavior.

### A2. Identify changed blocks per tablespace relation using WAL/BRT

Targets:

- `src/libpgmoneta/walfile/wal_summary.c`
- `src/libpgmoneta/walfile/wal_reader.c`
- `src/include/brt.h`

Plan:

1. reuse WAL summary output and BRT entries keyed by relation locator,
2. query changed blocks for each tablespace relation via existing BRT APIs,
3. preserve existing correctness rules for truncation and fork handling.

Result:

tablespace changed-block detection comes from existing infrastructure, not a parallel custom mechanism.

### A3. Move modified blocks from cluster to pgmoneta safely

Targets:

- legacy read/copy/write path in `src/libpgmoneta/wf_backup_incremental.c`

Plan:

1. reuse server-side binary reads and extension-backed file listing,
2. emit incremental files using existing format and semantics,
3. keep conservative full-copy fallback for unsupported/special cases.

Result:

correct and protocol-consistent data transfer for tablespace incremental files.

### B) End-to-end functional validation via scenario-driven tests

### B1. Real-world scenarios

1. full -> incremental where only tablespace objects are modified,
2. mixed updates across default and user tablespaces,
3. multi-incremental chain restore with tablespace mapping checks.

### B2. Extreme scenarios

1. incremental backup after timeline change,
2. incremental backup during active concurrent updates.

### B3. Edge scenarios

1. relation truncation in tablespace paths,
2. segment rollover (`.1`, `.2`, ...),
3. fork-specific behavior (`main`, `fsm`, `vm`, `init`),
4. metadata consistency and chain linkage.

### B4. Regression scope

1. no behavioral regression in existing restore path,
2. no regression in PG17+ incremental flow.

### C) Bonus: improving brute-force sequential behavior

Plan:

1. profile where sequential legacy pipeline spends most time,
2. identify safe parallelism boundaries (per-file tasks and worker reuse),
3. reduce memory churn in hot copy/write loops by buffer reuse,
4. validate correctness under parallel execution before adopting optimization.

Result:

measured performance gains with correctness preserved.

---

## PostgreSQL Protocol Compliance Checklist

To ensure native implementation soundness, the implementation and test plan will explicitly validate:

1. correct backup start/stop semantics for legacy PostgreSQL versions,
2. correct LSN boundary handling (parent checkpoint to current incremental start),
3. explicit timeline behavior during incremental backup after timeline changes,
4. fork/truncation semantics and conservative fallback behavior,
5. server API and extension-based read flow consistency (`pg_read_binary_file`, extension file discovery),
6. no protocol-semantic drift while introducing performance optimizations.

---

## Implementation Plan by Work Package

### WP1: Foundation and baselining

1. finalize acceptance criteria for each sub-problem,
2. map scenario matrix by PostgreSQL version and workload type,
3. establish reproducible baseline runs.

### WP2: Legacy tablespace incremental logic

1. implement parser/classification changes for tablespace relations,
2. wire tablespace relation identity into BRT-driven decisions,
3. validate block extraction and fallback behavior.

### WP3: Metadata and restore compatibility

1. verify tablespace metadata completeness in backup chain artifacts,
2. validate restore/combine behavior for new artifacts,
3. introduce minimal restore workflow changes only if needed.

### WP4: Scenario expansion and stabilization

1. implement real-world and extreme tests,
2. stabilize test reproducibility and diagnostics,
3. integrate version/scenario coverage consistently.

### WP5: Optimization and finalization

1. prototype selected performance improvements,
2. evaluate before/after behavior and safety,
3. finalize documentation and completion report.

---

## Approximate Schedule (360 Hours)

Assumption: 12 weeks, approximately 30 hours per week.

### Week 1 (30h): Baseline and scope lock

1. reproduce and document current behavior boundaries,
2. finalize acceptance checklist and scenario matrix.

### Week 2 (30h): Test harness preparation

1. prepare PG14/15/16 scenario execution baseline,
2. scaffold tablespace test utilities and workload runners.

### Week 3 (30h): Tablespace path parsing

1. implement robust tablespace relation parsing in legacy path,
2. add parser-focused tests and debug traces.

### Week 4 (30h): BRT integration for tablespaces

1. wire tablespace relations into changed-block lookup path,
2. verify block-selection correctness under controlled WAL activity.

### Week 5 (30h): Data transfer and incremental file correctness

1. validate changed-block fetch/write for tablespace relation files,
2. harden fallback semantics for unsafe/special files.

### Week 6 (30h): Metadata and chain consistency

1. validate tablespace metadata fields across backup chains,
2. verify incremental chain integrity and compatibility.

### Week 7 (30h): Restore integration cycle

1. run end-to-end restore/combine checks for tablespace incrementals,
2. implement workflow adaptations only if required.

### Week 8 (30h): Real-world test scenarios

1. add mixed tablespace/default workload scenarios,
2. add chain-restore assertions and result validation.

### Week 9 (30h): Extreme and edge scenarios

1. add timeline change scenario tests,
2. add active-write during incremental backup tests,
3. add truncation/segment/fork edge tests.

### Week 10 (30h): Coverage and stability

1. consolidate PG14-16 scenario runs,
2. improve flake resistance and failure diagnostics.

### Week 11 (30h): Bonus optimization pass

1. profile sequential bottlenecks,
2. prototype safe parallelism and memory optimizations,
3. verify correctness vs baseline.

### Week 12 (30h): Final polish

1. incorporate review feedback,
2. finalize docs and submission artifacts,
3. publish final deliverable evidence.

---

## Success Criteria

The project is successful when:

1. PostgreSQL 14-16 legacy incremental backups correctly include tablespace-backed changes.
2. Restores from resulting incremental chains are correct and reproducible.
3. Timeline-change and active-write scenarios are explicitly tested with defined expected outcomes.
4. Scenario coverage across PG14/15/16 is stable and maintainable.
5. Any optimization merged preserves correctness and demonstrates measurable benefit.
