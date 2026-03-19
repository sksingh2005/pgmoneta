# GSoC Proposal: Legacy Incremental Backup with Tablespace Support (PostgreSQL 14-16)

## Title

Complete tablespace-aware incremental backup support for PostgreSQL 14-16 in `pgmoneta`, with robust backup and restore test coverage and optional performance improvements.

## Description

`pgmoneta` already contains a legacy incremental backup path for PostgreSQL 14-16 and a restore/combine pipeline for incremental chains. The remaining gap is complete **tablespace-aware legacy incremental backup** support.

This project aims to complete that capability end-to-end by:

1. correctly identifying changed tablespace blocks from WAL-derived summaries,
2. mapping tablespace relation files to the same relation-identity model used by incremental logic,
3. transferring only modified blocks from cluster to backup storage using existing server APIs and the pgmoneta extension,
4. producing correct incremental artifacts and metadata for tablespace-aware chains,
5. ensuring that restore remains correct for these new artifacts,
6. building strong scenario coverage, especially for timeline changes and active-write workloads.

This approach stays aligned with PostgreSQL's own protocols and avoids reinventing change tracking.

Current status:

1. legacy incremental backup path for PG14-16 exists, but tablespace support is incomplete,
2. incremental restore workflow already works and is expected to need minimal or no changes, though compatibility will be explicitly validated,
3. PostgreSQL 17+ uses native server-side incremental backup, while PostgreSQL 14-16 uses pgmoneta's preview legacy block-tracking path; therefore, the project will prioritize correctness and compatibility first and pursue low-risk optimizations only afterward.

---

## List of Deliverables

### Core Deliverables

1. Tablespace-aware legacy incremental implementation:
Implement complete handling of `pg_tblspc` relation files in the PostgreSQL 14-16 incremental backup path.

2. WAL/BRT-driven tablespace block selection:
Use existing WAL summary and BRT infrastructure to identify modified blocks for tablespace-backed relations.

3. Correct backup artifacts and metadata for legacy tablespace incrementals:
Populate `backup->tablespaces*` metadata correctly for PostgreSQL 14-16 incrementals, ensure `backup_manifest` paths for tablespace files are correct, and keep backup-side tablespace metadata consistent with the existing restore/combine flow.

4. Restore compatibility validation:
Confirm that the current incremental restore/combine flow consumes newly generated legacy tablespace incrementals correctly and update the workflow only if required.

5. Correct modified-block transfer and file emission:
Use existing read/copy mechanisms (`pgmoneta_server_read_binary_file` and pgmoneta extension file discovery) to materialize incremental files for tablespace relations.

6. Comprehensive backup and restore test suite:
Add real-world and extreme cases covering timeline changes, active-write workloads, truncation, segment and fork edge behavior, and multi-step chain restore.

7. PG14-16 version coverage in test execution:
Ensure stable scenario coverage for PostgreSQL 14, 15, and 16, including explicit PG16 test-environment and CI alignment.

8. Documentation updates:
Document behavior, constraints, and scenario coverage for legacy incremental tablespace support, including:

- an administrator guide covering prerequisites (required permissions, tablespace mount accessibility, and expected server-side capabilities), operational constraints, and troubleshooting for common tablespace backup failures;
- architecture notes describing how `pg_tblspc` paths are discovered, how relation-file paths are parsed and mapped, how backup-side tablespace directories and links are laid out, and how WAL/BRT state is used for tablespace relations;
- compatibility guidance for existing backup chains, including when a new full backup is required before adopting tablespace-aware legacy incrementals.

### Stretch Deliverable

1. Targeted backup-latency improvements:
Implement at most a small set of low-risk improvements in the legacy incremental path where time permits, prioritizing batched reads, cached capability checks, file-size metadata reuse, and other optimizations that do not change backup semantics.

---

## Approach

Structured around the three priority areas defined by the maintainer.

---

### Priority 1: Tablespace Implementation and Artifact Correctness

The current legacy incremental backup path handles only `base/` and `global/` relation paths. All tablespace files under `pg_tblspc/` are currently full-copied instead of being incrementally backed up. The implementation can be broken into six focused tasks:

#### 1. Discover tablespace structure from the existing file listing

- Currently, the 14-16 path hardcodes `backup->number_of_tablespaces = 0` and never discovers tablespace information.
- The `get_paths()` function already processes the response from `pgmoneta_ext_get_files(".")`, which returns all server files including `pg_tblspc/` entries. It also already creates the corresponding directories in the backup.
- Tablespace OIDs can be extracted directly from the returned paths (e.g., `pg_tblspc/16385/PG_<major>_<catalog_version>/...` → `spcOid = 16385`). This will be used as the primary discovery mechanism for routing relation files.
- A small metadata lookup may still be required to populate stable human-readable tablespace names and original paths for backup metadata and restore mapping.
- During the file enumeration, collect unique tablespace OIDs and version directories and populate `backup->tablespaces_oids[]`, `backup->tablespaces[]`, and `backup->tablespaces_paths[]` consistently for legacy incrementals.
- The first implementation should match the existing full-backup tablespace layout: backup-side tablespace directories plus `pg_tblspc/<oid>` symlinks, because the current restore/combine path already assumes that model.
- Tablespace identity should remain OID-based. If a tablespace is dropped and recreated with the same name, the new OID should be treated as a new tablespace in backup metadata and restore mapping.

#### 2. Parse tablespace relation file paths into `rel_file_locator`

- `parse_relation_file` handles only `base/` and `global/` prefixes today. The `else` branch does `goto done;`, returning success without populating the `rlocator`, so tablespace paths are effectively excluded from incremental decisions.
- PostgreSQL stores tablespace relations under `pg_tblspc/<spcOid>/PG_<major>_<catalog_version>/<dbOid>/<relfilenode>` (as defined in PostgreSQL's `relpath.h`).
- The parser should not hardcode version-directory literals. It should accept the PostgreSQL `PG_<major>_<catalog_version>` form dynamically and reuse/fix the existing version-directory helper logic where possible.
- Add a third parsing branch to extract `spcOid`, `dbOid`, and `relfilenode` from this path structure, and create the corresponding backup directory.
- While working on this parser, also verify handling of segment numbering, forks, and temporary relation naming so the change does not accidentally hardcode only the simplest permanent-relation filename form.
- The important distinction is physical location: `pg_tblspc/<oid>` is only the cluster-relative entry point, while the real relation data lives in the mounted tablespace directory reached through that link. The legacy incremental path should therefore treat `pg_tblspc/<oid>/PG_<ver>/<dbOid>/<relfilenode>` as the logical cluster path for lookup and transfer, without assuming that the bytes physically reside inside `pg_tblspc` itself.
- **Symlink handling**: `pg_tblspc/<oid>` is part of the backup-side layout, not the main changed-block problem. Only relation files underneath the version directory (`pg_tblspc/<oid>/PG_<ver>/<dboid>/<relfilenode>`) go through incremental block-level treatment. The legacy incremental artifact should preserve the existing symlink-oriented tablespace layout expected by full backup and restore paths.

#### 3. Route tablespace files through the incremental decision path

- The main backup loop currently skips anything not starting with `base` or `global` by doing a full copy.
- Add `pg_tblspc` to the filter so tablespace relation files flow through the same incremental decision logic: parse → BRT lookup → incremental or full write.

#### 4. Reuse existing WAL/BRT relation tracking and validate `RM_TBLSPC` coverage

- `pgmoneta_wal_record_summary` in `wal_reader.c` iterates every WAL record's block references and calls `pgmoneta_brt_mark_block_modified(brt, &rlocator, forknum, blocknum)`.
- The `rlocator` extracted from WAL records already includes the correct `spcOid` when the block belongs to a tablespace relation. This is how PostgreSQL encodes block references in WAL.
- The existing special handling for `XLOG_DBASE_CREATE_*` and `XLOG_SMGR_CREATE` should already help classify newly created database/tablespace storage and newly created relation forks correctly.
- In addition, the WAL summarizer should be checked for explicit `RM_TBLSPC` (`XLOG_TBLSPC_CREATE` / `XLOG_TBLSPC_DROP`) handling. This will be treated as a verification task first; dedicated summarizer or BRT handling will be added only if it is required for correctness in create, drop, or move tablespace scenarios.

#### 5. Ensure legacy backup artifacts remain correct for tablespace chains

- For PostgreSQL 14-16, `pgmoneta` generates the incremental `backup_manifest` locally, so tablespace-aware changes must preserve correct relative paths and checksums for restore and combine.
- Legacy incremental chains should preserve the tablespace metadata and path information already used by restore/combine, especially for tablespace creation, drop, or movement across backups.
- The goal is to keep tablespace-aware legacy incrementals fully reconstructable by the existing restore and combine flow without introducing hidden format mismatches between backup, manifest, and restore metadata.

#### 6. Transfer modified tablespace blocks to the pgmoneta side efficiently

- First evaluate whether the existing admin/server file APIs are sufficient for tablespace-backed relation files through cluster-relative paths under `pg_tblspc/...`.
- If `pgmoneta_server_read_binary_file` and related server-side file access can read the required tablespace relation files correctly, reuse that path and avoid extra extension-specific transfer logic.
- If the server APIs do not cover the required access pattern for tablespace data, fall back to extension-assisted transfer for those files.
- For file discovery, `create_standard_directories` already calls `pgmoneta_ext_get_files(ssl, socket, ".", &qr)` which lists `pg_tblspc/` entries and can be reused to drive the local backup-side tablespace directory creation.
- This should be checked against tablespaces on different mount and permission setups so the backup path remains reliable even when tablespace directories differ from the main data directory.

#### PostgreSQL protocol compliance

- Uses `pg_backup_start`/`pg_backup_stop`, the correct backup APIs for PG14-16.
- WAL summarization uses the previous checkpoint LSN up to the current start LSN, matching pgmoneta's legacy change-capture model for PG14-16.
- Tablespace path parsing follows PostgreSQL's `relpath.h` convention.
- FSM forks are excluded from incremental tracking because PostgreSQL does not fully WAL-log them.
- Block reads should prefer PostgreSQL's standard server-side file access APIs where they are sufficient; extension fallback is reserved for cases those APIs cannot cover.
- Backup-side tablespace layout should match the existing full-backup model: local tablespace directories plus `pg_tblspc/<oid>` symlinks. If manifest generation needs adjustment, that should be solved without changing the expected layout model.
- Only relation files get incremental block-level treatment; non-relation files and layout artifacts follow full-backup semantics.
- Unchanged tablespace relation files should follow the same legacy incremental state machine as unchanged relation files elsewhere: emit empty header-only incremental artifacts rather than relying on symlinks or parent-file reuse.
- **Newly created tablespace/database check**: PostgreSQL's `GetFileBackupMethod` checks if a database OID/tablespace OID pairing was created since the previous backup (BRT entry with `relNumber=0`). If so, everything in that pairing must be fully backed up. This should be evaluated and adopted as feasible.
- **Artifact correctness**: backup metadata, generated manifests, and restore-time tablespace mapping behavior must remain consistent with the final restore/combine workflow.
- **Full-backup threshold**: PostgreSQL-inspired heuristics such as switching to full-copy when a very large fraction of blocks changed will be considered only as stretch optimization work after correctness is complete.
- If timeline continuity cannot be guaranteed safely, fail fast with a clear error and require a new full backup; explicit support for cross-timeline legacy incrementals is not assumed as a core deliverable.

---

### Priority 2: Test Coverage

Current test coverage is minimal. Only basic chain-creation and restore scenarios are tested. There is no dedicated coverage for tablespaces, timeline changes, active-write workloads, or important edge cases. The following scenarios will be added:

#### Real-world scenarios

**1. Tablespace-only modifications**

1. Create a user tablespace and a table within it, take a full backup.
2. Modify only tablespace data, take an incremental backup.
3. Verify: only tablespace blocks appear in incremental, metadata includes tablespace info.
4. Restore and verify data integrity and tablespace symlink reconstruction.

**2. Tablespace creation between backups**

1. Full backup (no user tablespaces), then create a tablespace and table, insert data.
2. Incremental backup.
3. Verify: new tablespace fully captured, metadata includes the new OID and path.
4. Verify that backup metadata and `pg_tblspc/<oid>` links are updated consistently.
5. Restore and verify.

**3. Tablespace drop between backups**

1. Full backup with user tablespace, then move table to default tablespace and drop the user tablespace.
2. Incremental backup.
3. Verify: restore handles absence of dropped tablespace, data is accessible in the default tablespace, and stale tablespace metadata is not retained.

**4. Mixed default and tablespace updates**

1. Tables in both default and user tablespace, full backup, update both, incremental backup.
2. Restore and verify data integrity across both locations.

**5. Relation moved between tablespaces**

1. Full backup with a table in the default tablespace.
2. Move it with `ALTER TABLE ... SET TABLESPACE`, then take an incremental backup.
3. Verify the new storage is treated as new relation data rather than as an in-place modification of the old file.
4. Restore and verify the table is accessible from the new tablespace location.

#### Extreme scenarios

**1. Incremental backup after timeline change**

1. Full backup on timeline 1.
2. Promote a standby (creates timeline 2), insert data on new timeline.
3. Incremental backup.
4. Verify the expected behavior explicitly:
   - if supported: correct changed-block capture and correct restore output,
   - if not supported safely: explicit fail-fast with clear diagnostic and no partial/incorrect artifacts.

**2. Incremental backup during active concurrent updates**

1. Full backup, start `pgbench -T 60 -c 4` workload.
2. While pgbench is running, trigger incremental backup.
3. Restore and verify consistency at the backup stop LSN, with no corruption or missing blocks.

#### Edge scenarios

**1. Large relation spanning multiple segments**

1. Create a table exceeding `relseg_size` (multiple `.1`, `.2`, ... segments) in a tablespace.
2. Full backup, modify blocks across different segments, incremental backup.
3. Verify: only modified segments' blocks in the incremental, segment number calculation correct, restore reconstructs the multi-segment relation.

**2. Fork-specific behavior**

1. Full backup, then `VACUUM` to update the visibility map fork.
2. Verify: `vm` fork incrementally backed up (WAL-logged), `fsm` fork full-copied (not fully WAL-logged), `init` fork handled correctly.

**3. Relation truncation in tablespace**

1. Full backup with large tablespace table, `TRUNCATE` the table, incremental backup.
2. Verify: BRT `limit_block` set to 0 from SMGR truncation record, full backup triggered for the truncated file, restore produces empty table.

#### Regression scenarios

**1. Multi-step incremental chain restore**

1. Full → incremental1 → incremental2 → incremental3 (tablespace mods at each step).
2. Restore from latest, verify chain reconstruction is correct.
3. Verify that `backup_manifest` includes the generated tablespace incremental artifacts with correct relative paths, sizes, and checksums.
4. Verify that tablespace metadata saved in the chain matches the final chain state.
5. Verify that the `pg_tblspc/<oid>` links produced in the backup and restore layout are valid and point to the expected local tablespace directories.

**2. No regression in PG17+ flow**

1. Run existing tests against PG17+ to ensure tablespace changes don't affect the modern path.

All of these scenario families, except the PG17+ regression check, will be parameterized for PG14, 15, and 16 using the existing MCTF test framework.

---

### Priority 3: Reducing Backup Latency

The current `incr_backup_execute_14_to_16` implementation processes all files sequentially in a single loop. Several factors contribute to backup latency and can be improved:

These improvements are intentionally treated as stretch work. The primary project goal remains correctness and test coverage for legacy tablespace-aware incrementals. Only low-risk changes that do not broaden the project beyond 360 hours will be pursued.

#### 1. Batch contiguous block reads

- `write_incremental_file` issues one `pgmoneta_server_read_binary_file` call per modified block, which means one network round-trip per block.
- Since the block array is already sorted, contiguous block ranges can be detected and fetched in a single call by specifying a larger byte range.
- This reduces network round-trips from N blocks to the number of contiguous ranges.

#### 2. Pre-allocate a reusable block buffer

- The block array (`incr_blocks`) is allocated via `malloc` and freed for every relation file in the loop.
- Pre-allocating a single buffer sized to `rel_seg_size` before the loop and reusing it across all files eliminates per-file allocation overhead.

#### 3. File-level parallelism via worker pool

- Each file's processing (parse → BRT lookup → block read → write) is independent.
- pgmoneta already provides a worker pool (`workers.h`) used in the restore path (`do_reconstruct_backup_file`, `do_copy_backup_file`).
- A worker function can encapsulate the per-file logic and dispatch files to the pool.
- Constraint: each worker needs its own server connection since block reads go through `pg_read_binary_file`. Connection management adds complexity.
- Parallelism is safe because the BRT is read-only during the backup loop (summarization is complete before it starts).
- Because of the connection-management and failure-handling complexity, this is a late stretch goal or prototype candidate rather than an expected core deliverable.

#### 4. Reorder BRT checks before file stat

- The main loop currently calls `pgmoneta_server_file_stat` (network round-trip) for every relation file, then checks BRT.
- Moving the BRT check before the stat call allows earlier short-circuit for unchanged files.
- `pgmoneta_ext_get_files` already returns size metadata; if cached during `create_standard_directories`, unchanged-file paths can use cached size and avoid extra `pgmoneta_server_file_stat` calls.
- If cached size is unavailable or considered stale for a path, fall back to `pgmoneta_server_file_stat` for correctness.

#### 5. Overlap server reads with local writes

- Currently, block reads from the server and writes to the local file happen strictly sequentially: read a block, write it, then read the next.
- A double-buffering approach can overlap network I/O with disk I/O: while one buffer is being written to disk, the next block is fetched by a prefetch stage.
- This requires asynchronous prefetch (reader/writer split or non-blocking flow); a single blocking loop alone will not create true overlap.
- This hides network latency behind disk write time and can improve throughput for backups with many modified blocks.

#### 6. Cache server read capability checks per connection

- `pgmoneta_server_read_binary_file` re-checks `pg_read_server_files` role membership and `EXECUTE` privilege on `pg_read_binary_file(...)` on every call.
- In the legacy incremental path, this happens repeatedly during block fetches for incremental files and chunk reads for full-copy fallbacks.
- Cache these capabilities for the lifetime of the backup connection so repeated reads do not issue identical privilege queries.
- This is especially relevant once tablespace relation files are routed through the same read path, because the optimization applies uniformly to both default and tablespace-backed relations.

#### Evaluation approach

1. Profile the sequential pipeline to measure where backup time is spent.
2. Implement batch reads, capability-check caching, and stat-skip optimization first because they are the lowest-risk changes with the highest expected payoff.
3. Implement buffer reuse next if profiling shows that allocator overhead is still meaningful.
4. Treat double-buffering and file-level parallelism as optional late-stage prototypes only if the low-risk optimizations are complete and stable.
5. Verify correctness against baseline restore results before and after each change.

#### Latency breadcrumbs (how gains will be measured)

1. End-to-end incremental backup duration for fixed workloads (PG14/15/16).
2. Number of server read calls per backup (`pgmoneta_server_read_binary_file` count).
3. Number of capability/privilege-check queries issued during backup-side file reads.
4. Number of `pgmoneta_server_file_stat` round-trips per backup.
5. Effective data throughput (MiB/s) from cluster to backup directory.
6. Restore correctness parity before and after each optimization pass.
7. A simple "optimization accepted / deferred" decision based on whether the gain is measurable and the code-path complexity remains low.

---

## Implementation Plan

### 1. Foundation and baselining

1. Finalize acceptance criteria for each sub-problem.
2. Map scenario matrix by PostgreSQL version and workload type.
3. Establish reproducible baseline runs.

### 2. Legacy tablespace incremental logic

1. Implement tablespace enumeration and metadata population.
2. Implement the `parse_relation_file` tablespace branch.
3. Update main loop routing for `pg_tblspc` files.
4. Verify BRT lookup correctness for tablespace relations.
5. Validate file transfer for tablespace paths.
6. Validate generated manifests and full-backup-compatible tablespace layout behavior for legacy incremental chains.

### 3. Metadata and restore compatibility

1. Verify tablespace metadata completeness in backup chain artifacts.
2. Verify generated `backup_manifest` path correctness for legacy incrementals.
3. Validate restore/combine behavior for create, drop, and move scenarios using the existing tablespace metadata and restore-time mapping mechanisms.
4. Confirm that the backup-side symlink layout matches what restore/combine expects.
5. Introduce minimal restore workflow changes only if needed.

### 4. Scenario expansion and stabilization

1. Implement real-world scenarios.
2. Implement extreme scenarios.
3. Implement edge scenarios.
4. Implement regression scenarios.
5. Align CI/test infra for PG14/15/16 scenario execution (including PG16 matrix coverage).
6. Stabilize test reproducibility and diagnostics.
7. Draft operator-facing troubleshooting notes from the new test scenarios.

### 5. Optimization and finalization

1. Profile sequential pipeline.
2. Implement low-risk optimizations such as batch block reads, capability-check caching, and metadata/stat reuse.
3. Prototype buffer reuse, file-level parallelism, or overlapped I/O if time allows.
4. Finalize documentation and completion report.

---

## Approximate Schedule (360 Hours)

Assumption: 12 weeks at approximately 30 hours per week.

### Community Bonding Period (~3 weeks before coding)

1. Set up PG14, PG15, and PG16 test environments.
2. Trace the current legacy incremental and restore paths end-to-end.
3. Document current behavior boundaries and edge cases.
4. Align with mentor on acceptance criteria.

### Week 1 (30h): Baseline and scope lock

1. Reproduce and document current behavior boundaries.
2. Finalize acceptance checklist and scenario matrix.
3. Prepare PG16 test/CI additions needed for the project.

### Week 2 (30h): Tablespace enumeration and metadata

1. Implement tablespace OID/path discovery.
2. Implement backup metadata population for legacy incrementals.
3. Implement and document the existing full-backup-compatible on-disk layout for tablespace artifacts.

### Week 3 (30h): Tablespace path parsing

1. Implement the `parse_relation_file` tablespace branch.
2. Handle dynamic `PG_<major>_<catalog_version>` parsing.
3. Validate fork, segment, and temporary-relation filename parsing while touching the parser.
4. Add parser-focused tests and debug traces.

### Week 4 (30h): Main loop routing + BRT integration

1. Update the main loop filter for `pg_tblspc` files.
2. Verify BRT lookup correctness for tablespace relations.
3. Check whether RM_TBLSPC records need explicit summarizer support.
4. End-to-end incremental backup test with tablespace.

### Week 5 (30h): Data transfer and incremental file correctness

1. Validate block fetch/write for tablespace paths.
2. Validate unchanged-file header-only behavior and full-copy fallbacks.
3. Validate generated `backup_manifest` entries and backup-side `pg_tblspc` link layout.
4. Harden fallback semantics for edge cases.

### Midterm Evaluation

Expected state: tablespace-aware incremental backup works end-to-end for PG14-16 in core scenarios, with correct metadata emission, correct legacy backup artifacts, and basic restore validation in place.

### Week 6 (30h): Restore integration and metadata consistency

1. Run end-to-end restore and combine checks for tablespace incrementals.
2. Validate chain integrity across backup types.
3. Verify `backup_manifest` and `pg_tblspc` link correctness.
4. Implement workflow adaptations only if required.

### Week 7 (30h): Real-world test scenarios

1. Implement tablespace-only, creation, drop, and mixed scenarios.
2. Add moved-between-tablespaces scenario.
3. Add assertions for backup metadata and restore correctness.

### Week 8 (30h): Extreme and edge scenarios

1. Timeline change tests, with explicit validation of supported behavior versus fail-fast behavior.
2. Active-write during backup tests.
3. Segment, fork, and truncation edge tests.

### Week 9 (30h): Regression and version coverage

1. Multi-step chain restore validation.
2. PG17+ non-regression testing.
3. Consolidate PG14-16 scenario runs.
4. Finalize PG16 CI/test coverage.

### Week 10 (30h): Coverage stabilization

1. Improve flake resistance and failure diagnostics.
2. Address edge case failures from earlier weeks.
3. Draft administrator and troubleshooting documentation from test outcomes.

### Week 11 (30h): Bonus optimization pass (stretch goal)

1. Profile sequential pipeline.
2. Implement at most the lowest-risk optimizations first: batch reads, capability-check caching, and metadata/stat reuse.
3. Implement buffer reuse, and only prototype file-level parallelism or overlap I/O if time allows and the core project is already stable.
4. Verify correctness against baseline.

### Week 12 (30h): Final polish

1. Incorporate review feedback.
2. Finalize documentation and submission artifacts.
3. Publish final deliverable evidence.

---

## Success Criteria

The project is successful when:

1. PostgreSQL 14-16 legacy incremental backups correctly include tablespace-backed changes.
2. Resulting backup metadata and legacy-generated artifacts remain correct for tablespace-aware incremental chains.
3. Restores from resulting incremental chains are correct and reproducible.
4. Timeline-change and active-write scenarios are explicitly tested with defined expected outcomes.
5. Scenario coverage across PG14/15/16 is stable and maintainable.
6. Any optimization merged preserves correctness and demonstrates measurable benefit without compromising the core 360-hour scope.
