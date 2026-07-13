# discover-attack-surface — Java / JVM

## Workflow

### 1. Settle built-in coverage

Built-in java source rules live under `java/lib/{generic,spring}/` within the `opentaint health --rules` root — grep there for a member's FQN. The project's own custom rules are under `.opentaint/rules/java`

### 2. Classify the plan's members

- Each plan member is `{ method, signature }` — an `owner.Class#method` ref and its JVM descriptor, copy both into the source tracking unit
- Confirm each package's dependency identity and read its signatures/docs in the resolved jar under `.opentaint/project/dependencies` — `unzip -l <jar> | grep <package-as-path>` (the dotted package with `.` → `/`)

### 3. Write the source units

- `dependencies` is the package's Maven GAV, `group:artifact:version` (e.g. `io.vertx:vertx-core:4.5.26`).
