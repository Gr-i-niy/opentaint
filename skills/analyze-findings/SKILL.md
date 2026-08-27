---
name: analyze-findings
description: Triage OpenTaint findings statically. Use when scan findings need a TP/FP verdict
license: Apache-2.0
metadata:
  author: opentaint
  version: "0.3.0"
---

# Skill: Analyze Findings

A finding file bundles all of one rule's results. Read each result's code flow, split the bundle into distinct vulnerabilities, and give each a TP/FP verdict on its own evidence

## Inputs

Provided by the caller, fall back to the default value when omitted. Ask back only when a required input is missing and has no sensible default

- `project-root` (optional) — root of the target project. Opentaint keeps all analysis artifacts under the fixed `<project-root>/.opentaint/` directory, so every `.opentaint/...` path below resolves there. Default: current directory
- `language` (required) — target language for this project and language-specific instructions
- `findings` (required) — the finding file(s) to triage, each `.opentaint/tracking/findings/<name>.yaml` bundling one rule's SARIF results in `sarif_hashes`

## Workflow

### 1. Reconcile before judging

A finding whose `notes` open with a `reconcile` line is a rescan result under a rule whose other findings are already triaged — most often the same vulnerability with a shifted hash, not a new one. Before judging it fresh, read the rule's already-triaged finding files and compare flows (source → sink, same essential path): if one matches, move this finding's `sarif_hashes` into that finding, drop this file, and let the inherited verdict stand — don't re-judge a flow already triaged. Only when no triaged finding matches do you treat it as new and continue below.

### 2. One result at a time — STOP checklist

For each hash in the bundle, before any verdict, read its raw result from `.opentaint/results/report.sarif`:

- find its SARIF result via `sarif_hashes` — each entry is the leading 16 chars of that result's `vulnerabilitySourceSinkHash`/`vulnerabilityWithTraceHash` fingerprint, so match it against the result's `fingerprints`/`partialFingerprints` — then read the raw `codeFlows[]`
- walk every step, source → hops → sink, confirming it's the same tainted value end to end; confirm the flow against the application source (the built project's own sources under `.opentaint/project/sources/`) and dependency code, not the trace text alone
- judge each result on its own trace — no verdict shared across results just because they share the rule

### 3. Split the bundle into logical findings

The results in the file all fired one rule, but may be several different vulnerabilities. Keep results that are the same vulnerability (same sink, same essential flow) together as one finding; move genuinely distinct ones into their own finding file with a new name and their `sarif_hashes` (per Tracking).

### 4. Classify and record

A vulnerability is more than a reachable source-to-sink flow. It exists when an attacker-controlled value or action follows a feasible path across an intended trust or privilege boundary, reaches the sink's exact dangerous interpretation or protected operation, and violates a concrete security property without an effective control.

Check each logical finding in this order:

- actor and source provenance — the attacker can control the relevant value or action under realistic preconditions
- reachability — the current code supports the path and its transformations preserve the relevant attacker influence
- boundary and impact — identify the intended trust boundary, the exact dangerous sink behavior, and the concrete consequence
- defenses — inspect the closest validation, authorization, parameterization, encoding, escaping, or sanitization control; confirm that it covers this value and sink context, runs before the dangerous interpretation, and is not undone by later decoding or reparsing

Assign TP only when those elements form a supported boundary violation. Assign FP only with concrete counterevidence: no attacker control, an infeasible path, a sink that is not dangerous in this exact context, or an effective control. Do not treat a sanitizer name or analyzer recognition as proof, and do not invent an unobserved mitigation.

Set `verdict` and append a `triage:` rationale to `notes` that names the decisive current evidence, below the analyzer report already seeded there (per Tracking).

## Output

### Artifacts

- `.opentaint/tracking/findings/<name>.yaml` — each triaged finding with `verdict` set and the rationale appended to `notes`; a split also writes new finding file(s) (per Tracking)

### Summary

- one line per finding: name, verdict, one-clause reason

## Tracking

This skill writes only each finding's `verdict` and the reasoning appended to `notes`. A split additionally creates a new finding file — a fresh docker-like name, the moved `sarif_hashes`, and `rule_id` copied from the bundle, carrying the seeded analyzer report into its `notes` and leaving `poc` pending. Never touch the `poc` field, or the `sarif_hashes` of a finding you keep.

`.opentaint/tracking/findings/<name>.yaml` — one finding, bundling one rule's SARIF results and carrying it through triage and PoC. A script seeds each file from the scan — its `sarif_hashes`, `rule_id`, and the analyzer report in `notes`. Triage sets `verdict` and appends its reasoning. The PoC stage sets `poc` and appends its outcome. Keep it clear from comments

```yaml
sarif_hashes: [a1b2c3d4, e5f6a7b8]
rule_id: java/security/sqli.yaml:sqli
verdict: TP
notes: >
  <analyzer report for these results — seeded from the scan>
  triage: @RequestParam orderBy is attacker-controlled; reaches ${} in SelectProvider unsanitized → TP
  poc: logged in as a seeded user, then GET /api/orders?orderBy=id);SELECT pg_sleep(5)-- delayed ~5s → confirmed
poc: confirmed
```

## Constraints

- Verdicts and notes go in the finding files only — never write `.opentaint/vulnerabilities.md`; the orchestrator assembles it from the verdicts
- Judge each result on its own trace, never share one verdict across results just because they fired the same rule

## Gotchas

- Bulk verdicts are the most common triage error — many results marked under one shared rationale with the traces unread
- A rule's bundle is not one finding — split distinct vulnerabilities apart, but keep true duplicates (same sink and flow) together as one finding with multiple `sarif_hashes`
