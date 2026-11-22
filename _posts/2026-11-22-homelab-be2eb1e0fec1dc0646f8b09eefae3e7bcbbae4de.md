---
layout: post
title: "Reproducible Helm chart publication with timestamp normalization"
date: 2026-11-22 08:14 -0800
categories: [Homelab]
tags: [homelab, helm, harbor, ci, gitops, reproducibility]
author: rodman
---

Restored reproducible reviewed Helm publication by normalizing archive timestamps for charts sourced from pinned Git revisions. The change ensures the same Git commit always produces the same package hash, which the offline check validates before any protected dispatch publishes to Harbor.

## Timestamp normalization for Git-sourced charts

When Helm packages a chart from a Git checkout, it embeds filesystem mtimes into the gzip and tar headers. Those timestamps make the package hash non-reproducible even when the source revision is pinned. I added a `normalize_git_package` function that re-writes the archive with mtime set to zero and strips the pax mtime headers, preserving only the chart file contents. The publisher now calls this normalizer for every chart declared with a `revision` field before computing the reviewed hash. The manifest validation logic also shifted to require a reviewed package for every declared Argo CD chart, failing fast if any chart is missing its Harbor package.

## Test coverage for validation and normalization

The test suite now exercises the full validation pipeline: it asserts that a complete manifest passes, that dropping a chart package raises a clear error about the missing Harbor package, and that package identity is stable after normalization. A fixture helper builds test packages with an explicit gzip mtime so the normalizer has something to strip, confirming the hash stays constant across runs. These tests run in the offline check gate via `harbor-chart-publish.py --check` and the new `harbor-chart-publish-test.py` invocation.

## Platform engineering transfer

Any team publishing Helm charts from Git sources to an OCI registry can adopt the same pattern: pin the chart revision in a manifest, normalize archive timestamps at publish time, and gate merges on an offline check that verifies every declared chart has a reviewed package with a matching hash. The approach decouples reproducibility from the build environment and makes the reviewed artifact the single source of truth for GitOps sync.
