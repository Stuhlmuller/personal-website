---
layout: post
title: "One Offline Check Script Instead of Two Overlapping Ones"
date: 2026-11-23 03:58 -0800
categories: [Homelab]
tags: [homelab, ci, platform-engineering, nix, terragrunt, documentation]
author: rodman
---

I folded the offline static checks back into the main offline check script, so the harness that gates a push now has a single entry point instead of two scripts that shared fixtures, generated Terragrunt stacks, and source contracts. The change is a consolidation of test tooling, not a change to what the cluster runs.

## Folding static checks into `offline-checks.sh`

The commit removes `scripts/ci/offline-static-checks.sh` and moves its shell fixtures and source contracts into `offline-checks.sh`, which keeps its own Python test and then runs the fixture groups in a scoped subshell. The groups keep their `::group::` markers, so log output still reads as a series of labeled blocks rather than one undifferentiated run. The Terragrunt work is unchanged in shape: formatting and validation run, the count of explicit stack units is compared against the stack files and against `terragrunt find`, and the empty Kustomize block guard still fails the build. Generated Terragrunt units remain fixtures, with no provider initialization or authenticated cloud and cluster calls, which is what keeps the whole harness honest about being offline.

## Repointing the test and the docs that named the old file

Because the harness is itself inspected by tests and cited by documentation, renaming had a wider blast radius than the script alone. `terragrunt-plan-stage-test.py` read the old helper by name and now reads `offline-checks.sh`; that assertion is about wiring, not about whether the fixtures pass. The wiki pages and the operations notes that declared `repo://scripts/ci/offline-static-checks.sh` as a source now point at the consolidated file, and the Nix cache notes lost the two-owner split in favor of one sentence describing a single script that owns Python tests, shell fixtures, and source contracts. I rewrote that prose deliberately rather than leaving a stale claim that fixtures and contracts live elsewhere.

## What the evidence does and does not establish

What I have here is the merge of the consolidation: the deleted file, the additions inside `offline-checks.sh`, and the updated referents. That is a fact about repository structure. It is not yet evidence that the combined harness passes on a hosted runner, and a single script running more groups sequentially is exactly the case where a stray unbound variable or a fixture writing outside its temp directory would surface. The next check is a protected-main run of the full offline result, confirming every fixture group reports success and that the signed Nix result is published for the merged tree. Live plans and runtime acceptance stay separate gates regardless, so passing this says nothing about the cluster converging.

## Where this transfers to a platform team

Two check scripts that quietly share a generated stack is a common failure mode: each passes alone, the pair drifts, and the split becomes an argument about ownership rather than about what is verified. If a team finds itself writing a second harness to hold the half that did not fit, collapsing it back into one entry point with named groups and asserting the resulting unit counts is a cheap way to make the gate auditable. The part I would copy is the ordering discipline: update the tests and the documents that name the script in the same commit, so nobody reads a source declaration pointing at a file that no longer exists.
