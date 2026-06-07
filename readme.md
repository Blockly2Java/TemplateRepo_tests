[![licensebadge](https://img.shields.io/badge/license-MIT-blue)](https://choosealicense.com/licenses/mit/)


### Test Repository Instructions



#### Architecture Overview
Tests live in a **separate repository** (`*_tests`) and are consumed as **Maven JAR dependencies** by the `parent/` coordinator repo. At build time, `parent/build.gradle` downloads the test JAR from Maven Central, extracts `TestManager.class`, and uses it for JUnit dynamic test discovery. This "remote testing" pattern lets you maintain one test suite per exercise while reusing it across template, solution, and local builds.

#### Test Framework
- **JUnit 5** (Jupiter) + **AssertJ** for assertions
- Custom **levenshtein-testing-framework** for structural/name deviation checks (`@LevenshteinTest`, `@TestFactory`)
- **ByteBuddy** for dynamic class instantiation

#### Project Structure
```
tests/
├── pom.xml                          # Maven build (groupId, artifactId, version — customize per exercise)
├── test/b2j/
│   ├── test/
│   │   ├── TestManager.java         # Main test class (@Test, @TestFactory, @BeforeAll)
│   │   ├── Tests.java               # Test implementations (static test* methods with assertions)
│   │   ├── Messages.java            # Bilingual feedback messages (DE/EN)
│   │   └── TestSettings.java        # Configuration: BASE_PACKAGE, deviation thresholds, language
│   └── wrappers/
│       ├── MainWrapper.java         # Wrapper per student class (extends ClassWrapper)
│       └── ...
```

#### How to Write Tests
1. **Wrappers** (`test/b2j/wrappers/`): Declare expected class, method, attribute signatures for each student class. Extend `ClassWrapper<T>`.
2. **Test methods** (`test/b2j/test/Tests.java`): Static methods named `test*()` that use wrappers to invoke student code and AssertJ assertions.
3. **Test class** (`test/b2j/test/TestManager.java`): Reference wrappers + test methods via `@Test`, `@TestFactory` (for dynamic/structural tests), or `@LevenshteinTest`.

#### How to Run Tests
| Command                                             | Description                                             |
| --------------------------------------------------- | ------------------------------------------------------- |
| `cd parent && gradle testSolution --no-daemon`      | Run remote tests against `solution/`                    |
| `cd parent && gradle testTemplate --no-daemon`      | Run remote tests against `template/` (failures allowed) |
| `cd parent && gradle testSolutionLocal --no-daemon` | Run local test sources against `solution/`              |
| `cd tests && mvn test -P solution`                  | Run Maven tests directly (requires `../solution/src`)   |

Maven profiles: `-P template` or `-P solution` point source dirs to the respective sibling repos.

#### Project Structure Requirement
The **package structure of test classes must match** the package structure of the template/solution repos. Otherwise Maven won't resolve imported classes during the test run.

#### Adjustments After Forking

When forking this repo for a new exercise (e.g., `{exercise}_tests`), update the following:

1. **`pom.xml`** — Change:
   - `artifactId` (e.g., `template_tests` → `myexercise_tests`)
   - `<name>`, `<description>`, and `<url>` in both root and `<scm>`
2. **Wrappers** (`test/b2j/wrappers/`) — Rename/recreate to match your student classes. Each extends `ClassWrapper<T>` and declares expected class name, package, and modifiers.
3. **Test methods** (`test/b2j/test/Tests.java`) — Replace with your test logic referencing your wrappers.
4. **Test class** (`test/b2j/test/TestManager.java`) — Update imports, reference your test methods via `@Test`/`@TestFactory`, adjust `@LevenshteinTest` structural tests.
5. **`TestSettings.java`** — Update `BASE_PACKAGE` if your student code uses a non-default package. Adjust thresholds/language as needed.
6. **`Messages.java`** — Optional: customize feedback messages.
7. **GitHub Secrets** — On the **new fork's** repo settings, configure: `OSSRH_USER`, `OSSRH_TOKEN`, `GPG_PRIVATE_KEY`, `GPG_PASSPHRASE`.
8. **Parent repo CI** — Ensure the parent coordinator repo can find your `{exercise}_tests` repo (naming convention: `{org}/{exercise}_tests`).

> **Recommendation**: Fork this repo for new exercises to make features and fixes available across all exercises. Copying test files is possible but requires separate maintenance per exercise.

---

### Deployment to Maven Central

#### GitHub Workflow: `tests/.github/workflows/publish-mvn-central.yml`
Triggers on **release published**, **push to `main`**, or **manual dispatch** (`workflow_dispatch`).

**Steps:**
1. Checkout code, set up JDK 25 (Temurin)
2. Determine version: tag → release version; otherwise `1.0.0-SNAPSHOT`
3. Update `pom.xml` version via `mvn versions:set`
4. Deploy to Sonatype Nexus (`mvn clean deploy -DskipTests`), which auto-releases to Maven Central

#### Required GitHub Secrets
Configure these on the **tests repo** settings:

| Secret            | Purpose                                               |
| ----------------- | ----------------------------------------------------- |
| `OSSRH_USER`      | Sonatype OSSRH username                               |
| `OSSRH_TOKEN`     | Sonatype OSSRH access token                           |
| `GPG_PRIVATE_KEY` | ASCII-armored GPG private key (for signing artifacts) |
| `GPG_PASSPHRASE`  | Passphrase for the GPG key                            |

#### Required GitHub Variables (for parent repo workflows)
The parent coordinator repo (`{exercise}`) derives sub-repo names automatically — **no extra variables needed**:
- Tests repo: `{org}/{exercise}_tests`
- Solution repo: `{org}/{exercise}_solution`
- Template repo: `{org}/{exercise}_template`

#### Parent Repo CI Workflows
Located in `parent/.github/workflows/`:
- **`test-solution.yml`**: Checks out `tests` + `solution` repos, runs `gradle testSolution`
- **`test-template.yml`**: Checks out `tests` + `template` repos, runs `gradle testTemplate`

Both use the `ghcr.io/valentinherrmann/artemis-maven-docker:latest` container and post test results to the GitHub step summary.