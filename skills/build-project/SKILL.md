---
name: build-project
description: Build a Java/Kotlin project for opentaint analysis and produce a project.yaml model. Use whenever an opentaint scan needs a project model and `opentaint compile` may need help
license: Apache-2.0
metadata:
  author: opentaint
  version: "0.2"
---

# Skill: Build Project

Build a target project into an opentaint project model. The model is this skill's only output

## Inputs

From the caller; if omitted, fall back to the default. Ask only when a required input is missing and has no sensible default

- Project root `<project-root>` — the project to build. Default: current directory
- Model output directory `<model-out>` — where to write the model. Default: `.opentaint/project`
- Build constraints (optional) — required Java version, submodules to initialize, `--package` filters for `opentaint project`

## Workflow

### 1. Determine project type

- `build.gradle` / `build.gradle.kts` → Gradle
- `pom.xml` → Maven
- pre-compiled JAR/WAR → classpath mode
- existing `project.yaml` → reuse it only when the sources are unchanged since it was built (the model is older than every source file, or the tree is clean at the model's commit); if the code moved on, the model is stale — delete `<model-out>` first, then rebuild, so leftover files from the old model can't bleed into the new one

### 2a. Gradle/Maven — autobuilder

```bash
opentaint compile <project-root> -o <model-out>
```

Strongly prefer this path — it runs the project's real build, so it captures the actual module reactor and resolved dependencies. The manual fallback (2b) yields a weaker model (no dependency resolution, hand-guessed package scope that silently drops code), so exhaust 2a first.

A failure here is almost always a fixable build problem, not grounds to switch to 2b — and the autobuilder's wrapper message is terse, so don't judge fixability from it. Reproduce the project's own build directly (`./gradlew build -x test` or `mvn package -DskipTests`) to surface the real error, fix it, then re-run `opentaint compile`. Try this before concluding the autobuilder can't build the project.

### 2b. Manual build + `opentaint project` — last resort

Only when the project's own build cannot be made to pass at all (so the autobuilder can't either). Build manually, then create the model from the artifacts. Always pass `--package` to restrict analysis to project code — without it the analyzer walks third-party libraries and hangs. Take the roots from the packages the classes actually declare, not the Gradle `group` or the source-folder layout — forked or vendored code often declares packages that differ from its directory. Pass one `--package` per declared root and cover all of them; an omitted root is left out of the model

```bash
./gradlew build -x test     # Gradle
mvn package -DskipTests     # Maven

opentaint project \
  --output <model-out> \
  --source-root <project-root> \
  --classpath <app.jar> \
  --package <root.one> --package <root.two>
```

Multi-module: repeat `--classpath` and `--package` per module

### 3. Verify

`<model-out>/project.yaml` exists, is non-empty, and its `packages:` list includes every root the project's classes declare

## Output

The project model directory containing `project.yaml` (default `.opentaint/project`, or the caller's path). Report that path back

## Gotchas

- Analysis hangs → `--package` was omitted in `opentaint project`; the analyzer is processing third-party libraries. Re-run with `--package`
- Build tool not found → use the wrapper (`./gradlew`, `./mvnw`) or install the tool
- Compilation errors → check the autobuilder log, fix the build, retry; if it can't be fixed, fall back to 2b
- Java version mismatch → set `JAVA_HOME` to the version the project needs (opentaint itself needs Java 21+)
- Missing dependencies → initialize submodules (`git submodule update --init`)
