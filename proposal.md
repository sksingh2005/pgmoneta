# GSoC Proposal: Legacy Incremental Backup with Tablespace Support (PostgreSQL 14-16)

## Title

Complete tablespace-aware incremental backup support for PostgreSQL 14-16 in `pgmoneta`, with robust backup/restore test coverage and optional performance improvements.

## Detailed Description

`pgmoneta` already contains a legacy incremental backup path for PostgreSQL 14-16 and a restore/combine pipeline that handles incremental chains. The remaining gap is complete **tablespace-aware legacy incremental backup** behavior.

The project will complete this capability end-to-end by:

1. correctly identifying changed tablespace blocks from WAL-derived summaries,
2. mapping tablespace relation files to the same relation-identity model used by incremental logic,
3. transferring only modified blocks from cluster to backup storage using existing server APIs and the pgmoneta extension,
4. validating that restore remains correct for these new artifacts,
5. building strong scenario coverage, especially for timeline changes and active-write workloads.

This approach stays aligned with PostgreSQL's own protocols and avoids reinventing change tracking.

Current scope clarification:

1. legacy incremental backup path for PG14-16 exists, but tablespace support is incomplete,
2. incremental restore workflow already works and is expected to need minimal or no changes, though compatibility will be explicitly validated.

---

## List of Deliverables

### D1. Tablespace-aware legacy incremental implementation

Implement complete handling of `pg_tblspc` relation files in the PostgreSQL 14-16 incremental backup path.

### D2. WAL/BRT-driven tablespace block selection

Use existing WAL summary and BRT infrastructure to identify modified blocks for tablespace-backed relations.

### D3. Correct modified-block transfer and file emission

Use existing read/copy mechanisms (`pgmoneta_server_read_binary_file` and pgmoneta extension file discovery) to materialize incremental files for tablespace relations.

### D4. Restore compatibility validation (and minimal adaptation if needed)

Verify that current incremental restore/combine flow consumes newly generated legacy tablespace incrementals correctly; update workflow only if required.

### D5. Comprehensive backup+restore test suite

Add real-world and extreme cases covering timeline changes, active-write workloads, truncation, segment/fork edge behavior, and multi-step chain restore.

### D6. PG14-16 version coverage in test execution

Ensure version-aware scenario runs and stable coverage for PostgreSQL 14, 15, and 16.
This includes explicit test-environment and CI alignment for PG16.

### D7. Optional performance improvements

Propose and, where feasible, prototype optimizations such as parallelism via worker pool, batched reads, and memory reuse.

### D8. Documentation updates

Document behavior, constraints, and scenario coverage for legacy incremental tablespace support.

---

## Approach

Structured around the three priority areas defined by the maintainer.

---

### Priority 1: Tablespace Implementation Sub-problems

The current legacy incremental backup (`incr_backup_execute_14_to_16` in `wf_backup_incremental.c`) handles only `base/` and `global/` relation paths. All tablespace files under `pg_tblspc/` are full-copied (see line 335) instead of incrementally backed up. There are five sub-problems to solve:

#### S1: Discover tablespace OIDs and paths from existing file listing

- Currently, the 14-16 path hardcodes `backup->number_of_tablespaces = 0` and never discovers tablespace information.
- The `get_paths()` function (line 1650) already processes the response from `pgmoneta_ext_get_files(".")`, which returns all server files including `pg_tblspc/` entries. It also already creates the corresponding directories in the backup.
- Tablespace OIDs can be extracted directly from the returned paths (e.g., `pg_tblspc/16385/PG_16_202307071/...` → `spcOid = 16385`). This avoids a separate SQL query like `SELECT FROM pg_tablespace` and saves API calls.
- During the file enumeration, collect unique tablespace OIDs and their version directories to populate `backup->tablespaces[]` and `backup->tablespaces_oids[]`.
- For `backup->tablespaces_paths[]` (the physical symlink target needed by the restore pipeline), `get_paths()` alone does not provide this. Similarly, `backup->tablespaces[]` (tablespace name) derivation — whether OID-based stable naming or resolved real names — is tied to the same metadata resolution decision. The resolution method (e.g., reading symlink targets via the extension, or a minimal metadata query for discovered OIDs) is pending confirmation with the maintainer.

#### S2: Parse tablespace relation file paths into `rel_file_locator`

- `parse_relation_file` currently handles only `base/` and `global/` prefixes. The `else` branch does `goto done;` — returning success but without populating the `rlocator`, so tablespace paths are effectively not processed for incremental decisions.
- PostgreSQL stores tablespace relations under `pg_tblspc/<spcOid>/PG_<major>_<catver>/<dbOid>/<relfilenode>` (as defined in PostgreSQL's `relpath.h`).
- Add a third branch to extract `spcOid`, `dbOid`, and `relfilenode` from this path structure, and create the corresponding backup directory:

```c
else if (!strcmp(results[0], "pg_tblspc"))
{
   rlocator.spcOid = pgmoneta_atoi(results[1]);
   rlocator.dbOid = pgmoneta_atoi(results[3]);
   relation_file = pgmoneta_append(relation_file, results[4]);
   // create pg_tblspc/<oid>/<ver_dir>/<dboid> in backup_data
}
```

- The rest of the relation-file parsing (fork suffix, segment number) remains unchanged since it uses the same `<relfilenode>[_fork][.segno]` format regardless of location.
- **Symlink handling**: entries in `pg_tblspc/` are symlinks pointing to actual tablespace locations (e.g., `pg_tblspc/16385` → `/data/tablespace1`). Following PostgreSQL's own `basebackup.c` behavior (see `sendDir`), symlink semantics must be preserved (copied as symlink metadata or reconstructed during restore). Only the actual relation files underneath the version directory (`pg_tblspc/<oid>/PG_<ver>/<dboid>/<relfilenode>`) go through incremental block-level treatment. Non-relation files and the symlinks themselves are always full-copied.

#### S3: Route tablespace files through the incremental decision path

- The main backup loop at line 335 currently skips anything not starting with `base` or `global` by doing a full copy.
- Add `pg_tblspc` to the filter so tablespace relation files flow through the same incremental decision logic: parse → BRT lookup → incremental or full write.

#### S4: WAL/BRT already tracks tablespace block changes — no WAL reader changes needed

- `pgmoneta_wal_record_summary` in `wal_reader.c` iterates every WAL record's block references and calls `pgmoneta_brt_mark_block_modified(brt, &rlocator, forknum, blocknum)`.
- The `rlocator` extracted from WAL records already includes the correct `spcOid` when the block belongs to a tablespace relation — this is how PostgreSQL encodes block references in WAL.
- Once S2 and S3 are in place (so `parse_relation_file` produces the correct `spcOid` and the main loop routes tablespace files to BRT lookup), the existing `pgmoneta_brt_get_entry` call will find the correct changed blocks with no changes to WAL reading, summarization, or BRT logic.

#### S5: Transfer modified tablespace blocks using pgmoneta extension

- The existing `pgmoneta_server_read_binary_file` call works with any relative path PostgreSQL can resolve, including tablespace paths. No transfer mechanism change needed.
- For file discovery, `create_standard_directories` already calls `pgmoneta_ext_get_files(ssl, socket, ".", &qr)` which lists server files including `pg_tblspc/` entries. If needed, targeted `pgmoneta_ext_get_files` calls per tablespace can supplement this.

#### PostgreSQL protocol compliance

- Uses `pg_backup_start`/`pg_backup_stop` — the correct backup APIs for PG14-16.
- WAL summarization uses previous checkpoint LSN to current start LSN — matching PostgreSQL's incremental semantics.
- Tablespace path parsing follows PostgreSQL's `relpath.h` convention.
- FSM forks are excluded from incremental tracking because PostgreSQL does not fully WAL-log them.
- Block reads use `pg_read_binary_file` — PostgreSQL's standard server-side file access API.
- Symlink semantics in `pg_tblspc/` are preserved; only relation files get incremental block-level treatment — matching `basebackup.c` `sendDir` behavior.
- **Newly created tablespace/database check**: PostgreSQL's `GetFileBackupMethod` checks if a database OID/tablespace OID pairing was created since the previous backup (BRT entry with `relNumber=0`). If so, everything in that pairing must be fully backed up. This should be evaluated and adopted as feasible.
- **Full-backup threshold**: PostgreSQL sends the whole file instead of incremental when 90%+ of blocks need sending (simpler, faster, and the incremental overhead isn't worth it). This optimization should be evaluated and adopted as feasible.
- If timeline continuity cannot be guaranteed safely, fail fast with a clear error and require a new full backup.

---

### Priority 2: Test Coverage — Real-World and Extreme Scenarios

Current test coverage is minimal — only basic chain creation and restore are tested. No tablespace, timeline, active-workload, or edge case coverage exists. The following scenarios will be added:

#### Real-world scenarios

**T1. Tablespace-only modifications**

1. Create a user tablespace and a table within it, take a full backup.
2. Modify only tablespace data, take an incremental backup.
3. Verify: only tablespace blocks appear in incremental, metadata includes tablespace info.
4. Restore and verify data integrity and tablespace symlink reconstruction.

**T2. Tablespace creation between backups**

1. Full backup (no user tablespaces), then create a tablespace and table, insert data.
2. Incremental backup.
3. Verify: new tablespace fully captured, metadata includes the new OID and path.
4. Restore and verify.

**T3. Tablespace drop between backups**

1. Full backup with user tablespace, then move table to default tablespace and drop the user tablespace.
2. Incremental backup.
3. Verify: restore handles absence of dropped tablespace, data accessible in default tablespace.

**T4. Mixed default and tablespace updates**

1. Tables in both default and user tablespace, full backup, update both, incremental backup.
2. Restore and verify data integrity across both locations.

#### Extreme scenarios

**T5. Incremental backup after timeline change**

1. Full backup on timeline 1.
2. Promote a standby (creates timeline 2), insert data on new timeline.
3. Incremental backup.
4. Verify expected behavior explicitly:
   - if supported: correct changed-block capture and correct restore output,
   - if not supported safely: explicit fail-fast with clear diagnostic and no partial/incorrect artifacts.

**T6. Incremental backup during active concurrent updates**

1. Full backup, start `pgbench -T 60 -c 4` workload.
2. While pgbench is running, trigger incremental backup.
3. Restore and verify consistency at backup stop LSN — no corruption or missing blocks.

#### Edge scenarios

**T7. Large relation spanning multiple segments**

1. Create a table exceeding `relseg_size` (multiple `.1`, `.2`, ... segments) in a tablespace.
2. Full backup, modify blocks across different segments, incremental backup.
3. Verify: only modified segments' blocks in the incremental, segment number calculation correct, restore reconstructs the multi-segment relation.

**T8. Fork-specific behavior**

1. Full backup, then `VACUUM` to update the visibility map fork.
2. Verify: `vm` fork incrementally backed up (WAL-logged), `fsm` fork full-copied (not fully WAL-logged), `init` fork handled correctly.

**T9. Relation truncation in tablespace**

1. Full backup with large tablespace table, `TRUNCATE` the table, incremental backup.
2. Verify: BRT `limit_block` set to 0 from SMGR truncation record, full backup triggered for the truncated file, restore produces empty table.

#### Regression scenarios

**T10. Multi-step incremental chain restore**

1. Full → incremental1 → incremental2 → incremental3 (tablespace mods at each step).
2. Restore from latest, verify chain reconstruction is correct.

**T11. No regression in PG17+ flow**

1. Run existing tests against PG17+ to ensure tablespace changes don't affect the modern path.

All tests T1-T10 will be parameterized for PG14, 15, and 16 using the existing MCTF test framework.

---

### Priority 3 (Bonus): Reducing Backup Latency

The current `incr_backup_execute_14_to_16` processes all files sequentially in a single loop. Several areas contribute to backup latency that can be improved:

#### B1: Batch contiguous block reads

- `write_incremental_file` issues one `pgmoneta_server_read_binary_file` call per modified block — one network round-trip per block.
- Since the block array is already sorted, contiguous block ranges can be detected and fetched in a single call by specifying a larger byte range.
- This reduces network round-trips from N blocks to the number of contiguous ranges.

#### B2: Pre-allocate reusable block buffer

- The block array (`incr_blocks`) is allocated via `malloc` and freed for every relation file in the loop.
- Pre-allocating a single buffer sized to `rel_seg_size` before the loop and reusing it across all files eliminates per-file allocation overhead.

#### B3: File-level parallelism via worker pool

- Each file's processing (parse → BRT lookup → block read → write) is independent.
- pgmoneta already provides a worker pool (`workers.h`) used in the restore path (`do_reconstruct_backup_file`, `do_copy_backup_file`).
- A worker function can encapsulate the per-file logic and dispatch files to the pool.
- Constraint: each worker needs its own server connection since block reads go through `pg_read_binary_file`. Connection management adds complexity.
- Parallelism is safe because the BRT is read-only during the backup loop (summarization is complete before it starts).

#### B4: Reorder BRT check before file stat to short-circuit earlier

- The main loop currently calls `pgmoneta_server_file_stat` (network round-trip) for every relation file, then checks BRT.
- Moving the BRT check before the stat call allows earlier short-circuit for unchanged files.
- `pgmoneta_ext_get_files` already returns size metadata; if cached during `create_standard_directories`, unchanged-file paths can use cached size and avoid extra `pgmoneta_server_file_stat` calls.
- If cached size is unavailable or considered stale for a path, fall back to `pgmoneta_server_file_stat` for correctness.

#### B5: Overlap server reads with local writes

- Currently, block reads from the server and writes to the local file happen strictly sequentially — read a block, write it, then read the next.
- A double-buffering approach can overlap network I/O with disk I/O: while one buffer is being written to disk, the next block is fetched by a prefetch stage.
- This requires asynchronous prefetch (reader/writer split or non-blocking flow); a single blocking loop alone will not create true overlap.
- This hides network latency behind disk write time and can improve throughput for backups with many modified blocks.

#### Evaluation approach

1. Profile the sequential pipeline to measure where backup time is spent.
2. Implement batch reads (B1) and stat-skip optimization (B4) first — lowest risk, highest expected improvement.
3. Implement buffer reuse (B2) and double-buffering (B5) next.
4. Prototype file-level parallelism (B3) if time allows.
5. Verify correctness against baseline restore results before/after each change.

#### Latency breadcrumbs (how gains will be measured)

1. End-to-end incremental backup duration for fixed workloads (PG14/15/16).
2. Number of server read calls per backup (`pgmoneta_server_read_binary_file` count).
3. Number of `pgmoneta_server_file_stat` round-trips per backup.
4. Effective data throughput (MiB/s) from cluster to backup directory.
5. Restore correctness parity before/after each optimization pass.

---

## Implementation Plan by Work Package

### WP1: Foundation and baselining

1. Finalize acceptance criteria for each sub-problem.
2. Map scenario matrix by PostgreSQL version and workload type.
3. Establish reproducible baseline runs.

### WP2: Legacy tablespace incremental logic

1. Implement tablespace enumeration and metadata population (S1).
2. Implement `parse_relation_file` tablespace branch (S2).
3. Update main loop routing for `pg_tblspc` files (S3).
4. Verify BRT lookup correctness for tablespace relations (S4).
5. Validate file transfer for tablespace paths (S5).

### WP3: Metadata and restore compatibility

1. Verify tablespace metadata completeness in backup chain artifacts.
2. Validate restore/combine behavior for new artifacts.
3. Introduce minimal restore workflow changes only if needed.

### WP4: Scenario expansion and stabilization

1. Implement real-world tests T1-T4.
2. Implement extreme tests T5-T6.
3. Implement edge tests T7-T9.
4. Implement regression tests T10-T11.
5. Align CI/test infra for PG14/15/16 scenario execution (including PG16 matrix coverage).
6. Stabilize test reproducibility and diagnostics.

### WP5: Optimization and finalization

1. Profile sequential pipeline.
2. Implement batch block reads (B1) and buffer reuse (B2).
3. Prototype file-level parallelism (B3) if time allows.
4. Finalize documentation and completion report.

---

## Approximate Schedule (360 Hours)

Assumption: 12 weeks, approximately 30 hours per week.

### Community Bonding Period (~3 weeks before coding)

1. Set up PG14, PG15, and PG16 test environments.
2. Prototype `parse_relation_file` tablespace branch locally.
3. Document current behavior boundaries and edge cases.
4. Align with mentor on acceptance criteria.

### Week 1 (30h): Baseline and scope lock

1. Reproduce and document current behavior boundaries.
2. Finalize acceptance checklist and scenario matrix.

### Week 2 (30h): Tablespace enumeration and metadata

1. Implement tablespace OID/path discovery (S1).
2. Implement backup metadata population.

### Week 3 (30h): Tablespace path parsing

1. Implement `parse_relation_file` tablespace branch (S2).
2. Add parser-focused tests and debug traces.

### Week 4 (30h): Main loop routing + BRT integration

1. Update main loop filter for `pg_tblspc` files (S3).
2. Verify BRT lookup correctness for tablespace relations (S4).
3. End-to-end incremental backup test with tablespace.

### Week 5 (30h): Data transfer and incremental file correctness

1. Validate block fetch/write for tablespace paths (S5).
2. Harden fallback semantics for edge cases.

### Midterm Evaluation

Expected state: tablespace incremental backup produces correct incremental files for PG14-16, with basic end-to-end restore validation.

### Week 6 (30h): Restore integration and metadata consistency

1. Run end-to-end restore/combine checks for tablespace incrementals.
2. Validate chain integrity across backup types.
3. Implement workflow adaptations only if required.

### Week 7 (30h): Real-world test scenarios (T1-T4)

1. Implement tablespace-only, creation, drop, and mixed scenarios.
2. Add assertions for backup metadata and restore correctness.

### Week 8 (30h): Extreme and edge scenarios (T5-T9)

1. Timeline change tests (T5).
2. Active-write during backup tests (T6).
3. Segment, fork, and truncation edge tests (T7-T9).

### Week 9 (30h): Regression + version coverage (T10-T11)

1. Multi-step chain restore validation.
2. PG17+ non-regression testing.
3. Consolidate PG14-16 scenario runs.

### Week 10 (30h): Coverage stabilization

1. Improve flake resistance and failure diagnostics.
2. Address edge case failures from earlier weeks.

### Week 11 (30h): Bonus optimization pass (stretch goal)

1. Profile sequential pipeline.
2. Implement batch reads (B1) and buffer reuse (B2).
3. Prototype file-level parallelism (B3) if time allows.
4. Verify correctness against baseline.

### Week 12 (30h): Final polish

1. Incorporate review feedback.
2. Finalize docs and submission artifacts.
3. Publish final deliverable evidence.

---

## Success Criteria

The project is successful when:

1. PostgreSQL 14-16 legacy incremental backups correctly include tablespace-backed changes.
2. Restores from resulting incremental chains are correct and reproducible.
3. Timeline-change and active-write scenarios are explicitly tested with defined expected outcomes.
4. Scenario coverage across PG14/15/16 is stable and maintainable.
5. Any optimization merged preserves correctness and demonstrates measurable benefit.
