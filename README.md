# Develocity Artifact Cache GitHub Action

Cache Gradle, Maven, npm, pip, and Sonar dependencies and build outputs in GitHub
Actions with the [Develocity Artifact Cache](https://docs.develocity.ai/artifact-cache/),
in a single workflow step. The action restores the cache before your build and stores
it afterward automatically.

## Why use this action?

The action integrates the Develocity Artifact Cache the way you already integrate other
tooling in GitHub Actions: a single step you add to a job, in the same shape as
`setup-gradle`, `setup-node`, or `setup-java`. It manages the whole cache lifecycle for
you:

- **One step, both phases.** The action restores in its main step and stores in an
  automatic post step. You do not write or order a separate store step, and cache
  activity never fails the build.
- **Every cache type, autodetected.** It caches Gradle, Maven, npm, pip, and Sonar
  content automatically, with no per-type configuration.
- **Cache images that work out of the box.** Images are generated per job and isolated by
  runner operating system and architecture, with smart fallbacks to the base branch and
  the repository default branch, so new branches restore instead of starting cold.
- **A verified, tested CLI.** The action downloads a checksum-verified CLI JAR and defaults
  to the version pinned and tested for this release, so you do not track CLI versions or
  download URLs yourself.
- **Visible results.** A run's cache activity is summarized as a table in the job summary
  (whenever the action reaches the cache), and Gradle and Maven builds surface the same
  metrics in their Build Scan.

Already driving the CLI by hand? See
[Migrating from a custom setup](#migrating-from-a-custom-artifact-cache-cli-setup).

## Quick start

Add the action to a job **once**, before the steps whose dependencies you want cached.
It runs in two automatic phases:

- its **main** step, where you place it in the job, **restores** the cache, and
- a **post** step that GitHub runs at the **end of the job** **stores** the cache.

You do **not** add a separate step to store. The post step is registered automatically,
the same model as [`actions/cache`](https://github.com/actions/cache) and
[`gradle/actions`](https://github.com/gradle/actions).

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: actions/setup-java@v5 # provides the JDK your build runs on
        with:
          distribution: temurin
          java-version: '21'

      - uses: gradle/develocity-artifact-cache-github-action@v1
        with:
          develocity-url: https://develocity.example.com
          develocity-access-key: ${{ secrets.DEVELOCITY_ACCESS_KEY }}

      - run: ./gradlew build # uses the restored cache; its output is stored after the job
```

Place the action **before** your build so the restore happens first. The store then runs
only when the **job succeeds** (it is skipped if a job step failed) and **never fails the
build**. To make the cache **read-only** (restore, but never store), set
`cache-read-only: true`.

### Using with `setup-gradle`

When your workflow also uses [`gradle/actions/setup-gradle`](https://github.com/gradle/actions),
two things matter:

1. **Apply `setup-gradle` before this action.** `setup-gradle` establishes the Gradle User
   Home for the job — it creates the directory, writes its init scripts, and exports
   `GRADLE_USER_HOME`. Running it first lets this action's CLI auto-detect that same Gradle
   User Home, so the cache is **restored into, and stored from, the location your Gradle
   build actually uses**. If this action ran first, it would resolve the default Gradle User
   Home, which may not match the one `setup-gradle` goes on to configure — notably on
   Windows, where `setup-gradle` can relocate it.
2. **Set `cache-disabled: true` on `setup-gradle`.** This action already caches the Gradle
   dependencies that `setup-gradle`'s own Gradle User Home cache would otherwise store.
   Disabling `setup-gradle`'s caching defers to this action and avoids both mechanisms
   caching the same content.

```yaml
      - uses: gradle/actions/setup-gradle@v6
        with:
          cache-disabled: true # defer dependency caching to the Artifact Cache action

      - uses: gradle/develocity-artifact-cache-github-action@v1
        with:
          develocity-url: https://develocity.example.com
          develocity-access-key: ${{ secrets.DEVELOCITY_ACCESS_KEY }}

      - run: ./gradlew build
```

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `develocity-url` | yes | (none) | Develocity server URL. |
| `develocity-access-key` | yes | (none) | Access key for `develocity-url` in `host=key` form (the only key source). The action exchanges it for a short-lived token used **only** for its own cache restore/store — it is not exported to other steps. Required to enable caching; never fails the build if missing — it warns and skips (see [Authentication](#authentication)). |
| `develocity-token-expiry` | no | `2` | Lifetime, in hours, of the short-lived token obtained from `develocity-access-key`. Raise it only if a build could run long enough for the token to expire before the post-step store (see [Authentication](#authentication)). |
| `image-names` | no | (none) | Ordered image names, one per line: the first is the primary (stored under, tried first on restore), the rest are restore-only fallbacks. Replaces the auto-generated name (see [Cache image names](#cache-image-names)). |
| `cache-read-only` | no | `false` | When `true`, restore only: the post step skips the store. |
| `cli-version` | no | pinned general-availability version | Artifact Cache CLI version to download. Defaults to the tested version the action pins; usually leave unset (see [Versioning](#versioning)). |
| `cli-repository` | no | public URL | JAR download source. Set to an internal mirror for air-gapped runners. Must be an `https` URL unless `cli-repository-allow-insecure` is set. |
| `cli-repository-header` | no | (none) | A single `Name: Value` HTTP header for an authenticated mirror. Supply via a secret. |
| `cli-repository-allow-insecure` | no | `false` | Allow a plain-`http` `cli-repository`. The default rejects `http`; enable only for a trusted internal or GitHub Enterprise mirror without TLS (over `http` the download and any `cli-repository-header` credential are sent in cleartext). |
| `cli-sha256` | no | (built in) | Expected JAR SHA-256; needed only to verify a version the action does not already ship a checksum for. |
| `additional-cli-args` | no | (none) | Extra CLI arguments (one per line) appended to restore and store. The action-managed flags (`--dv-server`, `--image-name`, `--cache-metrics-file`, `--dv-edge`) are rejected. |

## Authentication

The action needs a Develocity access key for the server named by `develocity-url`,
supplied through the **`develocity-access-key` input**. This input is the **only** key
source: the action does **not** read the `DEVELOCITY_ACCESS_KEY` environment variable or
`keys.properties`, so it never authenticates with an ambient key you did not explicitly
pass to it.

Provide it in **`host=key` form** — e.g. `develocity.example.com=<key>`, or a
`;`-separated `host1=key1;host2=key2` list — matching the `DEVELOCITY_ACCESS_KEY` format
used across the Develocity ecosystem (the DV build agents, `keys.properties`, and
`setup-gradle`). A bare key value is not accepted. The action uses the entry for the
`develocity-url` host, so a key for another host is never sent to this server.

The input is **required** to enable caching, but the action is fail-safe: if it is missing,
not in `host=key` form, or has no entry for the `develocity-url` host, the action **warns
and skips all cache activity for the job** rather than failing the build.

### Short-lived access tokens

Develocity access keys are long-lived, so exposing one to a whole workflow is a risk if it
leaks. To avoid that, the action immediately exchanges the key you provide for a
**short-lived Develocity token** and uses that token **only for its own cache restore and
store** — it hands the token to the Artifact Cache CLI through a private, subprocess-scoped
environment variable. The long-lived key never leaves this action. If the exchange fails, the
action warns and skips rather than falling back to the long-lived key.

The action does **not** set `DEVELOCITY_ACCESS_KEY` (or the token) for any other step. A cache
action shouldn't propagate credentials to the rest of the build, and — unlike Gradle, which
has `setup-gradle` — other build tools have no equivalent that would expect it. If a later step
needs to authenticate with Develocity (for example, to publish a Build Scan), give it its own
credentials: `setup-gradle`, for instance, performs the same key-to-short-lived-token exchange
and exports it for Gradle builds.

The token is valid for **2 hours** by default; raise `develocity-token-expiry` only if a build
runs long enough that the token could expire before the post-step store runs.

## Security

The action authenticates with a Develocity access key, supplied to the Artifact Cache CLI through the
host-scoped `DEVELOCITY_ACCESS_KEY` environment variable. To limit its exposure:

- supply the key through a repository or organization **secret**, as above, rather than a
  literal;
- prefer an access key **scoped to only the servers and permissions** it needs.

The resolved key is registered with the runner as a masked value, so it is redacted from
the workflow log.

## Cache coverage and observability

The action caches **Gradle, Maven, npm, pip, and Sonar** automatically: it invokes the Artifact Cache
CLI, which autodetects the matching project content on the runner and caches each type it
finds, with no per-type enable input. To exclude a type (for example, to skip npm in a
large monorepo), pass `--no-autodetect <type>` through `additional-cli-args`.

Cache activity is visible in two places:

- The **job summary** is the closest observability layer: a run's cache activity is
  summarized as a table (see [Job summary](#job-summary) below).
- For finer-grained cache operation metrics, inspect the build's **Build Scan** in
  Develocity. Gradle and Maven builds that apply the Develocity Gradle plugin or Maven
  extension surface cache metrics there directly.

### Job summary

After a run, the action adds a single cache-activity table to the GitHub **job summary**,
sourced from the metrics the Artifact Cache CLI writes. The table has a **Phase** column
and one row per phase:

- **Restore**: Restored, Not found, Failed, Size, Duration.
- **Store**: Stored, Already present, Failed, Size, Duration.

A successful restore followed by a store shows both rows; when the store is skipped —
read-only cache, a restore that did not succeed, or a **failed build** — the table shows
the restore row only. An `Errors` column is added when a phase reports a message, and a
dash marks a field the CLI did not report. The post step always runs, so the table
appears even when the build fails (with the restore row only, since a failed build never
stores). The summary carries cache counts only, not a Build Scan link (Gradle and Maven
surface the same metrics in the Build Scan through the separate path above).

### Warnings

Cache activity is best-effort: a restore or store problem is surfaced as a **warning**,
never a build failure. (The one hard failure is a JAR checksum mismatch, a security
guard; see [Security](#security).) The action warns when an operation reports a problem:

- **Failures**: a non-zero failed-artifacts or failed-images count, or a transient error
  recorded by the CLI, warns that some cache activity did not complete. The build
  continues, just less cached.
- **Rejected access key**: when the Develocity server rejects the key (unauthorized), the
  action warns specifically and points at the `develocity-access-key` input; cache
  activity is skipped for that step and the build still runs.

A cache **miss** is expected: it is not a failure and never warns.

## Cache image names

Each cache is keyed by an **image name**. By default you set none: the Artifact Cache CLI
auto-generates one from the workflow, job, git ref, runner operating system, and runner
architecture, so every job gets its own cache and jobs on different operating systems or
architectures stay isolated.

### Fallback to another branch's cache

On **restore** the action tries the primary name and then a fallback, so a branch with no
cache of its own can still restore from a related branch:

- **Pull requests** fall back to the **base branch**'s cache, so a pull request's first
  build restores from its target branch.
- **Push and manually dispatched events** fall back to the repository **default branch**'s
  cache, so the first build of a new branch restores from your default branch instead of
  starting cold.

**Store** always writes to the single primary name.

### Overriding with `image-names`

Set `image-names` (one name per line) to take control of naming. It is a single
ordered list:

- the **first** entry is the **primary**, the name the cache is **stored** under and the
  first that **restore** tries;
- the **rest** are **restore-only fallbacks**, tried in order, stopping at the first hit.

Setting `image-names` **replaces** the auto-generated names entirely, including the
pull request and push fallbacks above, so list any fallbacks you want as additional
entries yourself:

```yaml
- uses: gradle/develocity-artifact-cache-github-action@v1
  with:
    develocity-url: https://develocity.example.com
    image-names: |
      my-project-${{ github.ref_name }}
      my-project-main
```

### Matrix builds share a cache across axes other than operating system and architecture

The auto-generated cache name isolates by job, runner operating system, and runner
architecture, but **not** by other matrix axes such as a Java or build-tool version. Jobs
that differ only in such an axis therefore **share one cache** and can overwrite each
other's content.

To **isolate** a sensitive axis, add a dedicated `image-names` entry that includes its
value. Since this replaces the auto-generated names, add your own fallback entry to keep
the cross-branch seeding described above:

```yaml
    image-names: |
      my-project-java${{ matrix.java-version }}-${{ github.ref_name }}
      my-project-java${{ matrix.java-version }}-main
```

## Compatibility

The action runs on the **Node 24** action runtime and invokes the Artifact Cache CLI under a JDK that
meets a **minimum floor** (currently **Java 21**). On GitHub-hosted runners
it resolves that JDK from the pre-set `JAVA_HOME_<major>_<arch>` variables, so **no
`setup-java` step is required for the action itself**, and a build pinned to an older JDK still caches as long
as the runner provides a floor-satisfying JDK. If none is found, the action skips caching
for the job rather than failing the build.

### Supported runners

The action is supported on the following runners, provided each has a floor-satisfying
JDK (GitHub-hosted runners do; see above):

| Runner operating system | Architecture |
| --- | --- |
| Linux | x64 |
| macOS | x64, ARM64 |
| Windows | x64 |

It also runs on self-hosted Linux (x64) runners. All five cache types (Gradle, Maven,
npm, pip, Sonar) are supported on every listed runner.

### JVM warmup

The Artifact Cache CLI runs on the JVM, so each `restore` (main step) and `store` (post step) pays a
**one-time JVM startup cost**, typically a few seconds per invocation, independent of
cache size.

## Versioning

The action pins a **default Artifact Cache CLI version** (`cli-version` defaults to that
pinned general-availability version), and that default advances with each Artifact Cache
CLI release. Two consequences for consumers:

- The action tracks a **tested default version**, so most users need not set
  `cli-version` at all.
- **A change to the minimum required JDK is a breaking change and forces a major version
  bump of the action.** Pinning a newer CLI whose floor your runners do not meet makes the
  action skip caching (it never fails the build), so a new major version signals a
  JDK-floor change you must accommodate.

To pin or override, set `cli-version` (and `cli-sha256` to verify a version the
action does not already ship a checksum for).

## Configuration handled by the CLI

Some configuration is handled by the CLI rather than exposed as an action input:

- **Cache cleanup** is automatic, so there is no cleanup input.
- **Cache-type selection** is by autodetection: the CLI detects Gradle, Maven, npm, pip,
  and Sonar. To exclude a type, pass `--no-autodetect <type>` through
  `additional-cli-args`.
- **Tool homes** are resolved from the standard environment variables CI sets. For a
  non-standard Maven local repository, pass `--maven-repository <path>` through
  `additional-cli-args`.

## Migrating from a custom Artifact Cache CLI setup

If you invoke the Artifact Cache CLI yourself today, typically a custom "setup + restore" step plus a
separate "store" step, configured through `DV_SERVER` / `ARTIFACT_CACHE_*` environment
variables, this action replaces all of it with **one step**.

The migration has the same shape wherever the CLI is used:

1. **Collapse two steps into one.** Replace the restore step, and the separate store step,
   with a single `uses:` of this action, placed where restore ran. Its post step stores at
   the end of the job automatically.
2. **Map the CLI configuration to inputs** using the table below.
3. **Drop the plumbing.** Remove the manual JAR download, checksum verification, and JDK
   wiring; the action does all of it.
4. **Move any restore-time condition.** If the old restore step had an `if:` gate for
   whether to use the cache, put the same condition on the action step. A
   store-only-on-success gate (`if: success()` on the old store step) needs nothing — the
   action always stores only when the job succeeds.
5. **Matrixed jobs:** if your matrix varies beyond the operating system and architecture,
   add a dedicated `image-names` entry to isolate that axis so those jobs do not share
   one cache (see
   [Matrix builds](#matrix-builds-share-a-cache-across-axes-other-than-operating-system-and-architecture)).

A custom setup may also do things this action does not cover, for example uploading the
CLI log as an artifact, or supplying Develocity credentials that other steps (such as
Build Scan publishing) also rely on. This action replaces only the cache restore and
store; **keep, or add, whatever additional steps your workflow needs for behavior outside
that scope.**

| Current (custom CLI setup) | This action |
| --- | --- |
| separate **restore** step **and** **store** step | one `uses:` step (main restores, post stores) |
| `DV_SERVER` | `develocity-url` input |
| `DEVELOCITY_ACCESS_KEY` (env) | pass it to the `develocity-access-key` input (the action does not read the env var) |
| `--image-name` / `ARTIFACT_CACHE_IMAGE` | `image-names` input |
| extra CLI flags (for example `--gradle-home`, often via `ARTIFACT_CACHE_OPTS`) | `additional-cli-args` input (one per line) |
| CLI version pin (for example `ARTIFACT_CACHE_CLI_VERSION`) | `cli-version`, usually omitted (defaults to the pinned version) |
| manual JAR download, SHA-256 verify, and JDK wiring | built in |

Two behavior differences to note:

- **Store timing.** The post step stores only when the **job succeeds**, the same effect
  as an `if: success()` gate on the old store step (best-effort, never failing the build).
  Set `cache-read-only: true` to never store.
- **Logs.** The action does not upload an `artifact-cache.log` artifact; cache activity
  appears in the **job summary** and workflow log. Drop any log-upload step, or add your
  own if you still want the artifact.

**Before**, two custom steps plus environment:

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

**After**, one step:

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

- [Develocity Artifact Cache documentation](https://docs.develocity.ai/artifact-cache/):
  product overview, CLI reference, and the GitHub Actions setup guide.
