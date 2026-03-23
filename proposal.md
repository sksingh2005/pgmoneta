# GSoC Proposal: Completing Legacy Incremental Backup with Tablespace Support for PostgreSQL 14-16

## Project Description

Complete legacy incremental backup support for PostgreSQL 14-16 in `pgmoneta`, with correct tablespace handling, restore/chain compatibility, and stronger cross-version test coverage.

## Index

1. Project Idea
2. Proposed Solution
3. Anticipated Problems and Mitigation
4. Test Plan
5. Optional Stretch Work: Performance Improvements
6. Schedule of Deliverables
7. Expected Outcome
8. Why This Is a Good GSoC Project
9. Preparation

## Project Idea

`pgmoneta` supports incremental backup through two paths: PostgreSQL 17+ uses PostgreSQL's native incremental backup protocol, while PostgreSQL 14-16 use `pgmoneta`'s legacy WAL-summary and block-level incremental path.

The remaining gap in the PostgreSQL 14-16 path is support for tablespace-backed relation files. The current legacy workflow routes only relation files under `base/` and `global/` through incremental handling. Tablespace-backed relation files are not yet routed through the same relation-based flow, even though WAL summarization, block reference tracking, and restore/combine already contain most of the required infrastructure.

This project will complete that missing piece. The goal is not to introduce a new backup format, but to finish the existing legacy design so that full backup, incremental backup, combine, and restore behave correctly for PostgreSQL 14-16 clusters that use tablespaces.

## Proposed Solution

The implementation will build on the existing PostgreSQL 14-16 incremental path in `wf_backup_incremental.c`. The design principle is to reuse the current legacy incremental format and restore model rather than introduce a parallel tablespace-specific workflow.

### 1. Tablespace metadata

The legacy path needs explicit tablespace identity. For each tablespace, the implementation should persist OID, name, and original location so that backup metadata remains consistent across incremental chains and stays compatible with the existing restore/combine model.

### 2. Backup-local tablespace layout

The legacy incremental path should preserve the same backup-local layout already used by full backup: a sibling tablespace directory for each tablespace and `data/pg_tblspc/<oid>` symlink entries pointing to those directories. This keeps the output compatible with the current restore/combine path instead of introducing a second layout model.

### 3. Tablespace-aware relation parsing and routing

`parse_relation_file()` should be extended beyond `base/` and `global/` so that logical `pg_tblspc/<oid>/...` relation paths can be parsed into the same relation identity model used by the existing incremental workflow. That parsing must cover tablespace OID, database OID, relfilenode, forks, and segment files. Once parsed, tablespace-backed relations can be routed through the existing block-reference-table lookup and full-vs-incremental decision flow while preserving current FSM and truncation handling.

### 4. Manifest and artifact correctness

For PostgreSQL 14-16, `pgmoneta` generates `backup_manifest` locally, so manifest correctness is part of the implementation rather than a later cleanup task. Locally generated manifests must use PostgreSQL's logical `pg_tblspc/<oid>/...` path model even though `pgmoneta` stores tablespace contents in backup-local sibling directories. Incremental artifact names, manifest entries, and backup-local layout must all remain compatible with chain reconstruction.

### 5. Restore and combine compatibility

The existing restore/combine pipeline already contains tablespace-aware logic. The project therefore should validate that the completed backup-side changes feed the existing restore/combine path correctly, and only introduce restore-side changes where compatibility issues are exposed by end-to-end validation.

### 6. Early technical validation

An early checkpoint in the project is validating whether `pgmoneta_ext_get_files()`, `pg_read_binary_file`, and `pg_stat_file` expose tablespace-backed files in the form required by the legacy workflow. If they do, the implementation can stay within the current architecture. If they do not, the project should define the minimum compatible extension-side changes before proceeding further.

## Anticipated Problems and Mitigation

- Tablespace-backed files may not be exposed by `pgmoneta_ext_get_files()`, `pg_read_binary_file`, or `pg_stat_file` in the exact form needed by the legacy incremental workflow.
  Mitigation: validate this early during community bonding and Milestone 1, and if necessary, define the minimum extension-side adjustments before continuing with the main implementation.

- Locally generated manifests may require explicit path translation so that PostgreSQL's logical `pg_tblspc/<oid>/...` representation remains correct even when `pgmoneta` stores tablespace contents in backup-local sibling directories.
  Mitigation: treat manifest correctness as a separate implementation and validation task instead of assuming it will automatically follow from filesystem layout changes.

- Backup-side changes may expose assumptions in combine and restore that are not visible from static code reading alone.
  Mitigation: validate combine and restore as soon as tablespace-aware artifacts are generated, and keep restore-side changes minimal and compatibility-focused.

- Cross-version differences between PostgreSQL 14, 15, and 16 may reveal edge cases in relation-path handling, truncation behavior, or test setup.
  Mitigation: keep the test matrix active throughout the implementation instead of postponing version-specific validation until the end.

## Test Plan

The test suite should be expanded across PostgreSQL 14, 15, and 16 with scenarios covering:

- full and incremental backup with one user tablespace,
- multi-step incremental chains with tablespace-backed relation changes,
- modifications confined to user tablespaces,
- mixed default-tablespace and user-tablespace changes,
- multi-segment relations in tablespaces,
- fork-specific behavior (`VM`, `FSM`, `INIT`),
- truncation scenarios,
- restore of the newest backup in a chain,
- combine and restore validation for multi-step chains,
- relation movement between tablespaces where supported by the workload,
- active-write workloads during incremental backup.

Validation should go beyond command success. For the main scenarios, it should verify backup metadata, backup-local `pg_tblspc/<oid>` symlink layout, manifest and incremental artifact correctness, and end-to-end restore/combine correctness at both filesystem and SQL levels across PostgreSQL 14, 15, and 16.

## Optional Stretch Work: Performance Improvements

If the core backup/restore correctness work is complete and stable, I would like to evaluate a small set of low-risk latency improvements in the legacy incremental path.

Promising candidates are:

- batching contiguous changed blocks into fewer `pg_read_binary_file()` calls,
- reducing avoidable `pg_stat_file()` round-trips,
- caching per-connection capability checks for file-read operations,
- measuring whether some file-handling work can be parallelized safely without increasing backup inconsistency risk.

These improvements are secondary to correctness and compatibility. They should be evaluated with measurements such as:

- total backup duration,
- number of server read calls,
- number of file-stat calls,
- throughput across different change patterns,
- impact on active-write workloads.

## Schedule of Deliverables

Note: This timeline assumes that the project will use the extended timeline.

### Community Bonding (May 4 - June 3)

- introduce myself to the community and discuss the execution plan, milestones, and acceptance criteria with my mentors,
- dig deeply into `pgmoneta`'s codebase and understand the existing backup, incremental backup, manifest, combine, restore, and tablespace-related code paths,
- experiment with the existing codebase and test setup to build intuition about WAL summarization, relation-path handling, manifest generation, and chain reconstruction,

### Milestone 1 (June 4 - June 24)

- set up PostgreSQL 14, 15, and 16 development and testing environments,
- add PostgreSQL 16 to the automated test setup and align the test matrix across supported versions,
- validate early assumptions about how tablespace-backed files are exposed through `pgmoneta_ext_get_files()`, `pg_read_binary_file`, and `pg_stat_file`,
- document the exact current failure mode and finalize the implementation plan for the first coding phase,
- prepare and add initial test cases for baseline behavior and environment validation.

### Milestone 2 (June 25 - July 15)

- populate legacy incremental tablespace metadata consistently by querying and persisting tablespace OID, name, and original location,
- implement the backup-local tablespace layout for legacy incrementals so it stays aligned with the existing full-backup layout,
- ensure `data/pg_tblspc/<oid>` symlinks are recreated correctly and remain compatible with restore assumptions,
- extend `parse_relation_file()` to recognize logical `pg_tblspc/<oid>/...` relation paths,
- validate tablespace-backed relation identity extraction for relfilenode, forks, and segment files,
- prepare and add new test cases for metadata population, path parsing, and basic tablespace-aware backup behavior.

### Milestone 3 (July 16 - August 5)

- route tablespace-backed relation files through the existing legacy incremental decision flow,
- reuse the current WAL summary and block reference table machinery instead of introducing a parallel workflow,
- verify that changed and unchanged tablespace-backed relation files produce the correct full or incremental artifacts,
- preserve existing handling for FSM, truncation, and segment-boundary cases,
- prepare and add test cases specifically for tablespace-aware incremental artifact generation.

### Milestone 4 (August 6 - August 26)

- validate local `backup_manifest` generation for tablespace-aware legacy incrementals,
- verify manifest path correctness and artifact naming correctness for chain reconstruction,
- verify combine behavior for incremental chains containing tablespace-backed relations,
- validate restore correctness end-to-end and introduce only minimal restore-side fixes if required,
- prepare and add new test cases for incremental chains, restore/combine, truncation, and mixed default-tablespace/user-tablespace scenarios.

### Milestone 5 (August 27 - September 16)

- expand the test suite across PostgreSQL 14, 15, and 16 with edge cases such as multi-segment relations, active-write workloads, and relation movement between tablespaces where supported by the workload,
- strengthen SQL-level restore validation in addition to filesystem validation,
- resolve bugs found during broader testing of the core implementation,
- improve diagnostics, reproducibility, and failure reporting for backup and restore scenarios.

### Milestone 6 (September 17 - October 7)

- finish any remaining implementation work from previous milestones,
- stabilize behavior across PostgreSQL 14, 15, and 16,
- tighten handling around supported scenarios, constraints, and operational caveats,
- improve diagnostics, reproducibility, and failure reporting for backup and restore scenarios,
- continue refining tests based on failures discovered during broader validation.

### Milestone 7 (October 8 - October 21)

- finalize user-facing and administrator-facing documentation,
- document supported behavior, constraints, and operational caveats for legacy tablespace-aware incrementals,
- polish implementation details based on mentor and community feedback,
- prepare the project deliverables in a stable, reviewable state.

### Milestone 8 (October 22 - November 1)

- complete final cleanup and hardening so the project deliverables are stable and reviewable,
- if the correctness work is already complete, evaluate low-risk performance improvements such as batching contiguous reads or reducing avoidable file-stat and capability-check round-trips,
- resolve any final bugs reported during review.

## Expected Outcome

At the end of the project, `pgmoneta` should provide complete and reliable full and incremental backup/restore support for PostgreSQL 14-16 clusters that use tablespaces, using the existing legacy incremental architecture and restore/combine model.
