---
layout: post
title: "Add schedule-check test and updated offsite backup handling for etcd"
date: 2026-11-22 18:01 -0800
categories: [Homelab]
tags: [homelab, etcd, backup, schedule, ci, documentation]
author: rodman
---

I introduced a schedule-check test and revised the etcd offsite backup scripts to validate source integrity and destination alignment, ensuring that migrations are safe and verifiable.

## Add schedule-check test for etcd offsite backups

The new `scripts/ci/etcd-offsite-schedule-check.py` imports `hashlib`, installs the schedule, and verifies that the active revision's source checksum matches the stored value and that the legacy copy aligns with the current destination. This test confirms that the schedule can be applied without corrupting existing backup data, providing an early safety net for any offsite migration.

## Update etcd offsite schedule and backup scripts

Changes in `scripts/etcd-offsite-schedule.py` add a `LEGACY_DESTINATION` constant and logic that selects the correct destination based on source checksums, allowing the installer to validate the active release before rollback. `scripts/etcd-offsite-backup.py` now references the IaC configuration file, and `scripts/platform-maintenance.py` updates the terragrunt render command to include the backup storage path. The platform‑maintenance stage script also adjusts bucket extraction to reflect the new configuration.

## Document the migration and verification process

Documentation in `docs/application-recovery.md`, `docs/etcd-offsite-schedule.md`, `docs/etcd-offsite-storage.md`, and `openwiki/architecture/storage-and-state.md` now explains that the shared S3 destination is defined in IaC, that installer validation occurs at the backup path, and that both application and etcd backup destinations reside alongside their respective workload stacks. Policy files `policy/workflows.rego` and `policy/workflows_test.rego` were updated to reflect the new verification steps.

## Pattern for safe offsite backup migrations

The team can reuse this technique for any service requiring offsite backup migration: add a schedule-check test that validates source integrity and destination consistency, modify backup and schedule scripts to handle legacy and current paths, document the flow in recovery docs, and let CI enforce correctness. This pattern ensures that migrations are repeatable, auditable, and resilient to configuration drift.
