# Develocity Artifact Cache GitHub Action

Cache Gradle, Maven, npm, pip, and Sonar dependencies and build outputs in GitHub
Actions with the [Develocity Artifact Cache](https://docs.develocity.ai/artifact-cache/),
in a single workflow step. The action restores the cache before your build and stores
it afterward automatically, and never fails the build.

## Why use this action?

If you cache Develocity Artifact Cache content in GitHub Actions today, you likely
maintain a hand-rolled setup: a step that downloads the CLI JAR, verifies it, wires up
a JDK, and runs `restore`, plus a second step that runs `store`. This action replaces
all of that with a single `uses:` step, and adds behavior that is tedious to reproduce
by hand:

- **One step, both phases.** The action restores in its main step and stores in an
  automatic post step. You do not write or order a separate store step, and cache
  activity never fails the build.
- **Every cache type, autodetected.** It caches Gradle, Maven, npm, pip, and Sonar
  content automatically, with no per-type configuration.
- **Cache names that work out of the box.** Names are generated per job and isolated by
  runner OS and architecture, with automatic fallback to a pull request's base branch
  and to the repository default branch, so new branches restore instead of starting
  cold.
- **A verified, tested CLI.** The action downloads a checksum-verified CLI JAR and
  defaults to the version pinned and tested for this release, so you do not track CLI
  versions or download URLs yourself.
- **Visible results.** Every run adds restore and store tables to the job summary, and
  Gradle and Maven builds surface the same metrics in their Build Scan.

See [Migrating from a custom setup](#migrating-from-a-custom-ac-cli-setup) for a
step-by-step conversion.

## Quick start

Add the action to a job **once**, before the steps whose dependencies you want cached.
It runs in two automatic phases:

- its **main** step — where you place it in the job — **restores** the cache, and
- a **post** step that GitHub runs at the **end of the job** **stores** the cache.

You do **not** add a separate step to store — the post step is registered
automatically, the same model as [`actions/cache`](https://github.com/actions/cache)
and [`gradle/actions`](https://github.com/gradle/actions).

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: gradle/develocity-artifact-cache-github-action@v1
        with:
          develocity-url: https://develocity.example.com
          develocity-access-key: ${{ secrets.DEVELOCITY_ACCESS_KEY }}

      - run: ./gradlew build # uses the restored cache; its output is stored after the job
```

Place the action **before** your build so the restore happens first; the store then
runs after all the job's steps (even if the build fails) and **never fails the build**.
To restore without storing, set `cache-read-only: true`.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `develocity-url` | yes | — | Develocity server URL (passed to the CLI as `--dv-server`). |
| `develocity-access-key` | no | — | Raw access key for `develocity-url`. Optional — falls back to the `DEVELOCITY_ACCESS_KEY` env var or `keys.properties` (see [Authentication](#authentication)). |
| `ac-image-names` | no | — | Ordered image names, one per line: the first is the primary (stored under, tried first on restore), the rest are restore-only fallbacks. Replaces the auto-generated name (see [Cache image names](#cache-image-names)). |
| `matrix` | no | — | Pass `${{ toJSON(matrix) }}` to get a best-effort warning when the build matrix has axes the auto-generated name does not isolate (see [Matrix builds](#matrix-builds-share-a-cache-across-non-osarch-axes)). |
| `cache-read-only` | no | `false` | When `true`, restore only — the post step skips the store. |
| `ac-cli-version` | no | pinned GA | AC CLI version to download. Defaults to the tested version pinned in the manifest; usually leave unset (see [Versioning](#versioning)). |
| `ac-cli-repository` | no | public URL | JAR download source. Set to an internal mirror for air-gapped runners. |
| `ac-cli-repository-header` | no | — | A single `Name: Value` HTTP header for an authenticated mirror. Supply via a secret. |
| `ac-cli-sha256` | no | manifest | Expected JAR SHA-256; verifies a pinned version the manifest does not list. |
| `additional-cli-args` | no | — | Extra CLI arguments (one per line) appended to restore and store. The action-managed flags (`--dv-server`, `--image-name`, `--cache-metrics-file`, `--dv-edge`) are rejected. |

## Authentication

The action needs a Develocity access key for the server named by `develocity-url`.
It resolves one exactly the way the AC CLI does, in descending precedence:

1. the **`develocity-access-key` input** — the **raw key value** for the
   `develocity-url` host (wins over everything below),
2. a non-empty **`DEVELOCITY_ACCESS_KEY` environment variable**,
3. **`keys.properties`** — `~/.gradle/develocity/keys.properties`, then
   `~/.m2/.develocity/keys.properties` — the entry for the `develocity-url` host.

So the `develocity-access-key` input takes precedence over both the
`DEVELOCITY_ACCESS_KEY` environment variable and `keys.properties`.
Only the first source that exists is consulted — a non-empty environment
variable, or the first key file present on disk — and resolution does not fall
through to a later source that lacks an entry for the host, matching the CLI.
The environment variable and key files are host-scoped, in the CLI's `host=key`
form.

The `develocity-access-key` input is **optional**: omit it to rely on the
environment variable or a key file. If no key is found in any source, the action
**skips all cache activity for the job** rather than failing the build.

## Security

The action uses the **raw, long-lived** Develocity access key and does **not**
perform short-lived access-token exchange — a deliberate, accepted downgrade.
The raw key is supplied to the AC CLI through the host-scoped
`DEVELOCITY_ACCESS_KEY` environment variable. To limit exposure:

- supply the key through a repository/organization **secret**, as above, rather
  than a literal;
- prefer an access key **scoped to only the servers** it needs.

The resolved key is registered with the runner as a masked value, so it is
redacted from the workflow log.

## Cache coverage and observability

The action caches **Gradle, Maven, npm, pip, and Sonar** automatically: it invokes the
AC CLI, which autodetects the matching project content on the runner and caches
each type it finds, with no per-type enable input. To exclude a type (for example,
to skip npm in a large monorepo), pass `--no-autodetect <type>` through
`additional-cli-args`.

Where cache activity is visible depends on the build tool:

- **Gradle and Maven** — cache metrics appear in the **Build Scan** when the
  build applies the Develocity Gradle plugin or the Develocity Maven extension,
  which emit them. The action does not inject the plugin or extension into an
  unconfigured build; such a build still caches, just without Build Scan metrics.
- **npm, pip, and Sonar** — cache activity is reported in the **workflow log and job
  summary only**. There is no Develocity Build Scan integration for these tools, so
  their cache metrics do not appear in a Build Scan.

### Job summary

After each restore and store, the action adds a cache-activity table to the
GitHub **job summary**, sourced from the metrics the AC CLI writes:

- **restore** — Restored, Not found, Failed, Size, Duration.
- **store** — Stored, Already present, Failed, Size, Duration.

A dash (`—`) means the CLI did not report that field. The summary carries cache
counts only, not a Build Scan link (Gradle and Maven surface the same metrics in
the Build Scan through the separate path above).

### Warnings

Cache activity is best-effort: a restore or store problem is surfaced as a
**warning**, never a build failure. (The one hard failure is a JAR checksum
mismatch — a security guard, see [Security](#security).) The action warns when
an operation reports a problem:

- **Failures** — a non-zero failed-artifacts or failed-images count, or a
  transient error recorded by the CLI, warns that some cache activity did not
  complete. The build continues, just less cached.
- **Rejected access key** — when the Develocity server rejects the key
  (unauthorized), the action warns specifically and points at the
  `develocity-access-key` input (or `DEVELOCITY_ACCESS_KEY`); cache activity is
  skipped for that step and the build still runs.

A cache **miss** is expected — it is not a failure and never warns.

## Cache image names

Each cache is keyed by an **image name**. By default you set none: the AC CLI
auto-generates one from the workflow, job, git ref, runner OS, and runner arch,
so every job gets its own cache and jobs on different OSes or architectures stay
isolated.

### Fallback to another branch's cache

On **restore** the action tries the primary name and then a fallback, so a branch
with no cache of its own can still restore from a related branch:

- **Pull requests** — the AC CLI natively falls back to the **base branch**'s
  cache (from `GITHUB_BASE_REF`), so a PR's first build restores from its target
  branch.
- **Push and manually-dispatched (`workflow_dispatch`) events** — the action
  adds a fallback to the repository **default branch**'s cache, so the first
  build of a new branch can restore from `main` (or your default branch) instead
  of starting cold. (Scheduled runs already run on the default branch, so their
  primary name targets that cache directly.)

**Store** always writes to the single primary name.

### Overriding with `ac-image-names`

Set `ac-image-names` (one name per line) to take control of naming. It is a
single ordered list:

- the **first** entry is the **primary** — the name the cache is **stored** under
  and the first that **restore** tries;
- the **rest** are **restore-only fallbacks**, tried in order, stopping at the
  first hit.

Setting `ac-image-names` **replaces** the auto-generated names entirely —
including the PR and push fallbacks above — so list any fallbacks you want as
additional entries yourself:

```yaml
- uses: gradle/develocity-artifact-cache-github-action@v1
  with:
    develocity-url: https://develocity.example.com
    ac-image-names: |
      my-project-${{ github.ref_name }}
      my-project-main
```

### Matrix builds share a cache across non-OS/arch axes

The auto-generated name isolates by job, runner OS, and runner arch — but **not**
by other matrix axes such as a Java or build-tool version. Jobs that differ only
in such an axis therefore **share one cache** and can overwrite each other's
content. Pass your matrix to get a best-effort warning when this happens:

```yaml
- uses: gradle/develocity-artifact-cache-github-action@v1
  with:
    develocity-url: https://develocity.example.com
    matrix: ${{ toJSON(matrix) }}
```

`matrix` is your workflow's `matrix` context serialized with `toJSON` — there is
**no schema the action defines**: it's whatever axes your `strategy.matrix`
declares, passed as a JSON object of `axis: value` for the current job. For
example:

```yaml
strategy:
  matrix:
    java-version: [17, 21]
    os: [ubuntu-latest]
steps:
  - uses: gradle/develocity-artifact-cache-github-action@v1
    with:
      develocity-url: https://develocity.example.com
      matrix: ${{ toJSON(matrix) }} # e.g. {"java-version":"17","os":"ubuntu-latest"}
```

These axis names are treated as **already isolated** by the auto-generated cache
name (they feed `RUNNER_OS`/`RUNNER_ARCH`), matched case-insensitively: `os`,
`arch`, `platform`, `runner`, `runs-on`, `runner-os`, `runner-arch`,
`operating-system`. **Any other axis** (e.g. `java-version`, `gradle-version`)
is non-isolating and triggers the warning. The check sees only the current job's
resolved values, so a single-valued extra axis can warn even when no collision is
possible — a harmless false positive.

To **isolate** a sensitive axis, add a dedicated `ac-image-names` entry that
includes its value. Since this replaces the auto-generated names, add your own
fallback entry to keep the cross-branch seeding described above:

```yaml
    ac-image-names: |
      my-project-java${{ matrix.java-version }}-${{ github.ref_name }}
      my-project-java${{ matrix.java-version }}-main
```

## Compatibility

The action runs on the **Node 24** action runtime and invokes the AC CLI under a
JDK that meets the manifest's **minimum floor** (currently **Java 21**). On
GitHub-hosted runners it resolves that JDK from the pre-set
`JAVA_HOME_<major>_<arch>` variables — **no `setup-java` step is required** — so a
build pinned to an older JDK still caches, as long as the runner provides a
floor-satisfying JDK. If none is found, the action skips caching for the job
rather than failing the build.

### Supported runners

The action is supported on the following runners, provided each has a
floor-satisfying JDK (GitHub-hosted runners do; see above):

| Runner OS | Architecture |
| --- | --- |
| Linux | x64 |
| macOS | x64, ARM64 |
| Windows | x64 |

It also runs on self-hosted Linux (x64) runners. All five cache types (Gradle,
Maven, npm, pip, Sonar) are supported on every listed runner.

### JVM warmup

The AC CLI runs on the JVM, so each `restore` (main step) and `store` (post step)
pays a **one-time JVM startup cost** — typically a few seconds per invocation,
independent of cache size. A future GraalVM native image would remove this cost.

## Versioning

The action pins a **default AC CLI version** in its `manifest.json`
(`ac-cli-version` defaults to that pinned GA version), and that default advances
with each AC CLI release through the release flow. Two consequences for consumers:

- The action tracks a **tested default version**, so most users need not set
  `ac-cli-version` at all.
- **A change to the minimum required JDK is a breaking change and forces a major
  version bump of the action.** Pinning a newer CLI whose floor your runners do
  not meet makes the action skip caching (it never fails the build), so a new
  major version signals a JDK-floor change you must accommodate.

To pin or override, set `ac-cli-version` (and `ac-cli-sha256` to verify a version
the manifest does not list).

## Unsupported inputs

The action deliberately omits several inputs; each has a rationale or a workaround:

- **No cleanup input** — AC journal filtering keeps the stored cache lean on its
  own, so a manual cleanup control is not needed.
- **No per-cache-type enable/disable input** — the CLI autodetects Gradle, Maven,
  npm, pip, and Sonar. To exclude a type, pass `--no-autodetect <type>` through
  `additional-cli-args`.
- **No first-class tool-home inputs** — the CLI resolves each tool home from the
  standard environment variables CI sets. The one gap is a Maven local repository
  set via `-Dmaven.repo.local` (which autodetect does not see); pass
  `--maven-repository <path>` through `additional-cli-args`.
- **No short-lived token or OIDC exchange** — the action consumes the raw,
  long-lived access key (see [Security](#security)); token exchange and OIDC are
  future work.

## Migrating from a custom AC CLI setup

If you invoke the AC CLI yourself today — typically a hand-rolled "setup + restore"
step plus a separate "store" step, configured through `DV_SERVER` / `ARTIFACT_CACHE_*`
environment variables — this action replaces all of it with **one step**.

The migration has the same shape wherever the CLI is used:

1. **Collapse two steps into one.** Replace the restore step — and the separate
   store step — with a single `uses:` of this action, placed where restore ran.
   Its post step stores at the end of the job automatically.
2. **Map the CLI configuration to inputs** using the table below.
3. **Drop the plumbing.** Remove the manual JAR download, checksum verification,
   and JDK wiring — the action does all of it.
4. **Preserve any step condition.** If the old steps had an `if:` gate, put the
   same condition on the single action step.
5. **Matrixed jobs:** pass `matrix: ${{ toJSON(matrix) }}` so you are warned when
   matrix axes beyond OS/arch would share one cache (see
   [Matrix builds](#matrix-builds-share-a-cache-across-non-osarch-axes)).

A custom setup may also do things this action does not cover — for example
uploading the CLI log as an artifact, or supplying Develocity credentials that
other steps (such as Build Scan publishing) also rely on. This action replaces
only the cache restore and store; **keep, or add, whatever additional steps your
workflow needs for behavior outside that scope.**

| Current (custom CLI setup) | This action |
| --- | --- |
| separate **restore** step **and** **store** step | one `uses:` step (main restores, post stores) |
| `DV_SERVER` | `develocity-url` input |
| `DEVELOCITY_ACCESS_KEY` (env) | keep the env var, or use the `develocity-access-key` input |
| `--image-name` / `ARTIFACT_CACHE_IMAGE` | `ac-image-names` input |
| extra CLI flags (e.g. `--gradle-home`, often via `ARTIFACT_CACHE_OPTS`) | `additional-cli-args` input (one per line) |
| CLI version pin (e.g. `ARTIFACT_CACHE_CLI_VERSION`) | `ac-cli-version` — usually omit (defaults to the pinned version) |
| manual JAR download + SHA-256 verify + JDK wiring | built in |

Two behavior differences to note:

- **Store timing.** The post step stores at the **end of the job regardless of
  build outcome** (best-effort, never failing the build). A custom
  store-only-on-success gate has no direct equivalent; use `cache-read-only: true`
  to never store.
- **Logs.** The action does not upload an `artifact-cache.log` artifact — cache
  activity appears in the **job summary** and workflow log. Drop any log-upload
  step, or add your own if you still want the artifact.

**Before** — two custom steps plus environment:

```yaml
env:
  DV_SERVER: https://develocity.example.com
  DEVELOCITY_ACCESS_KEY: ${{ secrets.DEVELOCITY_ACCESS_KEY }}
  ARTIFACT_CACHE_OPTS: "--gradle-home=/home/runner/.gradle"
steps:
  - uses: ./.github/actions/setup-and-restore-artifact-cache # download CLI + restore
  - run: ./gradlew build
  - uses: ./.github/actions/store-artifact-cache             # store
    if: success()
```

**After** — one step:

```yaml
steps:
  - uses: gradle/develocity-artifact-cache-github-action@v1
    with:
      develocity-url: https://develocity.example.com
      develocity-access-key: ${{ secrets.DEVELOCITY_ACCESS_KEY }}
      additional-cli-args: --gradle-home=/home/runner/.gradle
  - run: ./gradlew build
```

## Learn more

- [Develocity Artifact Cache documentation](https://docs.develocity.ai/artifact-cache/) — product overview, CLI reference, and the GitHub Actions setup guide.
