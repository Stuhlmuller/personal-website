---
layout: post
title: "Adding Homepage Image Publication Scope to Harbor Mirror Pipeline"
date: 2026-11-23 00:03 -0800
categories: [Homelab]
tags: [homelab, harbor, ci, image-mirroring, homepage]
author: rodman
---

The commit extends the Harbor mirror CI pipeline to support publishing the Homepage v2.8.0 container image, adding a new `homepage` scope alongside the existing mirrors for individual apps. This enables a single automated workflow to push the Homepage image to the internal registry, reducing manual steps and keeping the mirror process consistent with the project's supply‑chain policy.

## Change summary

The push touches ten files. The core of the change is adding `homepage` to every list of valid `image_scope` values: the GitHub Actions workflow `harbor-mirror.yml`, the documentation `harbor-image-mirroring.md`, the operations wiki `harbor-oci.md`, the OPA policy `workflows.rego`, the test file `workflows_test.rego`, the Python test `harbor-publish-test.py`, the shell wrapper `harbor-publish.sh`, the static‑checks script `offline-static-checks.sh`, and a new config file `harbor-homepage-images.json` that holds the image entry. Each modification wires the new scope through case statements, policy rules, and test assertions.

## Implementation detail

The workflow now accepts `image_scope=homepage` as a workflow dispatch input; the case statement in the shell script includes `homepage` alongside all other scopes, and the associated mirror command `mirror-homepage --tailscale` is invoked when the scope matches. The OPA policy `approved_tailnet_harbor_mirror` has been updated to allow the homepage image through the same tailnet‑authenticated path. The Python test adds an assertion that the generated workflow contains `homepage` in the allowed‑scope list and, when a manifest is present, writes it to `harbor-homepage-images.json`. The static‑checks script was updated to recognise the new scope in its validation loop.

## Verification approach

After merging, run the workflow with the new scope using the CLI: `gh workflow run harbor-mirror.yml --ref main -f image_scope=homepage`. Confirm that the pipeline completes successfully and that the Homepage v2.8.0 digest appears in the internal registry. The updated test `harbor-publish-test.py` should pass, asserting that `homepage` is present in the allowed scopes and that the manifest file is correctly populated. If any step fails, inspect the workflow run logs and the OPA decision logs to verify that the policy rule for `homepage` is evaluated as expected.

## Platform engineering note

Teams operating a Kubernetes GitOps homelab can apply the same pattern when extending a mirror or publish pipeline: add the new scope to every case list, update the policy rego to approve the image, add a corresponding test assertion, and document the scope in the operations wiki. This keeps the CI surface area small while giving operators a clear, audited path to promote new container images through the existing automated channel.
