---
layout: post
title: "Staging the n8n Restore Image and Labeling the Failing Phase"
date: 2026-11-23 01:49 -0800
categories: [Homelab]
tags: [homelab, ci, n8n, postgres, observability]
author: rodman
---

This main push tightens the paired checkpoint script for n8n in two ways that both target diagnosability: the pinned PostgreSQL image is now pulled up front, before capture resources are allocated, and every stage of the run records which phase it was in so a failure can be attributed to a named step rather than to a stack trace. The workflow picks that label up and prints it as a single sanitized line.

## Pre-pulling the pinned restore image

The script derives a fully-qualified image reference from the same digest it already pins, then pulls it while the phase is still `restore-image`, ahead of any volume or container creation. Later, when it starts the throwaway database for verification, it passes `--pull never` against that same reference. That ordering means the only network-dependent step happens early and explicitly, and the offline restore itself cannot silently reach out to a registry. The storage and state note now describes this staging as source preparation rather than a completed backup, which matches what the script actually guarantees.

## Reporting a failure phase without leaking output

A `phase` variable advances through the run: `source-validation`, `restore-image`, `capture`, `reader-cleanup`, `archive-validation`, `archive-publication`, `pvc-readback`, `database-start`, `database-restore`, `fence-recheck`, `manifest-publication`, and `isolation-cleanup`. On a non-zero exit the cleanup path writes the current value to a file in the runner temp directory, and a dedicated workflow step reads it back. The step allowlists the known labels and collapses anything else to `unknown`, so the log line stays a fixed string and no command output is echoed into a public job log.

## Tests changed, verification still pending

The offline test double now asserts the pull arguments match the pinned digest and that volume creation asks for `--pull never`. Two new modes, a pull failure and an offline restore failure, were added, and the first asserts that no receipt is emitted and that the recorded phase is `restore-image`. These are changes to the test harness, not evidence of a green run against a real cluster: I have not seen a live paired checkpoint execute end to end here. The check I want next is a real run that reaches `pvc-readback` and publishes the manifest, plus one deliberate pull failure to confirm the label surfaces correctly in the job.

## Where a platform team would reuse this

The pattern is worth copying whenever a long automation script spans several risky steps and only the tail of the log survives triage: keep a single phase variable, persist it on failure to a scratch path outside the build tree, and let CI translate it into a bounded label. Pairing that with an explicit image pre-pull and a no-pull guarantee for the offline portion turns an ambiguous failure into a specific one, which is exactly what you want when the same script runs unattended against production-adjacent storage.
