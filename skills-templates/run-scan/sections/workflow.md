### 1. Run the scan

Scan the pre-built model at `.opentaint/project`. Write the report to `.opentaint/results/report.sarif` and load both the built-in ruleset and the project's own rules under `.opentaint/rules`:

```bash
opentaint scan --project-model .opentaint/project \
  -o .opentaint/results/report.sarif \
  --ruleset builtin --ruleset .opentaint/rules \
  --track-external-methods
```

- `--rule-id <full-id>` — restrict to specific rules (repeatable, one per input rule ID); every unnamed rule is dropped, including library `refs`, so list every id the restricted rules depend on. Omit to run all loaded rules
- `--passthrough-approximations .opentaint/pass-through` — add when that directory exists: passThrough configs override built-ins at the rule level, a provided rule overriding a built-in only when it matches one
- `--dataflow-approximations .opentaint/dataflow` — add when that directory exists: code-based approximations (sources auto-compiled; pre-compiled `.class` dirs passed through as-is)

Both approximation-dir flags walk their trees recursively; pass each parent directory once, not every package or batch separately.

The scan is long — run it in the background and wait for it to finish. Leave `--timeout` at the engine default (900s); the CLI ends the analysis itself and writes whatever SARIF it has.

### 2. Retry once on out-of-memory

Start at the 8G default. Only after an out-of-memory failure, retry once with `--max-memory 16G` — never higher, more RAM won't improve results. One bump, no further. When the caller passed `max-memory`, run at it from the first attempt instead

### 3. Collect the report, or escalate

If a SARIF was produced — even alongside a timeout or OOM message — take it as-is and ignore the error, the results are already there. When the scan instead fails at config-load on a malformed approximation (e.g. an unexpected position modifier, a duplicate approximation class), it is not out-of-memory: don't retry at 16G — report the engine error and the offending file under `.opentaint/pass-through`/`.opentaint/dataflow` per Output, locating it from the error message (grep the artifacts for the reported symbol when the error doesn't name the file). Only when no valid SARIF comes out even at 16G is it a plain failure: report it per Output with the setup and don't retry beyond that one 16G attempt
