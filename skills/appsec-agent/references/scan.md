# Scan

Delegate run-scan. Inputs: model-dir `.opentaint/project`, ruleset `builtin` + `.opentaint/rules`, report `.opentaint/results/report.sarif`; plus config-dir `.opentaint/pass-through` and approx-dir `.opentaint/dataflow` whenever those dirs exist — apply every existing approximation on every scan, any level (the level gates whether new approximations are *generated*, not whether existing ones are *applied*; a lite rescan must still see a prior deep run's coverage). Both dir flags walk the tree recursively, so the parents apply every unit. Keep the return concise — finding counts per rule and any config load/parse errors, never the SARIF body; the files persist on disk for the next steps.

Only normal/deep runs need more, because only they iterate approximations: there, pass `--track-external-methods` and have the agent also report the methods still in `dropped-external-methods.yaml` that sit on a source→sink path. A lite run has no approximation work — omit `--track-external-methods` and skip the dropped-method analysis entirely; the SARIF and finding counts are all triage needs. Don't make a lite scan read code-flows to weigh dropped methods.

Cap the scan at `--timeout 600` (10 minutes), no longer. If it still errors or times out, don't retry or block on it — take whatever SARIF was produced and continue; an over-long or failed scan is a coverage gap, not a run-stopper.

Set `phases.scan: done`.

On deep runs, if the scan flags an issue with a created rule — a rule that failed to load/parse, a join that should fire but didn't, or an own rule that false-positives — dispatch create-rule to fix that rule (references/discover-rules.md), then rescan before continuing.
