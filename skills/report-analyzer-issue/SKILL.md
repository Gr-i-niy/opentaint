---
name: report-analyzer-issue
description: Write an OpenTaint engine-issue report from a confirmed diagnosis or a resource failure, optionally opening a GitHub issue. Use when engine-side issue got confirmed and requires report
license: Apache-2.0
metadata:
  author: opentaint
  version: "0.2"
---

# Skill: Report Analyzer Issue

Turn a confirmed engine-level problem into a self-contained `.opentaint/issues/<slug>.md` report, and optionally a GitHub issue. It runs no analysis of its own — it only writes the report from what the caller supplies. Two issue kinds:

- a taint-propagation issue — a confirmed debug-rule diagnosis (the default below)
- a resource-failure issue — a scan that ran out of memory even at `--max-memory 16G` (no valid SARIF). This needs no taint diagnosis; it documents the setup that triggered it so the engine team can reproduce the memory problem

## Inputs

From the caller; if omitted, fall back to the default. Ask only when a required input is missing and has no sensible default

- Diagnosis `<diagnosis>` — taint-propagation kind: debug-rule's engine-level conclusion: where taint dies (`file:line` + instruction), the fact-reachability trace up to the last reachable fact, and observed vs expected verdict
- Failing setup `<setup>` — resource-failure kind: what was running when the scan ran out of memory — the ruleset(s), the approximation dirs, the project model, the `--max-memory` reached (up to `16G`), and the commit hash (`git rev-parse HEAD`)
- Test project `<test-project>` / `<test-compiled>` — taint-propagation kind: the project the artifact was tested on and debug-rule traced, already built by create-test-project. Default: `.opentaint/test-projects/<name>` / `.opentaint/test-compiled/<name>`
- Artifact `<artifact>` — the rule or approximation the issue concerns: a rule's full id and ruleset, or the approximation's target method(s)
- Issue file `<issue-file>` — where to write the report. Default: `.opentaint/issues/<slug>.md`; `<slug>` is a short kebab-case symptom name (a filename — no spaces or hashes)
- Open a GitHub issue `<open-issue>` (optional) — whether to also file at github.com/seqra/opentaint; the main agent decides and passes this. Default: no

## Workflow

### 1. Gate

For a taint-propagation issue, file only for a diagnosis debug-rule already confirmed. The diagnosis must establish all three; if any is missing, return to the caller and ask for debugging first — don't verify or run anything yourself:

- not a rule fix — the rule's patterns are correct; debug-rule ruled out tightening or broadening it
- not a missing model — no method on the source→sink path remains in `dropped-external-methods.yaml`
- it is the engine — taint is dropped at an instruction the engine should propagate through

For a resource-failure issue, the gate is simpler: the caller confirms the scan ran out of memory and produced no valid SARIF even at `--max-memory 16G`. No diagnosis is required — write the report from `<setup>`.

### 2. Write the report

Write `<issue-file>` — this file is the deliverable; never return the report as chat text only.

For a taint-propagation issue, assemble from the inputs:

- Test project — `<test-project>` path, the test command (`test rule run` / `test approximation run`), and the failing `test-result.json` snippet (e.g. a positive sample stuck at `falseNegative`)
- Rule / approximation — the `<artifact>`: a rule's full id and ruleset, or the approximation's target method(s)
- Observed vs expected — e.g. expected a finding at `Sink.java:42`; observed none
- Where the dataflow dies — `file:line` and the instruction, quoted up to the last reachable fact
- Ruled-out causes — the three gate points
- Hypothesis — 1–3 sentences on what the engine is likely doing wrong there; a hypothesis, not a fix

For a resource-failure issue, assemble from `<setup>`:

- The setup — ruleset(s), approximation dirs, and the project model in play (with rough size: classes/modules if known)
- The memory reached — `--max-memory` of the failed attempts (up to `16G`)
- Commit hash — the `git rev-parse HEAD` the model was built from
- Observed — the scan ran out of memory and produced no valid SARIF

Keep it to about one screen (plus the test project, for the taint-propagation kind)

### 3. File on GitHub (only if asked)

When `<open-issue>` is set, file the same content to the fixed repo:

```bash
gh issue create --repo seqra/opentaint \
  --title "<slug>: <one-line symptom>" \
  --body-file <issue-file>
```

## Output

- The written `<issue-file>` (always), and the issue URL if one was filed
