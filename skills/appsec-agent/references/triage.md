# Triage

The scan must be stable first.

A zero-finding scan is not automatically a clean project. On a normal/deep run, when rules load and run cleanly but nothing fires, suspect a broken flow before accepting the result — taint that dies before reaching any sink yields zero findings just as a genuinely clean project does. The usual causes are an over-eager `skipped.yaml` entry or a method still dropped on a source→sink path. Check both, then trace where taint dies with debug-rule (references/escalation.md). Conclude the project is clean only once the flows are confirmed intact, or the cause is found and genuinely cannot be fixed.

## Generate finding files

Run this skill's bundled `scripts/sarif-to-findings.py` over `.opentaint/results/report.sarif` (`python3 <this skill's directory>/scripts/sarif-to-findings.py .opentaint/results/report.sarif -o .opentaint/tracking/findings` — the script lives in the skill directory, not the project; the project-relative paths are arguments). It writes one `tracking/findings/<finding_name>.yaml` per rule and is idempotent — a rescan adds new result hashes to an untriaged finding and resets it to `pending`, while leaving triaged verdicts intact. This is a deterministic script with no context cost, so run it yourself, not via a subagent.

When every finding for a rule is already triaged but the rescan brings new hashes, the script can't merge them without clobbering a verdict, so it writes them as a fresh `pending` finding whose `notes` open with a `reconcile:` line (its summary reports a `to reconcile` count). These are the hash-shift cases — the same vulnerability with a nudged hash, or a genuinely new one. Hand each such finding to its analyze-findings subagent to reconcile against the rule's triaged findings by flow: same vulnerability → it merges the hashes into that finding and inherits the verdict; genuinely new → it triages it normally.

## Classify — never in main

Fan out analyze-findings, one subagent per finding file (the rule bundle is the bucket). Inputs: `<findings>` = the finding file, report `.opentaint/results/report.sarif`. The agent reads each result's `codeFlows[]`, splits the bundle into distinct logical findings, and sets `verdict` + `notes` on each. Return: one line per logical finding (name, verdict, one-clause reason). Assign no verdicts yourself.

## Assemble the report

Refresh `.opentaint/vulnerabilities.md` yourself (never a subagent), so every run — static included — ends with a report on the current verdicts, not only dynamic ones. Overwrite it from the `verdict: TP` findings: one entry each — `finding_name`, rule / vuln class, the source→sink location, and the one-clause rationale from `notes`. Re-running drops findings now resolved and updates changed ones; if no TP remains, say so rather than leaving a stale file. On a static run each entry stands as an analysis-only (un-reproduced) result; on a dynamic run the PoC phase refreshes this again with reproduction evidence (references/poc.md). Set `phases.triage: done` once it's written.
