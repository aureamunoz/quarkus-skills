# Module: Check JDK Version

Verify that the installed JDK meets the minimum version required by the target Quarkus version.

## Preconditions

This phase has no preconditions -- it must **always** run as the very first step.

## Instructions

- **DO NOT** skip this phase.

### Pass 1: Absolute minimum check

- [ ] Run `java -version` (or `java --version`) and capture the installed major version.
- [ ] If `java` is not found or the major version is **< 17**:
  - **Warn the user**: "JDK 17 or later is required for any Quarkus 3.x+ migration. Currently installed: <detected version or 'none'>. Please install JDK 17+ and ensure it is on your PATH before retrying."
  - **Stop the migration** -- do not proceed to any subsequent phase.

### Pass 2: Resolve the minimum JDK for the target Quarkus version

1. **Determine the target Quarkus version.** Use the first source that provides a value:
   - `.quarkus-migration.yml` in the source project root (`quarkus_version` field)
   - Skill invocation argument
   - If neither is available, resolve from the Quarkus streams API (step 2)

2. **Query the Quarkus streams API** to get version metadata:
   ```
   curl -s https://code.quarkus.io/api/streams
   ```
   The response is a JSON array. Each entry contains:
   - `quarkusCoreVersion` -- the Quarkus version (e.g. `"3.39.5"`)
   - `javaCompatibility.versions` -- sorted list of supported JDK versions (first element is the minimum)
   - `javaCompatibility.recommended` -- the recommended JDK version
   - `recommended` -- `true` for the current recommended stream
   - `lts` -- `true` for LTS streams

   **If no target version was resolved in step 1**: use the stream where `recommended: true`. Extract its `quarkusCoreVersion` as the target Quarkus version.

   **Find the matching stream**: match the target Quarkus version's major.minor against the stream keys (e.g. target `3.39.x` matches `io.quarkus.platform:3.39`). Extract the **minimum JDK** from `javaCompatibility.versions[0]`.

3. **If the API call fails** (network error, timeout, non-200 response), use this static fallback table:

   | Quarkus major | Minimum JDK |
   |---------------|-------------|
   | 3.x           | 17          |
   | 4.x           | 21          |

4. **Check the installed JDK against the resolved minimum:**
   - [ ] If the installed JDK version is **>= minimum**, mark this phase as passed and proceed.
   - [ ] If the installed JDK version is **< minimum**:
     - **Warn the user**: "Quarkus <target-version> requires JDK <minimum> or later. Currently installed: <detected version>. Please upgrade your JDK before retrying."
     - **Stop the migration** -- do not proceed to any subsequent phase.

5. Log the resolved values:
   ```
   JDK check: installed=<detected>, target Quarkus=<version>, minimum JDK=<minimum>, source=<api|fallback>
   ```